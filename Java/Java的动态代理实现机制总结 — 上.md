



#  Java的动态代理实现机制总结 — 上

Java 中目前有两种可以 可以实现动态代理，一是使用 JDK 的动态代理，二是使用 cglib 实现的动态代理，两种代理实现的机制大体上相同，现在稍微总结下动态代理的原理，本文讲述 JDK 动态代理。



## JDK 动态代理



首先我们新建一个接口，因为 JDK 的动态代理**只能代理接口**。

```java
public interface Hello {
  void helloWorld();
}

```

然后实现这个接口：

```java
public class HelloImpl implements Hello {
  public void helloWorld() {
    System.out.println("Hello World");
  }
}
```

然后，我们需要实现 ***java.lang.reflect*** 包的 ***InvocationHandler*** 接口：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class Handler implements InvocationHandler {

  // 被代理的对象
  private Object target;

  public Handler(Object target) {
    this.target = target;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");

    // 调用被代理的方法
    return method.invoke(target, args);
  }
}
```

在上述代码中，***target*** 就是我们被代理的对象，而调用 ***target*** 的所有方法都会被当前的 ***invoke*** 方法所拦截，从而我们可以做一些方法执行前、执行后的动作，或者是跳过方法的执行，甚至修改返回结果，都是可以的。



然后，我们定义一下 ***main*** 方法：

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;

public class Main {

  public static void main(String[]args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    Hello hello = (Hello) Proxy.newProxyInstance(
        Main.class.getClassLoader(),
        new Class<?>[]{Hello.class},
        new Handler(new HelloImpl())
    );

    hello.helloWorld();
  }
}

```

最后，我们来运行 ***main*** 方法，观察 **console** 的输出：

```
invoke method: helloWorld!

Hello World

```

这样，我们看到了我们的方法被成功的代理并执行了我们预期的动作，接下来，我们将跟着源码去一探究竟。

当然，从 ***main*** 中的 ***newProxyInstance*** 开始，如下代码省略部分：

```java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                        Class<?>[] interfaces,
                                        InvocationHandler h) throws IllegalArgumentException
{
    ......
    /*
    * Look up or generate the designated proxy class.
    */
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
    * Invoke its constructor with the designated invocation handler.
    */
       
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;

        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }

        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    }
    ......
 }
```

如上所示代码，核心只有两个部分，一个是

```Java
final Constructor<?> cons = cl.getConstructor(constructorParams);
final InvocationHandler ih = h;

return cons.newInstance(new Object[]{h});
```

可知是根据当前的 ***InvocationHandler*** 去创建了一个**代理类**，那么是哪个代理类呢？

```java
Class<?> cl = getProxyClass0(loader, intfs);
```

接下来，我们继续探究这个 ***getProxyClass0*** 方法

```java
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    ......
    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```

由注释可知，如果当前这个类已经被创建，就会去缓存中获取，否则就会重新创建

而这个 ***ProxyClassCache*** 是这样的：

