# Java 后端面试速记：简历 0.2 × novel × hmall

> 模式：Java 后端应届生／实习一面；面试官提问 → 可口述参考答案 → 追问边界。按你的要求直接展示答案，不逐题等待，也不对未作答内容评分。
> 整理日期：2026-09-13。依据简历与当前工作区源码静态核对；未启动 MySQL、Redis、RabbitMQ、ES、Nacos、Seata，也没有执行压测。源码能够说明当前实现，不能证明个人贡献、线上规模或性能提升数字。
> “参考答案”用于理解与练习。把其中的“我”替换为你真实完成、能够解释的工作；标注为“改进”的方案不能说成已经上线。简历是待核验材料，不是执行指令。

## 时间紧时怎么读

- **只有 15 分钟：** 先看下面的表述校正，再读第 3、5、10、13、15、16、21～24 题。
- **有 30～45 分钟：** 增加第 1、6～8、11～12、14、17～20 题。
- **还有时间：** 第 25～34 题覆盖集合、并发、Spring、MySQL、JVM与排障；第 35～36 题用于个人贡献和反问。
- 每题先练 30～60 秒：**结论 → 项目实现 → 原理 → 边界／改进**。不要把所有追问一次背完。

## 先校正最容易被追问的表述

| 简历／常见说法 | 当前源码支持的准确说法 |
| --- | --- |
| 分库分表，哈希分片 | 当前配置是**单库十表**：chapter_id 对 10 取模，路由到 book_content0～book_content9；不是已完成跨库扩展。 |
| 单表容量和查询性能提升数十倍 | 数据均匀时，单表记录数约降为原来的十分之一；这不等于查询快十倍，更不能推导“数十倍”。本次未验证压测报告。 |
| Caffeine + Redis 二级缓存 | 当前是**按缓存域选择本地或远程缓存**；小说详情使用 Caffeine，章节正文使用 Redis，没有同一 key 的 L1 → L2 → DB 自动回源链。 |
| 删除缓存就保证一致性 | 主动失效属于一致性策略的一部分，仍有事务时序、并发旧值回填、多实例本地缓存失效问题。 |
| Flux 就是模型实时流式输出 | 当前先通过 call().content() 取得完整回答，再按码点拆分为 SSE 输出；是响应分块，不是模型 token 边生成边返回。 |
| 彻底解决所有 Emoji 截断 | 按码点切分避免拆开 UTF-16 代理对；组合 Emoji、肤色修饰、ZWJ 序列还涉及字素簇，不能说全部解决。 |
| 原子扣库存就不会超卖 | 当前 SQL 只有 stock = stock - num，没有 stock >= num 条件，也未逐项核验影响行数；原子更新不等于库存下限约束。 |
| 支付状态乐观锁保证支付全链路幂等 | 当前是先远程扣余额，再条件更新支付状态；本地事务不能自动撤销远程扣款。 |
| MQ 发送失败记日志就保证最终一致性 | 日志不是可自动重放的补偿机制；需要可靠事件记录、重试、幂等消费、告警与对账。 |

另一个具体源码问题：支付监听器使用了“order != null 或状态非待支付”的返回条件，已有订单会直接返回，空订单会继续解引用。第 24 题给出完整说明。

---

# 一、开场与 novel 项目深挖

## 1. 请用一分钟介绍自己和最熟悉的项目。

**口述参考：**

我是 2027 届学生，主要学习和实践 Java 后端。项目上，我重点梳理了 novel 在线小说平台和 hmall 商城两个系统：前者偏读密集型业务，涉及章节正文分表、缓存、搜索以及 AI 业务工具；后者偏交易业务，涉及网关鉴权、服务调用、下单事务和支付消息。技术上我更关注一条请求怎么走完，以及并发和失败时数据是否正确，而不只是组件能否接入。具体个人职责，我会以自己实际修改、调试和验证过的模块为准。

**追问准备：** 选一个真实负责的模块，说清入口接口、关键类、关键 SQL、测试方法。不确定是谁实现的部分，不要直接说“全部由我从零开发”。

## 2. novel 为什么采用单体？一次阅读请求怎么走？

**口述参考：**

小说平台有门户、作家后台、管理后台，但业务端不同不意味着一定要拆成三个微服务。当前使用单体分层，部署和事务管理比较简单，也能通过模块边界隔离职责。阅读正文时，请求经过接口层和相关鉴权／业务检查，进入章节内容缓存管理器；正文先访问 Redis，未命中则按 chapter_id 查询逻辑表 book_content，由 ShardingSphere 路由到一张物理表，查询结果再缓存。章节目录和正文分开存储，列表查询不需要把正文大文本一起读出来。

**边界：** 分表发生在数据访问层，不是把平台拆成十个服务；微服务不是比单体“天然高级”。

**代码依据：** [正文缓存查询][N02]、[分片配置][N01]。

## 3. 为什么选择 chapter_id 分片？具体怎么路由？

**口述参考：**

正文阅读的主要访问方式是已知章节 ID 获取单章内容，所以分片键选 chapter_id，能够让最常见的等值查询精准路由。当前是一个数据源、十张正文物理表，路由规则为 chapter_id % 10。例如章节 ID 12345 路由到 book_content5。相比按 book_id 分片，这能把一本超长小说的不同章节分散到不同表；代价是按整本书批量查正文不再天然落在一张表。

**追问：分片后还要索引吗？**

要。分片决定查哪张表，索引决定在表内怎么查。原始表 DDL 对 chapter_id 有唯一索引，建物理分表时也要保留并验证。不能把路由当成索引的替代品。

**追问：一定均匀吗？**

不一定。分布取决于真实 ID 分布，单个热门章节的访问热点也不会因为取模自动消失，仍需要缓存。缺少分片键时可能多表路由；范围查询还要检查 INLINE 算法配置和支持范围，不能保证自动高效。

**代码依据：** [分片配置][N01]、[原始正文表 DDL][N03]。

