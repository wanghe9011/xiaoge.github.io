---
layout: post
title: "Spring AOP实现原理（五）手写一个Spring AOP框架"
categories: misc
---

# 前言（可跳过）

在开始正文之前，先聊点其它的，原本规划的《Spring AOP实现原理》系列的最后一章节是讲解Spring AOP源码的。刚开始对此也是信心满满的，直到我深入读了源码之后才发现这事情没有那么简单。

首先，Spring AOP源码有些多，不够精简，这就给书面讲解造成很大麻烦。其次，完全基于Spring AOP源码讲解它的实现似乎也没有太大意义。

因此我决定另辟蹊径，从Spring AOP的特性和功能发起，然后结合着Spring AOP实现的思路，大致实现一个Spring AOP的架子。

特别声明：在实现的过程中，由于篇幅原因，砍掉了不少优化部分，特别是有关工厂，懒加载，缓存，并发等。

# 功能拆分

从上一章节中，我们得知了Spring AOP的特性以及其要完成功能，我们抽取出其中的重点列举一下：

* 支持注解和XML两种配置方式
* 能够与Spring IoC结合
* 支持JDK Dynamic Proxy和CGLIB两种代理方式
* 集成AspectJ的Annotation注解，包括切面（@Aspect）、切点（@Pointcut）、通知（@Advice）等
* 在动态代理中实现对Advice的调用



我们针对这些特性做一个功能分析，大致有如下功能：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/14/1734ca3669db9bef~tplv-t2oaga2asx-image.image)

* AOP，主要专注于基于方法的AOP功能实现，分为三个方面：
  * 集成AspectJ，包括集成表达式，使用AspectJ定义的Annotation，以及使用AspectJ提供的工具；
  * 抽象层，包括了Spring对AOP概念的封装和抽象，重点理解一下Advisor，这是Spring独有的概念；
  * 实现层，自定义方法拦截器MethodInterceptor实现对AOP目标方法以及Advice方法的调用

* 对象代理，支持JDK和CGLIB两种代理方式，实现根据目标对象（target）生成对应的代理对象

* 配置解析，支持XML和Annotation两种配置方式的解析，实现根据配置解析出对应的Advisor等

* IoC集成，集成BeanFactory，实现对IoC容器中bean的访问；实现BeanPostProcesser，将配置解析、创建代理对象等纳入到bean初始化流程



注：我们看下官网对Advisor的解释：*

*An advisor is like a small self-contained aspect that has a single piece of advice.*

*Advisor是具有**单个**Advice以及可以使用Pointcut的组合类，可以看作是一个特殊的Aspect。



在列出了Spring AOP功能之后，我们接下来讨论下功能实现的流程

# 流程拆分

我把流程简单分为两大部分，创建代理阶段和代理调用阶段，其中前者是由Spring IoC初始化触发的，后者是由程序调用触发的，详细流程参考下图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/14/1734ca3c8add658d~tplv-t2oaga2asx-image.image)


## 创建代理阶段

1. Spring IoC容器触发Bean初始化，通过BeanPostProcesser接口

2. 调用BeanPostProcesser接口实现方法postProcessAfterInitialization
3. 进入构建Advisor的流程，通过反射找到所有匹配的Advisor
4. 筛选出符合的Advisor
5. 进入创建代理的流程，将上一个流程中得到的Advisor集合传递给代理对象，并且根据规则判断使用哪种代理方式

## 代理调用阶段

1. 外部的方法调用。以日志打印切面举例，当调用日志切面中的方法时，会触发代理的调用
2. 调用回调方法，相当于将目标方法的调用委托给了回调方法。在生成代理对象时，都有对应的回调，比如，CGLIB中设置CallBack，JDK中设置InvocationHandler
3. 在回调方法中，构建方法拦截器链（后面会针对拦截器链重点讲解）
4. 进一步将调用权委托给拦截器链，由拦截器链完成执行

