---
layout: post
title: "Spring AOP 实现原理（四）特性解读"
categories: misc
---

# AspectJ简介

我们花了很多篇幅讲解了Spring AOP的实现原理动态代理，但是只有动态代理是不够的，比如说前面提到的Aspect、Join Point、Pointcut等，这些有关AOP的抽象概念也非常重要，我们知道这部分实际上是由AspectJ提供的，那么AspectJ是什么呢？

aspectj is

- a seamless aspect-oriented extension to the Javatm programming language
- Java platform compatible
- easy to learn and use

AspectJ都包括了哪些内容？

* 提供了面向切面的模型，像aspect、pointcut等
* 有自己的一套完整的语言体系，类似于execution(** com.aop.biz.*.*(..)) 表达式这种
* 拥有自己的一套开发编译工具
* 本质上也是通过访问字节码现的AOP

或许有人要问，既然AspectJ是需要编译的，那么Spring AOP怎么没有使用额外的编译工具呢？

那是因为Spring仅使用了AspectJ的模型和语言表达式部分。

AspectJ的相关内容非常多，几篇文章都不一定能说的完，这块不是我们讨论的重点，有兴趣的同学请参考附录资料。

# Spring AOP官方文档解读

在解读源码之前，我们先看下Spring AOP的官方文档，从中我们能获取到极为重要的信息。

<div style="background-color:#eee;">
Spring provides simple and powerful ways of writing custom aspects by using either a schema-based approach or the @AspectJ annotation style. Both of these styles offer fully typed advice and use of the AspectJ pointcut language while still using Spring AOP for weaving.
</div>

Spring 提供了schema和@AspectJ注解两种风格，这两种风格本质上使用的都是AspectJ提供的切入点语言。

<div style="background-color:#eee;">
Weaving: linking aspects with other application types or objects to create an advised object. This can be done at compile time (using the AspectJ compiler, for example), load time, or at runtime. Spring AOP, like other pure Java AOP frameworks, performs weaving at runtime
</div>

在Spring AOP的织入特性中，我们得知Spring AOP仅支持runtime织入，而AspectJ支持compile time，load time，runtime三种.

<div style="background-color:#eee;">
Spring AOP Capabilities and Goals

Spring AOP currently supports only method execution join points (advising the execution of methods on Spring beans). Field interception is not implemented, although support for field interception could be added without breaking the core Spring AOP APIs. If you need to advise field access and update join points, consider a language such as AspectJ.

Spring AOP never strives to compete with AspectJ to provide a comprehensive AOP solution. We believe that both proxy-based frameworks such as Spring AOP and full-blown frameworks such as AspectJ are valuable and that they are complementary, rather than in competition.
</div>

上面这段内容主要提到了，Spring AOP仅支持method接入点，不支持field接入点；Spring AOP的目标从来都不是提供完整的AOP解决方案，Spring AOP和AspectJ更像是互补的关系，而不是竞争关系.

<div style="background-color:#eee;">
AOP Porxies

Spring AOP defaults to using standard JDK dynamic proxies for AOP proxies. This enables any interface (or set of interfaces) to be proxied.
Spring AOP can also use CGLIB proxies. This is necessary to proxy classes rather than interfaces. By default, CGLIB is used if a business object does not implement an interface
</div>

Spring AOP默认使用JDK动态代理，同时也支持CGLIB，当代理类而不是接口的时候，默认使用CGLIB方式.



**从官方文档中我们能够得出来以下重要信息：**

* Spring AOP是一个纯Java的AOP框架
* Spring AOP是基于代理（proxy-based）的AOP框架
* Spring AOP并不是一个完整的AOP实现，它主要是提供了AOP和IoC之间的无缝集成
* Spring AOP的pointcut采用了AspectJ的语言
* Spring AOP仅支持runtime织入，不支持compile time和load time
* Spring AOP仅支持method执行的join point，不支持field的join point
* Spring AOP同时支持JDK proxy和CGLIB，代理接口时使用JDK proxy，代理类时使用CGLIB

# 引用资料

AspectJ：https://www.eclipse.org/aspectj/doc/released/progguide/index.html

cglib:https://github.com/cglib/cglib

Spring AOP：https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#aop

Spring Source Code：https://github.com/spring-projects/spring-framework/tree/master/spring-aop