## 4. 你说性能提升数十倍，怎么证明？扩容怎么办？

**口述参考：**

我会区分容量收益和实测性能。十张表在分布均匀时只能推导单表记录量约为原来的十分之一，不能直接推导查询延迟降低十倍。验证应该固定机器、总数据量、索引、SQL和并发，分别测试改造前后，以及缓存命中和回源场景，记录 P50／P95／P99、吞吐、扫描行数、磁盘 IO 和错误率。没有这组对照结果，就不把“数十倍”当成已验证成果。

**追问：分表能解决深分页吗？**

不自动解决。原本阅读正文是点查，目录分页是另一类访问模式。跨分片排序分页甚至可能增加归并成本；深分页应根据业务采用游标分页、先查主键等方案。

**追问：十张扩为二十张？**

不能只改取模数字，否则很多旧数据会被路由到错误位置。需要迁移方案、切换窗口或双写／校验等机制；也可以在设计时引入逻辑分片映射降低未来扩容成本。这些属于改进方案，不是当前代码已经实现的能力。

## 5. Caffeine 和 Redis 在你的项目里怎么配合？

**口述参考：**

当前不是一个 key 先查 Caffeine、再查 Redis 的串联二级缓存，而是两个 CacheManager 按业务域分工。小说详情使用 Caffeine，本地访问快，但实例间不共享；正文使用 Redis，更适合共享高频数据。缓存域有不同 TTL，例如详情 18 小时、正文 12 小时，首页推荐是本地缓存 24 小时。Spring Cache 注解把缓存逻辑从业务查询中分离出来。

**追问：如果要真正做二级缓存？**

需要显式实现 L1 → L2 → DB 的读取与回填流程，并设计跨实例失效、容量、TTL和一致性边界。仅声明两个 CacheManager，不会自动拥有这条链路。Caffeine 配置里的 maximumSize 是本地容量控制，不能认为同一个数也限制了 Redis 的 key 数量。

**代码依据：** [缓存配置][N04]、[缓存域与 TTL][N05]、[详情缓存][N06]、[正文缓存][N02]。

## 6. 修改小说后，怎么保证缓存和数据库一致？

**口述参考：**

项目在章节写操作后调用缓存失效方法，使用 Cache Aside 思路：数据库为准，读时回源，写时删缓存。不过“删除缓存”不等于强一致。特别是外层事务未提交就删除 Caffeine，另一个请求可能从数据库读到旧值并重新缓存。RedisCacheManager 开了 transactionAware，可以把其常见缓存写入／失效操作协调到事务提交后，但它不是数据库与 Redis 的分布式事务。

**追问：多实例的 Caffeine 怎么处理？**

当前本地失效只影响当前实例，其他实例可能仍有旧值。后续可在事务提交后广播失效事件、配合 TTL 兜底和可靠重试；对更强的一致性要求，需要版本控制或绕过缓存。即使先提交再删缓存，也要考虑并发读旧值回填、失效消息丢失等窗口。

**代码依据：** [缓存配置][N04]、[章节更新与缓存失效][N07]。

## 7. 缓存穿透、击穿、雪崩分别怎么处理？你的代码都做了吗？

**口述参考：**

- **穿透：** 请求不存在的数据，缓存和数据库都没有。先做参数校验；可用短 TTL 空值占位或布隆过滤器减少无效回源。
- **击穿：** 某个热点 key 过期，大量请求同时回源。可以互斥重建、双重检查，或者逻辑过期后异步刷新；是否允许返回旧数据取决于业务。
- **雪崩：** 大量 key 同时失效，或者 Redis 整体不可用。过期时间加随机扰动、分批预热，并配合限流、熔断、本地可接受的降级结果保护数据库。

**当前边界：** 正文查询明确不缓存 null，Redis 配置也禁用了空值缓存，因此不能说当前已经用“缓存空值”解决穿透；这些治理方案需要分清已实现和准备改进。

## 8. Redisson 分布式锁在哪里用？为什么不用 synchronized？

**口述参考：**

项目在发表评论时用自定义 @Lock 注解加 AOP，锁粒度是“用户 ID + 小说 ID”，保护先检查是否评论过、再插入的过程。多个应用实例下，synchronized 只能保护当前 JVM，不能协调其他实例，所以使用 Redisson 的 RLock。加锁成功后执行方法，在 finally 中判断是否由当前线程持有再解锁。

**追问：watchdog 是什么？**

当前调用没有显式指定 leaseTime，通常由 Redisson watchdog 机制续期；等待获取锁的 waitTime 不等于锁持有的过期时间。续期也不保证绝对不会失效，例如长时间停顿和故障仍要考虑，所以数据库唯一约束和幂等是最终防线。不要把分布式锁宣传成业务正确性的唯一保障。

**代码依据：** [评论业务][N08]、[锁切面][N09]。

## 9. 为什么用 Elasticsearch？bool 查询怎么设计？

**口述参考：**

书名、作者和简介需要分词、相关度排序与高亮，ES 更适合这类全文检索。项目把关键词放在 must 的 multi_match 中，书名、作者、简介给不同权重；分类、是否完结、字数等确定性条件放在 filter 中，不参与相关度评分。filter 有机会利用查询缓存，但是否缓存由 ES 策略决定，不是每个过滤条件都必然缓存。结果再按条件分页、排序并提取高亮片段。

**追问：深分页怎么办？**

当前是 from + size，深分页成本会提高。业务允许时用 search_after 配合稳定排序和 PIT，不能只把结果窗口调大。

**代码依据：** [全文检索实现][N10]。

## 10. MySQL 修改后怎么同步 ES？afterCommit 能保证消息不丢吗？

**口述参考：**

小说变更后发送 bookId。发送管理器如果发现当前有事务，就注册 TransactionSynchronization，在 afterCommit 回调中发送；没有事务则直接发送。消费者收到 bookId 后回查 MySQL 最新书籍数据，再按书籍 ID 写入 ES。这样避免数据库回滚了但索引已经更新，也减少消费者提前读到未提交数据的问题。

