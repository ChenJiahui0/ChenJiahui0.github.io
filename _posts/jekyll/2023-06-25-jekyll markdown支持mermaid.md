---
title: jekyll markdown支持mermaid
layout: post
author: 陈家辉
tags:
- jekyll
- mermaid
---



# 简介

该篇文章记载本站文章如何支持mermaid渲染。

# 实操

1. 在需要使用mermaid的界面中插入以下脚本，本站在post.html公共代码部分进行插入

   ```html
   <script type="module">
       import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10.0.2/+esm';
       mermaid.initialize({ startOnLoad: false });
       await mermaid.run({
         querySelector: '.language-mermaid',
       });
   </script>
   ```

# 测试

用例1

```mermaid
---
title: 标题
---
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```



用例2

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->>John: Hello John, how are you?
    loop Healthcheck
        John->>John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail!
    John-->>Alice: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
```

用例3

```mermaid
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label
```

更多用例请参考[mermaid官网](https://mermaid.js.org/intro/)
