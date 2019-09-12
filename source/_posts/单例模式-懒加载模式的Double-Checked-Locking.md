---
title: 单例模式-懒加载模式的Double-Checked Locking
date: 2019-09-12 13:43:58
tags:
- java
- 单例模式
- 设计模式
- 懒加载
- double check
- 指令重拍
categories:
- 设计模式
---

# 最简单的懒加载

```
// Single threaded version
class Foo { 
  private Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
        helper = new Helper();
    return helper;
  }
  // other functions and members...
}
```

这种加载方式在并发度很高的情况下会有问题，可能同时多个线程同时判断对象是否为空，都为空，然后都去实例化了对象。

# Double-Checked Locking 解决上述问题

```
// Broken multithreaded version
// "Double-Checked Locking" idiom
class Foo { 
  private Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
      synchronized(this) {
        if (helper == null) 
          helper = new Helper();
      }    
    return helper;
  }
  // other functions and members...
}
```

判断两次对象是否为空，第一次判断对象为空之后，后续的对象创建需要加锁。

double-check的作用：

- first check：如果没有第一个非空判断，即使单例对象创建成功了，多线程来获取该单例对象的时候，还是会产生大量的锁竞争；加上这个判断之后，只有在并发初始化对象的时候才会有锁竞争
- second check：真正保证产生的对象是单例的，同步代码块只有一个方法能进入，初次进入的时候对象为空，进行创建，创建完成释放了锁，后续的线程进入同步代码块的时候，判空将不通过，不会再重复创建对象

**为啥对象创建加锁了，在锁内部为啥还需要判断一次是否为空？？？**

如果不判断，还是会创建好多次，假设现在10个线程竞争，其中9个线程在等待锁，第一个线程执行完锁中的逻辑之后，释放所，9个线程其中一个获取到锁，进入锁内部的逻辑，如果不判断对象是否为空，还会继续创建对象，只有再判断一次对象是否为空，才能保证锁内部的对象创建不再被执行。

## Double-Checked Locking 存在的问题

上述描述似乎已经解决了我们面临的所有问题，但实际上，从JVM的角度讲，这些代码仍然可能发生错误。 
对于JVM而言，它执行的是一个个Java指令。在Java指令中创建对象和赋值操作是分开进行的，也就是说instance = new Singleton();语句是分两步执行的。但是JVM并不保证这两个操作的先后顺序，也就是说有可能JVM会为新的Singleton实例分配空间，然后直接赋值给instance成员，然后再去初始化这个Singleton实例。(即先赋值指向了内存地址，再初始化)这样就使出错成为了可能，我们仍然以A、B两个线程为例：

1. A、B线程同时进入了第一个if判断
2. A首先进入synchronized块，由于instance为null，所以它执行instance = new Singleton();
3. 由于JVM内部的优化机制，JVM先画出了一些分配给Singleton实例的空白内存，并赋值给instance成员（注意此时JVM没有开始初始化这个实例），然后A离开了synchronized块。
4. B进入synchronized块，由于instance此时不是null，因此它马上离开了synchronized块并将结果返回给调用该方法的程序。
5. 此时B线程打算使用Singleton实例，却发现它没有被初始化，于是错误发生了。

# 通过内部类实现多线程环境中的单例模式 

```
public class Singleton{        
    private Singleton(){        
        …        
    }        
    private static class SingletonContainer{        
        private static Singleton instance = new Singleton();        
    }        
    public static Singleton getInstance(){        
        return SingletonContainer.instance;        
    }        
} 
```

JVM内部的机制能够保证当一个类被加载的时候，这个类的加载过程是线程互斥的。这样当我们第一次调用getInstance的时候，JVM能够帮我们保证instance只被创建一次，并且会保证把赋值给instance的内存初始化完毕。

# 在新的JAVA内存模型中 volatile 的使用

```
// Single threaded version
class Foo { 
  private volatile Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
        helper = new Helper();
    return helper;
  }
  // other functions and members...
}
```

在JDK1.5或者更晚的版本中，扩展了volatile的语义，使得我们可以通过将helper属性字段设置为volatile来修复Double-Checked的问题。

---

### 参考

- [深刻理解双重检查锁定（double-checked locking）与单例模式](https://blog.csdn.net/gangjindianzi/article/details/78689713)
- [可以不要再使用Double-Checked Locking了](http://www.importnew.com/23980.html)