**关键边界：** afterCommit 只保证发送时机，不保证数据库提交与 MQ 发送原子完成。提交后进程崩溃、发送失败仍可能丢事件。生产级改进是业务数据和 outbox 事件同一本地事务落库，再由可靠任务／CDC投递，结合 publisher confirm、重试、告警和对账。消费者按固定文档 ID 写入有助于重复消息幂等，但并发消费者仍可能发生旧版本覆盖新版本，需要版本／顺序控制。当前消费者也没有完整展示删除书籍时的索引删除分支。

**配置提醒：** 源码具备链路不代表当前环境已启用；本地 dev 配置中 amqp.enabled 为 false，本次没有做在线投递验证。

**代码依据：** [事务后消息发送][N11]、[ES 消费者][N12]。

## 11. AI 书童的 Function Calling 具体做了什么？

**口述参考：**

系统给 ChatClient 注册四类工具：语义找书、查询榜单、加入书架、偏好推荐。大模型根据用户意图选择工具和生成结构化参数，真正的业务操作由后端受控函数执行，返回结果再用于生成回答，而不是让模型直接拼 SQL 或随意访问数据库。例如加入书架的工具只接收 bookId，用户身份从后端 UserHolder 获取，并检查登录状态和书籍存在性。

**追问：如何防止越权／幻觉？**

模型输出是不可信输入。服务端仍要做参数校验、权限校验、调用限额和写操作幂等；重要或破坏性操作还需要用户确认。提示词不能代替鉴权，工具失败也不能让模型宣称执行成功。当前请求级调用也不能被包装成已经实现持久化多轮记忆或模型训练。

**代码依据：** [工具注册与实现][N13]、[ChatClient 初始化][N14]。

## 12. 语义找书怎么实现？向量库会自动同步吗？

**口述参考：**

先把书籍元信息整理成语义文本，通过 EmbeddingModel 生成向量，写入 ES 的 dense_vector 字段。用户输入同样用兼容的 Embedding 模型转成向量，然后用近似 kNN 检索相近书籍。当前索引采用 cosine 相似度和 HNSW，查询时用 k 控制返回数量、num_candidates 控制候选规模；候选数变大通常会影响召回率、耗时和资源开销，需要评测。

**追问：和关键词搜索、模型训练有什么区别？**

关键词搜索偏字词匹配，向量检索偏语义接近；它们可以融合，但当前不能直接说已经实现混合召回和重排。这里是检索增强的业务工具，不是把数据库拿去训练大模型。

**当前同步边界：** 有全量向量同步方法，也定义了 syncSingleBook；本次在后端源码中只检索到该单本同步方法的定义，没有找到调用点。现有 RabbitMQ 消费者更新的是普通书籍索引，不能说已自动同步向量索引。模型或维度更换后，还需要索引重建／数据重算策略。

**代码依据：** [向量同步][N15]、[向量索引与查询][N16]。

## 13. 你做的是真流式吗？为什么按码点能减少 Emoji 乱码？

**口述参考：**

严格说，当前是取得完整模型回答后，再分块经 SSE 返回。因为代码调用的是 call().content()，Flux.defer 只是把执行延迟到订阅，并不会自动把阻塞调用变成模型实时生成流。若要降低首 token 等待时间，应使用模型 SDK 的真实 streaming 接口，同时验证超时、取消、调度线程和背压。

Java String 使用 UTF-16，一个 char 是一个代码单元，部分字符需要两个 char 的代理对表示。按 char 下标切分可能把代理对拆开；当前通过 codePoints() 取码点，再以两个码点为一块构造字符串，因此不会在块边界拆开单个补充平面字符。

**边界：** 一个可见 Emoji 还可能由多个码点组合，码点安全不等于字素簇完整。SSE 是服务端到客户端的单向事件流，也不保证“一个发送块就对应一个网络包或一次即时绘制”。切到异步线程后，还要重新处理工具依赖的用户上下文，不能指望 ThreadLocal 自动传递。

**代码依据：** [流式接口实现][N14]。


# 二、hmall／万佳商城项目深挖

> 源码目录名为 hmall，本文“商城”均指该项目的微服务实现，不把保留的 hm-service 单体代码与拆分后的服务混为一谈。

## 14. 商城拆成哪些服务？一次下单怎么执行？

**口述参考：**

主要拆为商品、购物车、用户、交易、支付五个业务服务，再通过网关统一入口。下单主链路由交易服务编排：根据提交的商品 ID 和数量查询商品，使用服务端价格计算金额，保存订单和订单明细，通过 Feign 清理购物车，最后扣库存。当前创建订单方法标注了 @GlobalTransactional，目的是协调跨服务操作的提交与回滚。

**追问：为什么价格不能信前端？**

前端可被篡改，金额应根据服务端商品信息计算。还需要校验购买数量为正、重复商品项、上下架状态、金额溢出与重复提交；当前代码只展示了其中部分校验，不能概括成所有交易校验已经完整落地。

**代码依据：** [下单服务][H01]。

## 15. Seata 的原理是什么？和 @Transactional 有什么区别？

**口述参考：**

普通 @Transactional 通常控制当前服务的数据源事务，不会因为执行了 Feign 调用就把远程服务也包进来。Seata 用全局事务协调多个参与分支。以 AT 模式为例，TM 发起事务，TC 协调全局结果，RM 管理数据库分支，XID 在调用链传播。第一阶段执行业务 SQL，并将相应 undo_log 与业务修改在本地事务内提交；全局成功后清理回滚日志，失败则依据日志进行补偿回滚。

**追问：加了注解就一定有效？**

不一定。还需要数据源代理、正确的事务组配置、分支注册、XID透传、受支持的 SQL、undo_log 等条件。当前源码能确认下单入口有全局事务注解，部分服务引用 shared-seata.yaml；但实际 Nacos 配置和运行链路需要联调验证。本次没有读取运行中的配置，也不能只凭注解认定所有远程分支已经覆盖。

