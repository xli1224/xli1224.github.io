---
layout: post
title: Java动态代理及几类实现
tags:
    - java
---

## 什么是代理
代理是很常见的一个词，可以作为功能性描述，比如代理服务器，代理对象，甚至代理（proxy）本身也是一个名词。在大部分情况，我所理解的代理都是由于它本身所代表的代理特性，或者说是符合了代理的模式。

代理模式简单来说，即不暴露核心的情况的使用某个功能，这里的不暴露，有可能是不想暴露（AOP之我要加料），也有可能是无法暴露（RMI）。代理类负责为委托类处理消息。

## 静态代理
代理中最简单的就是静态代理。看看下面这段例子，有一个服务类（委托类）A，我们不想它被其他代码直接调用，那么我们提供一个代理类B，它提供了所有A的服务，只是它不是A。(这有什么意义？后面就知道了）。

这是我们的服务类，它只有一个很基本的功能，但实际上在使用中，它应该是那些真正实现了你的关键逻辑的地方。
```java
package proxy.staticproxy;

/**
 * Created by xiang on 15/10/9.
 */
public class HideService {

    public String getSecret() {
        return "Secret";
    }

}
```

这是代理类，它继承了服务类，这样子它可以被转型为它的父类，使用者仍然能正常使用。
```java
package proxy.staticproxy;

/**
 * Created by xiang on 15/10/9.
 */
public class HideServiceProxy extends HideService{

    private HideService delegate;


    public HideServiceProxy(HideService delegate) {
        this.delegate = delegate;
    }

    @Override
    public String getSecret() {
        return "Proxyed " + this.delegate.getSecret();
    }

    public static void main(String[] args) {

        //Pretend this is the container that compose beans
        HideService service = new HideService();
        HideService proxyBean = new HideServiceProxy(service);

        //And the proxy bean will be given when requiring the service bean.
        System.out.println(proxyBean.getSecret());
    }

}
```

Output

    Proxyed Secret

细心的人会发现，用继承的话，使用者仍然能发现它不是他想要的类，因为instanceOf可以很容易的找出来它的真正class对象是什么。不用担心，那只是我偷懒不想多写一个接口，实际上静态代理更推荐接口，在DIP实践广泛的今天，接口才是主流。当使用接口时，静态代理类的代码并不会有太多改变，只是从继承变成了实现接口，内部的delegate变量的类型同时也变成接口类。而使用者在使用接口进行调用时，代理类同样也是接口的实现，没有任何问题。

为什么这种形式叫静态代理或者说什么是静态？静态是指运行前就确定了，代理在编译期就已经生成。这个代理就是这个样子，代码都已经写好。这样当然很好，代码清晰可读，一看就知道这里有个代理。缺点也很明显，首先，写它占用了时间，如果需要实现代理的类很多，一个个是写不过来的。其次，如果更改了接口，除了更改服务实现类以外，还得去更新这些代理类，维护上容易出纰漏。

## 动态代理
想要不自己费事写代理类，就需要动态代理。动态代理怎么动态呢？第一，代理类的创建是动态的，不需要手工的定义一个代理类；第二，对于代理方法的处理也是动态的，不需要为被代理类的每个方法都做一个代理实现。

在动态代理这里，针对接口代理和针对实体类的代理就有点不一样了，背后所需要的技术不一样导致了使用上的不一样。我们先来看JDK支持的动态代理，它适用于接口编程模式。也就是说被代理的类必须是实现了某个接口，代理类将基于此接口进行动态创建。

### JDK Dynamic Proxy
还是先上例子

这是接口
```java
public interface TestService {

    public void say();

}
```
然后是实现类
```java
public class TestServiceImpl implements TestService {

    private String name;

    public TestServiceImpl(String name) {
        this.name = name;
    }

    public void say() {
        System.out.println(name);
    }
}
```

最后是我们的动态代理
```java
public class JDKDynamicProxy implements InvocationHandler{

    private Object delegate;

    public JDKDynamicProxy(Object delegate) {
        this.delegate = delegate;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("In proxy");
        Object result = method.invoke(delegate, args);
        System.out.println("Exit proxy");

        return result;
    }

    public Object getProxy (ClassLoader cl, Class clasz) {
        return Proxy.newProxyInstance(cl, new Class[]{clasz}, this);
    }

    public static void main(String[] args) {
            TestService ts = new TestServiceImpl("service bean instance");

            if (ts instanceof TestService) {
                System.out.println("Yep, i'm the one you need");
            }

            TestService proxy = (TestService)new JDKDynamicProxy(ts).getProxy(TestService.class.getClassLoader(), TestService.class);

            proxy.say();

            if (proxy instanceof TestService) {
                System.out.println("Proxy is also seen as the bean");
            }

        }
}
```

Output

    Yep, i'm the one you need
    In proxy
    service bean instance
    Exit proxy
    Proxy is also seen as the bean

我们的代理类并没有去实现那个接口，相反，它根本连哪个接口要代理都不关心，它只关心一点，就是当我拦截了方法请求，即进入代理时，该做点啥？这就是invoke方法关心的，也是我们要实现的。而我们除了这点，还必须关心另一点，就是对于动态代理类，生成代理实例的时候，归根结底还是需要一个委托对象的，不然它又不是神，怎么知道到底跑哪个业务逻辑呢？在我们这个例子里，是通过构造函数扔进去的，即生成InvocationHandler实例时，需要提供一个委托对象作为参数，基于这个handler，我们通过newProxyInstance方法构造出一个动态代理对象。

我们该好奇的是，newProxyInstance做了啥事？拿出来的到底是什么东西，如何实现代理的？我们先进这个方法看看，关键的应该就是下面三行

```java
//获取动态代理类（一个新的类？）
Class<?> cl = getProxyClass0(loader, intfs);
//获取该类的构造器
final Constructor<?> cons = cl.getConstructor(constructorParams);
//通过构造器生成新的代理对象并返回，构造器的参数就是invocationhandler实例
return cons.newInstance(new Object[]{h});
```

再看getProxyClass0方法
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
注释写的非常清楚，所有创建过的动态代理class都会存在一个cache里面，如果有的话，凭借特有的loader和接口就可以找到，找不到的话就通过ProxyClassFactory创建一个。

```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

```java
String proxyName = proxyPkg + proxyClassNamePrefix + num;
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
```

generateProxyClass看不到源码，不过可以分析一个创建出来的动态类有什么特点。下面就是反编译了上面使用的动态代理类后得到的代码。

```java
public final class $Proxy100 extends Proxy implements TestService {

//四个method类型的静态变量，用来做invocation时候的参数
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

//构造函数接受一个invocation handler，传给父类的构造器。在Proxy类里将这个值赋予h变量
    public $Proxy100(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

//这就是接口定义方法的实现，可以看到将调用转到了传进来的invocation handler的invoke方法上
    public final void say() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("proxy.dynamicproxy.TestService").getMethod("say", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

从上面的代码，可以创建出来的代理类有几个特点
1. 基于Proxy这个父类，代理类的名字是$ProxyN，N为数字。
2. 对所有的接口类方法调用，都转交给invocation handler处理，所被调用的method对象和参数也一并传入。
3. invocation handler里通过invoke方法和反射机制，实现了添加额外统一处理逻辑和对原实现方法的调用。

### CGLIB
当委托类没有实现接口时，就无法使用JDK动态代理，这时候我们想想前面静态代理那个例子，就会发现其实子类也是一个可行的方案。当然，我们了解前面JDK动态代理的原理是通过反射，那么动态的继承了委托类的代理类该怎么创建？我们使用CGLIB这个工具。

```java
public class ServiceWithDefaultConstructor {
    public void say() {
        System.out.println("In real service");
    }
}
```

在JDK动态代理中有invocation handler，在CGLIB中也有类似的东西，就是MethodInterceptor。在新创建的代理类中，对所有方法的调用，也会交给它来处理。
```java
private static class MethodInterceptorImpl implements MethodInterceptor {

        public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
            System.out.println("In proxy");
            Object result = methodProxy.invokeSuper(proxy, args);
            System.out.println("In proxy");
            return result;
        }
    }
```

做好了代理逻辑，就可以去调用生成代理类了。
```java
public static void proxyServiceWithDefaultConstructor() {
        Enhancer en = new Enhancer();
        en.setSuperclass(ServiceWithDefaultConstructor.class);
        en.setCallback(new MethodInterceptorImpl());

        ServiceWithDefaultConstructor test = (ServiceWithDefaultConstructor)en.create();
        test.say();
    }
```

输出结果如下：

    In proxy
    In real service
    In proxy

是不是很简单？代理对象都要从enhancer里面获取，所以我们首先要告诉enhancer，它这个是要负责创建哪个委托类的代理，然后把MethodInterceptor对象设置成callback，这样创建出来的代理类上，方法调用的实现都会去找这个MethodInterceptor。最后让enhancer给你一个实例，就可以用了。

这里也与JDK动态代理有个明显不同的地方，就是关于委托类的实例。在JDK动态代理中，我们需要在Invocation handler里面保留一个委托类对象，因此在方法调用进来后，除了代理的逻辑，还能够调用到真正的服务实现。而在CGLIB中，我们发现MethodInterceptor根本没关心这事。为什么呢？因为在JDK动态代理时，一切都是基于接口去代理的，它无法知道你到底是要代理哪个实现了接口的委托类，所以必须由调用者去传入委托对象。而CGLIB则是基于继承，它要代理的一定就是要继承的那个类的实例，所以如果CGLIB能够直接创建一个委托类的对象，那就一切完美了。在这个例子里是不是这样？我们继续验证下去……

**没有无参构造器的类**
是的，如果没用无参构造器，我们也不提供参数，怎么办？
```java
public class ServiceWithoutDefaultConstructor {

    private String name;

    public ServiceWithoutDefaultConstructor(String name) {
        this.name = name;
    }

    public void say() {
        System.out.println("In real service");
    }

    public void getName() {
        System.out.println(this.name);
    }

}
```

然后试一下CGLIB来创建
```java
public static void proxyServiceWithoutDefaultConstructor() {
        Enhancer en = new Enhancer();
        en.setSuperclass(ServiceWithoutDefaultConstructor.class);
        en.setCallback(new MethodInterceptorImpl());

        ServiceWithoutDefaultConstructor test = (ServiceWithoutDefaultConstructor)en.create();
        test.say();
    }
```

Error：

    Exception in thread "main" java.lang.IllegalArgumentException: Superclass has no null constructors but no arguments were given

果然失败了，错误信息告诉我们是因为父类需要参数来构造，而我们没提供任何参数。那么，如何提供参数给CGLIB呢？在用 enhancer 创建代理对象的时候，提供了参数的入口，只要稍微改一下就可以了。
```java
public static void proxyServiceWithoutDefaultConstructor() {
        Enhancer en = new Enhancer();
        en.setSuperclass(ServiceWithoutDefaultConstructor.class);
        en.setCallback(new MethodInterceptorImpl());

        ServiceWithoutDefaultConstructor test =
                (ServiceWithoutDefaultConstructor)en.create(new Class[] {String.class}, new Object[] {"constructor parameteres"});
        test.say();
    }
```

## 总结
* 静态代理更偏向设计模式的演练，在实际使用中会显得繁琐不通用。
* 动态代理有很强的实用性，代码层面上可以集中在代理逻辑上的编写。
* jdk 动态代理适用于接口，cglib 适用于实体类。使用上 cglib 更直接，但是 jdk 动态代理与Spring 等框架鼓励的通过接口解耦的编程方式也合作的很好。
