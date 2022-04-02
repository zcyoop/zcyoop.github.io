---
layout: post
title:  'Java 并发 - CompletableFuture详解'
date:   2022-04-02
tags: [java]
categories: [编程语言,java]
---

# 1. 概述

## 1.1 Java8 之前的异步编程
在Java8之前，异步编程主要使用Future 类来描述一个异步计算的结果。你可以使用 isDone() 方法检查计算是否完成，或者使用 get() 方法阻塞住调用线程，直到计算完成返回结果，也可以使用 cancel() 方法停止任务的执行。
虽然 Future 提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们异步编程的初衷相违背，轮询的方式又会耗费无谓的 CPU 资源，而且也不能及时的获取结果。

## 1.2 CompletableFuture基本概述
在 Java 8 中，新增了一个包含 50 多个方法的类：CompletableFuture，提供了非常强大的 Future 扩展功能，可以帮助我们简化异步编程的复杂性，提供函数式编程的能力。在 Java 8 中，新增了一个包含 50 多个方法的类：CompletableFuture，提供了非常强大的 Future 扩展功能，可以帮助我们简化异步编程的复杂性，提供函数式编程的能力。

# 2. CompletableFuture 类功能概览
```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T>
```
CompletableFuture 实现了 Future 接口，拥有 Future 所有的特性，比如可以使用 get() 方法获取返回值等；还实现了 CompletionStage 接口，这个接口有超过 40 个方法，功能太丰富了，它主要是为了**编排任务的工作流**。

CompletableFuture提供了方法大约有50多个，单纯一个个记忆，是很麻烦的，因此将其划分为以下几类

**创建类**

- completeFuture 可以用于创建默认返回值
- runAsync 异步执行，无返回值
- supplyAsync 异步执行，有返回值
- anyOf 任意一个执行完成，就可以进行下一步动作
- allOf 全部完成所有任务，才可以进行下一步任务

**状态取值类**

- join 合并结果，等待
- get 合并等待结果，可以增加超时时间;get和join区别，join只会抛出unchecked异常，get会返回具体的异常
- getNow 如果结果计算完成或者异常了，则返回结果或异常；否则，返回valueIfAbsent的值
- isCancelled
- isCompletedExceptionally
- isDone

**控制类（用于主动控制CompletableFuture的完成行为）**

- complete
- completeExceptionally
- cancel

**接续类（ CompletableFuture 最重要的特性，没有这个的话，CompletableFuture就没意义了，用于注入回调行为）**

- thenApply, thenApplyAsync
- thenAccept, thenAcceptAsync
- thenRun, thenRunAsync
- thenCombine, thenCombineAsync
- thenAcceptBoth, thenAcceptBothAsync
- runAfterBoth, runAfterBothAsync
- applyToEither, applyToEitherAsync
- acceptEither, acceptEitherAsync
- runAfterEither, runAfterEitherAsync
- thenCompose, thenComposeAsync
- whenComplete, whenCompleteAsync
- handle, handleAsync
- exceptionally

上面方法很多，但是都有规律可循：

1. 带有Async，都是异步方法，对应的没有Async则是同步方法
1. 带有Async后缀结尾的方法，都有两个重载的方法，一个是使用内置的forkjoin线程池，另一个需传入线程池
1. 带有run开头的方法，其入口参数一定是无参的，并且没有返回值，类似于执行Runnable方法。
1. 带有supply的方法，入口也是没有参数的，但是有返回值
1. 带有Accept开头或者结尾的方法，入口参数是有参数，但是没有返回值
1. 带有Apply开头或者结尾的方法，入口有参数，有返回值
1. 带有either后缀的方法，表示谁先完成就消费谁
# 
3 CompletableFuture 接口详解
## 3.1 提交执行的静态方法

| 方法名 | 描述 |
| --- | --- |
| runAsync(Runnable runnable) | 执行异步代码，使用 ForkJoinPool.commonPool() 作为它的线程池 |
| runAsync(Runnable runnable, Executor executor) | 执行异步代码，使用指定的线程池 |
| supplyAsync(Supplier<U> supplier) | 异步执行代码，有返回值，使用 ForkJoinPool.commonPool() 作为它的线程池 |
| supplyAsync(Supplier<U> supplier, Executor executor) | 异步执行代码，有返回值，使用指定的线程池执行 |

上述四个方法，都是提交任务的，runAsync 方法需要传入一个实现了 Runnable 接口的方法，supplyAsync 需要传入一个实现了 Supplier 接口的方法，实现 get 方法，返回一个值。
### 3.1.1 run 和 supply 的区别
run 就是执行一个方法，没有返回值，supply 执行一个方法，有返回值。
### 3.1.2 一个参数和两个参数的区别
第二个参数是线程池，如果没有传，则使用自带的 ForkJoinPool.commonPool() 作为线程池，这个线程池默认创建的线程数是 CPU 的核数（也可以通过 JVM option:-Djava.util.concurrent.ForkJoinPool.common.parallelism 来设置 ForkJoinPool 线程池的线程数）

### 3.2 串行关系 api
> 一个任务接着一个任务执行。例如，接水->烧水

这些 api 之间主要是能否获得前一个任务的返回值与自己是否有返回值的区别。

| api | 是否可获得前一个任务的返回值 | 是否有返回值 |
| --- | --- | --- |
| thenApply | 能 | 有 |
| thenAccept | 能 | 无 |
| thenRun | 不能 | 无 |
| thenCompose | 能 | 有 |

