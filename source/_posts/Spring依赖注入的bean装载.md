---
title: Spring依赖注入的bean装载
date: 2018-03-22 18:10:56
categories:
- spring
tags:
- spring
- ioc
---

---
# 关于父类
---

通过spring装配的两个bean（A和B），都继承了一个父类，父类也是通过spring进行装配，spring容器初始化的时候，会装配这两个bean（A和B），这两个子类装配的时候，会分别初始化一次父类，也就是父类会初始化两次。再加上父类本身被装载，所以还有一次，总共三次。

---
# java类的加载过程
---

1. 静态变量
2. 静态初始化块
3. 变量
4. 初始化块
5. 构造器

# java子类加载过程

## 类加载

1. 父类--静态变量
2. 父类--静态初始化块
3. 子类--静态变量
4. 子类--静态初始化块

## 方法调用

1. 子类main方法

## 对象创建

1. 父类--变量
2. 父类--初始化块
3. 父类--构造器
4. i=9, j=0
5. 子类--变量
6. 子类--初始化块
7. 子类--构造器
8. i=9,j=20

---
# spring的bean初始化过程
---

1. 构造器
2. 属性注入

跟jdk的初始化不同的是，spring创建bean的时候，先调用构造器，然后在进行属性的注入。

- SpringBeanSub: 通过spring注入的一个类，有父类SpringBeanParent
- SpringBeanParent：通过spring注入的一个类
- SpringBeanOther：通过spring注入的一个类，没有父类

## spring的bean的配置文件

```
<bean id="springBeanSub" class="hong.study.java.spring的bean装载.SpringBeanSub"/>
<bean id="springBeanParent" class="hong.study.java.spring的bean装载.SpringBeanParent"/>
<bean id="springBeanOther" class="hong.study.java.spring的bean装载.SpringBeanOther"/>
```

## spring的bean的java代码

```
//SpringBeanSub
public class SpringBeanSub extends SpringBeanParent {

    public SpringBeanSub() {
        System.out.println("SpringBeanSub");
    }
}

//SpringBeanInterface
public class SpringBeanParent implements SpringBeanInterface {

    @Autowired
    private SpringBeanOther springBeanOther;

    public SpringBeanParent() {
        System.out.println("SpringBeanParent");
    }
}

//SpringBeanInterface
public class SpringBeanOther implements SpringBeanInterface {

    public SpringBeanOther() {
        System.out.println("SpringBeanOther");
    }
}

//SpringBeanInterface
public interface SpringBeanInterface {

}
```

## main方法

```
public class MyMain {

    public static void main(String[] args) throws IOException {
        SpringContextHolder.init();
        System.out.println(SpringContextHolder.getBean("springBeanParent") + "");
    }

}
```

## 输出结果

```
SpringBeanParent
SpringBeanSub
SpringBeanOther
SpringBeanParent
======
```

结果分析：

按照spring配置文件中的配置，bean的加载顺序为`springBeanSub -> springBeanParent -> springBeanOther`

- 加载`springBeanSub`：会先创建父类的bean，spring先调用`springBeanParent`的构造器，所以输出SpringBeanParent，父类构造器调用完成之后，还没有初始化父类中的属性`springBeanOther`，父类构造器调用完成之后，先去调用了子类的构造器，输出SpringBeanSub，然后再去初始化父类的属性`SpringBeanOther`，控制台输出SpringBeanOther，到此`springBeanSub`这个bean加载完成。
- 加载`springBeanParent`：先调用构造器，输出SpringBeanParent，到此可以看出，`springBeanParent`作为父类被加载了一次，自己又被spring容器加载了一次。构造器调用完成之后，会初始化属性`springBeanOther`，但是由于spring默认是单例模式，所以这个bean不会再去初始化，所以没有输出。
- 加载`springBeanOther`：由于在加载`springBeanSub`的时候已经加载过了，所以不会在初始化，也就没有输出。
