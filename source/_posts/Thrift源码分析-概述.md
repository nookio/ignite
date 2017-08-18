---
title: Thrift源码分析--概述
tags:
  - RPC
  - 原理
  - 源码
  - Thrift
date: 2017-08-14 00:21:39
---

#### 简介  
我司采用的RPC框架是apache开源的thrift，并在上层封装了服务注册和自动分配的功能，我将在两个部分分别介绍我司的RPC框架，首先是从底层进行分析整个的工作原理，下一步完成给thrift添加上下文，最终介绍我司的封装

<!--more-->    

Thrift源于Facebook, 目前已经作为开源项目提交给了Apahce。Thrift解决了Facebook各系统的大数据量传输通信和内部不同语言环境的跨平台调用。   

#### 官网  
Thrift的官方网站: http://thrift.apache.org/  

#### 特点  
作为一个高性能的RPC框架，Thrift的主要特点有
1. 基于二进制的高性能的编解码框架
2. 基于NIO的底层通信
3. 相对简单的服务调用模型
4. 使用IDL支持跨平台调用  

#### 核心组件
__Thrift其中包含了如下的几个核心组件：__

- **TProtocol:** 协议和编解码组件  
- **TTransport:** 传输组件  
- **TProcessor:** 服务调用组件  
- **TServer，Client:** 服务器和客户端组件  
- **IDL:** 服务描述组件，负责生产跨平台客户端

我会在后面的文章中依次介绍每一个组件  

#### 建议
在学习本系列文章之前，建议具备的知识点：
 - socket网络编程
 - NIO基础知识，以及nio、socket实现机制