### 3.2.1 thenApply 和 thenApplyAsync 使用
```java
public static void main(String[] args) throws Exception {
    CompletableFuture.supplyAsync(() -> "Hello")
            .thenApply(x->x+" World")
            .thenApplyAsync(y->y+"!");
}
```
### 3.2.2 thenApply 和 thenApplyAsync 的区别
这两个方法的区别，在于谁去执行任务。如果使用 thenApplyAsync，那么执行的线程是从 ForkJoinPool.commonPool() 或者自己定义的线程池中取线程去执行。如果使用 thenApply，又分两种情况，如果 supplyAsync 方法执行速度特别快，那么 thenApply 任务就使用主线程执行，如果 supplyAsync 执行速度特别慢，就是和 supplyAsync 执行线程一样。
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println("----------supplyAsync 执行很快");
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
        System.out.println(Thread.currentThread().getName());
        return "1";
    }).thenApply(s -> {
        System.out.println(Thread.currentThread().getName());
        return "2";
    });
    System.out.println(future1.get());

    System.out.println("----------supplyAsync 执行很慢");
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
        }
        System.out.println(Thread.currentThread().getName());
        return "1";
    }).thenApply(s -> {
        System.out.println(Thread.currentThread().getName());
        return "2";
    });
    System.out.println(future2.get());
}
```
执行结果
```java
----------supplyAsync 执行很快
ForkJoinPool.commonPool-worker-1
main
2
----------supplyAsync 执行很慢
ForkJoinPool.commonPool-worker-1
ForkJoinPool.commonPool-worker-1
2
```
### 3.2.3 thenCompose 的使用
假设有两个异步任务，第二个任务想要获取第一个任务的返回值，并且做运算，我们可以用 thenCompose。
```java
public static void main(String[] args) throws Exception {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
            .thenCompose(x -> CompletableFuture.supplyAsync(() -> x + " World"));
}
```
## 3.3 And 汇聚关系 Api
> 等待两件事同时做好后再执行后面的事。例如，烧水+拿茶 =>泡茶

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> thenComposeOne = CompletableFuture.supplyAsync(() -> 192);
    CompletableFuture<Integer> thenComposeTwo = CompletableFuture.supplyAsync(() -> 196);
    CompletableFuture<Integer> thenComposeCount = thenComposeOne
        .thenCombine(thenComposeTwo, (s, y) -> s + y);

    thenComposeOne.thenAcceptBoth(thenComposeTwo,(s,y)-> System.out.println("thenAcceptBoth"));
    thenComposeOne.runAfterBoth(thenComposeTwo, () -> System.out.println("runAfterBoth"));

    System.out.println(thenComposeCount.get());
}
```
### 3.3.1 thenCombine 的使用
加入我们要计算两个异步方法返回值的和，就必须要等到两个异步任务都计算完才能求和，此时可以用 thenCombine 来完成。

### 3.3.2 thenAcceptBoth
接收前面两个异步任务的结果，执行一个回调函数，但是这个回调函数没有返回值。

### 3.3.3 runAfterBoth
接收前面两个异步任务的结果，但是回调函数，不接收参数，也不返回值。

## 3.4 Or 汇聚关系 Api
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> thenComposeOne = CompletableFuture.supplyAsync(() -> 192);
    CompletableFuture<Integer> thenComposeTwo = CompletableFuture.supplyAsync(() -> 196);
    CompletableFuture<Integer> thenComposeCount = thenComposeOne
            .applyToEither(thenComposeTwo, s -> s + 1);

    thenComposeOne.acceptEither(thenComposeTwo,s -> {});

    thenComposeOne.runAfterEither(thenComposeTwo,()->{});

    System.out.println(thenComposeCount.get());
}
```
### 3.4.1 applyToEither
任何一个执行完就执行回调方法，回调方法接收一个参数，有返回值
### 3.4.2 acceptEither
任何一个执行完就执行回调方法，回调方法接收一个参数，无返回值
### 3.4.3 runAfterEither
任何一个执行完就执行回调方法，回调方法不接收参数，也无返回值

## 3.5 处理异常
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    CompletableFuture.supplyAsync(() -> {
        System.out.println("execute one ");
        return 100;
    })
            .thenApply(s -> 10 / 0)
            .thenRun(() -> System.out.println("thenRun"))
            .thenAccept(s -> System.out.println("thenAccept"))
            .exceptionally(s -> {
                System.out.println("异常处理");
                return null;
            });

    CompletableFuture.runAsync(() -> System.out.println("other"));
}
```
可以使用  exceptionally 来处理异常。
使用 handle() 方法也可以处理异常。但是 handle() 方法的不同之处在于，即使没有发生异常，也会执行。

- 

# 参考

- [Java 8 异步编程 CompletableFuture 全解析](https://mp.weixin.qq.com/s?__biz=MzA4MDczNDA5Mg==&mid=2247486139&idx=1&sn=0c69eb21cc7c448031fc4f911f843703&chksm=9f9ef858a8e9714e16660ed3c86856389daf9cc8f086b0832120b683a54e21063db045c88522&scene=21#wechat_redirect)
- [CompletableFuture使用大全，简单易懂](https://juejin.cn/post/6844904195162636295#heading-11)
