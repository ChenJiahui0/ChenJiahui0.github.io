---
title: ReentrantLock源码解析
layout: post
author: 陈家辉
tags:
- Java
- 锁
- 同步机制
- 源码
comment: true
---

Java中的大部分同步类（Lock、Semaphore、ReentrantLock等）都是基于AbstractQueuedSynchronizer（简称为AQS）实现的。AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。本文会从应用层逐渐深入到原理层，并通过ReentrantLock的基本特性和ReentrantLock与AQS的关联，来深入解读AQS相关独占锁的知识点，同时采取问答的模式来帮助大家理解AQS。

下面列出本篇文章的大纲和思路，以便于大家更好地理解：

# 1 ReetrantLock

```java
 class X {
   private final ReentrantLock lock = new ReentrantLock();
   // ...

   public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
     } finally {
       lock.unlock();
     }
   }
 }
```

## 1.1 ReentrantLock Vs synchronized

Vs Synchronized

|          | ReentrantLock                | Synchronized   |
| :------- | ---------------------------- | -------------- |
| 底层实现 | AQS                          | 监视器模式     |
| 灵活性   | 支持响应中断/超时/尝试获取锁 | 不灵活         |
| 释放方式 | 显式JJ调用                   | 代码块自动释放 |
| 锁类型   | 公平锁&非公平锁              | 非公平锁       |
| 条件队列 | 可关联多个条件队列           | 一个条件队列   |
| 可重入性 | 可重入                       | 可重入         |

```java
@Test
@SneakyThrows
public void test() {
    // 1.初始化选择公平锁、非公平锁
    ReentrantLock lock = new ReentrantLock(true);
    // 2.可用于代码块
    lock.lock();
    try {
        try {
            // 3.支持多种加锁方式，比较灵活; 具有可重入特性
            if(lock.tryLock(100, TimeUnit.MILLISECONDS)){ }
        } finally {
            // 4.手动释放锁
            lock.unlock();
        }
    } finally {
        lock.unlock();
    }
}
```



```java
		// 修饰方法
    public synchronized void syncCalculate() {
        setSum(getSum() + 1);
    }

    public void syncCalculate2() {
        // 修饰代码块
        synchronized (this) {
            sum++;
        }
    }
```

## ReetrantLock与AQS的关联

ReentrantLock有三个内部类，分别是Sync、NonfairSync和FairSync

```java
abstract static class Sync extends AbstractQueuedSynchronizer{...}

static final class NonfairSync extends Sync{...}

static final class FairSync extends Sync{...}
```

ReetrantLock根据传入的参数来创建Sync对象，使用Sync对象提供的API来实现加锁释放锁。

### NonfairSync

首先来看lock()方法加锁过程，使用者手动调用，进行加锁，

```java
public class ReentrantLock {
    public void lock() {
      	// 调用AQS的lock()
      	sync.lock();	
    }
    abstract static class Sync extends AbstractQueuedSynchronizer {
        ...
        final void lock() {
            // 检查可重入性并获取在公平与非公平规则下是否立即可用的锁
            // 锁定方法在中继到相应的 AQS 获取方法之前执行 initialTryLock 检查。
            if (!initialTryLock())
               acquire(1);
        }
        public final void acquire(int arg) {
            if (!tryAcquire(arg))
              	// AQS核心API
                acquire(null, arg, false, false, false, 0L);
        }
        ...
    }
    static final class NonfairSync extends Sync {
        // Sync的初始化逻辑
        final boolean initialTryLock() {
            Thread current = Thread.currentThread();
            // 1.使用CAS尝试获取锁，如果成功将AQS的STATE设置为1
            if (compareAndSetState(0, 1)) {
                // 设置独占线程
                setExclusiveOwnerThread(current);
                return true;
            }
            // 2. 发现STATE不为0，说明有线程占有了，判断是否是自身占有，如果是则也可以进入（可重入）
            else if (getExclusiveOwnerThread() == current) {
                int c = getState() + 1;
                if (c < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(c);
                return true;
            } else
                return false;
        }

        // 实现了AQS的抽象方法
        protected final boolean tryAcquire(int acquires) {
            // 这个地方之所以先检查再去做CAS，个人认为只是为了效率
            // 直接get，如果发现等于0，再去做CAS操作
            if (getState() == 0 && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
    }
}
```

#### 流程

![image-20211211152351886](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241220643.png)

可以看到ReetrantLock在使用AQS的acquire方法前会调用#initialTryLock()和#tryAcquire(int arg)方法尝试获取锁，这两个方法是AQS让用户自己去实现的逻辑，NonFairSync实际上都是调用AQS中的#compareAndSetState(int expect, int update)方法，使用CAS的方式获取锁，两次获取锁都失败后，则会调用AQS的Acquire方法。



# 2 AQS