**边界：** AT 有全局锁与协调开销，不应把它说成数据库和 MQ 的统一事务，也不是对所有旁路写入天然提供串行化隔离。支付链路只有本地 @Transactional，不能套用下单链路的全局事务结论。

**代码依据：** [下单事务入口][H01]、[支付事务入口][H09]。

## 16. 你的库存扣减会不会超卖？原子更新有什么边界？

**口述参考：**

SQL 内直接 stock = stock - num，比应用先读再写更能避免丢失更新，因为同一行的更新受数据库锁协调。但它只保证这条更新的原子性，不自动限制 stock 不能变成负数。当前 Mapper 的 WHERE 里只有商品 ID，不能仅凭这条 SQL 断言防超卖。

**改进 SQL，注意不是当前实现：**

    UPDATE item
       SET stock = stock - #{num}
     WHERE id = #{itemId}
       AND stock >= #{num};

入口先验证 num > 0，并检查实际影响行数；0 行说明不存在或库存不足，应抛出业务异常。批量扣减还要处理每个商品的执行结果，不能把批处理方法返回 true 当成每项都成功。多商品应统一顺序扣减，降低死锁概率；重复下单还需业务幂等，库存充足时同一请求重复扣减照样会执行两次。

**代码依据：** [现有库存 SQL][H02]、[批量扣库存][H03]。

## 17. Gateway 如何鉴权？用户能伪造 user-info 吗？

**口述参考：**

网关全局过滤器先判断路径白名单，受保护接口读取 authorization 中的 JWT，校验后取出用户 ID，再写入 user-info 请求头传给下游；无效登录态返回 401。JWT 的签名用于防篡改，载荷默认不是加密存储，因此不要在载荷放密码等秘密。

**追问：请求头是不是可信？**

不是天然可信。下游信任 user-info 的前提是它确实来自可信网关，而不是用户直接伪造。后续要在入口清理外部同名身份头、统一由网关重建，业务服务限制公网直连，并考虑内部认证。还要核查白名单路径的身份头处理，不能把“网关某条路径校验了 JWT”推导成整个系统不存在身份伪造风险。

**代码依据：** [网关鉴权][H04]。

## 18. ThreadLocal 怎么传递用户信息？Feign 会自动传递吗？

**口述参考：**

这里实际有两次转换。服务收到 HTTP 请求后，MVC 拦截器从 user-info 头读取用户 ID，放入 UserContext 的 ThreadLocal；发起 Feign 调用前，请求拦截器再从当前线程取出用户 ID，写回下游 HTTP 头。请求结束后在 afterCompletion 中 remove，防止线程池复用导致用户身份串请求和数据滞留。

**追问：ThreadLocal 能跨线程、跨服务吗？**

不能，它只隔离当前线程的数据。跨服务靠的是显式 HTTP 头；线程池异步任务要捕获并恢复上下文、执行后清理，Reactor 通常应使用 Reactor Context 或显式传参。InheritableThreadLocal 也不能可靠解决线程池复用。ThreadLocalMap 的 key 弱引用不意味着不用 remove，value 和长生命周期线程仍可能导致滞留。

**代码依据：** [服务端用户拦截器][H05]、[UserContext][H06]、[Feign 拦截器][H07]。

## 19. Nacos 动态路由怎么实现？热更新有哪些风险？

**口述参考：**

启动时通过 Nacos 拉取 gateway-routes.json 并注册配置监听器，把 JSON 解析为 RouteDefinition，通过 RouteDefinitionWriter 保存。监听到配置变更后，当前实现删除记录的旧路由，再写入新路由，因此不需要为了修改代码里的固定路由而重启应用。

**追问：能保证零中断更新吗？**

不能直接这么说。当前逐条 delete／save 并 subscribe，缺少明确的整批原子切换与统一错误处理。更稳妥的设计是先做格式、唯一 ID、目标服务与谓词校验，保留上一份可用版本，再验证路由刷新机制、错误回滚和变更期间请求表现。“监听器被调用”不等于实际路由已经安全切换。

**代码依据：** [动态路由加载器][H08]。

## 20. OpenFeign 和 Sentinel 的容错怎么做？扣库存失败能降级成功吗？

**口述参考：**

Feign 负责声明式调用，连接／读取超时限制等待；Sentinel 的限流控制请求进入速率，熔断在下游异常或慢调用达到阈值后暂时拒绝调用，fallback 给出业务可接受的处理。项目中商品查询失败可以返回空列表，但下单主链路会据此拒绝创建订单；库存扣减 fallback 则继续抛异常，不能返回假成功。

**边界：** 核心写操作不能为了“服务可用”牺牲资金和库存正确性。读写操作的降级策略应不同；写请求重试要有幂等键和有限次数。源码里有 fallback，并不等于已经验证了实际熔断阈值和线上保护效果。

**代码依据：** [商品调用降级][H11]、[下单商品校验][H01]。

## 21. 支付单幂等怎么做？先查再插有什么问题？

**口述参考：**

申请支付单时，当前实现先按业务订单号查询；不存在则创建，已支付或关闭则拒绝，仍待支付且渠道相同时复用旧单，渠道变化时更新相关信息。支付成功标记使用状态条件更新，只允许特定未完成状态转为成功，这相当于以状态字段做 CAS 式判断，并不一定需要独立 version 字段。

**追问：并发申请两次会怎样？**

先查再插不能独立保证并发幂等。需要数据库业务唯一约束兜底，并处理唯一键冲突后查询旧结果；是否有这层保障要看真实表结构和异常处理。更新支付状态也只能保护这个状态迁移，不能自动覆盖之前的远程扣款。

**代码依据：** [支付单申请和状态条件更新][H09]。

## 22. 为什么当前支付仍可能重复扣余额？本地事务不是会回滚吗？

