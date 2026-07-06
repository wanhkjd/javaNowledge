---
title: Java 知识体系：基础、集合、并发、JVM、IO 与新特性
source: https://javaguide.cn/java/
author:
  - "[[Guide]]"
published: 2026-05-27
created: 2026-07-06
description: Java 面试与知识体系学习路线，涵盖 Java 基础、集合源码、并发编程、JVM、IO/NIO 和 Java 新特性，适合校招、社招和 Java 后端面试复习。重点围绕 适合谁看、学习重点、建议阅读顺序、核心文章、Java 等内容展开。结合 JavaGuide 知识体系梳理 Java 的核心概念、实践方法等核心内容。
tags:
  - clippings
---
[![JavaGuide 官方知识星球](https://oss.javaguide.cn/xingqiu/xingqiu.png)](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html)

这份 **Java 知识体系** 面向 Java 后端学习和面试复习，按“基础语法 -> 集合容器 -> 并发编程 -> IO/NIO -> JVM -> 新特性”的顺序整理本站 Java 相关文章。

如果你时间有限，建议先看 Java 基础、集合、并发和 JVM 的面试题总结，快速建立高频问题清单；如果你想系统补基础，可以按下面的专题顺序阅读。

## 适合谁看

- 正在系统学习 Java 的后端开发者。
- 准备校招、社招、中大厂 Java 后端面试的同学。
- 想把 Java 基础、集合、并发、JVM、IO 和新特性串起来复习的读者。
- 已经写过 Java 项目，但对底层原理、源码设计和工程实践理解不够系统的工程师。

## 学习重点

- Java 基础语法、面向对象、异常、泛型、反射、代理、序列化等核心机制。
- List、Map、Queue、并发容器的使用边界、源码实现和常见面试题。
- Java 线程、锁、JMM、CAS、AQS、线程池、CompletableFuture 和虚拟线程。
- JVM 内存区域、类加载、垃圾回收、参数配置、监控工具和线上问题排查。
- BIO、NIO、AIO、IO 模型，以及装饰器、适配器等 IO 相关设计模式。
- Java 8 到 Java 26 的重要新特性，以及哪些特性真正影响日常开发。

## 建议阅读顺序

1. [Java 基础专题](https://javaguide.cn/java/basis/) ：先掌握语法、面向对象、泛型、反射、代理、序列化等基础能力。
2. [Java 集合专题](https://javaguide.cn/java/collection/) ：理解 ArrayList、LinkedList、HashMap、ConcurrentHashMap 等常用容器的使用和源码。
3. [Java 并发编程专题](https://javaguide.cn/java/concurrent/) ：系统学习线程、锁、JMM、CAS、AQS、线程池和并发工具类。
4. [JVM 专题](https://javaguide.cn/java/jvm/) ：理解内存区域、类加载、垃圾回收、JVM 参数和线上排查。
5. [Java IO 专题](https://javaguide.cn/java/io/) ：补齐 BIO、NIO、AIO、Reactor、多路复用和 IO 设计模式。
6. [Java 新特性专题](https://javaguide.cn/java/new-features/) ：按版本梳理 Lambda、Stream、模块化、var、Record、虚拟线程等关键特性。

## 核心文章

### Java 基础

- [Java 基础专题](https://javaguide.cn/java/basis/) ：从基础语法讲到核心机制和常见 Java 面试题。
- [Java基础常见面试题总结(上)](https://javaguide.cn/java/basis/java-basic-questions-01.html) ：覆盖 Java 语言特点、基础语法、面向对象和常用类。
- [Java基础常见面试题总结(中)](https://javaguide.cn/java/basis/java-basic-questions-02.html) ：继续梳理异常、泛型、反射、注解和常见细节。
- [Java基础常见面试题总结(下)](https://javaguide.cn/java/basis/java-basic-questions-03.html) ：补齐高级基础知识和常见易错点。
- [Java 值传递详解](https://javaguide.cn/java/basis/why-there-only-value-passing-in-java.html) ：厘清值传递、引用变量和对象修改之间的关系。
- [Java 序列化详解](https://javaguide.cn/java/basis/serialization.html) ：理解序列化机制、serialVersionUID、安全风险和替代方案。
- [Java 反射机制详解](https://javaguide.cn/java/basis/reflection.html) 和 [Java 代理模式详解](https://javaguide.cn/java/basis/proxy.html) ：掌握框架底层常见机制。

### Java 集合

- [Java 集合专题](https://javaguide.cn/java/collection/) ：串联集合框架、使用注意事项和常见源码分析。
- [Java集合常见面试题总结(上)](https://javaguide.cn/java/collection/java-collection-questions-01.html) 和 [Java集合常见面试题总结(下)](https://javaguide.cn/java/collection/java-collection-questions-02.html) ：覆盖 List、Set、Map、Queue 和并发集合高频问题。
- [Java集合使用注意事项总结](https://javaguide.cn/java/collection/java-collection-precautions-for-use.html) ：总结集合判空、遍历、扩容、线程安全和性能相关注意点。
- [ArrayList 源码分析](https://javaguide.cn/java/collection/arraylist-source-code.html) 、 [HashMap 源码分析](https://javaguide.cn/java/collection/hashmap-source-code.html) 、 [ConcurrentHashMap 源码分析](https://javaguide.cn/java/collection/concurrent-hash-map-source-code.html) ：从源码理解常用容器的设计取舍。

### Java 并发

- [Java 并发编程专题](https://javaguide.cn/java/concurrent/) ：围绕线程、锁、内存模型、线程池和并发工具展开。
- [Java并发常见面试题总结（上）](https://javaguide.cn/java/concurrent/java-concurrent-questions-01.html) 、 [Java并发常见面试题总结（中）](https://javaguide.cn/java/concurrent/java-concurrent-questions-02.html) 、 [Java并发常见面试题总结（下）](https://javaguide.cn/java/concurrent/java-concurrent-questions-03.html) ：建立并发面试问题清单。
- [JMM（Java 内存模型）详解](https://javaguide.cn/java/concurrent/jmm.html) ：理解可见性、原子性、有序性和 happens-before。
- [CAS 详解](https://javaguide.cn/java/concurrent/cas.html) 、 [AQS 详解](https://javaguide.cn/java/concurrent/aqs.html) 、 [Java 线程池详解](https://javaguide.cn/java/concurrent/java-thread-pool-summary.html) ：掌握并发底层高频考点。
- [虚拟线程常见问题总结](https://javaguide.cn/java/concurrent/virtual-thread.html) ：理解 Project Loom 对并发模型的影响。

### JVM 与 IO

- [JVM 专题](https://javaguide.cn/java/jvm/) ：围绕内存、类加载、GC、参数、工具和线上排查展开。
- [Java内存区域详解（重点）](https://javaguide.cn/java/jvm/memory-area.html) ：理解程序计数器、虚拟机栈、本地方法栈、堆和方法区。
- [JVM垃圾回收详解（重点）](https://javaguide.cn/java/jvm/jvm-garbage-collection.html) ：理解对象存活判断、垃圾收集算法和主流垃圾收集器。
- [类加载过程详解](https://javaguide.cn/java/jvm/class-loading-process.html) 和 [类加载器详解（重点）](https://javaguide.cn/java/jvm/classloader.html) ：掌握类生命周期和双亲委派模型。
- [Java IO 专题](https://javaguide.cn/java/io/) ：从 BIO、NIO、AIO 讲到 IO 模型和 IO 设计模式。
- [Java IO 基础知识总结](https://javaguide.cn/java/io/io-basis.html) 、 [Java NIO 核心知识总结](https://javaguide.cn/java/io/nio-basis.html) 、 [Java IO 模型详解](https://javaguide.cn/java/io/io-model.html) ：补齐网络编程和中间件学习前置知识。

### Java 新特性

- [Java 新特性专题](https://javaguide.cn/java/new-features/) ：按版本梳理 Java 8 之后的重要语言、标准库和 JVM 特性。
- [Java8 新特性实战](https://javaguide.cn/java/new-features/java8-common-new-features.html) ：掌握 Lambda、Stream、Optional、接口默认方法和新日期 API。
- [Java 11 新特性概览（重要）](https://javaguide.cn/java/new-features/java11.html) 、 [Java 17 新特性概览（重要）](https://javaguide.cn/java/new-features/java17.html) 、 [Java 21 新特性概览(重要)](https://javaguide.cn/java/new-features/java21.html) ：优先关注 LTS 版本中的长期可用特性。

## 高频问题

- Java 为什么是值传递？对象引用作为参数传递时到底发生了什么？
- `String` 、 `StringBuilder` 、 `StringBuffer` 有什么区别？
- `equals()` 和 `hashCode()` 有什么关系？
- `ArrayList` 和 `LinkedList` 如何选择？ `HashMap` 为什么线程不安全？
- `ConcurrentHashMap` 在 JDK 7 和 JDK 8 中有什么变化？
- `synchronized` 和 `ReentrantLock` 有什么区别？
- JMM 如何保证可见性、有序性和原子性？
- 线程池核心参数如何配置？为什么不建议直接使用 `Executors` ？
- JVM 内存区域如何划分？哪些区域可能发生 OOM？
- G1、ZGC、Shenandoah 分别适合什么场景？
- BIO、NIO、AIO 有什么区别？Reactor 模型解决什么问题？
- Java 8、11、17、21 中哪些新特性最值得掌握？

## 相关专题

- [计算机基础](https://javaguide.cn/cs-basics/)
- [系统设计](https://javaguide.cn/system-design/)
- [数据库](https://javaguide.cn/database/)
- [分布式系统知识体系](https://javaguide.cn/distributed-system/)
- [高性能系统知识体系](https://javaguide.cn/high-performance/)

## 写在最后

如果内容对你有帮助的话，欢迎顺手给 JavaGuide 点一个免费的 Star 支持一下： [GitHub](https://github.com/Snailclimb/JavaGuide) | [Gitee](https://gitee.com/SnailClimb/JavaGuide) 。

JavaGuide 已持续维护近七年，累计 **6100+** 次提交，来自 **620+** 位贡献者共同完善。你的 Star、反馈和 PR，都是这个项目继续更新的动力。

如果你正在准备后端/AI 应用开发面试，也可以了解一下我的 [知识星球](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html) ，里面包括后端和 AI 实战项目、简历优化、一对一提问和高频考点资料，已经持续维护六年。