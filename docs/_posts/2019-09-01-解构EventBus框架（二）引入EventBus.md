---
layout: post
title: "解构EventBus框架（二）引入EventBus"
categories: misc
---

# 总线的由来

总线的概念来源于计算机硬件，指的是各个硬件之间的交互方式。总线提供了一个通用的方式为各组件提供数据传输和控制逻辑。

我们引入“事件总线”，它为我们解决了两个核心的问题

* 统一的事件注册

* 统一的事件分发

# 引入EventBus

以Google的EventBus为例，我们直接看下实现

1、标记

```java
@Subscribe
public void openUserHandler(Event event) {
}
```

2、注册

```java
// eventbus 实例化
final EventBus eventBus = new EventBus("default");

//注册服务
eventBus.register(resourceService);
eventBus.register(smsPushService);
eventBus.register(workOrderService);
```

3、分发

```java
//开户业务
User user = new User();
user.setId("2");
user.setName("wang");

//开户之后发送Event
Event event = new Event(user);
eventBus.post(event);
```

使用EventBus之后，只需要少量的代码即可实现业务解耦。

# 对比之前的实现

##### 注册

* 之前实现

  没有实现统一注册，每个事件注册自己的事件处理器，例如：开户-开户处理器、修改密码-修改密码处理器等

* EventBus

  通过注解标记方法，完成事件和事件处理器的映射，实现了统一注册

##### 分发

* 之前实现

  事件的分发者需要关注具体应该发给谁

* EventBus

  调用方无须关注，可以将事件post出来

本章内容主要引入了EventBus以及和之前的实现做个对比，下章节中我们自己尝试动手实现一个EventBus。