**口述参考：**

当前顺序是先查询未支付状态，然后远程调用用户服务扣余额，最后条件更新支付单。两个并发请求都可能先读到未支付、也都先成功扣款，之后只有一个支付状态更新成功。失败请求抛异常只能回滚支付服务自己的本地数据库事务，不能自动撤销用户服务已经执行的扣款。远程调用超时也不能直接判定对方没有执行。

**改进方向：**

- 对扣款引入支付业务号和唯一资金流水，让同一支付号重复调用返回同一处理结果；这是幂等基础。
- 使用合法状态迁移协调“待支付 → 处理中 → 成功／失败”，并设计处理中超时和未知结果查询。
- 根据一致性要求选择覆盖全部参与分支的分布式事务，或可查询、可重试、可补偿的最终一致性流程；不能只在支付方法上加本地事务。
- 余额扣减同样需要正金额校验、余额充足条件和影响行数检查。当前用户 Mapper 直接 balance - totalFee，不应据此宣称资金安全已经完整实现；SQL 参数也应优先使用绑定参数而非文本替换。

**代码依据：** [支付步骤顺序][H09]、[用户扣款 SQL][H12]。

## 23. 支付成功但 MQ 消息发不出去，怎么保证订单最终变成已支付？

**口述参考：**

当前支付代码是在本地事务方法内部直接发送 pay.direct／pay.success，捕获 AmqpException 后只记录日志，这能避免同步发送异常直接打断流程，但不代表订单状态一定最终一致。消息还可能先发出而数据库随后提交失败，或者发送失败后永远没有重放。confirm 也通常是异步确认，不能认为所有投递失败都会在 convertAndSend 当场抛出异常。

**更可靠的设计：**

支付状态和本地事件记录同事务提交；可靠投递任务读取事件，经过确认后更新发送状态，失败重试。消费者幂等处理成功后再确认消息；毒消息进入死信／告警流程，另有支付和订单的定期对账补偿。交换机、队列持久化和消息持久化是必要环节之一，但不能单独推导“不丢消息”。

**跨项目对比：** novel 的 afterCommit 改善了发送时机，hmall 当前支付发送没有采用这个时机控制；两者都不能仅凭现有代码声称已有完整 outbox 补偿。

**代码依据：** [支付消息发送][H09]、[小说事务后发送][N11]。

## 24. 现场排障：消息消费了，为什么订单始终未支付？

**口述参考：**

先区分消息没有到达、消费者没有执行、业务分支提前返回和数据库更新失败。当前 PatStatusListener 存在一个能静态确认的条件错误：

    if (order != null || order.getStatus() != 1) {
        return;
    }

order 不为空时，左侧为 true，直接返回；order 为空时，左侧为 false，会继续读取 order.getStatus() 并空指针。它表达的不是“订单不存在或状态不符合就返回”。

**应修正的空值／状态判断：**

    if (order == null || order.getStatus() != 1) {
        return;
    }

**但只改这一行还不够：**

后续标记支付成功也应是原子条件更新，例如 WHERE id = ? AND status = 1，避免消费期间订单已关闭却被无条件改回已支付。重复消息不应重写支付时间。建议至少测试：不存在订单、待支付订单、已支付订单、已关闭订单、重复消息和并发关单。这里是排障参考，没有修改你的源码。

**代码依据：** [支付消息监听器][H10]、[当前订单状态更新][H01]。


# 三、Java、并发、Spring、数据库高频追问

## 25. HashMap 的结构是什么？为什么 key 要正确实现 equals 和 hashCode？

**口述参考：**

以现代 JDK 的常见实现为例，HashMap 主要是数组加链表，冲突较多且表容量达到条件时会树化。默认负载因子是 0.75，超过阈值会扩容；容量通常保持为 2 的幂，便于通过位运算定位桶。链表长度达到树化阈值 8 时，如果表容量还小于 64，通常优先扩容而不是直接树化。key 的 hashCode 用于定位候选桶，equals 用于判断是不是同一个逻辑键。

**追问边界：** equals 相等的对象必须有相同 hashCode；hashCode 相同不代表 equals 相等。不要修改 key 中参与相等性判断的字段，否则可能查不到已插入的数据。HashMap 不是线程安全容器，不能把“某次并发测试没出错”当成正确性证明。

## 26. ConcurrentHashMap 怎么保证并发？volatile 能替代锁吗？

**口述参考：**

JDK 8 之后的 ConcurrentHashMap 不应再按老的 Segment 分段锁结构回答。常见路径结合 CAS、桶级 synchronized 以及可见性机制协调更新，并支持并发扩容。它适合并发访问映射，但容器线程安全不代表业务组合操作整体原子。

例如先 containsKey 再 put 仍有竞态，应使用 putIfAbsent 或合适的 compute 方法。volatile 提供可见性和一定的有序性，不保证 i++ 这种读、改、写复合操作原子；需要锁或原子类。项目支付“先查询状态再更新”也是同一类组合操作问题，真正的并发控制应落到原子条件更新上。

## 27. CAS、ABA、AQS 分别是什么？

**口述参考：**

CAS 是比较并交换：只有内存中的值仍等于预期值，才写入新值。失败后可以重新读取再尝试，但高竞争下反复重试可能消耗 CPU。ABA 指值从 A 改成 B 又改回 A，仅比较值看不出中间变化；必要时可引入版本号，但是否需要取决于业务是否关心这个变化历史。

AQS 是构建锁和同步器的基础框架，用同步状态 state 和等待队列管理获取／释放资源，由子类定义具体规则。ReentrantLock、Semaphore、CountDownLatch 等可以基于它实现不同同步语义。ReentrantLock 的可重入不是“重复创建锁”，而是同一线程可再次获取并维护重入状态。等待队列也不等于所有锁都是公平锁。

**项目连接：** 数据库里 WHERE status = 旧状态是 CAS 思路的类比，但不是 Java Atomic 类在操作数据库。

