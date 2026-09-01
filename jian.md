# 万守洋

**求职意向：Java 后端开发实习生** ｜ 电话：178-3933-0662 ｜ 邮箱：3599808381@qq.com ｜ 2027 届本科 ｜ 可实习 6 个月以上

---

## 🎓 教育背景

**河南农业大学** ｜ 软件工程 · 本科 ｜ 2023.09 – 2027.06 (2027 届)
- **主修课程**：数据结构、计算机网络、操作系统、数据库原理、Java 程序设计、算法设计与分析、软件工程
- **综合素养**：通过大学英语四/六级（CET-4/6），具备扎实的计算机基础与良好的英文文档查阅能力

---

## 💻 专业技能

- **Java 基础**：熟练掌握集合框架（HashMap/ConcurrentHashMap 底层实现与扩容）、IO 与反射；深入理解 JVM 运行时内存模型、类加载机制与分代垃圾回收（CMS/G1）。
- **并发编程**：熟悉线程池核心参数与调优、ThreadLocal 上下文传递与生命周期清理、CAS 与 AQS 原理；具备并发安全编程意识与 Redisson 分布式锁落地经验。
- **后端框架**：熟练使用 Spring Boot 3、Spring MVC、MyBatis、MyBatis-Plus；熟练进行 RESTful API 契约设计、入参校验与基于 AOP 的全局统一异常拦截。
- **微服务体系**：熟悉 Spring Cloud Alibaba（Nacos/Gateway/OpenFeign/Sentinel/Seata）；理解微服务边界拆分、网关动态路由、全链路上下文透传与分布式事务。
- **数据库与缓存**：熟练使用 MySQL（InnoDB 存储引擎、B+ 树索引结构、事务 ACID、MVCC、锁机制与慢 SQL 优化）；熟悉 Redis 核心结构、持久化与缓存治理（击穿/穿透/雪崩），掌握 Caffeine + Redis 多级缓存实践。
- **中间件与检索**：熟悉 RabbitMQ 消息可靠投递（Confirm/ACK）、死信/延迟队列与基于 Spring 事务同步器的异步解耦；掌握 Elasticsearch 8 组合检索、高亮分页与索引同步，了解 XXL-JOB。
- **开发与规范**：熟练使用 Git、Maven、IntelliJ IDEA、Apifox、Docker、Linux；熟练使用 springdoc/Knife4j 维护接口文档；严格遵守《阿里巴巴 Java 开发手册》规约。

---

## 🚀 项目经历

### 1. Novel 互动小说平台（单体架构）
**2026.05 – 至今** ｜ Spring Boot 3 · Java 21 · MyBatis-Plus · MySQL 8 · Redis · Caffeine · Redisson · Elasticsearch 8 · RabbitMQ · XXL-JOB · Sentinel · Spring AI

前后端分离小说系统，涵盖前台门户、作家工作台与平台管理后台，支撑小说检索、章节阅读、作家创作与运营审核。在开源项目基础上进行二次开发，负责后台研发与性能优化，深度实践多级缓存、检索同步、安全防护与 AI 伴读。

