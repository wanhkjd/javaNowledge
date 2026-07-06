---
title: "分布式系统知识体系：入门、理论协议、RPC、网关、锁、事务与 ID"
source: "https://javaguide.cn/distributed-system/"
author:
  - "[[Guide]]"
published: 2026-06-25
created: 2026-07-06
description: "分布式系统面试与学习路线，涵盖分布式系统入门、中心化与去中心化、CAP、BASE、拜占庭将军问题、Paxos、Raft、ZAB、Gossip、一致性哈希、RPC、API 网关、分布式锁、分布式事务和 ZooKeeper。重点围绕 适合谁看、学习重点、这组文章怎么串起来？、建议阅读顺序、分布式 等内容展开。"
tags:
  - "clippings"
---
[![JavaGuide 官方知识星球](https://oss.javaguide.cn/xingqiu/xingqiu.png)](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html)

这份 **分布式系统知识体系** 面向后端学习、系统设计和面试复习，按“分布式入门 -> 通信调用 -> 服务治理 -> 一致性与协调 -> 工程实践”的顺序整理本站分布式相关文章。

如果你刚开始学分布式，建议先看 [分布式系统入门](https://javaguide.cn/distributed-system/distributed-system-intro.html) ，建立整体认知；如果你时间有限，建议先看 [分布式系统面试题总结](https://javaguide.cn/distributed-system/distributed-system-interview-questions.html) ，快速建立高频问题清单；如果你想系统补基础，可以按下面的专题顺序阅读。

## 适合谁看

- 正在系统学习分布式系统的后端开发者。
- 准备校招、社招、中大厂后端面试的同学。
- 想补齐分布式理论、工程实践和方案选型能力的工程师。
- 已经写过业务代码，但对 RPC、分布式锁、分布式事务、配置中心等底层原理不够熟的读者。

## 学习重点

- 分布式系统为什么需要在一致性、可用性、性能和复杂度之间做取舍？
- CAP、BASE、Paxos、Raft、ZAB、Gossip、一致性哈希分别解决什么问题？
- RPC、API 网关、配置中心、注册中心在微服务体系里分别承担什么职责？
- 分布式 ID、分布式锁、分布式事务在真实业务里有哪些常见方案和坑？
- 面试中如何把“概念、原理、场景、方案对比、落地经验”串成完整回答？

## 这组文章怎么串起来？

这组文章可以按 4 条线一起看。

第一条是 **理论线** ： [分布式系统入门](https://javaguide.cn/distributed-system/distributed-system-intro.html) 先解释多节点系统为什么会变复杂， [CAP 定理与 BASE 理论详解](https://javaguide.cn/distributed-system/protocol/cap-and-base-theorem.html) 解释分区发生时一致性和可用性怎么取舍， [分布式协调详解](https://javaguide.cn/distributed-system/protocol/centralized-and-decentralized.html) 再把 Leader、Quorum、Lease、Fencing Token、Gossip 放到同一条线上。

第二条是 **调用线** ： [RPC 远程过程调用专题](https://javaguide.cn/distributed-system/rpc/) 解决服务之间怎么互相调用， [API 网关详解](https://javaguide.cn/distributed-system/api-gateway.html) 解决外部流量怎么进入系统， [Spring Cloud Gateway 面试题总结](https://javaguide.cn/distributed-system/spring-cloud-gateway-questions.html) 则展开 Spring Cloud Gateway 的路由、断言、过滤器和限流。

第三条是 **数据一致性线** ： [分布式 ID 生成方案详解](https://javaguide.cn/distributed-system/distributed-id.html) 解决全局唯一标识问题， [分布式 ID 设计实战](https://javaguide.cn/distributed-system/distributed-id-design.html) 把 ID 放到订单号、优惠券、短网址这些业务里， [分布式锁](https://javaguide.cn/distributed-system/distributed-lock.html) 和 [分布式事务](https://javaguide.cn/distributed-system/distributed-transaction.html) 继续处理跨节点互斥和跨服务写入一致性。

第四条是 **协调组件线** ： [分布式配置中心详解](https://javaguide.cn/distributed-system/distributed-configuration-center.html) 讲配置发布和客户端容灾， [ZooKeeper 专题](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/) 讲协调组件的概念、ZAB、会话、Watcher 和 Curator 实战。

## 建议阅读顺序

1. [分布式系统入门](https://javaguide.cn/distributed-system/distributed-system-intro.html) ：先理解什么是分布式系统，以及为什么拆成多节点后会引入通信、故障和一致性问题。
2. [分布式系统面试题总结](https://javaguide.cn/distributed-system/distributed-system-interview-questions.html) ：建立高频问题清单，知道面试最常考哪些点。
3. [分布式协调详解](https://javaguide.cn/distributed-system/protocol/centralized-and-decentralized.html) ：理解 Leader、Quorum、Gossip、Lease、脑裂和 Fencing Token 之间的关系。
4. [CAP 定理与 BASE 理论详解](https://javaguide.cn/distributed-system/protocol/cap-and-base-theorem.html) ：理解分布式系统最核心的取舍逻辑。
5. [拜占庭将军问题详解](https://javaguide.cn/distributed-system/protocol/byzantine-generals-problem.html) ：理解共识问题里的恶意节点、故障假设和 BFT 容错边界。
6. [RPC 远程过程调用详解](https://javaguide.cn/distributed-system/rpc/rpc-intro.html) ：掌握服务之间如何通信，以及 RPC 框架解决了哪些工程问题。
7. [分布式 ID 生成方案详解](https://javaguide.cn/distributed-system/distributed-id.html) 、 [分布式锁入门](https://javaguide.cn/distributed-system/distributed-lock.html) 、 [分布式事务解决方案详解](https://javaguide.cn/distributed-system/distributed-transaction.html) ：补齐高频工程实践。
8. [ZooKeeper 入门指南](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/zookeeper-intro.html) 和 [分布式配置中心详解](https://javaguide.cn/distributed-system/distributed-configuration-center.html) ：理解分布式协调和配置治理。

## 核心文章

### 分布式基础与理论协议

这部分适合先建立分布式系统的底层认知，重点理解一致性、可用性、分区容错、共识算法和数据分布。

- [分布式系统入门](https://javaguide.cn/distributed-system/distributed-system-intro.html) ：理解分布式系统的定义、架构演进、典型特征、常见系统和学习路线。
- [分布式理论、算法与协议专题](https://javaguide.cn/distributed-system/protocol/) ：把 CAP、BASE、Paxos、Raft、ZAB、Gossip 和一致性哈希放在同一条学习线上。
- [分布式协调详解](https://javaguide.cn/distributed-system/protocol/centralized-and-decentralized.html) ：串起 Leader/Quorum、脑裂、Lease、Fencing Token 和 Gossip，理解“谁来做决定、状态怎么传播”这两个问题。
- [CAP 定理与 BASE 理论详解](https://javaguide.cn/distributed-system/protocol/cap-and-base-theorem.html) ：理解一致性、可用性、分区容错和最终一致性。
- [拜占庭将军问题详解](https://javaguide.cn/distributed-system/protocol/byzantine-generals-problem.html) ：理解恶意节点场景下的共识难点、 `3m + 1` 节点要求和 BFT 容错。
- [Raft 算法详解](https://javaguide.cn/distributed-system/protocol/raft-algorithm.html) ：用更易理解的共识算法入门 Leader 选举和日志复制。
- [Paxos 算法详解](https://javaguide.cn/distributed-system/protocol/paxos-algorithm.html) ：补齐经典共识算法的角色、流程和难点。
- [ZAB 协议详解](https://javaguide.cn/distributed-system/protocol/zab.html) ：理解 ZooKeeper 的原子广播、崩溃恢复和事务日志机制。
- [Gossip 协议详解](https://javaguide.cn/distributed-system/protocol/gossip-protocol.html) ：理解大规模节点之间的信息传播和最终一致性。
- [一致性哈希算法详解](https://javaguide.cn/distributed-system/protocol/consistent-hashing.html) ：理解分布式缓存、负载均衡和分库分表中的数据分布问题。

### RPC 与服务调用

RPC 解决的是远程服务调用的工程复杂度，包括序列化、网络传输、服务发现、负载均衡、超时重试和服务治理。

- [RPC 专题](https://javaguide.cn/distributed-system/rpc/) ：从 RPC 基础、Dubbo 到 HTTP 与 RPC 的关系，建立服务调用完整认知。
- [RPC 远程过程调用详解](https://javaguide.cn/distributed-system/rpc/rpc-intro.html) ：理解 RPC 调用流程、动态代理、序列化、网络传输和框架选型。
- [Dubbo 面试题总结](https://javaguide.cn/distributed-system/rpc/dubbo.html) ：串联 Dubbo 架构、服务暴露与引用、SPI、负载均衡和集群容错。
- [有了 HTTP 协议，为什么还要 RPC？](https://javaguide.cn/cs-basics/network/http-vs-rpc.html) ：厘清 HTTP 和 RPC 的层次关系与选型边界。

### API 网关与流量入口

API 网关负责统一接入、路由转发、认证鉴权、限流熔断、灰度发布和跨域处理，是微服务体系里的重要入口层。

- [API 网关详解](https://javaguide.cn/distributed-system/api-gateway.html) ：理解请求路由、认证鉴权、限流熔断、负载均衡和双层网关架构。
- [Spring Cloud Gateway 面试题总结](https://javaguide.cn/distributed-system/spring-cloud-gateway-questions.html) ：掌握 Predicate、GatewayFilter、GlobalFilter、限流熔断和常见生产问题。

### 分布式 ID、锁与事务

这部分偏工程落地，常见于订单、支付、秒杀、库存扣减、数据一致性和跨服务协作场景。

- [分布式 ID 生成方案详解](https://javaguide.cn/distributed-system/distributed-id.html) ：对比 UUID、数据库自增、号段模式、Redis、Snowflake、Leaf、Tinyid 等方案。
- [分布式 ID 设计实战](https://javaguide.cn/distributed-system/distributed-id-design.html) ：结合订单号、支付码、优惠券、一码付等业务场景理解业务 ID 设计。
- [分布式锁入门](https://javaguide.cn/distributed-system/distributed-lock.html) ：理解互斥语义、锁粒度、安全释放、超时续约和 Fencing Token。
- [分布式锁实现方案详解](https://javaguide.cn/distributed-system/distributed-lock-implementations.html) ：对比 Redis、Redisson、Redlock、ZooKeeper 和 Curator 分布式锁。
- [分布式事务解决方案详解](https://javaguide.cn/distributed-system/distributed-transaction.html) ：系统理解 XA、AT、TCC、Saga、本地消息表、事务消息和最大努力通知。

### 配置中心与 ZooKeeper

配置中心和 ZooKeeper 主要解决服务配置、注册发现、分布式协调和集群一致性问题。

- [分布式配置中心详解](https://javaguide.cn/distributed-system/distributed-configuration-center.html) ：对比 Apollo、Nacos、Spring Cloud Config 和 Kubernetes ConfigMap。
- [ZooKeeper 专题](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/) ：从 ZooKeeper 核心概念讲到 ZAB、Leader 选举、Curator 和分布式锁实践。
- [ZooKeeper 入门指南](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/zookeeper-intro.html) ：掌握 ZNode、Watcher、ACL 和典型应用场景。
- [ZooKeeper 进阶详解](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/zookeeper-plus.html) ：理解 ZAB 协议、Leader 选举、集群部署和会话管理。
- [ZooKeeper 实战教程](https://javaguide.cn/distributed-system/distributed-process-coordination/zookeeper/zookeeper-in-action.html) ：通过 Docker、zkCli、四字命令和 Curator 完成实践。

## 高频问题

- 分布式系统为什么不能同时完美满足一致性、可用性和分区容错？
- CAP 和 BASE 是什么关系？最终一致性适合哪些场景？
- 分布式系统为什么需要协调？中心化和去中心化分别适合哪些场景？
- Paxos、Raft、ZAB 有什么区别，分别用在什么地方？
- RPC 和 HTTP 有什么区别？为什么内部服务调用常用 RPC？
- 分布式 ID 如何保证全局唯一、趋势递增和业务可读？
- Redis 分布式锁有哪些坑？Redlock 是否一定可靠？
- 分布式事务有哪些方案？TCC、Saga、本地消息表分别适合什么场景？
- 配置中心和注册中心有什么区别？Apollo、Nacos、Spring Cloud Config 如何选型？

## 相关专题

- [系统设计](https://javaguide.cn/system-design/)
- [高可用系统设计](https://javaguide.cn/high-availability/)
- [高性能系统设计](https://javaguide.cn/high-performance/)
- [计算机网络](https://javaguide.cn/cs-basics/network/)

## 写在最后

如果内容对你有帮助的话，欢迎顺手给 JavaGuide 点一个免费的 Star 支持一下： [GitHub](https://github.com/Snailclimb/JavaGuide) | [Gitee](https://gitee.com/SnailClimb/JavaGuide) 。

JavaGuide 已持续维护近七年，累计 **6100+** 次提交，来自 **620+** 位贡献者共同完善。你的 Star、反馈和 PR，都是这个项目继续更新的动力。

如果你正在准备后端/AI 应用开发面试，也可以了解一下我的 [知识星球](https://javaguide.cn/about-the-author/zhishixingqiu-two-years.html) ，里面包括后端和 AI 实战项目、简历优化、一对一提问和高频考点资料，已经持续维护六年。