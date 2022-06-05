---
layout: post
title: "我们在谈论synchronized的时候，我们在谈论什么？"
categories: misc
---

## synchronized是做什么用的？
synchronized是Java中实现锁的一种方式，我们可以通过synchronized来给一个方法，一个属性，一个对象等资源进行加锁。

### 我们为什么需要加锁呢？
可能你会说，是因为当某个资源被多个线程访问时，我们需要同步协调线程访问的顺序，在这种情况下，我们要对该资源加锁。

比如，在火车票放票期间，禁止售票员访问票源，这个本质上就是将资源（火车票源）加锁，协调了售票员和管理员的操作顺序。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/23/16b83880fc4eaa72~tplv-t2oaga2asx-image.image)

如果给外行人解释，这么说已经足够了。但对于一个有态度的技术人来说，这种描述就太浅显了。这个问题，我们还得从源头上说起。

我们使用一段代码来表达上面的例子：
```Java
//火车票程序
public class TrainTicket {
  int beiJing = 0, shangHai = 0;
  
  //放票
  public void writer() {
    beiJing = 1;
    shangHai = 2;
  }

  //查票
  public void reader() {
    int r1 = beiJing;
    int r2 = shangHai;
  }
}
```

```Java
//T-1 放票线程
recordering.writer();
```

```Java
//T-2 查票线程
recordering.reader();
```

我们按“顺序”执行T1和T2，结果会是什么呢？
我们期望的结果r1=1, r2=2
但是结果很可能是r1=0, r2=0
也可能是r1=1, r2=0
也可能是...

为什么和我们预期的结果不一样呢，是哪里出了问题？

**这就是我们今天要重点说的一个概念——重排序**

重排序（Reordering）是编译器（Compiler）为了优化执行效率而做的一种策略。在单线程中，重排序要保证不影响程序的语义，因此对于没有依赖关系的语句，都可能被重排序。
比如
```Java
int a=0;
int b=1;
```
第一行语句和第二行语句并不构成依赖，所以编译器可以任意调换顺序。

重点来了，那么在多线程中环境中，涉及到重排序时，就会遇到线程安全的问题。
因此，Java编译器并不会保证线程安全，线程是否安全由程序员确保的。

**这不是甩锅嘛！！！**

**么办法，这锅就是程序员的！**

好吧，让我们再回到最初的问题：
怎么更好地背锅？

哦，不对。

**为什么，我们需要对共享资源加锁？**

敲黑板，划重点

**加锁是为了消除程序因重排序而产生的线程安全问题，最终保证语义的一致性和数据的一致性！**


说到这，好像说的比较清楚了，但是还有一个根本性的问题
### 当我们用了synchronized（锁），怎么就能做到线程安全呢？

#### happen-before原则
在Java的内存模型(JMM)中定义了一系列的happen-before原则，具体这个原则如何描述，笔者也不好把握，如果执意要下定义的话，我认为：
**happen-before是Java提供的一系列的确保局部有序的规则。**
再具体一点就是，如果A操作happens-before于B操作，那么也就意味着A的操作结果对B是可见的。

可以回到我们火车票的例子理解一下，如果**出票**操作happens-before于**查票**操作，那么出票的结果对查票来说一定是可见的，也就是说出票结果一定会被正确查到。

下面是具体的每一条规则

* Each action in a thread happens before every action in that thread that comes later in the program's order.
* An unlock on a monitor happens before every subsequent lock on that same monitor.
* A write to a volatile field happens before every subsequent read of that same volatile.
* A call to start() on a thread happens before any actions in the started thread.
* All actions in a thread happen before any other thread successfully returns from a join() on that thread.

这些规则的中文翻译网上有很多，我之所以贴英文，主要是考虑到反正这种条文没有人会去记，反倒是贴英文官方文档更合适一些，也能帮助到想查官方文档的同学。

针对第二条（关于锁）的规则扩展一下。
同一个锁的unlock操作在lock之前，也就是说
一个锁处于被锁定状态，那么必须先执行unlock操作后面才能进行lock操作。

正式因为有了这条规则，我们就可以通过加锁的方式实现线程安全，将以上代码改造一下

```Java
//火车票程序
public class TrainTicket {
  int beiJing = 0, shangHai = 0;
  
  //放票
  public synchronized void writer() {
    beiJing = 1;
    shangHai = 2;
  }

  //查票
  public synchronized void reader() {
    int r1 = beiJing;
    int r2 = shangHai;
  }
}
```

这样，将两个方法都加上锁，这样就实现了线程的安全。

#### 编译器做了什么？
synchronized代码块编译之后会生成一个monitorenter指令和一个或多个monitorexit指令，大致如下：
```Java
monitorenter
/*---------*/
     code
/*--------*/
monitorexit
```

对于monitorenter和monitorexit，我们可以理解为每个锁对象拥有一个锁的计数器和一个指向持有该锁的线程的指针。

当执行monitorenter指令时，如果目标锁对象的计数器为0，那么说明它没有被别的线程持有，在这个情况下，Java虚拟机会将该锁对象的持有线程设置为当前线程，并且将计数器加1.

在目标锁对象的计数器不等于0的情况下，如果其它线程访问，则需要等待，直到持有锁的线程释放该锁。（联想一下happen-before中的第二条规则）

### 写在最后
有几个问题是可扩展的

1、重排序有三个维度上的，分别是编译器，内存和处理器。本文中只提到了编译器的重排序，而没有提到内存的重排序。

2、从内存的维度上，是如何禁止重排序的？

3、happen-before原则中提到了volatile类型变量，这个类型的变量有什么特殊之处，我们能用它解决什么呢？

下次有时间再细说

**本文原创，转载请注明出处**

引用参考（reference）：

http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html
https://docs.oracle.com/javase/specs/jls/se10/html/jls-17.html#jls-17.4