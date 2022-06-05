---
layout: post
title: "Spring AOP实现原理（三）动态代理"
categories: misc
---

Spring AOP实际上是基于动态代理实现的，只不过Spring 同时支持JDK Proxy和cglib，下面我们来介绍一下这两种实现动态代理的方式

*注：本示例中使用JDK1.8*

## 动态代理代码示例

### JDK Proxy方式

```java
/**
 * 在代理的接口调用时的处理器类
 */
class MyHandler implements InvocationHandler {
    private final Object target;
    public MyHandler(final Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
        // TODO Auto-generated method stub
        System.out.println("-----before----");
        final Object o = method.invoke(this.target, args);
        System.out.println("-----after----");
        return o;
    }
}

public static void main(final String[] args) {
        final BizAImpl bizAImpl = new BizAImpl();
        final IBizA newBizA = (IBizA) 		Proxy.newProxyInstance(MyHandler.class.getClassLoader(),
                bizAImpl.getClass().getInterfaces(),
                new MyHandler(bizAImpl));
        newBizA.doSomething();
}
```

### cglib方式

```java
class MyHandler implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // TODO Auto-generated method stub
        System.out.println("----before----");
        proxy.invokeSuper(obj, args);
        System.out.println("----after----");
        return obj;
    }
}

public static void main(String[] args) {
    MyHandler myHandler = new MyHandler();
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(BizA.class);
    enhancer.setCallback(myHandler);

    BizA bizA = (BizA) enhancer.create();
    bizA.doSomething();
}
```

## 对比JDK Proxy和 cglib的使用

从示例代码中，我们可以得出以下结论

| JDK Proxy                                                    | Cglib                                        |
| ------------------------------------------------------------ | -------------------------------------------- |
| 要求代理类必须实现接口，参见IBizA和BizImpl                   | 支持类的代理，对接口实现没有要求             |
| 需要将代理目标对象（bizImpl）传递给InvocationHandler，在写法上不够灵活 | 可以直接根据类（BizA.class）生成目标代理对象 |

## 动态代理的实现原理

实际上，这两种代理的实现方式也大不一样

### JDK Proxy的实现原理

JDK Proxy是通过复制原有代理类，然后生成一个新类，在生成新类的同时，将方法的调用转给了InvocationHandler，在代理类执行方法时，实际上是调用了InvocationHandler的invoke方法。

我们看下JDK源码，从newProxyInstance方法开始

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) {
    	/*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
    	final Constructor<?> cons = cl.getConstructor(constructorParams);
        return cons.newInstance(new Object[]{h});
    
}
```

newProxyInstance方法只干了两件事

* 获取代理类
* 调用代理类的构造方法，并将handler作为参数传进去（这一点可以通过反编译class文件看到）

那么这里的重点是如果获取代理类呢，我们接着往下看

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```

优先从缓存中获取，如果缓存中没有，会通过工厂（ProxyClassFactory）生成，缓存部分代码略过，直接看生成代理类的流程

```java
	/**
     * A factory function that generates, defines and returns the proxy class given
     * the ClassLoader and array of interfaces.
     */
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
			
            //校验接口，访问权限，确定package等，省略...
            
            /*
             * Choose a name for the proxy class to generate.
             */
            //生成代理类名字
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```

将核心方法提取出来

```java
//生成字节码数据
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
//这是一个native方法
return defineClass0(loader, proxyName,proxyClassFile, 0, proxyClassFile.length);
```

进入方法ProxyGenerator.generateProxyClass()

```java
	public static byte[] generateProxyClass(final String name,
                                            Class<?>[] interfaces,
                                            int accessFlags)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
        final byte[] classFile = gen.generateClassFile();
		
        //这个标志是系统的一个配置，是否保存生成的文件，可以通过配置项
        //sun.misc.ProxyGenerator.saveGeneratedFiles拿到
        if (saveGeneratedFiles) {
            //...保存文件，非讨论重点
        }

        return classFile;
    }
