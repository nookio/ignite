---
title: ClassLoader分析
date: 2017-08-13 22:43:32
tags:
  - Java
  - 原理
---
## ClassLoader的分类
整体上一共有三种，也是classloader的加载顺序
- bootstrap classLoader  --这个是JVM级别的
- extension classLoader  --这个是扩展加载器
- system classLoader  --应用类加载器

<!--more-->

下面依次介绍这三个ClassLoader.
#### Bootstrap classLoader
首先来说，这个加载器的被调用时机
