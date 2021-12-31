---
layout: post
title:  'Java 并发 - synchronized关键字使用'
date:   2012-12-31
tags: [java]
categories: [编程语言,java]
---

## 1. synchronized的使用

### 1.1 `synchronized`的作用范围
> synchronized可以根据锁的对象，把锁分为两种，分别是类锁和对象锁

### 1.1 对象锁

>  **手动指定锁定对象，也可是是this,也可以是自定义的锁**

```java
private final Object lock1 = new Object();
 
public void printStr1() {
    synchronized (lock1) {
        for (int i = 0; i < 5; i++) {
            System.out.println(Thread.currentThread().getName() + "\t i:" + i);
        }
    }
}
```

> **synchronized修饰普通方法，锁对象默认为this**

```java
public synchronized void printStr1() {
    for (int i = 0; i < 5; i++) {
    	System.out.println(Thread.currentThread().getName() + "\t i:" + i);
    }
}
```



### 1.2 类锁

> synchronize修饰静态方法

```java
public synchronized static void printStr1() {
    for (int i = 0; i < 5; i++) {
        System.out.println(Thread.currentThread().getName() + "\t i:" + i);
    }
}
```

> synchronized指定锁对象为Class对象

```java
public void printStr1() {
   synchronized (SynchronizedClassLock.class){
        for (int i = 0; i < 5; i++) {
            System.out.println(Thread.currentThread().getName() + "\t i:" + i);
        }
    }
}
```

## 2. Synchronized加锁原理

### 2.1 加锁和释放锁的原理

> 通过查看字节码来了解加锁

代码：

```java
Object lock3 = new Object();
public void printStr3() {
    synchronized (lock3){
    }
    method1();
}
private static void method1(){}
```

字节码方案信息：

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picgo/20211231163208.png)

关注红色方框里的`monitorenter`和`monitorexit`即可。

`Monitorenter`和`Monitorexit`指令，会让对象在执行，使其锁计数器加1或者减1。每一个对象在同一时间只与一个monitor(锁)相关联，而一个monitor在同一时间只能被一个线程获得，一个对象在尝试获得与这个对象相关联的Monitor锁的所有权的时候，monitorenter指令会发生如下3中情况之一：

- monitor计数器为0，意味着目前还没有被获得，那这个线程就会立刻获得然后把锁计数器+1，一旦+1，别的线程再想获取，就需要等待
- 如果这个monitor已经拿到了这个锁的所有权，又重入了这把锁，那锁计数器就会累加，变成2，并且随着重入的次数，会一直累加
- 这把锁已经被别的线程获取了，等待锁释放

`monitorexit指令`：释放对于monitor的所有权，释放过程很简单，就是讲monitor的计数器减1，如果减完以后，计数器不是0，则代表刚才是重入进来的，当前线程还继续持有这把锁的所有权，如果计数器变成0，则代表当前线程不再拥有该monitor的所有权，即释放锁。

下图表现了对象，对象监视器，同步队列以及执行线程状态之间的关系：

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picgo/20211231162415.png)

该图可以看出，任意线程对Object的访问，首先要获得Object的监视器，如果获取失败，该线程就进入同步状态，线程状态变为BLOCKED，当Object的监视器占有者释放后，在同步队列中得线程就会有机会重新获取该监视器

### 2.2 可重入原理：加锁次数计数器

上面的demo中在执行完同步代码块之后紧接着再会去执行一个静态同步方法，而这个方法锁的对象依然就这个类对象，那么这个正在执行的线程还需要获取该锁吗? 答案是不必的，从上图中就可以看出来，执行静态同步方法的时候就只有一条monitorexit指令，并没有monitorenter获取锁的指令。这就是锁的重入性，即在同一锁程中，线程不需要再次获取同一把锁。

Synchronized先天具有重入性。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一

### 参考

- *[关键字: synchronized详解](https://pdai.tech/md/java/thread/java-thread-x-key-synchronized.html#关键字-synchronized详解)*
- *[Synchronized方法锁、对象锁、类锁区别](https://www.cnblogs.com/codebj/p/10994748.html)*
- *[synchronized到底锁住的是谁？](https://www.cnblogs.com/yulinfeng/p/11020576.html)*