```

生成类文件的核心方法

```java
	private byte[] generateClassFile() {

        /* ============================================================
         * Step 1: Assemble ProxyMethod objects for all methods to
         * generate proxy dispatching code for.
         * 第一步：添加代理方法（ProxyMethod）
         */
        //object公用方法
        addProxyMethod(hashCodeMethod, Object.class);
        addProxyMethod(equalsMethod, Object.class);
        addProxyMethod(toStringMethod, Object.class);

		//代理实现的接口方法
        for (Class<?> intf : interfaces) {
            for (Method m : intf.getMethods()) {
                addProxyMethod(m, intf);
            }
        }

        /* ============================================================
         * Step 2: Assemble FieldInfo and MethodInfo structs for all of
         * fields and methods in the class we are generating.
         * 步骤2：将代理方法（MethodProxy）转换为方法(MethodInfo)，MethodInfo实现了写入字节码
         */
        try {
            methods.add(generateConstructor());
            for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
                for (ProxyMethod pm : sigmethods) {

                    // add static field for method's Method object
                    fields.add(new FieldInfo(pm.methodFieldName,
                        "Ljava/lang/reflect/Method;",
                         ACC_PRIVATE | ACC_STATIC));
					
                    //代理方法（MethodProxy）转换为方法(MethodInfo)
                    // generate code for proxy method and add it
                    methods.add(pm.generateMethod());
                }
            }
            methods.add(generateStaticInitializer());
        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        /* ============================================================
         * Step 3: Write the final class file.
         * 步骤3：写入类文件
         */

        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        DataOutputStream dout = new DataOutputStream(bout);

        try {
            /*
             * Write all the items of the "ClassFile" structure.
             * See JVMS section 4.1.
             */
            //...依照JVM规范生成字节码，省略

        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        return bout.toByteArray();
    }
```

再次进入ProxyMethod转换为MethodInfo的方法generateMethod()，这里的内容较多，我们只贴出来关键的一处，将InvocationHandler的调用加入了新方法中

```java
	out.writeByte(opc_invokeinterface);
    out.writeShort(cp.getInterfaceMethodRef(
    "java/lang/reflect/InvocationHandler",
    "invoke",
    "(Ljava/lang/Object;Ljava/lang/reflect/Method;" +
    "[Ljava/lang/Object;)Ljava/lang/Object;"));
    out.writeByte(4);
    out.writeByte(0);
```

由此我们了解了整个JDK Proxy的执行过程，最终可以从生成的class文件（$Proxy0.class）中反编译，反编译结果如下：

```java
package com.sun.proxy;

import com.proxy.IBizA;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class Proxy0 extends Proxy
  implements IBizA
{
  public Proxy0()
    throws 
  {
    super(paramInvocationHandler);
  }

  public final void doSomething()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
  }

  public final String toString()
    throws 
  {
    try
    {
      return ((String)this.h.invoke(this, m2, null));
    }
  }
}
```

我们从反编译的文件中可以看出来，实际上代理类调用了InvocationHandler的invoke方法

*注：系统默认不输出代理class文件，如果要实现新增代理class文件的输出，需要在main方法中加上*

```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

*注：Proxy相关的代码可以直接通过jdk源码看到，ProxyGenerator的代码请参考：*

https://github.com/JetBrains/jdk8u_jdk/blob/master/src/share/classes/sun/misc/ProxyGenerator.java

### cglib的实现原理

cglib是通过asm库实现了代理class的生成，cglib会生成两个类，一个类是作为代理类使用，一个类是作为方法调用时使用。在调用方法的时候，cglib做了一些优化，没有采用反射调用方法，而是给每个代理方法增加一个索引，然后根据索引找到方法，并直接调用。

```wiki
注：asm是一个操作Java字节码的库
参考资料：
https://www.ibm.com/developerworks/cn/java/j-lo-asm30/index.html
https://asm.ow2.io/
```

#### 代理类的生成

我们首先看下cglib是如何生成代理类的，由于cglib的代码较为复杂，我们只贴出来关键部分.

cglib和JDK proxy有一点比较类似，他们都是优先从缓存中获取，如果取不到才会调用生成class的方法，我们直接看AbstractClassGenerator类的generate方法