- **管理后台研发与规范化落地**：独立设计并实现运营管理后台 30+ 核心接口（数据看板、用户封禁、作品与章节审核、动态推荐位、新闻与友链管理），统一通用响应模型、分页结构与错误码规范。
- **多端鉴权与上下文隔离**：基于 JWT + 拦截器 + 策略模式构建前台、作家、管理三端统一认证体系；使用 ThreadLocal 维护用户上下文，并在 `afterCompletion` 强制清理，彻底杜绝线程池复用导致的串号与内存泄漏；引入作家归属权校验切面防止横向越权。
- **Caffeine + Redis 多级缓存与一致性保障**：基于 Spring Cache 构建多级缓存，针对首页推荐、排行榜与小说详情等高频读场景配置差异化 TTL 与随机过期抖动，配合空值缓存解决缓存雪崩与击穿；高频读接口平均耗时从 45ms 降至 3ms，数据库读负载降低 90%+。
- **接口防刷限流与声明式分布式锁**：基于 Sentinel 自定义拦截器配置全局 QPS 匀速排队（2000 QPS）与单 IP 秒级/分钟级参数流控，防御恶意爬虫；基于 Redisson + 自定义 `@Lock` 注解 + AOP + SpEL 实现声明式分布式锁，保证章节发布、高频评论等并发写操作的幂等性。
- **全文检索与事务安全数据同步**：集成 Elasticsearch 8.x + IK 分词器实现多字段 Bool 组合检索、关键词高亮与动态分页；通过 RabbitMQ 异步增量同步索引，**借助 Spring 事务同步器（`TransactionSynchronizationManager`）确保 MySQL 事务提交后再发 MQ**，杜绝并发读脏数据；配合 XXL-JOB 定时全量兜底同步。
- **全链路安全防护与 Spring AI 智能伴读**：配置 XSS 过滤器净化入参，结合 AOP 切面与白名单机制校验动态排序字段，根绝 SQL 注入；接入 Spring AI (`ChatClient`) + Function Calling + ES 向量检索打造智能书童，支持自然语言语义找书与 Unicode 安全流式响应。

---

### 2. 万佳商城微服务电商系统（HMall）
**2026.03 – 2026.05** ｜ Spring Boot 3 · Spring Cloud Alibaba · Nacos · Gateway · OpenFeign · Sentinel · Seata · MyBatis-Plus · MySQL · Redis · RabbitMQ

基于 Spring Cloud Alibaba 构建的 B2C 微服务电商系统，划分为商品、购物车、用户、交易、支付 5 个微服务与统一网关，实现服务注册发现、网关动态路由、跨服务通信、分布式事务与异步解耦。

- **服务拆分与基础设施构建**：参与单体到微服务架构演进，明确划分 5 大微服务业务边界；基于 Nacos 实现全服务动态注册发现与多环境配置集中化管理，提升系统可用性与扩展性。
- **网关统一鉴权与动态路由热更新**：基于 Spring Cloud Gateway 全局过滤器实现 JWT 登录校验、白名单放行与用户身份 Header 透传；编写 Nacos 配置监听器（`DynamicRoutLoader`）实现路由规则动态热加载，规则变更无需重启网关。
- **通用组件沉淀与工程化**：抽取 `hm-common` 通用基础设施组件，利用 `spring.factories` 自动装配 MVC、MyBatis-Plus、RabbitMQ 序列化等基础配置；配置拦截器还原网关透传的 `UserContext`，统一全局异常与统一响应体，消除 40%+ 跨服务冗余代码。
- **跨服务 RPC 与高可用容错治理**：基于 OpenFeign 实现跨微服务调用，配置 Feign 请求拦截器透传用户上下文；集成 Sentinel 并实现 `FallbackFactory` 熔断降级逻辑，有效阻断依赖故障传播，防止微服务雪崩。
- **下单主链路与 Seata 分布式事务一致性**：负责订单创建核心流程（订单入库、清空购物车、扣减库存）；引入 Seata AT 模式（`@GlobalTransactional`）保障跨服务数据最终一致性；库存扣减采用行级原子更新 SQL 语句（`stock >= #{num}`）杜绝高并发超卖。
- **支付幂等控制与 RabbitMQ 异步解耦**：实现支付单幂等校验与基于状态机乐观锁的支付状态流转；支付成功后通过 RabbitMQ 异步通知交易服务修改订单状态，开启 Publisher Confirm 与手动 ACK 保障消息可靠投递，实现交易与支付链路的高性能解耦。

---

## 🌟 自我评价

- **基础扎实，执行力强**：具备扎实的 Java 后端开发功底，能独立完成业务模块从需求分析、接口设计、编码到前后端联调的闭环；具备良好的工程规范与分层设计意识，严格遵守代码规约。
- **钻研技术，善于协作**：对分布式事务、多级缓存、消息队列、分布式锁等底层机制具备强烈的钻研意愿；能快速熟悉并融入项目代码库，善于沟通与复盘总结，具备良好的团队协同与抗压能力。