# 实现部分

在列出了Spring AOP功能之后，我们接下来讨论功能的实现部分功能的实现

## AOP抽象概念定义

首先对AspectJ中的概念抽象，我们简单定义下Aspect、JoinPoint、Advice、Pointcut等类。

```java
public class Aspect {
}
public class JoinPoint {
}
public interface Pointcut {
    public String getExpression();
}
public class Advice {
    private Method adviceMethod;
    private Aspect aspect;
}
```

SpringAOP中引入了Advisor的概念，我们同时定义一个Advisor类

```java
public class Advisor {
    private Advice advice;
    private Pointcut pointcut;
    private Aspect aspect;
}
```

为了融合AspectJ的表达式，我们针对Pointcut进一步改造

增加字符串表达式转换为AspectJ的表达式(PointcutExpression)

```java
	import org.aspectj.weaver.tools.PointcutExpression;
	//....

	/**
     * 转换为AspectJ的切入点表达式
     * @return
     */
    public PointcutExpression buildPointcutExpression();
```

引入AspectJ解析类(PointcutParser)，实现buildPointcutExpression方法

```java
import org.aspectj.weaver.tools.PointcutExpression;
import org.aspectj.weaver.tools.PointcutParser;

public class AspectJPointcut implements Pointcut {

    public String expression;

    public AspectJPointcut(String expression) {
        this.expression = expression;
    }

    @Override
    public String getExpression() {
        return this.expression;
    }
	
    
    @Override
    public PointcutExpression buildPointcutExpression() {
        PointcutParser parser = PointcutParser
      .getPointcutParserSupportingAllPrimitivesAndUsingContextClassloaderForResolution();
        return parser.parsePointcutExpression(this.expression);
    }
    
}
```

以上，定义了几个基本类。有的小伙伴会说，为什么没有看到BeforeAdvice这些定义呢？这里先卖个关子，等到后面引入方法拦截器的时候再定义。

## 创建代理阶段

### IoC集成

集成本身比较简单，实现接口BeanPostProcessor和BeanFactoryAware，直接上代码

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.config.BeanPostProcessor;

public abstract class AbstractAOPProxyCreator implements BeanPostProcessor, BeanFactoryAware {
    
    //子类可实现
    protected void initBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }
    
    //获取匹配的Advisor
    protected abstract List<Advisor> getMatchedAdvisors();

    //创建代理对象
    protected abstract Object createProxy(List<Advisor> advisors, Object bean);

    @Override
    public void setBeanFactory(BeanFactory arg0) throws BeansException {
        
    }
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName){
		return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName){
		return bean;
    }

}
```

实现了BeanFactoryAware接口的setBeanFactory方法，以及BeanPostProcessor接口的postProcessAfterInitialization方法和postProcessBeforeInitialization方法。

下面我们引入**模板方法**设计模式，来制定处理流程：

```java
	public Object postProcessAfterInitialization(Object bean, String beanName){
        //构建所有Advisor
        List<Advisor> advisors = buildAdvisors();
        //获取符合的Advisor
        advisors = this.getMatchedAdvisors();
        //根据获取的Advisor生成代理对象
        Object object = createProxy(advisors,bean);
        //返回代理对象
		return object == null ? bean : object;
    }
```

### 解析配置

解析配置主要就是用到了反射，找到被@Aspect标记的类，进而找到@Advice，@Pointcut等，最终将这些组合成Advisor实例，实现起来并不复杂，不再赘述

```java
public class AnnotationParser implements ConfigParser {
	//避免重复构建，增加了缓存
    private final Map<String, List<Advisor>> cache = new ConcurrentHashMap<>();

    @Override
    public List<Advisor> parse() {
        if(cache != null) {
            return getAdvisorsFromCache();
        } 
		//获取所有被@Aspect注解的类
        List<Class> allClasses = getAllAspectClasses();
        for (Class class1 : allClasses) {
            cache.putIfAbsent(class1.getName(), getAdvisorsByAspect(class1));
        }
        
        return getAdvisorsFromCache();
    }
    
