---
title: spring aop与动态代理
date: 2018-03-20 22:30:22
categories:
- spring
tags:
- spring
- aop
---

---
# aspectj
---

AspectJ是静态代理的增强，所谓的静态代理就是AOP框架会在编译阶段生成AOP代理类，因此也称为编译时增强。

相比于动态代理，aspectJ有更好的性能，但是需要特定的编译器，编译之后生成的class相比于java的编译器生成的class，会添加代理相关的逻辑。

---
# java的动态代理
---

proxy（Proxy）是java中的代理类，handler（InvocationHandler）中封装的是具体的代理逻辑，根据被代理类和handler创建的代理类proxy，跟被代理类一样实现了相同的接口，但是生成的代理类中在被代理类的基础上加了代理逻辑，所以java的动态代理要求被代理类实现某个接口。

## 被代理类

```
//接口
public interface HelloWorld {

    void sayHello(String name);

}
//实现
public class HelloWorldImpl implements HelloWorld {

    @Override
    public void sayHello(String name) {
        System.out.println("hello: " + name);
    }
}
```

## 处理器(InvocationHandler)

```
//封装了实际的代理逻辑
public class CustomInvocationHandler implements InvocationHandler {

    private Object target;

    public CustomInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before invocation");
        System.out.println(proxy.getClass().getName());
        method.invoke(target, args);
        System.out.println("after invacation");
        return null;
    }
}
```

## 生成代理对象(Proxy)

```
public static void main(String[] args) {
    HelloWorld helloWorld = new HelloWorldImpl();

    InvocationHandler handler = new CustomInvocationHandler(helloWorld);
    HelloWorld helloWorldProxy = (HelloWorld) Proxy.newProxyInstance(ProxyTest.class.getClassLoader(),
            new Class[]{HelloWorld.class},
            handler);
    helloWorldProxy.sayHello("zhan");
}
```

---
# spring的aop
---

与AspectJ的静态代理不同，Spring AOP使用的动态代理，所谓的动态代理就是说AOP框架不会去修改字节码，而是在内存中临时为方法生成一个AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

Spring AOP中的动态代理主要有两种方式，JDK动态代理和CGLIB动态代理。JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口。JDK动态代理的核心是InvocationHandler接口和Proxy类。

java的动态代理上面已经做了解释，下面介绍一下通过CGLIB实现动态代理。

## CGLIB实现动态代理

如果被代理类没有实现接口，那么spring会通过CGLIB实现动态代理。除了这种情况，还可以手动设置通过CGLIB实现动态代理：

```
//官网说明
It is possible to force the use of CGLIB, in those (hopefully rare) cases where you need to advise a method that is not declared on an interface, or where you need to pass a proxied object to a method as a concrete type.
强制使用CGLIB也是可能的(希望这种情况很少)，此时你需要advise的方法没有被定义在接口中，或者你需要向方法中传入一个具体的对象作为代理对象。
```

CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。