```java
protected Class generate(ClassLoaderData data) {
        Class gen;
        Object save = CURRENT.get();
        CURRENT.set(this);
        try {
            ClassLoader classLoader = data.getClassLoader();
            synchronized (classLoader) {
              String name = generateClassName(data.getUniqueNamePredicate());              
              data.reserveName(name);
              this.setClassName(name);
            }
            if (attemptLoad) {
                try {
                    gen = classLoader.loadClass(getClassName());
                    return gen;
                } catch (ClassNotFoundException e) {
                    // ignore
                }
            }
            //关键代码，调用了strategy的generate方法
            byte[] b = strategy.generate(this);
            String className = ClassNameReader.getClassName(new ClassReader(b));
            ProtectionDomain protectionDomain = getProtectionDomain();
            synchronized (classLoader) { // just in case
                if (protectionDomain == null) {
                    gen = ReflectUtils.defineClass(className, b, classLoader);
                } else {
                    gen = ReflectUtils.defineClass(className, b, classLoader, protectionDomain);
                }
            }
            return gen;
        } catch (RuntimeException e) {
            throw e;
        } catch (Error e) {
            throw e;
        } catch (Exception e) {
            throw new CodeGenerationException(e);
        } finally {
            CURRENT.set(save);
        }
    }
```

调用了GeneratorStrategy的generate方法，默认策略是DefaultGeneratorStrategy，我们继续跟踪

```java
public byte[] generate(ClassGenerator cg) throws Exception {
    DebuggingClassWriter cw = getClassVisitor();
    transform(cg).generateClass(cw);
    return transform(cw.toByteArray());
}
protected ClassGenerator transform(ClassGenerator cg) throws Exception {
    return cg;
}
```

默认策略并没有做什么实际的工作，直接调用了ClassGenerator接口的generateClass方法。

我们回到AbstractClassGenerator类，该类并没有定义generateClass方法，但在他的实现类Enhancer中定义了generateClass方法，而实际上也是调用了Enhancer的方法。

```java
public void generateClass(ClassVisitor v) throws Exception {
        Class sc = (superclass == null) ? Object.class : superclass;

        // Order is very important: must add superclass, then
        // its superclass chain, then each interface and
        // its superinterfaces.
        List actualMethods = new ArrayList();
        List interfaceMethods = new ArrayList();
        final Set forcePublic = new HashSet();
        getMethods(sc, interfaces, actualMethods, interfaceMethods, forcePublic);

        List methods = CollectionUtils.transform(actualMethods, new Transformer() {
            public Object transform(Object value) {
                Method method = (Method)value;
				//...
                return ReflectUtils.getMethodInfo(method, modifiers);
            }
        });
		
    	//类的开始
        ClassEmitter e = new ClassEmitter(v);
        e.begin_class(Constants.V1_8,
                      Constants.ACC_PUBLIC,
                      getClassName(),
                      Type.getType(sc),
                      (useFactory ?
                       TypeUtils.add(TypeUtils.getTypes(interfaces), FACTORY) :
                       TypeUtils.getTypes(interfaces)),
                      Constants.SOURCE_FILE);
		
    	//声明属性
        e.declare_field(Constants.ACC_PRIVATE, BOUND_FIELD, Type.BOOLEAN_TYPE, null);
        e.declare_field(Constants.ACC_PUBLIC | Constants.ACC_STATIC, FACTORY_DATA_FIELD, OBJECT_TYPE, null);
        e.declare_field(Constants.PRIVATE_FINAL_STATIC, THREAD_CALLBACKS_FIELD, THREAD_LOCAL, null);
        e.declare_field(Constants.PRIVATE_FINAL_STATIC, STATIC_CALLBACKS_FIELD, CALLBACK_ARRAY, null);

        // This is declared private to avoid "public field" pollution
        e.declare_field(Constants.ACC_PRIVATE | Constants.ACC_STATIC, CALLBACK_FILTER_FIELD, OBJECT_TYPE, null);
		//构造函数
        emitDefaultConstructor(e);
    	
    	//实现回调
        emitSetThreadCallbacks(e);
        emitSetStaticCallbacks(e);
        emitBindCallbacks(e);

        //...
		
    	//类的结束
        e.end_class();
    }
```

本段代码的核心是使用ClassEmitter（对asm ClassVisitor的封装）构造出来一个类，这个类实现了代理目标类（BizA）的方法，然后也增加了方法拦截器的调用，从反编译代码中我们可以看出来

```java
public final String doSomething()
  {
    MethodInterceptor tmp4_1 = this.CGLIB$CALLBACK_0;
    if (tmp4_1 == null)
    {
      tmp4_1;
      CGLIB$BIND_CALLBACKS(this);
    }
    MethodInterceptor tmp17_14 = this.CGLIB$CALLBACK_0;
    if (tmp17_14 != null)
      return ((String)tmp17_14.intercept(this, CGLIB$doSomething$0$Method, CGLIB$emptyArgs, CGLIB$doSomething$0$Proxy));
    return super.doSomething();
  }
```