    /**
     * 根据Aspect类生成Advisor类
     * @param class1
     * @return
     */
    private List<Advisor> getAdvisorsByAspect(Class class1) {
        List<Advisor> advisors = new ArrayList<>();
        for (Method method : getAdvisorMethods(class1)) {
            Advisor advisor = getAdvisor(method, class1.newInstance());
            advisors.add(advisor);
        }
        return advisors;
    }
}
```

### 筛选符合的Advisor

我们要从所有的Advisor中过滤出来与代理目标Bean相关的，以Bean的方法和Advisor的Pointcut作为过滤条件，这里利用了AspectJ的表达式以及比对工具

```java
	import org.aspectj.weaver.tools.ShadowMatch;
	
	/**
     * 从所有的Advisor中获取匹配的
     * @param advisors
     * @return
     */
    public static List<Advisor> getMatchedAdvisors(Class cls, List<Advisor> advisors) {
        List<Advisor> aList = new ArrayList<>();
        for (Method method : cls.getDeclaredMethods()) {
            for (Advisor advisor : advisors) {
                ShadowMatch match = advisor.getPointcut()
                                    .buildPointcutExpression()
                                    .matchesMethodExecution(method);
                if(match.alwaysMatches()) {
                    aList.add(advisor);
                }
            }
        }
        return aList;
    }
```

### 创建代理对象

我们定义了一个工厂，代理对象转交给由工厂创建

```java
public class AOPProxyFactory {

    public Object getProxyObject(List<Advisor> advisors, Object bean) {
        if(isInterface()) {
           return new CglibProxyImpl(advisors,bean).getProxyObject();
        } else {
            return new JdkDynamicProxyImpl(advisors,bean).getProxyObject();
        }
    }

    private boolean isInterface() {
        return false;
    }
    
}
```

同时也实现了两种代理方式，JDK Dynamic Proxy和CGLIB，下面逐一讲解下

*注：可以先忽略掉方法拦截器链*

JDK 实现方式，需要实现InvocationHandler接口，并且在接口方法invoke中实现方法拦截器链的调用

```java
public class JdkDynamicProxyImpl extends AOPProxy implements InvocationHandler {

    public JdkDynamicProxyImpl(List<Advisor> advisors, Object bean) {
        super(advisors, bean);
    }

    @Override
    protected Object getProxyObject() {
        return Proxy.newProxyInstance(this.getClass().getClassLoader(), ReflectHelper.getInterfaces(this.getTarget().getClass()), this);
    }


    /**
     * 实现InvocationHandler的接口方法,将具体的调用委托给拦截器链MethodInterceptorChain
     */
	@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        
        MyMethodInterceptor[] iterceptors = 
            AdvisorHelper.getMethodInterceptors(this.getAdvisors(), method);
        
        Object obj = new MethodInterceptorChain(iterceptors)
                        .intercept(method,args,proxy);
        return obj;
    }

}
```

CGLIB是通过回调（Callback）实现的，需要实现CGLIB的MethodInterceptor

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class CglibProxyImpl extends AOPProxy {

    public CglibProxyImpl(List<Advisor> advisors, Object bean) {
        super(advisors, bean);
    }

    @Override
    protected Object getProxyObject() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.getTarget().getClass());
        enhancer.setCallback(new AOPInterceptor());
        return enhancer.create();
    }

    /**
     * 实现cglib的拦截器,在intercept中将拦截器调用委托给拦截器链MethodInterceptorChain
     */
    private class AOPInterceptor implements MethodInterceptor {

        @Override
        public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            MyMethodInterceptor[] iterceptors = AdvisorHelper.getMethodInterceptors(CglibProxyImpl.this.getAdvisors(), method);
            
            Object o = new MethodInterceptorChain(iterceptors)
                .intercept(method, args, obj);
            return o;
        }
    }
}
```

