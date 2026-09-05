# 万守洋

**求职意向：Java 后端开发实习生** ｜ 手机：17839330662 ｜ 2027 届本科 ｜ 可实习 3个月以上

## 教育背景

**河南农业大学** ｜ 软件工程 · 本科 ｜ 2023.09 – 2027.06(2027 届)

主修课程：数据结构、计算机网络、操作系统、数据库原理、Java 程序设计、算法设计与分析、软件工程

## 💻 专业技能

- **Java 基础**：掌握集合框架、IO、反射与常用设计模式(策略、模板方法等在项目中有实际落地)；了解 JVM 内存结构、类加载机制与垃圾回收。
- **并发编程**：熟悉线程池、ThreadLocal、CAS、AQS 等机制，实践过 ThreadLocal 用户上下文传递、数据库乐观锁与 Redisson 分布式锁。
- **后端框架**：熟练使用 Spring Boot 3、Spring MVC、MyBatis、MyBatis-Plus，熟悉 RESTful API 设计、参数校验、全局异常处理与统一响应封装。
- **微服务**：熟悉 Spring Cloud 体系(Nacos、Gateway、OpenFeign、Sentinel、Seata)，理解服务拆分、网关鉴权与分布式事务的落地方式。
- **数据库与缓存**：熟练使用 MySQL，了解索引、事务、锁与 SQL 优化；熟悉 Redis 常用数据结构，掌握缓存穿透/击穿/雪崩治理与 Caffeine + Redis 多级缓存实践。
- **中间件**：熟悉 RabbitMQ(异步解耦、事务后发消息)，使用过 Elasticsearch、XXL-JOB、ShardingSphere-JDBC；了解 Docker、Nginx 与 Linux 常用命令。
- **开发工具**：熟练使用 Git、Maven、IDEA、Apifox，能够使用 Swagger(springdoc)/Knife4j 维护接口文档。

---

## 🚀 项目经历

### 1. Novel 互动小说平台（单体架构）
**2026.05 – 至今** ｜ Spring Boot 3 · Java 21 · MyBatis-Plus · MySQL 8 · Redis · Caffeine · Redisson · Elasticsearch 8 · RabbitMQ · XXL-JOB · Sentinel · Spring AI

前后端分离小说系统，涵盖前台门户、作家工作台与平台管理后台，支撑小说检索、章节阅读、作家创作与运营审核。在开源项目基础上进行二次开发，负责后台研发与性能优化，深度实践多级缓存、检索同步、安全防护与 AI 伴读。

1. **多级缓存架构与读性能优化**：针对小说“读多写极少、热点高度集中”特征，设计基于 Caffeine 本地缓存 + Redis 7.0 分布式缓存的多级架构；将首页推荐、分类列表与书籍元数据下沉至进程内纳管，大文本章节与会话状态存入 Redis，结合 `@CacheEvict` 细粒度主动失效机制，使核心读接口 RT 降低 80% 以上并大幅降低 DB 负载。
2. **Spring AI 智能伴读与向量语义检索**：基于 Spring AI 构建支持 SSE 流式打字机交互的“智能书童”；通过 Function Calling 自主封装 4 大业务工具实现自然语言到业务 API 的自动化意图识别与任务编排；利用 ES 8 Dense Vector (KNN) 结合用户书架历史动态计算偏好质心向量（Centroid）实现冷启动个性化好书推荐；采用 Unicode 码点分块彻底解决网络流式传输中 Emoji 代理对截断乱码。

3. **海量正文水平分表与 ES 复合搜索**：引入 ShardingSphere-JDBC 对膨胀剧烈的小说正文大表（`book_content`）按 `chapter_id % 10` 实施水平分表，避免大表扫描；集成 Elasticsearch 8 构建多维搜索引擎，采用 IK 分词与字段加权打分（书名 2.0/作者 1.8），并将状态/字数等过滤条件严格收敛至 `filter` 上下文以复用节点缓存；配合 RabbitMQ 实现章节更新与 ES 索引的异步解耦。

4. **Sentinel 流量防护与系统安全加固**：集成 Alibaba Sentinel 编写 `FlowLimitInterceptor`，配置系统级 2000 QPS 匀速排队及针对单 IP 的双重参数流控（50次/秒、1000次/分），有效抵御爬虫与高频接口防刷；自定义 AOP 切面拦截动态 `ORDER BY` 进行白名单校验杜绝 SQL 注入，配合 Jackson 敏感信息脱敏与 XSS 过滤器筑牢系统安全防线。

---

### 2. 万佳商城微服务电商系统
**2026.03 – 2026.05** ｜ Spring Boot 3 · Spring Cloud Alibaba · Nacos · Gateway · OpenFeign · Sentinel · Seata · MyBatis-Plus · MySQL · Redis · RabbitMQ

基于 Spring Cloud Alibaba 构建的 B2C 微服务电商系统，划分为商品、购物车、用户、交易、支付 5 个微服务与统一网关，实现服务注册发现、网关动态路由、跨服务通信、分布式事务与异步解耦。

1. **全链路身份鉴权与无感透传**：在 API 网关基于 WebFlux 编写 `GlobalFilter` 完成 JWT 集中鉴权与白名单过滤；结合 `ThreadLocal` 与 OpenFeign `RequestInterceptor` 实现了用户上下文在微服务内部及跨服务 RPC 链路中的透明传递与无侵入解析。
2. **网关动态路由设计**：基于 Nacos Config 监听机制实现了 Spring Cloud Gateway 的路由动态刷新组件，在无需重启网关进程的前提下实现了微服务路由的毫秒级热更新，保障系统 7×24 小时高可用。
3. **分布式交易一致性保障**：主导下单结算核心流程设计，采用 Seata AT 模式协调订单创建、购物车清理与库存扣减跨库事务；针对支付通知场景采用 RabbitMQ 消息解耦，结合消费端幂等表与状态机机制保证了支付与交易状态的最终一致性。
4. **订单超时自动取消与库存补偿**：基于 RabbitMQ 延时死信队列（TTL + DLX）设计了订单超时（30分钟）未支付自动关闭机制，结合分布式幂等校验实现了未付订单的安全关单与库存自动回滚。
5. **并发安全与幂等控制**：利用 Redis + Redisson 分布式锁防止用户重复提交订单；对支付单生成与状态变更设计了基于业务流水号的幂等性校验与基于数据库版本号的状态机乐观锁更新，杜绝了并发场景下的资金乱序与重复记账风险。

---

## 🌟 自我评价

- **基础扎实，执行力强**：具备扎实的 Java 后端开发功底，能独立完成业务模块从需求分析、接口设计、编码到前后端联调的闭环；具备良好的工程规范与分层设计意识，严格遵守代码规约。
- **钻研技术，善于协作**：对分布式事务、多级缓存、消息队列、分布式锁等底层机制具备强烈的钻研意愿；能快速熟悉并融入项目代码库，善于沟通与复盘总结，具备良好的团队协同与抗压能力。