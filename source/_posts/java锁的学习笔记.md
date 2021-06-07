---
title: java锁的学习笔记
date: 2021-05-24 13:38:51
categories:
- java
tags:
- java
- 多线程
- thread
- 锁
- lock
- synchronized
---

# synchronized与reentrantLock的性能差距

结论：在JDK 1.6之后，虚拟机对于synchronized关键字进行整体优化后，在性能上synchronized与ReentrantLock已没有明显差距，因此在使用选择上，需要根据场景而定，大部分情况下我们依然建议是synchronized关键字，原因之一是使用方便语义清晰，二是性能上虚拟机已为我们自动优化。而ReentrantLock提供了多样化的同步特性，如超时获取锁、可以被中断获取锁（synchronized的同步是不能中断的）、等待唤醒机制的多个条件变量(Condition)等，因此当我们确实需要使用到这些功能是，可以选择ReentrantLock。[参考](https://juejin.cn/post/6844903859878363144)

两个问题：

1. synchronized在1.6之后（包括1.6）到底做了什么优化，提升了性能
2. synchronized在优化之前为什么跟lock差异如此之大

## synchronized在1.6之后（包括1.6）到底做了什么优化，提升了性能

1. 偏向锁：虽然存在竞争的可能，但是实际运行过程中，只有单线程运行
2. 轻量级锁：虽然多线程运行，但是线程之间不会存在竞争
3. 锁消除：根据逃逸分析等技术，虚拟机断定不会存在多线程的竞争
4. 自旋和自适应自旋：频繁的线程上下文、用户态和内核态的切换，性能差，不如自旋等待一会儿消耗点CPU，自旋也不可能太多，自适应自旋最为合理

## synchronized在优化之前为什么跟lock差异如此之大

老的synchronized，对于并发的线程，如果都是那种执行比较快，但是并发量又比较高的情况，线程的时间可能主要消耗在系统调用上；老的synchronized只有重量级锁，依赖操作系统底层的mutexlock进行线程的挂起，需要进行系统调用，用户态到内核态的切换，对于性能影响很大。

但是lock是java api层面的锁，内部基于AQS，多线程获取锁的时候，通过cas实现，不需要进行系统调用；cas失败的时候，将线程放入等待队列中，这时候并不是直接将线程挂起，而是判断当前线程是否是队列的头结点，如果是头结点，再次尝试cas获取锁，获取到之后，执行线程；如果线程不在头结点，那么将线程挂起；对于并发较高的小任务类型的多线程场景，能有效的减少线程挂起的次数，减少上下文切换带来的性能损耗。

```java
//尝试获取锁
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
//将任务添加到等待队列中
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
//检测队列状态，如果是头结点，再次尝试获取锁，如果不是，挂起等待
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
//将线程挂起，挂起之前还要判断下是否需要挂起shouldParkAfterFailedAcquire
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

