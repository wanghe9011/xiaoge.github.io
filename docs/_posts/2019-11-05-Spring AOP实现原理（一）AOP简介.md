---
layout: post
title: "Spring AOP实现原理（一）AOP简介"
categories: misc
---

# 前言

Spring AOP是Spring众多优秀特性中的一个，我一直对它的实现比较好奇，最近有些闲暇时间，就整理了一些有关Spring AOP实现的资料。在整理的过程中，又稍作深入的理解一些优秀的框架和工具，比如asm，CGLIB，AspectJ等，顿觉自己知识浅薄，当然也难以抑制分享的冲动，于是就决定做一个系列文章——Spring AOP实现原理。

文章总共分为5部分：

1. <a href="https://juejin.im/post/6847902217131802632">AOP简介</a>

2. <a href="https://juejin.im/post/6847902217131966471">实现一个简单的AOP</a>

3. <a href="https://juejin.im/post/6847902217135980557">动态代理：JDK Proxy 和 CGLIB</a>

4. <a href="https://juejin.im/post/6847902217140191239">Spring AOP特性解读</a>

5. <a href="https://juejin.im/post/6850418115131097102">手写一个Spring AOP框架</a>

笔者认为重点是3和5，这两部分针对实现原理做了深入的探讨，有一定基础的同学建议直接阅读重点部分。

# AOP简介

## AOP是什么

<div style="background-color:#eee;">aspect-oriented programming (AOP) is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code (an advice) without modifying the code itself, instead separately specifying which code is modified via a "pointcut" specification, such as "log all function calls when the function's name begins with 'set'". This allows behaviors that are not central to the business logic (such as logging) to be added to a program without cluttering the code, core to the functionality. AOP forms a basis for aspect-oriented software development.</div>

Wiki对AOP的定义，AOP是一种编程范式，目的是为了将跨领域的关注点分离出来以达到模块化。它可以向现存代码中增加行为逻辑而不用修改原有代码，它是通过指定切入点（pointcut）来实现的，例如向以set为开头的函数（function）中增加日志功能。它可以实现将一些不是核心的业务逻辑（如日志等）添加到程序中，而不会使核心代码混乱。AOP为面向方面的软件开发奠定了基础。

从定义中，我们提取几个关键字：

* 编程范式，和面向对象编程类似，AOP也是一种编程范式，它是面向切面编程
* 模块化，将程序模块化是AOP的追求目标
* 非核心业务逻辑，AOP应用于日志、监控等这种非核心业务中

我们以一个简图来说明：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/5/1731e03776795bbb~tplv-t2oaga2asx-image.image)

从图中可以看出，业务A和业务B中都有log业务，面向切面的编程思想就将这些log业务从主业务中剥离出来，单独形成一个切面（Aspect），然后在切面中进行处理（Advice）.

## AOP中有哪些概念

我们以AspectJ的接入点模型（join-point model）为例，说明一下AOP中的概念

* 接入点（Join Point）：方法调用，构造器调用，初始化class，实例化对象，成员变量的读写，异常处理等都可以成为接入点

* 切点（PointCuts）：代表了一些接入点（Join Point）的集合，比如：

  ```
  execution(* set*(*))
  ```

  以方法接入点为示例，表达式代表了匹配以set为开头并且只有一个参数的方法

* Advice：指的可以在接入点的前（before），中（around），后（after）执行的代码

* 切面（Aspect）：切面像是一个抽象出来的类，它不仅融合了以上的元素，它也可以包含自己的属性，方法等，当然，切面也可以被实例化。在Spring AOP中，切面的应用较为简单，为了便于理解，我们可以把切面理解为一个实现具体业务的类，例如计算Dao层方法执行时间的类。

*注：AspectJ中的AOP实现比Spring AOP复杂的多，本文主要以理解Spring AOP为目的，不做深入的探讨，有兴趣进一步学习的同学可以参考附件中的AspectJ的官方文档*

## AOP的应用场景

前面我们提到AOP主要应用一些非核心业务逻辑中，我们看看AOP常见的应用场景

* 监控
* 日志
* 缓存
* 鉴权
* 事务
* 异常处理
* 持久化

# 引用资料
Wiki：https://en.wikipedia.org/wiki/Aspect-oriented_programming

AspectJ：https://www.eclipse.org/aspectj/doc/released/progguide/index.html

Spring AOP：https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#aop