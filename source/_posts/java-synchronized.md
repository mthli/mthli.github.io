---
title : 对 Java synchronized 关键字的理解
date  : 2019.02.20
tag   : Java
---

关于 synchronized 关键字 Oracle 官方有[一篇简洁易懂的文档](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)，以下是我简单的整理。

首先我们需要知道的是，Java 中任意的 class（类）或 object（对象）都有一个内建的锁，称为 intrinsic lock 或 monitor lock. 这个锁你可以理解为是官方方便大家才内建的，如果没有这个内建的锁，你自己创建一个自定义的锁也可以达到同样的效果，只是比较繁琐罢了。

## 使用 synchronized 修饰方法

当我们使用 synchronized 修饰一个**非静态方法**，锁住的是这个方法从属的 object(this), 即**对象锁**，当该方法执行完毕（即使抛出异常）时，锁就会被释放。

当我们使用 synchronized 修饰一个**静态方法**，锁住的是这个方法从属的 class, 即**类锁**，因为静态方法从属的是 class 而不是 object. 类锁与对象锁是不冲突的，锁住了其中一个，不意味着另外一个也锁住了。

## 使用 synchronized 修饰代码块

使用 synchronized 修饰代码块时，必须显式指定一个对象来充当 intrinsic lock, 例如下面这段代码：

```java
public void addName(String name) {
    synchronized(this) {
        lastName = name;
        nameCount++;
    }
    nameList.add(name);
}
```

在上述代码中，addName 需要同步地改变 laseName 和 nameCount, 但是需要同步地调用其他对象的方法，同步地调用其他对象的方法可能会导致[死锁](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)等问题。如果没有同步代码块，开发者就得自己实现一个单独的、非同步的方法来调用 nameList.add.

使用 synchronized 修饰代码块同时可以提升并发的性能。举个例子：

```java
public class MsLunch {
    private long c1 = 0;
    private long c2 = 0;
    private Object lock1 = new Object();
    private Object lock2 = new Object();

    public void inc1() {
        synchronized(lock1) {
            c1++;
        }
    }

    public void inc2() {
        synchronized(lock2) {
            c2++;
        }
    }
}
```

MsLunch 有两个成员变量 c1 和 c2, 这两个变量绝对不会在一起使用，同时更新这两个变量需要确保同步，但我们绝没有必要在更新 c1 时也把 c2 锁住，这样就减少了不必要的阻塞。

## 可重入的锁

一个线程不能获得被另一个线程持有的锁，但是如果是自己持有的锁，则可以随意使用。这不仅在逻辑上是合理的，在代码实现上也简单很多，毕竟 Oracle 官方自己也承认：

> This describes a situation where synchronized code, directly or indirectly, invokes a method that also contains synchronized code, and both sets of code use the same lock. Without reentrant synchronization, synchronized code would have to take many additional precautions to avoid having a thread cause itself to block.