以上，基本上实现了创建代理对象的流程，那么我们思考一个问题

**在调用代理方法的时候，是如何实现调用我们定义的Advice的呢？**

## 思考：Advice方法调用的实现

### 简单实现示例

我们先用一种简单的实现方式说明一下，以JDK代理方式为例：

```java
	@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return obj;
    }
```

我们知道，执行代理对象的任何方法都会进入到invoke里（对动态代理还不清楚的同学可以回看第三章动态代理的实现），那么进入到invoke里面之后我们需要做如下判断：

1. 找到该代理相关的所有的Advisor（这个不难，代理类有advisors属性）
2. 遍历Advisor集合，逐一判断是否匹配。判断规则为method和pointcut（这里同样可以采用AspectJ工具类）
3. 找出匹配的Advisor，进一步找出Advice，然后执行Advice

我们以BeforeAdvice为例，展示一下如何实现Advice调用

```java
	@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        BeforeAdvice beforeAdvice = getBeforeAdvice(method);
		beforeAdvice.before(proxy, method, args);//调用Advice处
        return method.invoke(this.bean, args);
    }
	
	
	/**
     * 以返回BeforeAdvice为例
     * @return
     */
    private BeforeAdvice getBeforeAdvice(Method method) {
        for (Advisor advisor : this.getAdvisors()) {
            if(AdvisorHelper.isMatch(advisor, method) 
               && advisor.getAdvice() instanceof BeforeAdvice) {
                return (BeforeAdvice) advisor.getAdvice();
            }
        }
        return null;
    }
```

那么，问题来了，如果我们获取到匹配的Advice中还有AfterAdvice呢？我们向invoke方法中增加代码

```java
	@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        BeforeAdvice beforeAdvice = getBeforeAdvice(method);
		beforeAdvice.before(proxy, method, args);//调用Advice处
        
        Object o = method.invoke(this.bean, args);
        
        AfterAdvice afterAdvice = getAfterAdvice(method);
		afterAdvice.after(proxy, method, args);//调用Advice处
        return o;
    }
```

那么，如果我们获得到两个或者多个相同类型的Advice呢？并且相同类型的Advice间有执行顺序需求。上面这种简单实现就无法满足了，我们需要引入**方法拦截器链**

### 职责链设计模式

这里做一个简单的扩展，很多拦截器（interceptor），过滤器（filter）的实现都是基于职责链模式实现的，在定义方法拦截器链之前，我们先看看Tomcat是如何实现过滤器（filter）的。

*注：确切来说是，Tomcat基于JavaEE标准实现的*

Java Sevlet接口

```java
public interface FilterChain {
    public void doFilter(ServletRequest request, ServletResponse response)
            throws IOException, ServletException;

}

public interface Filter {
    public void init(FilterConfig filterConfig) throws ServletException;

    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;

    public void destroy();
}
```

Tomcat过滤器链实现

```java
public final class ApplicationFilterChain implements FilterChain {
    
    private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
    
    void addFilter(ApplicationFilterConfig filterConfig) {
        //..
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {
        // Call the next filter if there is one
        //C-1
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            Filter filter = filterConfig.getFilter();
			filter.doFilter(request, response, this);
            return;
        }

        // We fell off the end of the chain -- call the servlet instance
        servlet.service(request, response);
        
    }
}
```

Session初始化过滤器

```java
public class SessionInitializerFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        ((HttpServletRequest)request).getSession();
        //C-2
        chain.doFilter(request, response);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // NO-OP
    }

    @Override
    public void destroy() {
        // NO-OP
    }
}
```

以上实现有几个关键点：

* ApplicationFilterChain控制着职责链的执行，这里也可以实现对元素的排序或者规则匹配等，参考C-1代码段
* ApplicationFilterChain通过递归调用实现了链式调用
* 每个Filter都可以注册/添加到执行链当中，待自己执行完之后，再将执行转交给ApplicationFilterChain（chain.doFilter）

