---
layout: post
title: "解构EventBus框架（三）自己实现一个EventBus"
categories: misc
---

# EventBus实现的思路

定义类：

* MyEventBus 事件总线

  方法：

  * register 注册
  * unregister 取消注册
  * post 分发

* Event 事件

* EventHandler 事件处理器

以下以两张图来表达实现的原理

注册：


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/21/172d4b9cfb8ec4a3~tplv-t2oaga2asx-image.image)

分发：


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/21/172d4b9fe4ae2b22~tplv-t2oaga2asx-image.image)

实现EventBus最关键的地方是，维护一个事件/处理器的注册表。在收到注册请求的时候，将内容加到注册表中；在收到调用请求的时候，从注册表中取出对应的处理器并调用。

# 动手实现一个EventBus

定义事件-事件处理器：

```java
public class Event<T> {
    private T entity;
    public Event() {
    }
    public Event(T entity) {
        this.entity = entity;
    }
}
```

```java
public interface EventHandler {
    void handle(Event event);
}
```

定义EventBus：

```java
	//注册集合
    private Map<Class<? extends Event>, List<EventHandler>> map = new HashMap<Class<? extends Event>, List<EventHandler>>();
    private Executor executor = Executors.newFixedThreadPool(10);

    public void register(Class<? extends Event> event, EventHandler eventHandler) {
        List<EventHandler> list = map.get(event);
        if(list != null) {
            list.add(eventHandler);
        } else {
            list = new ArrayList<EventHandler>();
            list.add(eventHandler);
            map.put(event,list);
        }
    }

    public void post(final Event event) {
        List<EventHandler> list = map.get(event.getClass());
        if(list != null) {
            for (final EventHandler eventHandler : list) {
                executor.execute(new Runnable() {
                    public void run() {
                        eventHandler.handle(event);
                    }
                });
            }
        }
    }

    public void unRegister(Class<? extends Event> event, EventHandler eventHandler) {
        List<EventHandler> list = map.get(event);
        if(list != null) {
            list.remove(eventHandler);
        }
    }
```

流程调用：

```java
 		//实例化，并注册
        MyEventBus myEventBus = new MyEventBus();
        EventHandler handler = new EventHandler() {
            public void handle(Event event) {
                User user = (User) event.getEntity();
                System.out.println(user.getId());
            }
        };
        myEventBus.register(OpenUserEvent.class,handler);



        //开户业务
        User user = new User();
        user.setId("3");
        user.setName("test");

        //实例化事件，并发送
        OpenUserEvent openUserEvent = new OpenUserEvent();
        openUserEvent.setEntity(user);
        myEventBus.post(openUserEvent);
```

以上我们实现了一个简单的EventBus，但这个EventBus还存在以下问题：

* 线程安全，在注册和取消注册的时候，会产生线程安全的问题
* 注册时，需要显示指定事件和事件处理器
* 事件在分发过程（异步调用）中，如果出现异常，无法被调用者感知

下章节中，以Google的EventBus为例，说明如何解决以上问题