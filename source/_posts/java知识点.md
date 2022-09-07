---
title: java知识点
date: 2021-08-03 16:45:01
categories:
- java
tags:
- java
- 闭包
- 知识点
---

# 闭包

闭包(closure)是一个可调用的对象，它记录了一些信息，这些信息来自于创建它的作用域。通过这个定义，可以看出内部类是面向对象的闭包，因为它不仅包含外围类对象(创建内部类的作用域)的信息，还自动拥有一个指向此外围类对象的引用，在此作用域内，内部类有权操作所有的成员，包括private成员。

闭包（Closure）是一种能被调用的对象，它保存了创建它的作用域的信息。JAVA并不能显式地支持闭包，但是在JAVA中，闭包可以通过“接口+内部类”来实现。

````java
/**
 * java闭包学习
 * 闭包(closure)是一个可调用的对象，它记录了一些信息，这些信息来自于创建它的作用域。
 * 通过这个定义，可以看出内部类是面向对象的闭包，因为它不仅包含外围类对象(创建内部类的作用域)的信息，
 * 还自动拥有一个指向此外围类对象的引用，在此作用城内，内部类有权操作所有的成员，包括private成员。
 * @author : zhanpenghong
 * @date : 2021/8/3 下午4:48
 */
public class ClosureStudy {

    private String name;
    public int age;

    public void study() {

        new ClosureInterface() {
            @Override
            public void run() {
                /* 内部类能够闭包，访问外部类的任何的成员变量，包括私有的 */
                System.out.println(name);
            }
        };

    }
}

/**
 * 闭包的学习接口类
 *
 * @author : zhanpenghong
 * @date : 2021/8/3 下午4:49
 */
@FunctionalInterface
public interface ClosureInterface {

    void run();

}

/**
 * 无法闭包的正常类
 *
 * @author : zhanpenghong
 * @date : 2021/8/3 下午4:49
 */
public class NoClosureClass {

    void run() {
        ClosureStudy closureStudy = new ClosureStudy();
        /* 下面的代码报错，因为正常的类不能访问目标类的私有成员变量 */
        /*closureStudy.name;*/
    }

}
````

# double&float的精度丢失问题

```java
double yuan = 2.3;
double fen = yuan * 100;
// 结果为229
System.out.println(Double.valueOf(fen).intValue());

BigDecimal b01 = BigDecimal.valueOf(yuan);
BigDecimal b02 = b01.multiply(BigDecimal.valueOf(100));
// 结果为230
System.out.println(b02.intValue());
```

double在进行计算（包括加减和乘除运算）的时候，会丢失精度，具体原因先不做太多展开，要想运算的时候不丢失精度，需要使用BigDecimal

# 领域实体

分层领域模型规约：

- DO（ Data Object）：与数据库表结构一一对应，通过DAO层向上传输数据源对象。
- DTO（ Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。
- BO（ Business Object）：业务对象。 由Service层输出的封装业务逻辑的对象。
- AO（ Application Object）：应用对象。 在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高。
- VO（ View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象。
- POJO（ Plain Ordinary Java Object）：在本手册中， POJO专指只有setter/getter/toString的简单类，包括DO/DTO/BO/VO等。
- Query：数据查询对象，各层接收上层的查询请求。 注意超过2个参数的查询封装，禁止使用Map类来传输。

# 数据库的并发查询

通过线程池对数据库的查询做并发控制

- 每个数据源一个线程池
- 一个请求中可能有很多个数据库查询，并发提交到统一的线程池中

# 参考链接

- [Java内部类之间的闭包和回调详解](https://www.jb51.net/article/92231.htm)
