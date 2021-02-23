[TOC]



# 前言

`HashMap` 在我们日常的开发中使用频率最高的一个工具类之一，然而使用 `HashMap` 最大的问题之一就是它是线程不安全的，如果我们想要线程安全应该怎么办呢？这时候就可以选择使用 `ConcurrentHashMap`，`ConcurrentHashMap` 和 `HashMap` 的功能是基本一样的，`ConcurrentHashMap` 是 `HashMap` 的线程安全版本。

因 `ConcurrentHashMap` 和 `HashMap` 排除线程的安全性方面，所以有很多相同的设计思想本文不会做太多重复介绍，如果大家不了解 `HashMap` 底层实现原理，建议在阅读本文可以先阅读[手把手轻松读懂HashMap源码](https://zhouwenxing.github.io/corejava/手把手轻松读懂HashMap源码)了解 `HashMap` 的设计思想。

# ConcurrentHashMap 源码分析

`ConcurrentHashMap` 是 `HashMap` 的线程安全版本，其内部和 `HashMap` 一样，也是采用了数组+链表+红黑树的方式来实现。

如何实现线程的安全性？加锁。但是这个锁应该怎么加呢？在 `HashTable` 中，是直接在 `put` 和 `get` 方法上加上了 `synchronized`，理论上来说 `ConcurrentHashMap` 也可以这么做，但是这么做锁的粒度太大，会非常影响并发性能，所以在 `ConcurrentHashMap ` 中并没有采用这么直接简单粗暴的方法，其内部采用了非常精妙的设计，大大减少了锁的竞争，提升了并发性能。

`ConcurrentHashMap` 中的初始化和 `HashMap` 中一样，而且容量也会调整为 2 的 N 次幂，在这里不做重复介绍这么做的原因。

## JDK1.8 版本 ConcurrentHashMap 做了什么改进

在 `jdk1.7` 版本中，`ConcurrentHashMap ` 采用了分段的设计思路，即将一个 `ConcurrentHashMap` 中所有的桶分割成不同的段，每次加锁只锁当前线程操作的段，以此来降低锁的粒度，提升并发效率。但是这么做的缺陷就是每次通过 `hash` 确认位置时需要 `2` 次才能定位到当前 `key` 应该落在哪个槽：

1. 通过 `hash` 值和 `段数组长度-1` 进行位运算确认当前 `key` 属于哪个段，即确认其在 `segments` 数组的位置。
2. 再次通过 `hash` 值和 `table` 数组（即 `ConcurrentHashMap` 底层存储数据的数组）长度 - 1进行位运算确认其所在桶。

为了进一步优化性能，在 `jdk1.8` 版本中，对 `ConcurrentHashMap ` 做了优化，取消了分段锁的设计，取而代之的是通过 `cas` 操作和 `synchronized` 关键字来实现优化，而扩容的时候也利用了一种分而治之的思想来提升扩容效率。

 ## ConcurrentHashMap 如何保证线程的安全性

在



# 总结