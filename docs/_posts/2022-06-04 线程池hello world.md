---
layout: post
title: "线程池Hello World"
categories: theadpool,java 核心技术
---

## 线程池如何使用

简单示例，分别以单例启动一个线程，线程池以及主线程打印一个字符串：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newSingleThreadExecutor(new NamedTheadFactory());
        String echoMessage = "你还好吗? ";

        //新启一个单线程执行
        new Thread(() -> System.out.println(Thread.currentThread().getName() + ":" + echoMessage)).start();

        //线程池执行
        executorService.submit(new EchoTask(echoMessage)).get();

        //主线程
        System.out.println(Thread.currentThread().getName() + ":" + echoMessage);

    }
```


执行结果：
```
Thread-0:你还好吗? 
echo-thead:你还好吗? 
main:你还好吗? 
```

## 池化技术
池化技术是一种普遍被应用的技术，他主要解决了以下问题
* 提升性能，减少创建资源的消耗，比如创建线程，TCP连接等
* 统一管理，减少随意的资源创建，将资源（线程，TCP连接等）更方便，更集中的管理
* 资源隔离，比如Netflix的Hystrix采用线程池实验了资源隔离

常见池化技术的应用：
* 线程池
* 数据库连接池
* TCP长连接池

## Java中的线程池如何实现的
Java线程池的核心组件：
* 线程（Thread）/工厂（ThreadFactory）/任务（Worker）
* 任务队列（BlockingQueue）/拒绝策略（RejectedExecutionHandler）
* 状态管理（ctl）
* 执行器（ExecutorService）