在进入AQS之前，先从宏观的角度来理解一下AQS的实现。

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241220644.png)



## 2.1 原理概览

AQS核心思想是，如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。

AQS中的队列是CLH变体的虚拟双向队列（FIFO），AQS是通过将每条请求共享资源的线程封装成一个节点来实现锁的分配。

![img](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241220645.png)

AQS使用一个Volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，通过CAS完成对State值的修改。

### 2.1.1 AQS数据结构

**Node**

![image-20211211202205778](https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241220646.png)

**方法和属性值的含义**

| 方法和属性值       | 含义                                           |
| ------------------ | ---------------------------------------------- |
| prev               | 前驱节点                                       |
| next               | 后继节点                                       |
| waiter             | 等待线程                                       |
| status             | 节点状态                                       |
| #casPrev           | CAS替换前驱节点                                |
| #casNext           | CAS替换前驱节点                                |
| #getAndUnsetStatus | 获取当前状态并且更新                           |
| #clearStatus       | 清空状态                                       |
| #setStatusRelaxed  | Adds node to condition list and releases lock. |

**线程两种锁的模式**

| 模式      | 含义     |
| --------- | -------- |
| SHARED    | 共享模式 |
| EXCLUSIVE | 独占模式 |

**status bits**

| Bit        | 含义                      |
| ---------- | ------------------------- |
| 1          | WAITING                   |
| 0x80000000 | CANCELLED                 |
| 2          | COND：in a condition wait |

### 2.1.2 同步状态State

在了解数据结构后，接下来了解一下AQS的同步状态——State。AQS中维护了一个名为state的字段，意为同步状态，是由Volatile修饰的，用于展示当前临界资源的获锁情况。

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer

private volatile int state;
```

下面提供了几个访问这个字段的方法：

| 方法名                                                       | 描述                 |
| ------------------------------------------------------------ | -------------------- |
| protected final boolean compareAndSetState(int expect, int update) | 使用CAS方式更新state |
| protected final void setState(int newState)                  | 设置state            |
| protected final int getState()                               | 获取state            |

可以通过修改state的值，实现多线程的独占或者共享

<div>
  <center class="half">
	<img src="https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241221452.png" width="300"/>
	<img src="https://cdn.jsdelivr.net/gh/Chenjiahui0/picture@main/202306241221004.png" width=300/>
</center>
</div>




```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    final int acquire(Node node, int arg, boolean shared,
                      boolean interruptible, boolean timed, long time) {
        Thread current = Thread.currentThread();
        byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
        boolean interrupted = false, first = false;
        Node pred = null;                // predecessor of node when enqueued

        for (;;) {
            if (!first && (pred = (node == null) ? null : node.prev) != null &&
                !(first = (head == pred))) {
                if (pred.status < 0) {
                    cleanQueue();           // predecessor cancelled
                    continue;
                } else if (pred.prev == null) {
                    Thread.onSpinWait();    // ensure serialization
                    continue;
                }
            }
           	// 1.if条件成立
            if (first || pred == null) {
                boolean acquired;
                try {
                    if (shared)
                        acquired = (tryAcquireShared(arg) >= 0);
                    else
                      	// 2.调用tryAcquire
                        acquired = tryAcquire(arg);
                } catch (Throwable ex) {
                  	// 如果对象没有实现tryAcquire(arg)抛出异常
                    cancelAcquire(node, interrupted, false);
                    throw ex;
                }
                if (acquired) {
                    if (first) {
                        node.prev = null;
                        head = node;
                        pred.next = null;
                        node.waiter = null;
                        if (shared)
                            signalNextIfShared(node);
                        if (interrupted)
                            current.interrupt();
                    }
                    return 1;
                }
            }
            if (node == null) {                 // allocate; retry before enqueue
                if (shared)
                    node = new SharedNode();
                else
                    node = new ExclusiveNode();
            } else if (pred == null) {          // try to enqueue
                node.waiter = current;
                Node t = tail;
                node.setPrevRelaxed(t);         // avoid unnecessary fence
                if (t == null)
                    tryInitializeHead();
                else if (!casTail(t, node))
                    node.setPrevRelaxed(null);  // back out
                else
                    t.next = node;
            } else if (first && spins != 0) {
                --spins;                        // reduce unfairness on rewaits
                Thread.onSpinWait();
            } else if (node.status == 0) {
                node.status = WAITING;          // enable signal and recheck
            } else {
                long nanos;
                spins = postSpins = (byte)((postSpins << 1) | 1);
                if (!timed)
                    LockSupport.park(this);
                else if ((nanos = time - System.nanoTime()) > 0L)
                    LockSupport.parkNanos(this, nanos);
                else
                    break;
                node.clearStatus();
                if ((interrupted |= Thread.interrupted()) && interruptible)
                    break;
            }
        }
        return cancelAcquire(node, interrupted, interruptible);
    }
```