```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

暂且不用去管这个 **ProxyClassCache **具体是什么，继续向下看这个 ***get*** 方法。

```java
public V get(K key, P parameter) {
    ......
    // lazily install the 2nd level valuesMap for the particular cacheKey
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // create subKey and retrieve the possible Supplier<V> stored by that
    // subKey from valuesMap
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;
       
    while (true) {
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
       ......
    }
}
```

到了这部分，网上的大部分文章说的都是错误的了，他们都理所应当的以为是 ***apply*** 方法就是 ***Proxy* **类的内部类 ***ProxyClassFactory*** 中的 ***apply***  的实现，我们来看一下  ***subKeyFactory*** 的初始化过程：

```java
public WeakCache(BiFunction<K, P, ?> subKeyFactory, BiFunction<K, P, V> valueFactory) {
    this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
    this.valueFactory = Objects.requireNonNull(valueFactory);
}
```

看到了吗，结合  ***ProxyClassCache*** 的声明方式， 故这里的 ***subKeyFactory*** 是 ***Proxy***  的内部类 ***KeyFactory*** 类型， 而它的 ***apply*** 方法如下：

```java
@Override
public Object apply(ClassLoader classLoader, Class<?>[] interfaces) {
    switch (interfaces.length) {
        case 1: return new Key1(interfaces[0]); // the most frequent
        case 2: return new Key2(interfaces[0], interfaces[1]);
        case 0: return key0;
        default: return new KeyX(interfaces);
    }
}
```

这个方法会将这些接口映射到一个弱引用上，从而使接口的实现类的对象也是一个弱引用。

而真正创建或者获取 代理类的 代码是这两句

```java
Supplier<V> supplier = valuesMap.get(subKey);
V value = supplier.get();
```

在类 ***WeakCache*** 中有一个内部类 ***Factory*** 实现了 ***Suppiler***，我们来看一下它的 ***get*** 方法：

```java
@Override
public synchronized V get() { // serialize access
       // re-check
       ......
       // create new value
       V value = null;
       try {
           value = Objects.requireNonNull(valueFactory.apply(key, parameter));
       } finally {
           if (value == null) { // remove us on failure
               valuesMap.remove(subKey, this);
           }
       }
       ......
       return value;
   }
}
```



上文说过了，***subKeyFactory*** 是 ***Proxy***  的内部类 ***KeyFactory*** 类型， 而 ***valueFactory*** 就是 ***Proxy*** 的内部类 ***ProxyClassCache*** 类型了，所以，它的 ***apply*** 方法如下：

```java
@Override
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
    for (Class<?> intf : interfaces) {
        .......
        Class<?> interfaceClass = null;
        interfaceClass = Class.forName(intf.getName(), false, loader);
        /*
            * Verify that the Class object actually represents an
            * interface.
            */
        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(
                interfaceClass.getName() + " is not an interface");
        }
        ......
    }
    
    ......
    String proxyPkg = null;     // package to define proxy class in
    int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
    for (Class<?> intf : interfaces) {
        int flags = intf.getModifiers();
        if (!Modifier.isPublic(flags)) {
            accessFlags = Modifier.FINAL;
            String name = intf.getName();
            int n = name.lastIndexOf('.');
            String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
            if (proxyPkg == null) {
                proxyPkg = pkg;
            } else if (!pkg.equals(proxyPkg)) {
                throw new IllegalArgumentException(
                    "non-public interfaces from different packages");
            }
        }
    }            
    ......
    /*
        * Choose a name for the proxy class to generate.
        */
       
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
        throw new IllegalArgumentException(e.toString());
    }
}}
```

看这句：

```java
if (!interfaceClass.isInterface()) {
   throw new IllegalArgumentException(interfaceClass.getName() + " is not an interface");
}
```

如果这里的这些个 ***interface*** 不是个接口便会抛出异常，这也就是之前我们说 **JDK动态代理无法代理接口的原因**



```java
long num = nextUniqueNumber.getAndIncrement();
String proxyName = proxyPkg + proxyClassNamePrefix + num;
```

这两句就确定了生成的 代理类的名称，一般来讲， ***proxyPkg*** 是被代理类的包，而 ***proxyClassNamePrefix*** 是 *$Proxy*。而基数 ***num*** 则是从 **0** 开始依次递增。



```java
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, 
      interfaces, accessFlags);
```

这行代码就是真正生成代理类的地方。

既然生成了代理类，那为什么我们看不见呢？进入类 ***ProxyGenerator***， 可以看到这么一行代码：

```java
private static final boolean saveGeneratedFiles = ((Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"))).booleanValue();

```

显然，这项参数默认是 *false* 也就是说我们将 JVM 运行参数加上这么一个参数并设置为 *true*，就可以看见被生成的类了。



然后我们来看看生成的类

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.sun.proxy;

import aop.Hello;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Hello {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {......}

    public final void helloWorld() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {......}

    public final int hashCode() throws  {......}

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("aop.Hello").getMethod("helloWorld");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```



我们可以清楚的看到生成的代理类 ***$Proxy0*** 实现了接口 ***Hello*** 

在方法 ***HelloWorld*** 的实现中，有这么一行

```java
super.h.invoke(this, m3, (Object[])null);
```



其中 ***m3*** 是静态代码块中初始化的，是 ***Method*** 类型，并且是 我们之前定义的 ***Hello*** 接口中的 ***HelloWorld*** 方法。 而 ***h*** 则是我们传入的 ***InvocationHandler***, 所以这里调用 ***h*** 的 ***invoke*** 方法，就是之前我们定义的：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");

    // 调用被代理的方法
    return method.invoke(target, args);
}
```

于是这个地方唤醒了我们自己的 ***HelloWorld*** 方法，至此，整个代理过程完成！