#  Java的动态代理实现机制总结 — 下

Java 中目前有两种可以 可以实现动态代理，一是使用 JDK 的动态代理，二是使用 cglib 实现的动态代理，两种代理实现的机制大体上相同，现在稍微总结下动态代理的原理，本文讲述 cglib 动态代理。



## Cglib 动态代理

上文已经讲述过，jdk 动态代理只能代理接口，而不能代理类，故上文使用接口来做实验。而 cglib 可以动态代理*类* ，所以本文使用 *类* 来做实验。



首先，我们定义这个实验类

```java
public class Hello {
  void helloWorld() {
    System.out.println("hello world");
  }
}
```



然后，我们仿照上文，去实现 ***InvocationHandler*** (注意：此处的 ***InvocationHandler*** 是 **cglib** 包的 版本，为了防止误解，代码中会带上 ***import*** 语句)

```java
import net.sf.cglib.proxy.InvocationHandler;
import java.lang.reflect.Method;

public class Handler implements InvocationHandler {

  private Object target;

  public MyInvocationHandler(Object target) {
    this.target = target;
  }

  
  public Object invoke(Object o, Method method, Object[] args) throws Throwable {
	System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");
    return method.invoke(target, args);
  }
}
```

一如既往，我们在方法被 invoke 之前做了一些操作，来保证看见当前方法确实是被 ***invoke*** 方法截取了。

当然，在 cglib 中，你也可以实现 ***MethodIntercepter*** 来达到动态代理的目的。但是为了和 JDK 代理统一，本文只讲述实现 ***InvocationHandler*** 这种方式。

```java
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class Intercepter implements MethodInterceptor {
    private Object target;
    
    public Intercepter(Object target) {
        this.target = target;
    }

    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");
        return method.invoke(target, args);
    }
}
```



之后我们便写 ***main*** 方法来调用：

```java
import net.sf.cglib.proxy.Enhancer;

public class Main {
  public static void main(String[] args) throws Exception {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Hello.class);//继承被代理的那个类
    enhancer.setCallback(new Handler(new Hello())); //设置回调类
    /*
     * 下面这条语句也可起到同样的作用
     * enhancer.setCallback(new Intercepter(new Hello()));
     */
    Hello target = (Hello) enhancer.create(); //创建代理对象

    target.helloWorld();
  }
}

```

观察 **console** 的输出

```
invoke method: helloWorld!

hello world

```

之后我们开始解析 cglib 的代理过程， 首先是 ***main*** 方法中的 ***enhancer.create()***

```java
public Object create() {
    this.classOnly = false;
    this.argumentTypes = null;
    return this.createHelper();
}
```

然后来看 ***createHepler***  方法

```java
private Object createHelper() {
    this.preValidate();
    Object key = KEY_FACTORY.newInstance(this.superclass != null ? this.superclass.getName() : null, ReflectUtils.getNames(this.interfaces), this.filter == ALL_ZERO ? null : new WeakCacheKey(this.filter), this.callbackTypes, this.useFactory, this.interceptDuringConstruction, this.serialVersionUID);
    this.currentKey = key;
    Object result = super.create(key);
    return result;
}
```

核心代码还在下一层，继续从 ***super.create()*** 深入：

```java
protected Object create(Object key) {
        try {
            .......
            Object obj = data.get(this, getUseCache());
            if (obj instanceof Class) {
                return firstInstance((Class) obj);
            }
            return nextInstance(obj);
        } catch (RuntimeException e) {
            throw e;
        ......
    }
```

我么来看这个 ***get*** 方法：

```java
public Object get(AbstractClassGenerator gen, boolean useCache) {
    if (!useCache) {
      return gen.generate(ClassLoaderData.this);
    } else {
      Object cachedValue = generatedClasses.get(gen);
      return gen.unwrapCachedValue(cachedValue);
    }
}
```

从上文可以看出，如果使用缓存，就从缓存中返回，否则将重新生成 代理类。

后文还有两层实现，但是大抵上原理如此了，还是去生成代理类。

然后我们修改一下 jvm 启动参数：

```
-Dcglib.debugLocation=
-Dnet.sf.cglib.core.DebuggingClassWriter.traceEnabled=true
```

