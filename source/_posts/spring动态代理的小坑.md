---
title: spring动态代理的小坑
date: 2019-11-08 18:55:55
categories:
- spring
tags:
- spring
- springboot
- aop
- 动态代理
- CGLib
- jdk
---

# 动态代理的方式

springboot貌似2.0前后，采用的动态代理方案不太一样：

- 2.0之前：实现接口的类的代理采用jdk动态代理，没有实现接口的类采用CGLib动态代理
- 2.0以及以后：不管是否实现接口，都默认采用CGLib代理

ps：这个分界版本可能不太准，具体可以测试一下

# jdk动态代理的小坑

```
@Component("abstractTaskImpl")
public class AbstractTaskImpl implements AbstractTaskInter {

    private static Random random = new Random();

    @Override
    @Async
    public void doTaskOne() throws InterruptedException {
        System.out.println("开始做任务一");
        long start = currentTimeMillis();
        sleep(random.nextInt(10000));
        long end = currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
    }
}
```

上面的类中，`doTaskOne`方法添加了`@Async`注解，所以实际运行过程中会被代理，保证`doTaskOne`方法会进行异步调用。

实现了接口，使用的是jdk动态代理（测试版本：springboot-1.5.8），所以代理类是实现了`AbstractTaskInter`接口，注入的时候，通过接口的类型可以正常注入。

```
@Autowired
@Qualifier("abstractTaskImpl")
private AbstractTaskInter abstractTaskInter;
```

此时，我想给类添加一个非接口的异步调用方法，如下：

```
@Component("abstractTaskImpl")
public class AbstractTaskImpl implements AbstractTaskInter {

    private static Random random = new Random();

    @Async
    public void other() {
        System.out.println("other async");
    }

    @Override
    @Async
    public void doTaskOne() throws InterruptedException {
        System.out.println("开始做任务一");
        long start = currentTimeMillis();
        sleep(random.nextInt(10000));
        long end = currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
    }
}
```

因为`other`方法不是接口的方法，所以通过接口类型注入的时候，无法调用到该方法，所以需要通过实现类具体类型来注入：

```
@Autowired
@Qualifier("abstractTaskImpl")
private AbstractTaskImpl abstractTaskInter;
```

此时运行程序的时候，会出现异常，显示`abstractTaskImpl`这个bean的类型不是`AbstractTaskImpl`：

```
Error creating bean with name 'hong.study.utils.springboot.async.AsyncTaskTest': Unsatisfied dependency expressed through field 'abstractTaskInter'; nested exception is org.springframework.beans.factory.BeanNotOfRequiredTypeException: Bean named 'abstractTaskImpl' is expected to be of type 'hong.study.utils.springboot.async.AbstractTaskImpl' but was actually of type 'com.sun.proxy.$Proxy79'
```

原因在于，生成的代理类是通过jdk动态代理生成的，他是实现了`AbstractTaskInter`接口的类，跟`AbstractTaskImpl`类是平等的两个实现类，所以会出现上面的bean类型错误

# 小坑的解决方案

## 实现另一个接口

上面的`AbstractTaskImpl`类去实现另一个接口`OtherInter`，该接口中有方法`other`：

```
@Component("abstractTaskImpl")
public class AbstractTaskImpl implements AbstractTaskInter, OtherInter {

    private static Random random = new Random();

    @Override
    @Async
    public void other() {
        System.out.println("other async");
    }

    @Override
    @Async
    public void doTaskOne() throws InterruptedException {
        System.out.println("开始做任务一");
        long start = currentTimeMillis();
        sleep(random.nextInt(10000));
        long end = currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
    }
}
```

注入的时候，通过这个新的接口类型进行注入，这样可以成功。

```
@Autowired
@Qualifier("abstractTaskImpl")
private OtherInter abstractTaskInter;
```

## 将动态代理的方式改为CGLib

CGLib的动态代理是通过生成被代理类的子类来实现的，代理类的类型就是被代理类，所以注入的时候通过`AbstractTaskImpl`进行注入是可以的，不会出现上面的代理类和注入的类型不一致的情况。

```
@Autowired
@Qualifier("abstractTaskImpl")
private AbstractTaskImpl abstractTaskInter;
```

### 开启CGLib

springboot2.0之后，默认都是CGLib的方式了，但是对于上面提到的`springboot-1.5.8`版本，需要进行一些手动设置：

- 添加全局配置：`spring.aop.proxy-target-class=true`，但是这个经过测试不太好使，而且即使生效了也是全局修改，可能会影响到其他的类的动态代理的生成
- 给bean添加单独配置：类上添加<mark>`@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)`</mark>，只会对当前类生效，测试有效，也不会影响到其他类，推荐这个方式
