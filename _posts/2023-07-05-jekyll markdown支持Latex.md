---
title: jekyll markdown支持Latex
layout: post
author: 陈家辉
tags:
- jekyll
- Latex
---



# 简介

该篇文章记载本站文章如何支持latex渲染。

# 实操

1. 在需要使用mermaid的界面中插入以下脚本，本站在post.html公共代码部分进行插入

   ```html
   <script type="text/javascript" async
        src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
   </script>
   ```

# 测试
$$
index(child) = 2k+1 = 2k+2\\
index(parent) = (k-1)/2\\
comment:第k个节点的子节点坐标为 2k+1或2k+2，父节点坐标为(k-1)/2
$$