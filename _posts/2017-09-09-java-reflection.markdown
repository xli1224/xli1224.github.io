---
layout: post
title: Java 反射介绍
tags:
- java
---

## 何为反射
反射是jdk提供的一组API，可以允许开发者获取程序运行时的内部结构，包括了类/接口/方法/属性/注解/泛型信息。另外，反射还允许动态创建Java对象，获取某个field的值，甚至是调用某个方法。

## 使用反射
获取类的属性、构造器、方法、私有方法和属性。

一个简单使用反射的例子如下：
```java
package me.xiang.features.reflection;
 
import java.lang.reflect.Method;
 
/**
 * Created by xiang on 02/06/2017.
 */
public class MyClass {
    private String var1;
    private String var2;

    public String getVar1() {
        return var1;
    }

    public void setVar1(String var1) {
        this.var1 = var1;
    }

    public String getVar2() {
        return var2;
    }

    public void setVar2(String var2) {
        this.var2 = var2;
    }

    public static void main(String[] args) {
        Method[] methods = MyClass.class.getMethods();

        for (Method method: methods) {
            System.out.println("Method: " + method.getName());
        }
    }
}
```

反射的第一步就是拿到目标类的class对象，从class上我们可以继续去获取其他的东西。我们先把该加的一些特性加到我们的目标类上，包括各种注解、私有共有属性、私有共有方法。

```java
@MyAnnotation(name="CreatedClass", value="first one")
public class MyClass {
    private String var1;
    private String var2;

    public String getVar1() {
        return var1;
    }

    public void setVar1(String var1) {
        this.var1 = var1;
    }

    public String getVar2() {
        return var2;
    }

    public void setVar2(String var2) {
        this.var2 = var2;
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MyAnnotation {
    public String name();
    public String value();
}
```

看下该如何通过反射获取各类信息：

首先是类的信息，可以获取类的名字、作用域、实现接口和父类
```java
		//获取class name
         String name = MyClass.class.getSimpleName();
         //获取作用域，作用域有多个，比如是否是static，是否是public，所有的值按照bit 0或1算出后，放到一个int值里面。
         //可以通过Modifer的方法来解析返回值
         int modifier = MyClass.class.getModifiers();
         Modifier.isAbstract(modifier);
         Modifier.isPublic(modifier);
         //获取所在包
         Package p = MyClass.class.getPackage();
         //获取实现接口：父类所实现的接口并不会包含在内。
         Class[] interfaces = MyClass.class.getInterfaces();
```

获取构造器和方法的时候，有一些比较好玩的点，就在于Java支持多态，所以可能存在多个同名方法或构造器。所以获取出来的默认是一个数组，可以二次过滤或者直接传入指定参数类型来获得匹配的方法。
```java
		//获取所有构造器
         Constructor[] constructors = MyClass.class.getConstructors();
         //获取指定参数类型的构造器
         Class[] params = new Class[] {String.class};
         try {
             MyClass.class.getConstructor(params);
         }
         catch (NoSuchMethodException nsme) {
             nsme.printStackTrace();
         }
 
         //获取所有方法
         Method[] methods = MyClass.class.getMethods();
         //获取指定参数的指定方法
         params = new Class[] {String.class};
         try {
             MyClass.class.getMethod("setVar1", params);
         }
         catch (NoSuchMethodException nsme) {
             nsme.printStackTrace();
         }
```

获取到了Method对象后，还可以直接进行调用。根据method是静态还是动态，可以选择是否需要先构造一个对象。
```java
Method method = MyObject.class.getMethod("doSomething", String.class);

//静态方法，所以invoke的对象是null
Object returnValue = method.invoke(null, "parameter-value1");
```

对于bean来说，还可以通过对方法名和参数个数的判断，来找出getter和setter方法。

接下来是属性，属性可以包括public和private属性。
```java
Field privateStringField = MyClass.class.getDeclaredField("var1");
privateStringField.setAccessible(true);
```

使用反射，还可以获得注解的信息，这也是runtime annotation的一种处理手段。

获得类的注解信息
```java
Annotation[] annotations = MyClass.class.getAnnotations();
for (Annotation annotation : annotations) {

    if (annotation instanceof MyAnnotation) {
        MyAnnotation myAnnotation = (MyAnnotation)annotation;

        System.out.println(myAnnotation.name());
        System.out.println(myAnnotation.value());
    }
}
```

可以看到输出如下：
```
CreatedClass
first one
```

对于方法自身还有方法参数上的注解，可以在获取了method对象后调用相应方法拿到
```java
Annotation[] annotations = method.getDeclaredAnnotations();
//获取某个annotation
Annotation annotation = method.getAnnotation(MyAnnotation.class);

//参数注解本身和参数并没有绑定在一起，还是要通过方法获取所有的参数注解。该返回值是一个二维的注解数组，对应到每一个参数上都是一个注解数组

Annotation[][] parameterAnnotations = method.getParameterAnnotations();
Class[] parameterTypes = method.getParameterTypes();

int i=0;
for(Annotation[] annotations : parameterAnnotations){
  Class parameterType = parameterTypes[i++];

  for(Annotation annotation : annotations){
    if(annotation instanceof MyAnnotation){
        MyAnnotation myAnnotation = (MyAnnotation) annotation;
        System.out.println("param: " + parameterType.getName());
        System.out.println("name : " + myAnnotation.name());
        System.out.println("value: " + myAnnotation.value());
    }
  }
}
```