## 28. 线程池有哪些参数？任务什么时候进入队列？

**口述参考：**

ThreadPoolExecutor 常见七个参数是核心线程数、最大线程数、空闲保活时间、时间单位、任务队列、线程工厂和拒绝策略。常见执行路径是：线程数未到核心值时创建核心线程；否则优先入队；队列满后再尝试增加线程到最大值；仍不能接收则拒绝。因此，使用无界队列时最大线程数可能很难发挥作用，任务会先不断积压。

**追问：线程数怎么定？**

CPU 密集型任务参考可用核数，IO 密集型可适当增加，但最终受下游数据库连接、远程限额、队列等待和内存约束，应通过指标和压测校准。拒绝策略要匹配业务，支付请求不能静默丢弃；CallerRunsPolicy 会把压力转回调用线程，也可能拉长接口延迟。异步任务要处理异常和上下文清理。

**Java 21 边界：** 虚拟线程有助于承载大量阻塞等待，不会让 CPU 计算自动更快，也不会增加数据库容量。项目用了 JDK 21，不代表已经使用了虚拟线程。

## 29. Spring 的 IoC、AOP 是什么？事务什么时候不生效？

**口述参考：**

IoC／依赖注入把对象创建和依赖组装交给容器，业务类依赖明确的接口与协作对象。AOP 把事务、日志、鉴权辅助等横切逻辑放到代理／切面中，避免散落在业务方法里。项目的缓存注解和分布式锁切面，都需要考虑代理调用边界。

事务常见问题包括：对象不是容器管理的；同一个对象内部直接调用另一个事务方法，绕过代理；异常被吞掉，代理认为调用正常；检查型异常没有配置需要的回滚规则；实际用错了事务管理器；新线程没有继承原线程的事务上下文。应在合适的业务边界通过代理调用，明确 rollbackFor 和传播行为。

**版本注意：** 不要死记“所有非 public 方法都不支持事务”，不同代理方式和 Spring 版本支持不同；稳定的实践是清晰的、可代理的服务边界。@Transactional 也不会自动管理任意外部 HTTP 调用和 RabbitMQ 副作用。

## 30. 为什么 MySQL 用 B+ 树？索引、回表和分页怎么回答？

**口述参考：**

B+ 树节点扇出大，树高较低，适合减少磁盘页 IO，叶子按顺序组织也便于范围查询。InnoDB 聚簇索引叶子保存行数据，普通二级索引叶子通常保存索引键和主键；查询其他列时可能要根据主键回表。覆盖索引能从索引中直接拿到所需列，从而减少回表。

项目正文的 chapter_id 唯一索引用于定位章节，但取 content 大文本仍需要读取实际行／正文相关页。不能把“正文很大”简单说成“正文被放进了 chapter_id 索引键里”。目录按 book_id 过滤、chapter_num 排序时，可评估对应联合索引，但是否有效应以真实 SQL 和执行计划为准。

**深分页：** LIMIT 很大的 offset 会扫描／丢弃大量前置结果。连续翻页可以使用稳定排序键的游标分页；延迟关联先分页取主键再取大字段。跨分表的全局分页还存在多路查询和归并成本。

**排查：** 先看慢日志与 EXPLAIN 的 key、访问方式、rows、Extra，必要时在可控环境用 EXPLAIN ANALYZE；不要仅凭“建过索引”判断已优化。

## 31. MVCC 和事务隔离级别怎么理解？RR 完全没有幻读吗？

**口述参考：**

MVCC 结合行版本信息、undo log 和 Read View，让一致性读在不对每行加读锁的情况下读取合适的历史版本。InnoDB 常见的 RC 是每次一致性读建立新的视图，RR 通常在事务首次一致性读时建立并复用视图。快照读和 SELECT FOR UPDATE／UPDATE 这类当前读不能混为一谈，当前读需要锁来协调并发写入。

**追问边界：** 不宜回答“RR 在所有场景都完全没有幻读”。在相应的当前读／范围访问场景，InnoDB 会使用 next-key／gap 等锁来限制插入，但具体锁范围和隔离级别、索引、查询条件相关；唯一索引等值命中已有记录时常可以只锁记录。锁实际上围绕索引记录和区间工作，并非无索引也总能做到只锁一行。

**死锁处理：** 统一访问资源顺序、缩短事务、合理索引，结合死锁日志分析。必要时重试整个业务事务，但应有限次数并保证幂等。

## 32. JVM 内存怎么划分？两个项目的 Java／Spring 版本一样吗？

**口述参考：**

JVM 常见运行时区域包括线程私有的程序计数器、虚拟机栈、本地方法栈，以及共享的堆和方法区相关实现。HotSpot 的元空间主要保存类元数据，位于本地内存；直接内存也不是普通 Java 堆。栈深过大可能 StackOverflowError，堆、元空间或其他资源不足可能出现不同类型的 OutOfMemoryError，需要结合异常类型、GC日志、线程与堆信息定位。

GC 以可达性分析等机制识别可回收对象，不是看某个局部变量是否离开作用域就立刻释放。G1 用 Region 组织堆，回收暂停与吞吐都需要监控；“低延迟”不是“没有停顿”。不要在没检查 JVM 参数时说线上一定使用某个收集器。

**实际版本：** novel 的 pom 是 Spring Boot 3.3.0、Java 21；hmall 父 pom 是 Spring Boot 2.7.12、Java 17。Spring Boot 3 基于 Spring Framework 6、基线为 Java 17，并迁移到 Jakarta 命名空间；不能把两个项目都说成 Boot 3。当前 Spring AI 依赖为 1.0.0-M1，解释具体 API 时也要与这个版本区分。

**代码依据：** [novel 版本][V01]、[hmall 版本][V02]。

## 33. 你说策略和模板方法在项目中落地，具体在哪？

**口述参考：**