### 实现方法拦截器链

我们引入职责链模式，**将Advice抽象成一个MethodInterceptor**。这对功能实现上有如下好处：

* 可以动态添加Advice
* 可以针对Advice进行排序，规则匹配等
* 各个Advice可以单独实现自己的业务逻辑（例如BeforeAdvice），后续也很容易扩展新的Advice

定义拦截器，为了和CGLIB的拦截器区分开，我们命名为MyMethodInterceptor

```java
public interface MyMethodInterceptor {

    public Object intercept(Method method, Object[] arguments, Object target, MethodInterceptorChain chain);
    
}
```

定义BeforeAdvice和AfterAdvice

```java
public class BeforeAdvice extends Advice implements MyMethodInterceptor {
    public BeforeAdvice(Method adviceMethod, Aspect aspect) {
        super(adviceMethod, aspect);
    }

    public void before(final Object target, final Method method, final Object[] args) {
        this.invokeAspectMethod(target, method, args);
        ;
    }

    @Override
    public Object intercept(Method method, Object[] arguments, Object target, MethodInterceptorChain chain) {
        this.before(target, method, arguments);
        return chain.intercept(method, arguments, target);
    }
}

public class AfterAdvice extends Advice implements MyMethodInterceptor {
    
    public AfterAdvice(Method adviceMethod, Aspect aspect) {
        super(adviceMethod, aspect);
    }

    public void after(final Object target, final Method method, final Object[] args) {
        this.invokeAspectMethod(target, method, args);
    }

    @Override
    public Object intercept(Method method, Object[] arguments, Object target, MethodInterceptorChain chain) {
        Object obj = chain.intercept(method, arguments, target);
        this.after(target, method, arguments);
        return obj;
    }
}
```

实现方法拦截器链

```java
public class MethodInterceptorChain {

    private MyMethodInterceptor[] methodInterceptors;

    public MethodInterceptorChain(MyMethodInterceptor[] methodInterceptors) {
        this.methodInterceptors = methodInterceptors;
    }

    private int index = 0;

    public Object intercept(Method method, Object[] arguments, Object target) {
        if (index == methodInterceptors.length) {
            // call method
            return method.invoke(target, arguments);
        } else {
            return methodInterceptors[index++]
                	.intercept(method, arguments, target, this);
        }
        return null;
    }
}
```

那么回到我们最开始的思考题，我们可以把原本简单的实现替换成MethodInterceptorChain，如下：

```java
	@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MyMethodInterceptor[] iterceptors = 
            AdvisorHelper.getMethodInterceptors(this.getAdvisors(), method);
        
        Object obj = new MethodInterceptorChain(iterceptors)
                        .intercept(method,args,proxy);
        return obj;
    }
```

这样，我们就将代理方法的调用转移到了MethodInterceptorChain

最后，这里面还隐藏一个小问题，就是代理对象中的Advisor是所有和这个类相关的，我们仍然需要根据method和pointcut找到与方法相匹配的拦截器，这和前面筛选Advisor的实现是一样的，都是基于AspectJ具



## 最后

在讲完方法拦截器链之后，代理调用的流程也就清晰了，也就不再赘述。

我们本次实现仅仅是基于AspectJ的Annotation配置，Spring AOP同时也支持基于Schema配置。时间与篇幅原因，就不再做深入探讨。

除了本文重点提到的职责链模式，Spring AOP还运用了大量的工厂模式、模板方法模式、适配器模式等。特别是大量使用工厂（比如Aspect工厂，Advisor工厂等）同时配合Spring IoC的情况下，能够支持类和对象（Aspect、Advisor等）强大的管理，包括了加载策略，比如单例，多例，懒加载等。笔者认为，这些值得大家深入学习和研究的。



# 附录

代码地址：https://github.com/wanghe9011/myaop