注意这一段代码

```java
tmp17_14.intercept(this, CGLIB$doSomething$0$Method, CGLIB$emptyArgs, CGLIB$doSomething$0$Proxy));
```

实现了方法拦截器（MethodInterceptor）的调用。

在生成代理类的同时，cglib同时生成了另外一个类FastClass，这个会在下面详细介绍

#### 代理类的调用

从cglib方式的代码示例中，我们看到，实际上在intercept方法中，我们是这么调用的

```java
proxy.invokeSuper(obj, args);
```

我们进一步看看这个方法做了什么

```java
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
    try {
        init();
        FastClassInfo fci = fastClassInfo;
        return fci.f2.invoke(fci.i2, obj, args);
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    }
}
```

忽略掉init，实际上是调用了FastClass的invoke方法

```java
/**
     * Invoke the method with the specified index.
     * @see getIndex(name, Class[])
     * @param index the method index
     * @param obj the object the underlying method is invoked from
     * @param args the arguments used for the method call
     * @throws java.lang.reflect.InvocationTargetException if the underlying method throws an exception
     */
    abstract public Object invoke(int index, Object obj, Object[] args) throws InvocationTargetException;
```

这是一个抽象方法，方法的实现是在动态生成的FastClass子类中（稍后会贴出反编译字节码）。从方法描述中，我们能看出来，FastClass根据不同的索引调用不同的方法（**注意，这里是直接调用而不是反射**）。我们看下反编译代码

```java
public Object invoke(, Object paramObject, Object[] paramArrayOfObject)
    throws InvocationTargetException
  {
    // Byte code:
    //   0: aload_2
    //   1: checkcast 133	com/cglib/BizA?EnhancerByCGLIB?eca2fdc
    //   4: iload_1
    //   5: tableswitch	default:+304 -> 309, 0:+99->104, 1:+114->119....
    
    //....
    
    //   198: invokevirtual 177	com/cglib/BizA?EnhancerByCGLIB?eca2fdc:doSomething	()Ljava/lang/String;
	
    
   	//...
    
    //   272: aconst_null
    //   273: areturn
    //   274: invokevirtual 199	com/cglib/BizA?EnhancerByCGLIB?eca2fdc:CGLIB$toString$2	()Ljava/lang/String;
    //   277: areturn
    //   278: invokevirtual 201	com/cglib/BizA?EnhancerByCGLIB?eca2fdc:CGLIB$clone$4	()Ljava/lang/Object;
    //   281: areturn
    //   330: athrow
    //
    // Exception table:
    //   from	to	target	type
    //   5	312	312	java/lang/Throwable
  }
```

我们看到，这里使用了tableswitch，可以认为是switch语句。根据不同的值调用不同的方法。从198行中我们可以看到调用了doSomething方法。

这是cglib的一大优势，通过创建索引，减少了反射的过程，也提升了动态代理的性能。另外cglib代码中大量使用了缓存，懒加载等机制，这对提升性能也有不少帮助。



cglib的代码比较复杂，由于文章篇幅原因，我们只是从动态代理实现的角度简单说明。cglib代码质量非常高，其中运用了大量的设计模式，对于线程安全，缓存，懒加载等都有涉及，有兴趣的同学可以直接阅读下源码，我在附录中贴出了源码的链接。

```
注：通过设置类输出目录，可以将cglib生成的类输出
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "C:\\classes");
```

## 对比JDK Proxy和cglib实现方式

再研究了各自实现原理之后，我们再次做个对比

| JDK Proxy                                                    | cglib                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 通过操作字节码实现，在生成的class中插入InvocationHandler的调用 | 在asm库的基础上进一步抽象封装，进而生成代理类（Enhancer代理）和FastClass。当然cglib本质上也是操作字节码。 |
| 代理接口方法调用时采用反射调用，反射性能较低                 | 根据自建的索引直接调用方法，性能高                           |

*注：思考一下，为什么反射影响性能呢？*

# 引用资料

cglib:https://github.com/cglib/cglib

Spring AOP：https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#aop

Spring Source Code：https://github.com/spring-projects/spring-framework/tree/master/spring-aop