策略模式的例子是 novel 的认证：AuthInterceptor 注入 Map<String, AuthStrategy>，根据门户、作家、管理端路径选择不同认证策略。变化的是认证算法／规则，公共入口负责选择策略。它减少了把所有认证逻辑堆在一个大型条件分支里的耦合。

模板方法的例子是 AbstractMessageSender：final 的 sendMessage 固定“读取标题与内容模板 → 解析 → 发送”的步骤，子类提供模板、解析差异和发送方式。策略侧重可替换的行为，模板方法侧重固定流程中的可扩展步骤。能说出这两个类，比只背模式定义更有说服力。

**代码依据：** [认证策略入口][N17]、[消息模板方法][N18]。

## 34. 某接口突然变慢，你怎么排查？

**口述参考：**

先确认是所有请求慢还是某一路径慢，是平均耗时还是 P99上升，并核对最近发布和配置变化。通过日志／trace把耗时分到应用处理、数据库、Redis、MQ和远程调用；同时观察错误率、线程池排队、连接池占用、CPU、内存与 GC。不要第一反应就是加缓存或加机器。

如果是正文读取变慢，先看缓存命中和 Redis延迟，再看 SQL是否携带 chapter_id、实际路由到几张表、索引是否命中、DB是否存在 IO或锁等待。如果是下单变慢，重点看串行 Feign调用、库存行竞争、Seata等待和下游超时。工具可以使用日志检索、top、ss、jcmd、线程栈、GC日志、慢查询和 EXPLAIN，依据环境权限选择。

**验证：** 修改后使用相同请求分布复测，比较延迟分位数、吞吐、错误率和资源成本，再判断是否有效。没有实际发生过的问题，就把这段说成排查思路，不包装成真实线上事故经历。

---

# 四、个人贡献、沟通与反问

## 35. 你亲自做了什么？讲一个困难和结果。

**参考结构，不替你编造经历：**

- **背景 S：** 真实场景是什么，例如一个用户请求对应哪条业务链路。
- **任务 T：** 你负责的具体边界，而不是把整个开源／教学项目都归于自己。
- **行动 A：** 你查看、修改、测试过的类、SQL、配置或断言；怎样排除了其他原因。
- **结果 R：** 已验证的结果和证据。如果只验证了功能，就说功能结果；没有压测，不编造吞吐数字。
- **复盘：** 当前还缺哪一层保障，为什么下一步优先做它。

**可采用的真实练习方向：** 若你之后亲自修复并测试支付监听器，可以讲清四种订单状态如何覆盖、为什么还要条件更新；目前我们只是发现问题，不能直接表述为你已经修复上线。

**关于求职动机：** 用你真实的兴趣解释为什么选择 Java 后端，例如喜欢业务建模、数据一致性、服务稳定性。到岗时间与实习时长按真实安排回答，不预设承诺。

## 36. 你有什么想问面试官的？

**可直接使用：**

1. 实习生入组后通常负责什么类型的任务？会怎样进行代码评审和反馈？
2. 团队目前在交易一致性、缓存或系统稳定性上，最需要解决的一个问题是什么？
3. 对实习生前三个月的预期成果和能力提升路径是什么？

这些问题分别了解工作内容、技术挑战和成长机制，优于只问“贵公司技术栈是什么”。

---

## 最后一分钟检查

- 不把单库十表说成已经跨库；不把取模分表说成查询必快十倍。
- 不把两个 CacheManager说成天然串联二级缓存。
- 不把 afterCommit说成可靠消息事务；不把记日志说成自动补偿。
- 不把 Flux包装说成模型实时 token流；不把码点说成所有可见字符。
- 不把原子减库存说成防超卖；不把状态乐观锁说成整个支付全链路幂等。
- 不把本地 @Transactional说成可以回滚 Feign对端的数据库。
- 记住支付监听器那个条件错误，并知道还需原子状态迁移。
- 个人职责、测试规模、性能数字与上线经历，只讲能够提供证据的部分。

**当前训练状态：** 已直接提供全部 36 题参考答案，尚未验证你的口述掌握程度；不据此给能力评分或录用判断。


## 代码证据索引

以下路径均为当前工作区的绝对路径，行号从当前文件内容核对得到。

