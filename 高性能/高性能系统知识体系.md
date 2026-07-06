---
title: "高性能系统知识体系：CDN、负载均衡、数据库优化、缓存与消息队列"
source: "https://javaguide.cn/high-performance/"
author:
  - "[[Guide]]"
published: 2026-06-11
created: 2026-07-06
description: "高性能系统面试与架构学习路线，涵盖 CDN、负载均衡、读写分离、分库分表、冷热分离、深度分页、SQL 优化、消息队列和主流 MQ。重点围绕 适合谁看、学习重点、面试回答主线、建议阅读顺序、高性能 等内容展开。结合 JavaGuide 知识体系梳理 高性能 的核心概念、实践方法、常见问题和高频面试考点，覆盖原理分析等核。"
tags:
  - "clippings"
---
[![JavaGuide 官方知识星球](https://oss.javaguide.cn/xingqiu/xingqiu.png)](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html)

这份 **高性能系统知识体系** 面向后端学习、系统设计和面试复习，围绕“减少延迟、提升吞吐、削峰填谷、降低数据库压力、优化数据访问路径”整理本站高性能相关文章。

如果你时间有限，建议先看 [高性能系统设计面试题总结](https://javaguide.cn/high-performance/high-performance-interview-questions.html) ，快速建立高频问题清单；如果你想系统补基础，可以按下面的阅读顺序推进。

学习这部分内容时，不建议只记“加缓存、加 MQ、分库分表”这些方案名。高性能优化更像一条链路分析题：请求从用户侧进来，经过 CDN、负载均衡、应用服务、缓存、数据库、消息队列，每一段都有可能成为瓶颈。能讲清楚瓶颈在哪里、为什么选这个方案、会引入什么新问题，才算真的掌握。

## 适合谁看

- 正在系统学习高性能系统设计的后端开发者。
- 准备校招、社招、中大厂后端面试的同学。
- 想补齐 CDN、负载均衡、数据库优化、消息队列等工程能力的工程师。
- 已经遇到慢 SQL、深分页、热点流量、消息积压、数据库压力等问题，但缺少系统解法的读者。

## 学习重点

- 高性能系统到底是在优化延迟、吞吐、资源利用率，还是在优化用户感知？
- CDN、负载均衡、缓存、数据库优化、消息队列分别解决链路上的哪些瓶颈？
- 读写分离、分库分表、冷热分离、深度分页优化分别适合什么场景？
- Kafka、RocketMQ、RabbitMQ、Disruptor 的定位和选型差异是什么？
- 面试中如何从“瓶颈定位 -> 方案选择 -> 取舍分析 -> 落地风险”回答高性能问题？

## 面试回答主线

回答高性能系统设计题时，可以按下面这条线展开：

1. **先确认目标** ：QPS、RT、P99、数据量、读写比例、一致性要求、成本约束。
2. **再定位瓶颈** ：入口带宽、应用线程池、慢 SQL、锁竞争、缓存命中率、MQ 积压、下游依赖。
3. **然后选方案** ：入口层用 CDN/负载均衡，应用层用缓存/限流/异步化，数据层用索引/读写分离/分库分表/冷热分离，削峰层用 MQ。
4. **最后讲取舍** ：方案带来的复杂度、一致性风险、运维成本、回滚方案和监控指标。

面试里最忌讳一上来就堆技术名词。比如“订单查询慢”不一定要分库分表，可能只是缺索引、深分页、历史数据太多或者热点商家查询集中。先把场景问清楚，再给方案，答案会稳很多。

## 建议阅读顺序

1. [高性能系统设计面试题总结](https://javaguide.cn/high-performance/high-performance-interview-questions.html) ：先建立缓存、数据库、消息队列、负载均衡等高频问题清单。
2. [CDN 工作原理详解](https://javaguide.cn/high-performance/cdn.html) 和 [负载均衡原理及算法详解](https://javaguide.cn/high-performance/load-balancing.html) ：理解流量入口和请求分发。
3. [读写分离和分库分表详解](https://javaguide.cn/high-performance/read-and-write-separation-and-library-subtable.html) 、 [常见 SQL 优化手段总结](https://javaguide.cn/high-performance/sql-optimization.html) 、 [深度分页介绍及优化建议](https://javaguide.cn/high-performance/deep-pagination-optimization.html) ：补齐数据库性能优化主线。
4. [消息队列基础知识总结](https://javaguide.cn/high-performance/message-queue/message-queue.html) ：理解异步处理、解耦、削峰、消息可靠性、顺序性和幂等。
5. 再根据技术栈深入 [Kafka 常见问题总结](https://javaguide.cn/high-performance/message-queue/kafka-questions-01.html) 、 [RocketMQ 常见问题总结](https://javaguide.cn/high-performance/message-queue/rocketmq-questions.html) 、 [RabbitMQ 常见问题总结](https://javaguide.cn/high-performance/message-queue/rabbitmq-questions.html) 。

## 核心文章

### 流量入口与请求分发

- [CDN 工作原理详解](https://javaguide.cn/high-performance/cdn.html) ：理解 GSLB 调度、缓存策略、预热刷新、命中率优化和防盗链。
- [负载均衡原理及算法详解](https://javaguide.cn/high-performance/load-balancing.html) ：理解四层/七层负载均衡、服务端/客户端负载均衡和常见调度算法。

### 数据库与数据访问优化

- [读写分离和分库分表详解](https://javaguide.cn/high-performance/read-and-write-separation-and-library-subtable.html) ：理解主从复制、读写分离、垂直拆分、水平拆分和分库分表后的问题。
- [数据冷热分离详解](https://javaguide.cn/high-performance/data-cold-hot-separation.html) ：理解冷热数据判定、分层存储、数据迁移一致性和冷数据查询优化。
- [常见 SQL 优化手段总结](https://javaguide.cn/high-performance/sql-optimization.html) ：梳理慢 SQL 定位、索引优化、查询重写和分页优化等实战方法。
- [深度分页介绍及优化建议](https://javaguide.cn/high-performance/deep-pagination-optimization.html) ：理解深分页性能问题，以及范围查询、子查询优化、延迟关联和覆盖索引方案。

### 消息队列与异步削峰

- [消息队列专题](https://javaguide.cn/high-performance/message-queue/) ：从消息队列基础讲到 Kafka、RocketMQ、RabbitMQ 和 Disruptor 的使用边界。
- [消息队列基础知识总结](https://javaguide.cn/high-performance/message-queue/message-queue.html) ：理解应用场景、消息模型、消息可靠性、顺序性、幂等和积压处理。
- [Kafka 常见问题总结](https://javaguide.cn/high-performance/message-queue/kafka-questions-01.html) ：掌握 Kafka 架构、高性能原理、消息可靠性、顺序性和 Rebalance。
- [RocketMQ 常见问题总结](https://javaguide.cn/high-performance/message-queue/rocketmq-questions.html) ：理解 RocketMQ 架构、消息类型、存储机制、可靠性和 5.x 新特性。
- [RabbitMQ 常见问题总结](https://javaguide.cn/high-performance/message-queue/rabbitmq-questions.html) ：理解 AMQP、Exchange 类型、确认机制、死信队列、延迟队列、Quorum Queue 和 Streams。
- [Disruptor 常见问题总结](https://javaguide.cn/high-performance/message-queue/disruptor-questions.html) ：理解 RingBuffer、Sequencer、WaitStrategy、无锁设计和缓存行填充。

## 高频问题

- 高性能系统优化时，应该先定位哪些指标？
- CDN 和负载均衡分别解决什么问题？
- 四层负载均衡和七层负载均衡有什么区别？
- 读写分离会带来哪些一致性问题？如何处理主从延迟？
- 分库分表后如何处理分布式 ID、跨库 JOIN 和分布式事务？
- 深度分页为什么慢？有哪些优化方案？
- 消息队列如何保证消息不丢、不重复、不乱序？
- Kafka、RocketMQ、RabbitMQ 如何选型？
- 消息积压应该如何定位和处理？

## 相关专题

- [高可用系统知识体系](https://javaguide.cn/high-availability/)
- [分布式系统知识体系](https://javaguide.cn/distributed-system/)
- [数据库](https://javaguide.cn/database/)
- [系统设计](https://javaguide.cn/system-design/)

## 写在最后

如果内容对你有帮助的话，欢迎顺手给 JavaGuide 点一个免费的 Star 支持一下： [GitHub](https://github.com/Snailclimb/JavaGuide) | [Gitee](https://gitee.com/SnailClimb/JavaGuide) 。

JavaGuide 已持续维护近七年，累计 **6100+** 次提交，来自 **620+** 位贡献者共同完善。你的 Star、反馈和 PR，都是这个项目继续更新的动力。

如果你正在准备后端/AI 应用开发面试，也可以了解一下我的 [知识星球](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html) ，里面包括后端和 AI 实战项目、简历优化、一对一提问和高频考点资料，已经持续维护六年。