通过反射还可以获得对象的一些泛型信息，虽然由于类型擦除机制的存在，泛型类中的类型参数等信息，在运行时刻是不存在的。JVM看到的都是原始类型。对此，Java 5对Java类文件的格式做了修订，添加了Signature属性，用来包含不在JVM类型系统中的类型信息。比如以java.util.List接口为例，在其类文件中的Signature属性的声明是<E:Ljava/lang/Object;>Ljava/lang/Object;Ljava/util/Collection<TE;>;; ，这就说明List接口有一个类型参数E。在运行时刻，JVM会读取Signature属性的内容并提供给反射API来使用。 

```java
Method method = MyClass.class.getMethod("getStringList", null);

Type returnType = method.getGenericReturnType();

if (returnType instanceof ParameterizedType) {
    ParameterizedType type = (ParameterizedType) returnType;
    Type[] typeArguments = type.getActualTypeArguments();
    for (Type typeArgument : typeArguments) {
        Class typeArgClass = (Class) typeArgument;
        System.*out*.println("typeArgClass = " + typeArgClass);
    }
}
```
上面的例子判断的是方法返回值的具体类型。如果是方法的参数的话，可以用下面的办法获取：
```java
method = Myclass.class.getMethod("setStringList", List.class);

Type[] genericParameterTypes = method.getGenericParameterTypes();

for(Type genericParameterType : genericParameterTypes){
    if(genericParameterType instanceof ParameterizedType){
        ParameterizedType aType = (ParameterizedType) genericParameterType;
        Type[] parameterArgTypes = aType.getActualTypeArguments();
        for(Type parameterArgType : parameterArgTypes){
            Class parameterArgClass = (Class) parameterArgType;
            System.out.println("parameterArgClass = " + parameterArgClass);
        }
    }
}
```

如果是个泛型类里面的属性，在实例化以后拿到该对象的field，就可以看到到底是什么类型。
```java
Field field = MyClass.class.getField("stringList");

Type genericFieldType = field.getGenericType();

if(genericFieldType instanceof ParameterizedType){
    ParameterizedType aType = (ParameterizedType) genericFieldType;
    Type[] fieldArgTypes = aType.getActualTypeArguments();
    for(Type fieldArgType : fieldArgTypes){
        Class fieldArgClass = (Class) fieldArgType;
        System.out.println("fieldArgClass = " + fieldArgClass);
    }
}
```

## 动态代理
前面我们大部分都是在获取各种信息，其实我们可以开始考虑，能否通过构造器来创建一个对象？拿到方法对象后，能不能直接调用方法？反射API的另外一个作用是在运行时刻对一个Java对象进行操作。 这些操作包括动态创建一个Java类的对象，获取某个域的值以及调用某个方法。在Java源代码中编写的对类和对象的操作，都可以在运行时刻通过反射API来实现。

 代理对象和被代理对象一般实现相同的接口，调用者与代理对象进行交互。代理的存在对于调用者来说是透明的，调用者看到的只是接口。代理对象则可以封装一些内部的处理逻辑，如访问控制、远程通信、日志、缓存等。

JDK 5引入的动态代理机制，允许开发人员在运行时刻动态的创建出代理类及其对象。在运行时刻，可以动态创建出一个实现了多个接口的代理类。每个代理类的对象都会关联一个表示内部处理逻辑的InvocationHandler接口的实现。当使用者调用了代理对象所代理的接口中的方法的时候，这个调用的信息会被传递给InvocationHandler的invoke方法。在 invoke方法的参数中可以获取到代理对象、方法对应的Method对象和调用的实际参数。invoke方法的返回值被返回给使用者。这种做法实际上相当于对方法调用进行了拦截。

动态代理可以用于许多用途，比如数据库连接和事务管理、单元测试还有其他的AOP类型的方法拦截的目的。

动态代理可以通过Proxy.newProxyInstance()方法创建，newProxyInstance()接受三个参数：
* 用来装载动态代理类的ClassLoader
* 用来实现的接口列表
* 一个实现了InvocationHandler接口的实例，用来处理所有调用代理对象的请求。

classLoader可以从要代理的对象中获得，第二个是要代理的对象继承了哪些接口，第三个是真正的动态代理类要实现的业务逻辑。

*创建动态代理的过程中，要注意的是被代理的实例创建并不是要关心的点，它可以从外部传入，也可以动态代理自己通过反射去创建，但总归要有一个。*

我们看一个例子：
```java
interface DemoInterface {
	public void echo();
}

interface DemoInterface2 {
	public void echo2();
}

class DemoApp implements DemoInterface {
	public void echo () { 
		System.out.println("Demo app");
	}
}

class DemoApp2 implements DemoInterface {
	public void echo2() {
		System.out.println("Demo app2");
	}
}

class DemoProxy implements InvocationHandler {
	private Object target;
	
	public Object getInstance(Object target){
		this.target = target;
		return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object result = null;
		System.out.println("Before call app");
		result = method.invoke(this.target, args);
		System.out.println("After call app");
		return result;
	}
}
```
