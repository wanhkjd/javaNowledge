Searched for "@Cacheable"
Searched for "@CacheEvict"
Searched for files: *Cache*
Searched for files: *Cache*
Viewed CacheConfig.java:1-92
Viewed CacheConsts.java:1-150
Viewed CacheConsts.java:151-176

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-16)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-18)

Error Unknown: There was a network issue connecting to the server, please try again. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-20)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-22)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-24)

Error Unknown: There was a network issue connecting to the server, please try again. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-26)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-28)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-30)

Error Unknown: There was a network issue connecting to the server, please try again. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-32)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-34)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-36)

Error Unknown: Agent execution terminated due to error. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-38)

Viewed AuthorInfoCacheManager.java:1-60

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-42-2010)

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-44)

Error: Our servers are experiencing high traffic right now, please try again in a minute. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-46-2010)

Viewed BookInfoCacheManager.java:1-91
Searched for "evict"

Error: The stream was interrupted. Please continue the task you were working on. (Error ID: 79ad1d63-4750-4933-b249-55406ba45a99-52-2010)

Searched for "@EnableCaching"
Viewed BookContentCacheManager.java:1-46
Viewed HomeBookCacheManager.java:70-89

结合本项目 [novel](file:///C:/Users/35998/Desktop/job/project/novel) 的源码，对 Spring Cache、Redis 的关系、核心注解、缓存时间配置以及增删改查策略进行详细解答：

---

### 一、Spring Cache 是什么？和 Redis 有什么关系？

1. **Spring Cache 是什么**
   - Spring Cache 是 Spring 3.1 引入的**缓存抽象规范 / 统一门面（Facade）**。
   - 它通过 **AOP（动态代理）** 机制提供了**声明式缓存**支持。开发者只需要在方法上打上注解（如 `@Cacheable`、`@CacheEvict`），Spring 就会自动拦截方法调用，完成“查缓存 -> 命中返回 / 未命中查库 -> 写入缓存”的逻辑，避免将缓存代码与业务逻辑强耦合。
   - 在主启动类 [`NovelApplication.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/NovelApplication.java#L20) 上，通过标注 `@EnableCaching` 开启 Spring Cache 功能。

2. **Spring Cache 和 Redis 的关系**
   - **接口抽象 vs 具体实现**：Spring Cache 本身**不存储数据**，它定义了 `CacheManager` 和 `Cache` 接口；而 Redis 是当下最主流的**分布式内存数据库**。
   - **插件式集成**：Spring Cache 通过 `spring-data-redis` 提供了 `RedisCacheManager` 和 `RedisCache`，将缓存数据的增删改查具体落地到底层 Redis 服务中。
   - **项目中的多级/多管理器架构**：
     在本项目 [`CacheConfig.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java) 中同时配置了两套 `CacheManager`：
     - **Caffeine 本地缓存**（[`caffeineCacheManager`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java#L35-L53)）：作为默认 `@Primary` 缓存管理器，存储高频且体积较小的数据（如首页推荐、小说分类、小说基本信息等）。
     - **Redis 分布式缓存**（[`redisCacheManager`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java#L59-L89)）：存储多节点共享或占用内存较大的数据（如小说内容 `bookContentCache`、用户信息 `userInfoCache`、作家信息 `authorInfoCache`）。

---

### 二、`@Cacheable` 注解是什么？有哪些参数？

#### 1. 注解作用
标注在查询方法上。执行时先到指定缓存中查找：
- **缓存命中**：直接返回缓存中的值，**不执行**方法体。
- **缓存未命中**：执行方法体查询数据库，并将方法的返回值**存入缓存**后返回。

#### 2. 常用核心参数

| 参数名 | 类型 | 说明 | 项目中实战示例 |
| :--- | :--- | :--- | :--- |
| `value` / `cacheNames` | `String[]` | 缓存的命名空间（Cache Name） | `value = CacheConsts.AUTHOR_INFO_CACHE_NAME` |
| `cacheManager` | `String` | 指定由哪个 `CacheManager` 处理 | `cacheManager = CacheConsts.REDIS_CACHE_MANAGER` |
| `key` | `String` (SpEL) | 缓存键，支持 SpEL。缺省时使用参数作为 key | 默认或传入如 `#userId`、`#id` |
| `unless` | `String` (SpEL) | **方法执行后**判定，条件为 true 则**不写入缓存** | `unless = "#result == null"`（防止缓存 null 结果） |
| `condition` | `String` (SpEL) | **方法执行前**判定，条件为 true 时才启用缓存 | 如 `condition = "#id > 0"` |
| `sync` | `boolean` | 是否异步/加锁同步加载（防止缓存击穿并发查库） | 默认为 false |

#### 3. 项目中的典型使用
参考 [`AuthorInfoCacheManager.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/AuthorInfoCacheManager.java#L30-L32)：
```java
@Cacheable(
    cacheManager = CacheConsts.REDIS_CACHE_MANAGER,
    value = CacheConsts.AUTHOR_INFO_CACHE_NAME, 
    unless = "#result == null"
)
public AuthorInfoDto getAuthor(Long userId) {
    // 缓存未命中时执行查库逻辑
    ...
}
```
再比如本地缓存小说基本信息 [`BookInfoCacheManager.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/BookInfoCacheManager.java#L36-L38)：
```java
@Cacheable(
    cacheManager = CacheConsts.CAFFEINE_CACHE_MANAGER,
    value = CacheConsts.BOOK_INFO_CACHE_NAME, 
    unless = "#result == null"
)
public BookInfoRespDto getBookInfo(Long id) { ... }
```

---

### 三、`@CacheEvict` 注解是什么？有哪些参数？

#### 1. 注解作用
标注在清除/更新方法上，用于触发从缓存中**驱逐/删除**数据。

#### 2. 常用核心参数

| 参数名 | 类型 | 说明 | 项目中实战示例 |
| :--- | :--- | :--- | :--- |
| `value` / `cacheNames` | `String[]` | 目标缓存命名空间 | `value = CacheConsts.AUTHOR_INFO_CACHE_NAME` |
| `cacheManager` | `String` | 指定执行删除的 `CacheManager` | `cacheManager = CacheConsts.REDIS_CACHE_MANAGER` |
| `key` | `String` (SpEL) | 要清除的单个 key | `key = "#userId"` |
| `allEntries` | `boolean` | **是否清空该 cacheName 下的所有缓存**，默认为 false | `allEntries = true` |
| `beforeInvocation` | `boolean` | 是否在方法执行前清除缓存，默认为 false（即方法抛异常时不清除） | 默认为 false |
| `condition` | `String` (SpEL) | 满足条件时才执行清除 | - |

#### 3. 项目中的典型使用
参考 [`AuthorInfoCacheManager.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/AuthorInfoCacheManager.java#L47-L57)：
- **精准删除单个用户的缓存**：
  ```java
  @CacheEvict(cacheManager = CacheConsts.REDIS_CACHE_MANAGER,
      value = CacheConsts.AUTHOR_INFO_CACHE_NAME, key = "#userId")
  public void evictAuthorCache(Long userId) {
      // 执行此方法时，自动清除该 userId 对应的 Redis 缓存
  }
  ```
- **清空整个命名空间的全部条目**（如管理员后台重置所有作家或首页推荐）：
  ```java
  @CacheEvict(cacheManager = CacheConsts.REDIS_CACHE_MANAGER,
      value = CacheConsts.AUTHOR_INFO_CACHE_NAME, allEntries = true)
  public void evictAllAuthorCache() {
      // allEntries = true 会清空 authorInfoCache 下的所有缓存
  }
  ```
  在首页推荐变更时 [`HomeBookCacheManager.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/manager/cache/HomeBookCacheManager.java#L82-L86)：
  ```java
  @CacheEvict(cacheManager = CacheConsts.CAFFEINE_CACHE_MANAGER,
      value = CacheConsts.HOME_BOOK_CACHE_NAME, allEntries = true)
  public void evictHomeBookCache() {
      // 清空所有首页小说推荐缓存
  }
  ```

---

### 四、缓存过期时间（TTL）是怎么设置的？

Spring Cache 注解原生并没有提供 `ttl` 属性，**TTL 是在配置具体的 `CacheManager` 时定制的**。

本项目通过在常量枚举类中集中声明 TTL，并在配置类中为每个缓存项独立装配过期时间：

1. **统一声明枚举定义 TTL**：
   在 [`CacheConsts.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/constant/CacheConsts.java#L101-L150) 中定义了 `CacheEnum`：
   ```java
   public enum CacheEnum {
       HOME_BOOK_CACHE(0, HOME_BOOK_CACHE_NAME, 60 * 60 * 24, 1), // 24小时
       BOOK_VISIT_RANK_CACHE(2, BOOK_VISIT_RANK_CACHE_NAME, 60 * 60 * 6, 1), // 6小时
       BOOK_CONTENT_CACHE(2, BOOK_CONTENT_CACHE_NAME, 60 * 60 * 12, 3000), // 12小时
       ...
   }
   ```

2. **在 `CacheConfig` 中为 Redis 和 Caffeine 分别注入 TTL**：
   参考 [`CacheConfig.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/core/config/CacheConfig.java#L44-L46,L71-L81)：
   - **Caffeine** 通过 `caffeine.expireAfterWrite(Duration.ofSeconds(c.getTtl()))` 设置写入后过期。
   - **Redis** 通过 `RedisCacheConfiguration.entryTtl(Duration.ofSeconds(c.getTtl()))` 为不同的缓存命名空间配置独立的 TTL，统一传入 `RedisCacheManager`：
   ```java
   for (var c : CacheConsts.CacheEnum.values()) {
       if (c.isRemote()) {
           if (c.getTtl() > 0) {
               cacheMap.put(c.getName(),
                   RedisCacheConfiguration.defaultCacheConfig()
                       .disableCachingNullValues()
                       .prefixCacheNameWith(CacheConsts.REDIS_CACHE_PREFIX)
                       .entryTtl(Duration.ofSeconds(c.getTtl()))); // <-- 这里配置 Redis TTL
           }
       }
   }
   RedisCacheManager redisCacheManager = new RedisCacheManager(redisCacheWriter, defaultCacheConfig, cacheMap);
   ```

---

### 五、项目中是如何删除和更新缓存的？

在实际业务开发中，维护缓存一致性通常遵循 **Cache-Aside Pattern（旁路缓存模式）**：**写数据库，然后删除缓存（失效旧缓存）**，而不是直接在写操作中更新缓存。

#### 1. 删除缓存
- **单条删除**：
  在章节更新或删除后，需要主动使相关缓存失效。例如 [`BookServiceImpl.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/BookServiceImpl.java#L532-L536)：
  ```java
  // 1. 更新数据库章节与小说信息
  ...
  // 2. 主动调用各 CacheManager 的 @CacheEvict 方法删除旧缓存
  bookChapterCacheManager.evictBookChapterCache(chapterId); // 清除章节基本信息
  bookContentCacheManager.evictBookContentCache(chapterId); // 清除章节正文
  bookInfoCacheManager.evictBookInfoCache(chapter.getBookId()); // 清除小说基本信息
  ```
- **全量删除（重置）**：
  后台管理员修改了首页小说推荐后，调用 [`AdminServiceImpl.java`](file:///C:/Users/35998/Desktop/job/project/novel/src/main/java/novel/service/impl/AdminServiceImpl.java#L369)：
  ```java
  homeBookCacheManager.evictHomeBookCache(); // 触发 allEntries = true 清空整个命名空间
  ```

#### 2. 更新缓存
- **被动更新（延迟加载，项目中主要采用）**：
  修改数据时仅做 `@CacheEvict` 删除缓存。当下一次用户请求到达该数据时，由于缓存已被删除，`@Cacheable` 拦截到未命中，会重新执行数据库查询并将最新的数据写入 Redis/Caffeine 中。
  - **优势**：避免了并发写操作导致的数据不一致；如果被更新的数据短期内没人访问，也不会浪费内存去提前计算和存储。
- **主动更新（`@CachePut`）**：
  Spring Cache 还提供了 `@CachePut` 注解（每次都执行方法体，并将返回结果强制覆写到缓存中）。适用于每次数据库更新后都能完整返回需要缓存的实体对象场景。
- **自然过期轮询更新**：
  对于榜单类、新闻类等数据（如 `BookRankCacheManager`），业务层无需频繁调用 evict，数据在到达 `entryTtl` 设定的时间后由 Redis / Caffeine 自动驱逐过期，下一次查询自动获取最新排行。