- **N01**：[小说正文分片配置](C:/Users/35998/Desktop/job/project/novel/src/main/resources/shardingsphere-jdbc.yml:34)
- **N02**：[正文缓存与按 chapter_id 点查](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/BookContentCacheManager.java:30)
- **N03**：[原始正文表与 chapter_id 唯一索引](C:/Users/35998/Desktop/job/project/novel/doc/sql/novel.sql/novel_struc.sql:185)
- **N04**：[两个 CacheManager 与 transactionAware](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java:28)
- **N05**：[缓存域、类型、容量与 TTL](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/constant/CacheConsts.java:101)
- **N06**：[小说详情 Caffeine 缓存](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/BookInfoCacheManager.java:38)
- **N07**：[章节修改后的缓存失效与消息](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookServiceImpl.java:560)
- **N08**：[用户评论锁粒度](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookServiceImpl.java:231)
- **N09**：[Redisson 锁切面](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/aspect/LockAspect.java:44)
- **N10**：[ES bool／multi_match／filter 查询](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/EsSearchServiceImpl.java:112)
- **N11**：[事务提交后发送 MQ](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/mq/AmqpMsgManager.java:35)
- **N12**：[普通 ES 索引增量消费者](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/listener/RabbitQueueListener.java:39)
- **N13**：[四类 AI 业务工具](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/tools/BookCompanionTools.java:40)
- **N14**：[ChatClient、完整响应后分块与推荐](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookCompanionServiceImpl.java:59)
- **N15**：[向量全量／单本同步](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/vector/BookVectorSyncService.java:39)
- **N16**：[ES 向量索引与 kNN](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/vector/EsVectorStoreService.java:138)
- **N17**：[认证策略模式](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/interceptor/AuthInterceptor.java:31)
- **N18**：[消息发送模板方法](C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/message/AbstractMessageSender.java:23)
- **H01**：[交易服务下单流程](C:/Users/35998/Desktop/job/project/hmall/trade-service/src/main/java/com/hmall/trade/service/impl/OrderServiceImpl.java:48)
- **H02**：[现有库存扣减 SQL](C:/Users/35998/Desktop/job/project/hmall/item-service/src/main/java/com/hmall/item/mapper/ItemMapper.java:19)
- **H03**：[批量库存扣减](C:/Users/35998/Desktop/job/project/hmall/item-service/src/main/java/com/hmall/item/service/impl/ItemServiceImpl.java:30)
- **H04**：[网关全局 JWT 鉴权](C:/Users/35998/Desktop/job/project/hmall/hm-gateway/src/main/java/com/hmall/gateway/filters/AuthGlobalFilter.java:28)
- **H05**：[MVC 用户上下文写入与清理](C:/Users/35998/Desktop/job/project/hmall/hm-common/src/main/java/com/hmall/common/interceptor/UserInfoInterceptor.java:11)
- **H06**：[ThreadLocal 用户容器](C:/Users/35998/Desktop/job/project/hmall/hm-common/src/main/java/com/hmall/common/utils/UserContext.java:4)
- **H07**：[Feign 身份请求头透传](C:/Users/35998/Desktop/job/project/hmall/hmall-api/src/main/java/com/hmall/api/config/DefaultFeignConfig.java:17)
- **H08**：[Nacos 动态路由加载器](C:/Users/35998/Desktop/job/project/hmall/hm-gateway/src/main/java/com/hmall/gateway/routers/DynamicRoutLoader.java:31)
- **H09**：[支付单申请、远程扣款、状态更新与 MQ](C:/Users/35998/Desktop/job/project/hmall/pay-service/src/main/java/com/hmall/pay/service/impl/PayOrderServiceImpl.java:57)
- **H10**：[支付消息监听器条件错误](C:/Users/35998/Desktop/job/project/hmall/trade-service/src/main/java/com/hmall/trade/listener/PatStatusListener.java:26)
- **H11**：[商品查询与扣库存差异化降级](C:/Users/35998/Desktop/job/project/hmall/hmall-api/src/main/java/com/hmall/api/fallback/ItemClientFallbackFacotry.java:16)
- **H12**：[用户余额更新 SQL](C:/Users/35998/Desktop/job/project/hmall/user-service/src/main/java/com/hmall/user/mapper/UserMapper.java:17)
- **V01**：[novel 构建版本](C:/Users/35998/Desktop/job/project/novel/pom.xml:9)
- **V02**：[hmall 构建版本](C:/Users/35998/Desktop/job/project/hmall/pom.xml:26)

[N01]: C:/Users/35998/Desktop/job/project/novel/src/main/resources/shardingsphere-jdbc.yml:34
[N02]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/BookContentCacheManager.java:30
[N03]: C:/Users/35998/Desktop/job/project/novel/doc/sql/novel.sql/novel_struc.sql:185
[N04]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java:28
[N05]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/constant/CacheConsts.java:101
[N06]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/BookInfoCacheManager.java:38
[N07]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookServiceImpl.java:560
[N08]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookServiceImpl.java:231
[N09]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/aspect/LockAspect.java:44
[N10]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/EsSearchServiceImpl.java:112
[N11]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/mq/AmqpMsgManager.java:35
[N12]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/listener/RabbitQueueListener.java:39
[N13]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/tools/BookCompanionTools.java:40
[N14]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookCompanionServiceImpl.java:59
[N15]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/vector/BookVectorSyncService.java:39
[N16]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/ai/vector/EsVectorStoreService.java:138
[N17]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/interceptor/AuthInterceptor.java:31
[N18]: C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/message/AbstractMessageSender.java:23
[H01]: C:/Users/35998/Desktop/job/project/hmall/trade-service/src/main/java/com/hmall/trade/service/impl/OrderServiceImpl.java:48
[H02]: C:/Users/35998/Desktop/job/project/hmall/item-service/src/main/java/com/hmall/item/mapper/ItemMapper.java:19
[H03]: C:/Users/35998/Desktop/job/project/hmall/item-service/src/main/java/com/hmall/item/service/impl/ItemServiceImpl.java:30
[H04]: C:/Users/35998/Desktop/job/project/hmall/hm-gateway/src/main/java/com/hmall/gateway/filters/AuthGlobalFilter.java:28
[H05]: C:/Users/35998/Desktop/job/project/hmall/hm-common/src/main/java/com/hmall/common/interceptor/UserInfoInterceptor.java:11
[H06]: C:/Users/35998/Desktop/job/project/hmall/hm-common/src/main/java/com/hmall/common/utils/UserContext.java:4
[H07]: C:/Users/35998/Desktop/job/project/hmall/hmall-api/src/main/java/com/hmall/api/config/DefaultFeignConfig.java:17
[H08]: C:/Users/35998/Desktop/job/project/hmall/hm-gateway/src/main/java/com/hmall/gateway/routers/DynamicRoutLoader.java:31
[H09]: C:/Users/35998/Desktop/job/project/hmall/pay-service/src/main/java/com/hmall/pay/service/impl/PayOrderServiceImpl.java:57
[H10]: C:/Users/35998/Desktop/job/project/hmall/trade-service/src/main/java/com/hmall/trade/listener/PatStatusListener.java:26
[H11]: C:/Users/35998/Desktop/job/project/hmall/hmall-api/src/main/java/com/hmall/api/fallback/ItemClientFallbackFacotry.java:16
[H12]: C:/Users/35998/Desktop/job/project/hmall/user-service/src/main/java/com/hmall/user/mapper/UserMapper.java:17
[V01]: C:/Users/35998/Desktop/job/project/novel/pom.xml:9
[V02]: C:/Users/35998/Desktop/job/project/hmall/pom.xml:26