将上述参数加入启动参数中，便可以在指定的目录中看到 cglib 为你生成的代理类。

cglib 生成的类较多，代理类的前缀名为 ***你的类名 + EnhancerByCGLIB + 随机串***

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package cglib;

import java.lang.reflect.Method;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.InvocationHandler;
import net.sf.cglib.proxy.UndeclaredThrowableException;

public class Hello$$EnhancerByCGLIB$$45e72743 extends Hello implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private InvocationHandler CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$helloWorld$0;
    ......

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$helloWorld$0 = Class.forName("cglib.Hello").getDeclaredMethod("helloWorld");
        ......
    }

    final void helloWorld() {
        try {
            InvocationHandler var10000 = this.CGLIB$CALLBACK_0;
            if (this.CGLIB$CALLBACK_0 == null) {
                CGLIB$BIND_CALLBACKS(this);
                var10000 = this.CGLIB$CALLBACK_0;
            }

            var10000.invoke(this, CGLIB$helloWorld$0, new Object[0]);
        } catch (Error | RuntimeException var1) {
            throw var1;
        } catch (Throwable var2) {
            throw new UndeclaredThrowableException(var2);
        }
    }
  
    ......
    public void setCallback(int var1, Callback var2) {
        switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (InvocationHandler)var2;
        default:
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (InvocationHandler)var1[0];
    }

    static {
        CGLIB$STATICHOOK1();
    }
}

```



从上述代码中可以得知， 首先执行了 ***CGLIB$STATICHOOK1()*** 方法，在方法中，参数 ***CGLIB$helloWorld$0*** 被设置为我们的 测试类 ***Hello*** 中的 ***HelloWorld*** 方法。

而本代理类继承了 ***Hello*** ，自己也实现了一套 ***HelloWorld*** 方法，首先它会去获取 ***CGLIB$CALLBACK_0*** 这个参数， 它是一个 ***InvocationHandler*** ，但是这个 **Handler** 是哪里来的呢？

别急，在我们的 ***main*** 方法中，有这么一句

```java
enhancer.setCallback(new Handler(new Hello()));//设置回调类
```

我们来看一下这个 ***setCallback*** 方法

```java
public void setCallback(final Callback callback) {
   setCallbacks(new Callback[]{ callback });
}
```

继续

```java
public void setCallbacks(Callback[] callbacks) {
    if (callbacks != null && callbacks.length == 0) {
        throw new IllegalArgumentException("Array cannot be empty");
    }
    this.callbacks = callbacks;
}
```

可以得知， ***callback*** 是 ***Enhancer*** 的属性。

而在生成代理类的核心代码中，有这么一段

```java
Object obj = data.get(this, getUseCache());
if (obj instanceof Class) {
   return firstInstance((Class) obj);
}
return nextInstance(obj);
```

而这个 ***obj*** 就是我们生成的 代理类。就是说这个地方会返回一个代理类的实例。而无论是哪种返回方式，都会去设置 ***callbacks*** (具体的十分复杂，不再铺开叙述)



在代理类中：

```java
public void setCallback(int var1, Callback var2) {
     switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (InvocationHandler)var2;
     default:
    }
}
```

所以这个 ***CGLIB$CALLBACK_0*** 就是我们设置的 实现了 ***InvocationHandler*** 的 ***Handler*** 类。

因此当代理类调用

```java
var10000.invoke(this, CGLIB$helloWorld$0, new Object[0]);  
```



便就是调用了 ***Handler*** 类的如下方法:

```java
public Object invoke(Object o, Method method, Object[] args) throws Throwable {
    System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");
    return method.invoke(target, args);
}
```

而实现了 ***MethodIntercepter*** 的方式， 在代理类中上述调用即为

```java
var10000.intercept(this, CGLIB$helloWorld$0$Method, CGLIB$emptyArgs, CGLIB$helloWorld$0$Proxy);
```

转而在 ***Intercepter*** 中便是调用

```java
public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("\n\ninvoke method: " + method.getName() + "!\n\n");
        return method.invoke(target, args);
}
```

其他步骤原理皆是一模一样，不再详谈。



以上。