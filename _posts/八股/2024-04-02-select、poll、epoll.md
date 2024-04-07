---
title: select、poll、epoll
layout: post
author: 陈家辉
tags:
- 八股
---

# select、poll、epoll优缺点总结

![img](https://pic3.zhimg.com/80/v2-4dc8249995904c19ad50c927e6eec32e_1440w.webp)

![img](https://pic3.zhimg.com/80/v2-69bed0bd4a124b99dc28e2825169bc86_1440w.webp)

# select（可以跨平台）

函数原型：

```cpp
/*
 * 数据结构 (bitmap)
 * fd_set保存了相关的socket事件
*/
#define __FD_SETSIZE 1024
typedef long int __fd_mask;
#define __NFDBITS (8 * (int)sizeof(__fd_mask))

typedef struct
{
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];//1024
}fd_set;


/**
* select是一个阻塞函数
*/
// 返回值就绪描述符的数目
int select(
    int max_fd,  // 最大的文件描述符值，遍历时取0-max_fd
    fd_set *readset,  // 读事件列表
    fd_set *writeset,  // 写事件列表
    fd_set *exceptset,  // 异常列表
    struct timeval *timeout  // 阻塞超时时间
)
```

**fd_set仅包含一个long类型的数组（bitmap），该数组的每一个元素的每一位标记一个文件描述符**，最大文件描述符数量由__FD_SETSIZE决定。

## select缺点总结如下：

1. select的参数类型fd_set没有将文件描述符和事件绑定，它仅仅是一个文件描述符集合，因此select需要提供3个这种类型的参数来分别传入和输出**可读、可写及异常**等事件。这一方面使得select不能处理更多类型的事件。
2. fd_set集合不可重用，由于内核对fd_set集合的在内核中会被修改，应用程序下次调用select前不得不重置这3个fd_set集合。
3. 最大文件描述符数量限制：32位系统1024，64位系统2048。
4. select监听到有事件发生时返回，用户态会以轮询的方式，消耗O(n)的时间复杂度来寻找就绪事件。
5. 每次调用select函数时都要内核向传递监视对象信息，文件描述符集合会从用户态拷贝到内核态。
6. 只有LT触发模式。

```cpp
socktd = socket(Ah_lNEl , s0CK_SlREAM，0);
memset(&addr，0，sizeof (addr));
addr .sin_family = AF_INET;
addr.sin_port = htons(2000);
addr.sin_addr.s_addr = INADDR_ANY;
bind(sockfd ,(struct sockaddr* )&addr ,sizeof(addr));
listen (sockfd，5);
for (i=0; i<5;i++)
{
    memset(&client，0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client，&addrlen);
    if(fds[i] > max) max = fds[i];
}
while(1)
{
    FD_ZERO(&rset);
    for (i = 0; i< 5; i++ ) 
    {
        FD_SET( fds[i],&rset);
    }

    select(max+1，&rset，NULL，NULL，NULL);

    for(i=0;i<5; i++) 
    {
        if (FD_ISSET(fds[i]，&rset))
        {
            memset(buffer ,0,MAXBUF);
            read( fds[i]，buffer，MAXBUF);puts(buffer);
        }
    }
}
```

# poll

1. 解决了select中fd_set集合不可重用的缺点，因为每一个fd采用的结构体的封装形式，内核每次修改的是pollfd结构体的revents成员，而events成员保持不变，因此下次调用poll时应用程序无须重置pollfd类型的事件集参数。在使用完成后即可置位，不像select使用额外的O(n)时间复杂度来置位。
2. 文件描述符没有限制。
3. 将每一个fd与事件绑定到一起，任何事件统一处理，使接口变得简洁很多。
4. 只有LT触发模式。

```cpp
struct pollfd 
{
    int fd;
    short events;//由用户设置
    short revents;//由内核设置
}

for (i=0 ; i<5;i++)
{
    memset(&client，0，sizeof (client));addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd,(struct sockaddr*)&client，&addrlen);
    pollfds[i].events = POLLIN;
}

while(1)
{
    poll(pollfds，5，50000);
    for(i=0;i<5;i++) 
    {
        if (pollfds[i].revents & POLLIN)
        {
            pollfds[i].revents = 0;
            memset(buffer ,0,MAXBUF);
            read(pollfds[i].fd，buffer，MAXBUF);
        }
    }
}
```

# epoll（不能够跨平台）

有三个关键函数：

```cpp
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_events* event);
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

1. 在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。
2. poll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。***epoll保证了每个fd在整个过程中只会拷贝一次。\***
3. epoll_wait则不同，它采用的是回调的方式。内核检测到就绪的文件描述符时，将触发回调函数，回调函数就将该文件描述符上对应的事件插入内核就绪事件队列。内核最后在适当的时机将该就绪事件队列中的内容拷贝到用户空间。因此***epoll_wait无须轮询整个文件描述符集合来检测哪些事件已经就绪，其算法时间复杂度是О(1)\***。
4. 但是，当活动连接比较多的时候，epoll_wait 的效率未必比 select和 poll高，因为此时回调函数被触发得过于频繁。
5. ***所以epoll_wait适用于连接数量多，但活动连接较少的情况。\***
6. 没有最大文件描述符限制。
7. 可以使用高效的ET触发模式。
8. 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递，即epoll使用mmap减少复制开销，epoll是通过内核于用户空间mmap同一块内存，避免了无畏的内存拷贝；

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。执行epoll_create时，创建了红黑树和就绪链表，执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据。执行epoll_wait时立刻返回准备就绪链表里的数据即可。

## epoll为什么要有EPOLLET触发模式？

如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。

而采用EPOLLET这种边沿触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。

这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。

