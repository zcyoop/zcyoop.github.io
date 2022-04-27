---
layout: post
title:  'Java  - Guava Cache使用'
date:   2022-04-27
tags: [java]
categories: [编程语言,java]
---

# 1. 概述

## 1.1 缓存
缓存在现有业务系统中非常常见，主要是通过保存一个计算或索引代价很高的值且会被多次使用的值，从而减少该数据获取时间，最终提高系统整体速度。缓存常分为本地缓存和远端缓存，常见的远端缓存有Redis、Memcache，本地缓存一般是使用Map的方式保存在本地内存中。

## 1.2 Guava Cache
Guava Cache 与 ConcurrentMap 很相似，但也不完全一样，前者增加了更多的元素失效策略，后者只能显示的移除元素，所以如果不需要 Cache 中的特性，使用 ConcurrentHashMap 有更好的内存效率


# 2. Guava Cache的使用
以下代码展示了如何创建一个缓存对象并使用
```java
public static void main(String[] args) {
    LoadingCache<String, String> build = CacheBuilder.newBuilder()
            .concurrencyLevel(8)
            //设置缓存容器的初始容量为10
            .initialCapacity(10)
            //设置缓存最大容量为100，超过100之后就会按照LRU最近虽少使用算法来移除缓存项
            .maximumSize(100)
            //是否需要统计缓存情况,该操作消耗一定的性能,生产环境应该去除
            .recordStats()
            //设置写缓存后n秒钟过期
            .expireAfterWrite(60, TimeUnit.SECONDS)
            //设置读写缓存后n秒钟过期,实际很少用到,类似于expireAfterWrite
            //.expireAfterAccess(17, TimeUnit.SECONDS)
            //只阻塞当前数据加载线程，其他线程返回旧值
            //.refreshAfterWrite(13, TimeUnit.SECONDS)
            //设置缓存的移除通知
            .removalListener(notification -> {
                System.out.println(notification.getKey() + " " + notification.getValue() + " 被移除,原因:" + notification.getCause());
            })
            //build方法中可以指定CacheLoader，在缓存不存在时通过CacheLoader的实现自动加载缓存
            .build(getCacheLoader());
    
        System.out.println(build.getUnchecked("Jack"));
}

public static CacheLoader<String, String> getCacheLoader() {
    return new CacheLoader<String, String>() {
        @Override
        public String load(String key) throws Exception {
            return "Hello " + key;
        }
    };
}
```
LoadingCache 是 Cache 的子接口，相比较于 Cache，当从 LoadingCache 中读取一个指定 key 的记录时，如果该记录不存在，则 LoadingCache 可以自动执行加载数据到缓存的操作

在调用 CacheBuilder 的 build 方法时，代码中有传递一个 CacheLoader 类型的参数，CacheLoader 的 load 方法需要我们提供实现。当调用 LoadingCache 的 get 方法时，如果缓存不存在对应 key 的记录，则 CacheLoader 中的 load 方法会被自动调用从外存加载数据，load 方法的返回值会作为 key 对应的 value 存储到 LoadingCache 中，并从 get 方法返回。

当然如果你不想指定重建策略，那么你可以使用无参的 build() 方法，它将返回 Cache 类型的构建对象。
## 2.1 可选配置解析
### 2.1.1 缓存的并发级别
```java
CacheBuilder.newBuilder()
		// 设置并发级别为cpu核心数
		.concurrencyLevel(Runtime.getRuntime().availableProcessors()) 
		.build();
```
Guava 提供了设置并发级别的 api，使得缓存支持并发的写入和读取。同 ConcurrentHashMap 类似 Guava cache 的并发也是通过分离锁实现。在一般情况下，将并发级别设置为服务器 cpu 核心数是一个比较不错的选择。

### 2.1.2 缓存的初始容量设置
```java
CacheBuilder.newBuilder()
		// 设置初始容量为100
		.initialCapacity(100)
		.build();
```
我们在构建缓存时可以为缓存设置一个合理大小初始容量，由于 Guava 的缓存使用了分离锁的机制，扩容的代价非常昂贵，所以合理的初始容量能够减少缓存容器的扩容次数。
### 2.1.3 设置最大存储
Guava Cache 可以在构建缓存对象时指定缓存所能够存储的最大记录数量。当 Cache 中的记录数量达到最大值后再调用 put 方法向其中添加对象，Guava 会先从当前缓存的对象记录中选择一条删除掉，腾出空间后再将新的对象存储到 Cache 中。

