---
layout: post
title: "Spring AOP实现原理（二）实现一个简单的AOP"
categories: misc
---

# Spring AOP 简单示例

在了解AOP之后，我们以注解的方式写一个Spring AOP的示例，这种例子网上很多，本文只贴一些关键性的代码

```java
package com.aop.biz;

class BizA {
    public void doSomething() {
        //do something...
    }   
}

class BizB {
    public void doSomething() {
        //do something...
    }    
}
```

定义两个业务类

```java
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class LogAspect {
    
    @Before("execution(** com.aop.biz.*.*(..))")
    public void before() {
        System.out.println("before");
    }

    @AfterReturning("execution(** com.aop.biz.*.*(..))")
    public void after() {
        System.out.println("after");
    }
}
```

实现一个切面（@Aspect），包括一个前置处理（@Before），一个后置处理（@AfterReturning）和一个切点

```
（execution(** com.aop.biz.*.*(..))）
```

匹配biz包下的所有方法。

当然也可以进一步抽取，将切点抽成一个方法myPointcut，其它的Advice引入这个方法

```java
@Aspect
public class LogAspect {

    @Pointcut("execution(** com.aop.LogAspect.doSomething(..))")
    public void myPointcut() {

    }
  
    @Before("mypointcut()")
    public void before() {
        System.out.println("before");
    }

    @AfterReturning("mypointcut()")
    public void after() {
        System.out.println("after");
    }
}
```

以上就是一个简单的示例，为了更进一步理解Spring AOP，下章节中我们自己尝试实现一个AOP

# 自己实现一个AOP


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/5/1731e055b43b876f~tplv-t2oaga2asx-image.image)

## 定义注解

Aspect 注解类

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Aspect {
    public String value() default "";
}
```

Before注解类（仅定义Before一种advice）

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Before {
    String value();
}
```

Pointcut注解类

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Pointcut {
    String value() default "";
}
```

定义一个日志切面

```java
@Aspect
public class MyLogAspect {
    @Pointcut("com.aop.biz.*.*")
    public void pointcut() {
    }

    @Before("pointcut")
    public void beforeAdvice(Method method,Object object) {
        System.out.println("-------before------");
        System.out.println(object.getClass().getName());
    }
}
```

## 实现AOP


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/5/1731e0586b9da924~tplv-t2oaga2asx-image.image)

```java
final ClassPath cp = ClassPath.from(AspectLoader.class.getClassLoader());
final ImmutableSet<ClassPath.ClassInfo> allClasses = cp.getAllClasses();

//1.获取具有@Aspect注解的类
Class aspectClass = getAspectClass(allClasses.asList());

//2-1.获取before
Before before = getBefore(aspectClass);

//2-2.获取before相关的方法
Method beforeMethod = getBeforeMethod(aspectClass);

//2-3.获取切点
String methodName = before.value().replace("()", "");
Pointcut pointcut = getPointcut(aspectClass, methodName);

//3.构建advice和pointcut关系
//...

//4.定位切点目标类
List<Class> targets = getTargetClasses(allClasses.asList(),pointcut.value());

//5-1.生成aspect实例
Object aspectObject = aspectClass.newInstance();

//5-2.通过动态代理，将advice添加到代理类的interceptor中，并最终把代理对象放入容器中以供调用
for (Class c : targets) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(c);
    enhancer.setCallback(new MethodInterceptor(){
        @Override
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            // TODO Auto-generated method stub
            beforeMethod.invoke(aspectObject,method,obj);
            proxy.invokeSuper(obj, args);
            return obj;
        }
    });

    Object o = enhancer.create();
    this.beanContainer.put(c.getSimpleName(),o);
}
```

前面的1到4步都是根据Java反射的特性实现的，这里不再赘述。关键是第5步，基于cglib实现了对象的代理，将before类型的advice动态加到了目标方法调用之前。

## 调用

```java
public static void main(final String[] args) throws Exception {
    AspectLoader aspectLoader = new AspectLoader();
    aspectLoader.init();
    BizA bizA = (BizA) aspectLoader.beanContainer.get(BizA.class.getSimpleName());
    bizA.doSomething();

    BizB bizB = (BizB) aspectLoader.beanContainer.get(BizB.class.getSimpleName());
    bizB.doSomething();
}
```

调用结果：

```
-------before------
com.aop.biz.BizA?EnhancerByCGLIB?6fba1eda
-------before------
com.aop.biz.BizB?EnhancerByCGLIB?7b80fb85
```

*注：我们可以看到，原本BizA类，实际上执行的时候变成了BizA?EnhancerByCGLIB?xxx，这个其实就是cglib生成的动态代理类*