# EventBus的主要模块

#### Subscribe

注解，可以标注哪个方法可以被注册和通知。它要求被注解的方法有且只有一个参数，并且该参数就是要注册监听的事件，例如：

```java
class EventBusChangeRecorder {
  @Subscribe 
  public void recordCustomerChange(ChangeEvent e) {
    recordChange(e.getChange());
  }
}
```

注册时，无须明确指定事件

```java
eventBus.register(new EventBusChangeRecorder());
```

#### Subscriber

被@Subscribe注解的方法，可以被调用执行，相当于事件处理器

```java
private EventBus bus;
final Object target;
private final Method method;
private final Executor executor;

final void dispatchEvent(final Object event) {
    this.executor.execute(new Runnable() {
        public void run() {
            try {
                Subscriber.this.invokeSubscriberMethod(event);
            } catch (InvocationTargetException var2) {
                Subscriber.this.bus.handleSubscriberException(var2.getCause(), Subscriber.this.context(event));
            }
        }
    });
}
```

target，事件注册的实例

method，Java反射中的方法实例

dispatchEvent，对外暴露的调用（事件分发）方法

#### SubscriberRegistry

事件的注册表类，主要提供了注册方法，取消注册方法

```java
void register(Object listener) {
    Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

    for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
      Class<?> eventType = entry.getKey();
      Collection<Subscriber> eventMethodsInListener = entry.getValue();

      //引入了CopyOnWriteArraySet解决集合的线程安全问题
      CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);
        
      if (eventSubscribers == null) {
        CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
      }
	  
      eventSubscribers.addAll(eventMethodsInListener);
    }
  }
```

```java
void unregister(Object listener) {
    Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

    for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
      Class<?> eventType = entry.getKey();
      Collection<Subscriber> listenerMethodsForType = entry.getValue();

      CopyOnWriteArraySet<Subscriber> currentSubscribers = subscribers.get(eventType);
      if (currentSubscribers == null || !currentSubscribers.removeAll(listenerMethodsForType)) {
        throw new IllegalArgumentException();
      }
    }
  }
```

从代码中我们可以看到，这两个方法都实现了线程安全，通过使用CopyOnWriteArraySet巧妙解决了注册/取消注册时的线程安全问题。

CopyOnWriteArraySet，在写入数据的时候，会创建一个新的 set，并且将原始数据 clone 到新的 set 中，在新的 set 中写入数据完成之后，再用新的 set 替换老的 set。这样就能保证在写入数据的时候，不影响数据的读取操作，以此来解决读写并发问题。

*后续有时间解析一下CopyOnWriteArraySet的源码*

#### EventBus

事件总线的组合类，组合了事件分发器（后面要提的Dispatcher）、事件注册表等，统一对外提供注册、分发等功能

```java
private final String identifier;
private final Executor executor;
private final SubscriberExceptionHandler exceptionHandler;

private final SubscriberRegistry subscribers = new SubscriberRegistry(this);
private final Dispatcher dispatcher;

public void register(Object object) {
   subscribers.register(object);
}

public void unregister(Object object) {
   subscribers.unregister(object);
}

public void post(Object event) {
    Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
      // the event had no subscribers and was not itself a DeadEvent
      post(new DeadEvent(this, event));
    }
}
```

总结一下，以上几个模块构成了EventBus的主体框架，基于这个框架解决了我们上一章节提到的

* 线程安全问题：通过引入线程安全集合CopyOnWriteArraySet

* 显式注册问题：通过定义注解类Subscribe，标记可以被注册的方法，并且将要监听的Event作为方法的唯一参数，再利用Java反射的特性，实现了隐式注册

# 其它模块

#### Dispatcher

EventBus提供专门的事件分发器，并且为事件的分发提供了两种策略，一种是广度优先，一种是深度优先。

```java
private static final class PerThreadQueuedDispatcher extends Dispatcher {

  private final ThreadLocal<Queue<Event>> queue;
  private final ThreadLocal<Boolean> dispatching;

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    Queue<Event> queueForThread = queue.get();
    queueForThread.offer(new Event(event, subscribers));

    if (!dispatching.get()) {
      dispatching.set(true);
      try {
        Event nextEvent;
        while ((nextEvent = queueForThread.poll()) != null) {
          while (nextEvent.subscribers.hasNext()) {
            nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
          }
        }
      } finally {
        dispatching.remove();
        queue.remove();
      }
    }
  }
}
```

上面的分发器，在被同一个线程分发（同一个线程调用post）时，能够保证事件分发的有序性。同时也引入了Queue实现了广度优先，下面我们看一下另外一个深度优先的实现

```java
private static final class ImmediateDispatcher extends Dispatcher {

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    while (subscribers.hasNext()) {
      subscribers.next().dispatchEvent(event);
    }
  }
}
```

#### SubscriberExceptionHandler

上章节中我们同样提到了调用时异常处理的问题，guava同样给处理解决办法。guava定义了一个接口

```java
public interface SubscriberExceptionHandler {
  /** Handles exceptions thrown by subscribers. */
  void handleException(Throwable exception, SubscriberExceptionContext context);
}
```

在调用出现异常时，回调这个接口（完整代码请参考Subscriber的dispatchEvent方法）

```java
try {
    Subscriber.this.invokeSubscriberMethod(event);
} catch (InvocationTargetException var2) {
    //出现异常，调用自定义异常处理
    Subscriber.this.bus.handleSubscriberException(var2.getCause(), Subscriber.this.context(event));
}
```

同时在实例化EventBus时，可传入自定义异常处理

```java
public EventBus(SubscriberExceptionHandler exceptionHandler) {
    this("default", MoreExecutors.directExecutor(), Dispatcher.perThreadDispatchQueue(), exceptionHandler);
}
```

以上，结合上章节提出的问题，对guava的EventBus做了分析，我们能看到Google实现的Eventbus代码很优雅，程序也很健壮，他们在设计的时候会考虑到很多方面，这对我们自己编程以及代码框架会有不少启发。

# 最后

有关EventBus的整个系列都写完了，在写作的过程中，我不断回看guava的源码，收获甚多。建议大家也去读一读guava源码，了解一下世界上顶级的Java开发者是如何写代码的。

https://github.com/google/guava