1. **基于容量的清除 (size-based eviction):** 通过 CacheBuilder.maximumSize(long) 方法可以设置 Cache 的最大容量数，当缓存数量达到或接近该最大值时，Cache 将清除掉那些最近最少使用的缓存;
1. ** 基于权重的清除: ** 使用 CacheBuilder.weigher(Weigher) 指定一个权重函数，并且用 CacheBuilder.maximumWeight(long) 指定最大总重。比如每一项缓存所占据的内存空间大小都不一样，可以看作它们有不同的 “权重”（weights）。

### 2.1.4 缓存清除策略

-  基于存活时间的清除
   - expireAfterWrite 写缓存后多久过期
   - expireAfterAccess 读写缓存后多久过期
   - refreshAfterWrite 写入数据后多久过期, 只阻塞当前数据加载线程, 其他线程返回旧值

这几个策略时间可以单独设置, 也可以组合配置。

- 上面提到的基于容量的清除
- 显式清除
   -  个别清除：Cache.invalidate(key) 	
   -  批量清除：Cache.invalidateAll(keys) 
   -  清除所有缓存项：Cache.invalidateAll() 
- 基于引用的清除（Reference-based Eviction）：在构建 Cache 实例过程中，通过设置使用弱引用的键、或弱引用的值、或软引用的值，从而使 JVM 在 GC 时顺带实现缓存的清除，不过一般不轻易使用这个特性。
   - CacheBuilder.weakKeys()：使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式，使用弱引用键的缓存用而不是 equals 比较键。
   - CacheBuilder.weakValues()：使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式，使用弱引用值的缓存用而不是 equals 比较值。
   - CacheBuilder.softValues()：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用 == 而不是 equals 比较值。

## 2.2 缓存数据清理
> 使用 CacheBuilder 构建的缓存不会 "自动" 执行清理和回收工作，也不会在某个缓存项过期后马上清理，也没有诸如此类的清理机制。相反，它会在写操作时顺带做少量的维护工作，或者偶尔在读操作时做——如果写操作实在太少的话。
>
> 这样做的原因在于：如果要自动地持续清理缓存，就必须有一个线程，这个线程会和用户操作竞争共享锁。此外，某些环境下线程创建可能受限制，这样 CacheBuilder 就不可用了。

```java
LoadingCache<String, String> build = CacheBuilder.newBuilder()
        //设置写缓存后n秒钟过期
        .expireAfterWrite(3, TimeUnit.SECONDS)
        .build(new CacheLoader<String, String>() {
            @Override
            public String load(String key) throws Exception {
                return key + " value";
            }
        });

Thread t1 = new Thread(() -> {
    try {
        build.put("a", "va");
        Thread.sleep(6 * 1000);
        build.put("b", "vb");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});

t1.start();


for (int i = 0; i < 10; i++) {
    System.out.println("当前缓存长度：" + build.size());
    Thread.sleep(1000);
}

System.out.println("a值："+build.getIfPresent("a"));
System.out.println("b值："+build.get("b"));


====================输出结果=========================
当前缓存长度：0
当前缓存长度：1
当前缓存长度：1
当前缓存长度：1
当前缓存长度：1
当前缓存长度：1
当前缓存长度：2
当前缓存长度：2
当前缓存长度：2
当前缓存长度：2
a值：null
b值：b value
```
上面代码说明了，如果数据清理并不会主动发生，而是在使用到该条数据时才会进行清理或刷新

## 2.3 给移出操作添加一个监视器
```java
CacheBuilder.newBuilder()
                //设置缓存的移除通知
                .removalListener((notification) -> {
                    System.out.println(notification.getKey() + " " + notification.getValue() + " 被移除,原因:" + notification.getCause());
                })
                .build();
```
**但是要注意的是：**
**默认情况下，监听器方法是在移除缓存时同步调用的。因为缓存的维护和请求响应通常是同时进行的，代价高昂的监听器方法在同步模式下会拖慢正常的缓存请求。在这种情况下，你可以使用**`**RemovalListeners.asynchronous(RemovalListener, Executor)**`**把监听器装饰为异步操作。**

## 2.4 调用统计
```java
Cache<String, String> cache = CacheBuilder.newBuilder()
        .maximumSize(3)
        .recordStats() //开启统计信息开关
        .build();
cache.put("1", "v1");
cache.put("2", "v2");
cache.put("3", "v3");
cache.put("4", "v4");

cache.getIfPresent("1");
cache.getIfPresent("2");
cache.getIfPresent("3");
cache.getIfPresent("4");
cache.getIfPresent("5");
cache.getIfPresent("6");

System.out.println(cache.stats()); //获取统计信息
```

# 参考

- [_Guava cache使用总结_](https://www.cnblogs.com/rickiyang/p/11074159.html)
- [CachesExplained](https://github.com/google/guava/wiki/CachesExplained#caches)