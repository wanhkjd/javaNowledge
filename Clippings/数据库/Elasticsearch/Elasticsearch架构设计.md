---
title: "Elasticsearch架构设计"
source: "https://hhzh.github.io/elasticsearch/01-elasticsearch-architecture.html"
author:
  - "[[Mr.Hope]]"
published:
created: 2026-07-06
description: "Elasticsearch架构设计 引言 当你的电商网站有上千万个商品，用户输入\"iPhone 15 Pro 256G 蓝色\"，如何在 50 毫秒内返回相关度最高的结果？当你的日志系统每天写入 TB 级别的日志，运维需要在秒级定位\"过去一小时所有 500 错误\"？ LIKE '%keyword%' 全表扫描早已撑不住这种量级，关系型数据库的 B+Tre..."
tags:
  - "clippings"
---
## 引言

当你的电商网站有上千万个商品，用户输入"iPhone 15 Pro 256G 蓝色"，如何在 50 毫秒内返回相关度最高的结果？当你的日志系统每天写入 TB 级别的日志，运维需要在秒级定位"过去一小时所有 500 错误"？

`LIKE '%keyword%'` 全表扫描早已撑不住这种量级，关系型数据库的 B+Tree 索引也不是为全文检索而生的。Elasticsearch 凭什么能在海量数据中实现毫秒级搜索？

读完本文，你将掌握：

- **倒排索引** ：从"文档到词条"到"词条到文档"的思维翻转
- **分片与副本** ：如何实现数据水平扩展和高可用
- **Query/Fetch 两阶段搜索流程** ：理解一次搜索请求在分布式集群中的完整旅程

这是从"只会 CRUD"到"掌握搜索架构"的关键一跃，也是面试中区分"用过 ES"和"理解 ES"的分水岭。

---

## 深度解析 Elasticsearch 架构设计：分布式搜索与分析的强大引擎

### 全文检索与实时分析的挑战

关系型数据库虽然在结构化数据管理和事务处理方面表现出色，但在以下场景下常常力不从心：

- **全文检索：** 根据文本内容快速、准确地查找相关文档，支持分词、模糊匹配、相关度排序等。
- **实时分析 (Aggregations)：** 对海量数据进行实时的统计、聚合、分组计算，例如计算某个时间段内用户的平均消费、特定商品的销售额趋势等。
- **处理非结构化/半结构化数据：** RDBMS 严格的表结构不适合存储和查询日志、JSON 文档等格式灵活的数据。
- **大规模数据下的扩展性：** 当数据量达到 TB/PB 级别时，RDBMS 的水平扩展非常复杂。

构建一个能够同时满足以上需求且具备高可用和可伸缩性的系统，是分布式领域的巨大挑战。

Elasticsearch 的出现，正是为了提供一个专门用于解决这些挑战的平台。

> **💡 核心提示** ：Elasticsearch 的核心优势不在于"存储更多数据"，而在于"用完全不同的索引结构（倒排索引）让搜索更快"。理解这一点，就能理解它为什么能替代数据库做搜索。

### Elasticsearch 是什么？定位与核心理念

Elasticsearch 是一个分布式、RESTful 风格的 **搜索与分析引擎** 。

- **定位：** 它是一个构建在 Apache Lucene 之上的 **分布式、实时** 的搜索和分析平台。它提供强大的全文检索、结构化搜索、实时分析和数据可视化能力。
- **核心理念：** 将数据存储为 **JSON 文档** ，对所有字段 **默认进行索引** ，通过 **分片 (Sharding)** 实现水平扩展，通过 **副本 (Replication)** 实现高可用，所有操作都通过 **RESTful API** 进行。

### 为什么选择 Elasticsearch？优势分析

- **卓越的性能：** 写入速度快，搜索和聚合查询响应迅速，尤其是在大数据量下。
- **极高的可伸缩性：** 通过简单的增加节点即可实现数据的自动分片和集群扩展。
- **近乎实时 (Near Real-time - NRT)：** 数据索引后，在很短的时间内（通常是 1 秒）即可被搜索到。
- **功能丰富：** 支持全文检索、结构化搜索、地理位置搜索、强大的聚合分析功能。
- **RESTful API：** 所有操作都通过标准的 HTTP RESTful API 进行，易于开发和集成。
- **强大的生态系统：** 作为 ELK Stack (Elasticsearch, Logstash, Kibana) 的核心组件，方便进行日志收集、分析和可视化。
- **灵活的数据模型：** 支持 JSON 文档，对 schema 约束较弱（支持动态 Mapping）。

### Elasticsearch 核心概念详解

理解 Elasticsearch 的架构，需要掌握其几个核心概念：

#### Document (文档)

- **定义：** Elasticsearch 中 **最小的数据单元** 。一个 Document 是一个 JSON 对象。类似于关系型数据库中的一行记录。
- **作用：** 存储需要索引和搜索的数据。每个 Document 在其 Index 中有唯一的 ID。
- **示例 (JSON):**
```json
{
  "user_id": 1001,
  "username": "zhangsan",
  "email": "zhangsan@example.com",
  "age": 30,
  "interests": ["coding", "music"],
  "about": "A software engineer interested in distributed systems.",
  "join_date": "2023-01-01T10:00:00Z"
}
```

#### Field (字段) & Mapping (映射)

- **Field：** Document 中的一个键值对。数据是按字段进行索引的。
- **Mapping：** 定义 Document 中每个 Field 的 **数据类型** （如 `text`, `keyword`, `date`, `long`, `boolean` 等）以及该字段如何被 **索引** 。
	- **作用：** 决定了如何存储、索引和搜索字段数据。 `text` 类型会被分词进行全文检索， `keyword` 类型会精确匹配。
		- **动态 Mapping：** 当索引一个新文档时，如果遇到了新的字段，Elasticsearch 会尝试自动推断字段类型并创建 Mapping。
		- **显式 Mapping：** 开发者可以预先定义 Mapping，更精确地控制索引行为。

> **💡 核心提示** ： `text` 和 `keyword` 是最容易混淆的两种类型。 `text` 会被分析器分词，适合全文检索； `keyword` 不分词，适合精确匹配、排序和聚合。如果需要对一个字段同时支持全文检索和精确匹配，可以使用 multi-fields 定义。

#### Index (索引)

- **定义：** 一个逻辑上的 **文档集合** 。它包含了一组具有相似特性的文档。
- **作用：** 文档在 Index 中被存储和索引。搜索操作针对一个或多个 Index 进行。分片和副本是在 Index 层面进行配置和管理的。
- **比喻：** 类似于关系型数据库中的一个"数据库"或旧版本中的"表"。（注意：在 Elasticsearch 7.x 及以后版本，"Type" 概念已被弃用，推荐一个 Index 只存储一类 Document）。

#### Shard (分片)

- **定义：** 一个 Index 被划分为一个或多个 Shard。每个 Shard 都是一个完全独立的 **Lucene 索引** 。
- **作用：**
	- **水平扩展：** Shard 是 Elasticsearch 实现水平扩展的基本单位。Index 的数据通过某种规则（如 Document ID 的 Hash 值）分布到不同的 Primary Shard 上。通过增加 Shard 数量，可以将数据分散到更多节点上，提升读写性能和存储容量。
		- **并行处理：** 搜索和聚合等操作可以并行在 Index 的所有 Shard 上执行，然后将结果合并。
- **比喻：** 一本书被分成多个独立章节，每个章节是一个 Shard。

#### Replica (副本)

- **定义：** 一个 Primary Shard 的 **拷贝** 。一个 Primary Shard 可以有零个或多个 Replica Shard。
- **作用：**
	- **高可用 (HA)：** 当 Primary Shard 或其所在的节点发生故障时，Replica Shard 可以被提升为新的 Primary Shard，保证数据和服务的高可用。
		- **读扩展：** 搜索请求可以由 Primary Shard 和其所有 Replica Shard 共同处理，分担读负载，提高搜索吞吐量。
- **比喻：** 章节的原件 (Primary Shard) 和复印件 (Replica Shard)。

#### Node (节点) & Cluster (集群)

- **Node：** 一台运行 Elasticsearch 实例的服务器。
- **Cluster：** 一组 Node 组成一个 Elasticsearch 集群。集群中的节点相互通信，共同管理数据（索引、分片、副本）和处理请求。
- **节点角色：** 节点可以扮演不同的角色，如 Master eligible node (有资格成为 Master)、Data node (存储数据)、Ingest node (预处理文档)、Coordinating node (协调请求) 等。一个节点可以扮演多种角色。

#### Primary Shard (主分片) & Replica Shard (副本分片)

- **区别：**
	- **Primary Shard：** 负责处理该分片所属数据的 **所有索引请求** (增、删、改)。
		- **Replica Shard：** 负责 **复制** 其对应的 Primary Shard 的数据，并可以处理该分片所属数据的 **读请求** (搜索)。
- 一个 Index 的 Primary Shard 数量在 Index 创建时确定，之后不能改变。Replica Shard 数量可以在 Index 创建后动态调整。

### Elasticsearch 集群架构

<svg id="mermaid-334" width="100%" xmlns="http://www.w3.org/2000/svg" style="max-width: 808.4921875px;" viewBox="0 0 808.4921875 1170.625" role="graphics-document document" aria-roledescription="class"><g><defs><marker id="mermaid-334_class-aggregationStart" refX="18" refY="7" markerWidth="190" markerHeight="240" orient="auto"><path d="M 18,7 L9,13 L1,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-aggregationEnd" refX="1" refY="7" markerWidth="20" markerHeight="28" orient="auto"><path d="M 18,7 L9,13 L1,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-aggregationStart-margin" refX="15" refY="7" markerWidth="190" markerHeight="240" orient="auto" markerUnits="userSpaceOnUse"><path d="M 18,7 L9,13 L1,7 L9,1 Z" style="stroke-width: 2;"></path></marker></defs><defs><marker id="mermaid-334_class-aggregationEnd-margin" refX="1" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse"><path d="M 18,7 L9,13 L1,7 L9,1 Z" style="stroke-width: 2;"></path></marker></defs><defs><marker id="mermaid-334_class-extensionStart" refX="18" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse"><path d="M 1,7 L18,13 V 1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-extensionEnd" refX="1" refY="7" markerWidth="20" markerHeight="28" orient="auto"><path d="M 1,1 V 13 L18,7 Z"></path></marker></defs><marker id="mermaid-334_class-extensionStart-margin" refX="18" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse" viewBox="0 0 20 14"><polygon points="10,7 18,13 18,1" style="stroke-width: 2; stroke-dasharray: 0;"></polygon></marker><defs><marker id="mermaid-334_class-extensionEnd-margin" refX="9" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse" viewBox="0 0 20 14"><polygon points="10,1 10,13 18,7" style="stroke-width: 2; stroke-dasharray: 0;"></polygon></marker></defs><defs><marker id="mermaid-334_class-compositionStart" refX="18" refY="7" markerWidth="190" markerHeight="240" orient="auto"><path d="M 18,7 L9,13 L1,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-compositionEnd" refX="1" refY="7" markerWidth="20" markerHeight="28" orient="auto"><path d="M 18,7 L9,13 L1,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-compositionStart-margin" refX="15" refY="7" markerWidth="190" markerHeight="240" orient="auto" markerUnits="userSpaceOnUse"><path viewBox="0 0 15 15" d="M 18,7 L9,13 L1,7 L9,1 Z" style="stroke-width: 0;"></path></marker></defs><defs><marker id="mermaid-334_class-compositionEnd-margin" refX="3.5" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse"><path d="M 18,7 L9,13 L1,7 L9,1 Z" style="stroke-width: 0;"></path></marker></defs><defs><marker id="mermaid-334_class-dependencyStart" refX="6" refY="7" markerWidth="190" markerHeight="240" orient="auto"><path d="M 5,7 L9,13 L1,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-dependencyEnd" refX="13" refY="7" markerWidth="20" markerHeight="28" orient="auto"><path d="M 18,7 L9,13 L14,7 L9,1 Z"></path></marker></defs><defs><marker id="mermaid-334_class-dependencyStart-margin" refX="4" refY="7" markerWidth="190" markerHeight="240" orient="auto" markerUnits="userSpaceOnUse"><path d="M 5,7 L9,13 L1,7 L9,1 Z" style="stroke-width: 0;"></path></marker></defs><defs><marker id="mermaid-334_class-dependencyEnd-margin" refX="16" refY="7" markerWidth="20" markerHeight="28" orient="auto" markerUnits="userSpaceOnUse"><path d="M 18,7 L9,13 L14,7 L9,1 Z" style="stroke-width: 0;"></path></marker></defs><defs><marker id="mermaid-334_class-lollipopStart" refX="13" refY="7" markerWidth="190" markerHeight="240" orient="auto"><circle fill="transparent" cx="7" cy="7" r="6"></circle></marker></defs><defs><marker id="mermaid-334_class-lollipopEnd" refX="1" refY="7" markerWidth="190" markerHeight="240" orient="auto"><circle fill="transparent" cx="7" cy="7" r="6"></circle></marker></defs><defs><marker id="mermaid-334_class-lollipopStart-margin" refX="13" refY="7" markerWidth="190" markerHeight="240" orient="auto" markerUnits="userSpaceOnUse"><circle fill="transparent" cx="7" cy="7" r="6" stroke-width="2"></circle></marker></defs><defs><marker id="mermaid-334_class-lollipopEnd-margin" refX="1" refY="7" markerWidth="190" markerHeight="240" orient="auto" markerUnits="userSpaceOnUse"><circle fill="transparent" cx="7" cy="7" r="6" stroke-width="2"></circle></marker></defs><g><g></g><g><path d="M414.641,150.944L365.146,171.014C315.651,191.083,216.661,231.221,167.167,258.723C117.672,286.224,117.672,301.089,117.672,308.521L117.672,315.953" id="mermaid-334-id_Cluster_MasterNode_1" style=";;;" data-edge="true" data-et="edge" data-id="id_Cluster_MasterNode_1" data-points="W3sieCI6NDE0LjY0MDYyNSwieSI6MTUwLjk0NDM5MjIyNDYwMzU4fSx7IngiOjExNy42NzE4NzUsInkiOjI3MS4zNTkzNzV9LHsieCI6MTE3LjY3MTg3NSwieSI6MzIxLjk1MzEyNX1d" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path><path d="M550.617,233.563L554.058,239.862C557.498,246.161,564.378,258.76,567.818,292.289C571.258,325.818,571.258,380.276,571.258,434.734C571.258,489.193,571.258,543.651,571.258,595.977C571.258,648.302,571.258,698.495,571.258,748.688C571.258,798.88,571.258,849.073,573.62,879.553C575.981,910.033,580.705,920.801,583.067,926.184L585.429,931.568" id="mermaid-334-id_Cluster_DataNode_2" style=";;;" data-edge="true" data-et="edge" data-id="id_Cluster_DataNode_2" data-points="W3sieCI6NTUwLjYxNzQ5MDg5NjA5MDUsInkiOjIzMy41NjI1fSx7IngiOjU3MS4yNTc4MTI1LCJ5IjoyNzEuMzU5Mzc1fSx7IngiOjU3MS4yNTc4MTI1LCJ5Ijo0MzQuNzM0Mzc1fSx7IngiOjU3MS4yNTc4MTI1LCJ5Ijo1OTguMTA5Mzc1fSx7IngiOjU3MS4yNTc4MTI1LCJ5Ijo3NDguNjg3NX0seyJ4Ijo1NzEuMjU3ODEyNSwieSI6ODk5LjI2NTYyNX0seyJ4Ijo1ODcuODM5Mjk0MTEwNTg5NCwieSI6OTM3LjA2MjV9XQ==" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path><path d="M427.441,233.563L424.001,239.862C420.561,246.161,413.681,258.76,410.241,272.492C406.801,286.224,406.801,301.089,406.801,308.521L406.801,315.953" id="mermaid-334-id_Cluster_CoordinatingNode_3" style=";;;" data-edge="true" data-et="edge" data-id="id_Cluster_CoordinatingNode_3" data-points="W3sieCI6NDI3LjQ0MTEwMjg1MzkwOTQsInkiOjIzMy41NjI1fSx7IngiOjQwNi44MDA3ODEyNSwieSI6MjcxLjM1OTM3NX0seyJ4Ijo0MDYuODAwNzgxMjUsInkiOjMyMS45NTMxMjV9XQ==" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path><path d="M563.418,173.039L586.744,189.426C610.07,205.813,656.723,238.586,680.049,260.272C703.375,281.958,703.375,292.557,703.375,297.857L703.375,303.156" id="mermaid-334-id_Cluster_Index_4" style=";;;" data-edge="true" data-et="edge" data-id="id_Cluster_Index_4" data-points="W3sieCI6NTYzLjQxNzk2ODc1LCJ5IjoxNzMuMDM5Mzg1MTk0MDg2M30seyJ4Ijo3MDMuMzc1LCJ5IjoyNzEuMzU5Mzc1fSx7IngiOjcwMy4zNzUsInkiOjMwOS4xNTYyNX1d" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path><path d="M703.375,560.313L703.375,566.612C703.375,572.911,703.375,585.51,703.375,597.109C703.375,608.708,703.375,619.307,703.375,624.607L703.375,629.906" id="mermaid-334-id_Index_Shard_5" style=";;;" data-edge="true" data-et="edge" data-id="id_Index_Shard_5" data-points="W3sieCI6NzAzLjM3NSwieSI6NTYwLjMxMjV9LHsieCI6NzAzLjM3NSwieSI6NTk4LjEwOTM3NX0seyJ4Ijo3MDMuMzc1LCJ5Ijo2MzUuOTA2MjV9XQ==" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path><path d="M703.375,861.469L703.375,867.768C703.375,874.068,703.375,886.667,701.013,898.35C698.651,910.033,693.928,920.801,691.566,926.184L689.204,931.568" id="mermaid-334-id_Shard_DataNode_6" style=";;;" data-edge="true" data-et="edge" data-id="id_Shard_DataNode_6" data-points="W3sieCI6NzAzLjM3NSwieSI6ODYxLjQ2ODc1fSx7IngiOjcwMy4zNzUsInkiOjg5OS4yNjU2MjV9LHsieCI6Njg2Ljc5MzUxODM4OTQxMDYsInkiOjkzNy4wNjI1fV0=" data-look="classic" marker-end="url(#mermaid-334_class-dependencyEnd)" fill="none" stroke="currentColor"></path></g><g><g transform="translate(117.671875, 271.359375)"><g data-id="id_Cluster_MasterNode_1" transform="translate(-37.1953125, -12.796875)"><foreignObject width="74.390625" height="25.59375"><p>"选举一个"</p></foreignObject></g></g><g transform="translate(571.2578125, 598.109375)"><g data-id="id_Cluster_DataNode_2" transform="translate(-37.1953125, -12.796875)"><foreignObject width="74.390625" height="25.59375"><p>"包含多个"</p></foreignObject></g></g><g transform="translate(406.80078125, 271.359375)"><g data-id="id_Cluster_CoordinatingNode_3" transform="translate(-61.1953125, -12.796875)"><foreignObject width="122.390625" height="25.59375"><p>"任一节点可充当"</p></foreignObject></g></g><g transform="translate(703.375, 271.359375)"><g data-id="id_Cluster_Index_4" transform="translate(-37.1953125, -12.796875)"><foreignObject width="74.390625" height="25.59375"><p>"包含多个"</p></foreignObject></g></g><g transform="translate(703.375, 598.109375)"><g data-id="id_Index_Shard_5" transform="translate(-45.1953125, -12.796875)"><foreignObject width="90.390625" height="25.59375"><p>"划分为多个"</p></foreignObject></g></g><g transform="translate(703.375, 899.265625)"><g data-id="id_Shard_DataNode_6" transform="translate(-29.1953125, -12.796875)"><foreignObject width="58.390625" height="25.59375"><p>"分配在"</p></foreignObject></g></g></g><g><g id="mermaid-334-classId-Cluster-4" data-look="classic" transform="translate(489.029296875, 120.78125)"><g><path d="M-74.388671875 -112.78125 L74.388671875 -112.78125 L74.388671875 112.78125 L-74.388671875 112.78125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-74.388671875 -112.78125 C-35.446363091041675 -112.78125, 3.4959456929166493 -112.78125, 74.388671875 -112.78125 M-74.388671875 -112.78125 C-33.90466953116411 -112.78125, 6.57933281267178 -112.78125, 74.388671875 -112.78125 M74.388671875 -112.78125 C74.388671875 -60.737347360121895, 74.388671875 -8.69344472024379, 74.388671875 112.78125 M74.388671875 -112.78125 C74.388671875 -53.88570250406377, 74.388671875 5.0098449918724555, 74.388671875 112.78125 M74.388671875 112.78125 C18.247165583273393 112.78125, -37.894340708453214 112.78125, -74.388671875 112.78125 M74.388671875 112.78125 C16.158256231206742 112.78125, -42.072159412586515 112.78125, -74.388671875 112.78125 M-74.388671875 112.78125 C-74.388671875 51.0590531592607, -74.388671875 -10.663143681478601, -74.388671875 -112.78125 M-74.388671875 112.78125 C-74.388671875 60.458978639096166, -74.388671875 8.136707278192333, -74.388671875 -112.78125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-24.390625, -88.78125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="48.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 98px; text-align: center;"><span style=""><p>«集群»</p></span></div></foreignObject></g></g><g transform="translate(-26.61328125, -63.1875)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="53.2265625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 103px; text-align: center;"><span style=""><p>Cluster</p></span></div></foreignObject></g></g><g transform="translate(-62.388671875, -13.59375)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="98.1640625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 160px; text-align: center;"><span style=""><p>+clusterName</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="94.28125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 157px; text-align: center;"><span style=""><p>+masterNode</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="95.125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 157px; text-align: center;"><span style=""><p>+dataNodes[]</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="70.046875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 129px; text-align: center;"><span style=""><p>+indices[]</p></span></div></foreignObject></g></g><g transform="translate(-62.388671875, 112.78125)"></g><g style=""><path d="M-74.388671875 -37.59375 C-36.59890054547982 -37.593495997808695, 1.1908707840403565 -37.59324199561738, 74.388671875 -37.59275 M-74.388671875 -37.59375 C-34.8032542789113 -37.59348392845578, 4.782163317177407 -37.593217856911565, 74.388671875 -37.59275" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-74.388671875 88.78125 C-20.649913792097855 88.78161120256438, 33.08884429080429 88.78197240512874, 74.388671875 88.78225 M-74.388671875 88.78125 C-22.310641330009297 88.78160004006143, 29.767389214981407 88.78195008012285, 74.388671875 88.78225" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g><g id="mermaid-334-classId-MasterNode-5" data-look="classic" transform="translate(117.671875, 434.734375)"><g><path d="M-109.671875 -112.78125 L109.671875 -112.78125 L109.671875 112.78125 L-109.671875 112.78125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-109.671875 -112.78125 C-27.498481646793934 -112.78125, 54.67491170641213 -112.78125, 109.671875 -112.78125 M-109.671875 -112.78125 C-62.61741744451615 -112.78125, -15.5629598890323 -112.78125, 109.671875 -112.78125 M109.671875 -112.78125 C109.671875 -37.08250017775501, 109.671875 38.61624964448998, 109.671875 112.78125 M109.671875 -112.78125 C109.671875 -39.57744146939184, 109.671875 33.626367061216314, 109.671875 112.78125 M109.671875 112.78125 C26.131239765832532 112.78125, -57.409395468334935 112.78125, -109.671875 112.78125 M109.671875 112.78125 C61.61726822073561 112.78125, 13.56266144147122 112.78125, -109.671875 112.78125 M-109.671875 112.78125 C-109.671875 49.77591504245277, -109.671875 -13.229419915094454, -109.671875 -112.78125 M-109.671875 112.78125 C-109.671875 29.495201883842853, -109.671875 -53.790846232314294, -109.671875 -112.78125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-50.5625, -88.78125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="101.125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 151px; text-align: center;"><span style=""><p>«Master 节点»</p></span></div></foreignObject></g></g><g transform="translate(-43.96484375, -63.1875)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="87.9296875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 140px; text-align: center;"><span style=""><p>MasterNode</p></span></div></foreignObject></g></g><g transform="translate(-97.671875, -13.59375)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="120.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 180px; text-align: center;"><span style=""><p>+集群元数据管理</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="112.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 170px; text-align: center;"><span style=""><p>+索引创建/删除</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="104.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 164px; text-align: center;"><span style=""><p>+分片分配决策</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="144.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 202px; text-align: center;"><span style=""><p>+节点加入/离开处理</p></span></div></foreignObject></g></g><g transform="translate(-97.671875, 112.78125)"></g><g style=""><path d="M-109.671875 -37.59375 C-46.87021623983673 -37.59346368384666, 15.931442520326542 -37.593177367693315, 109.671875 -37.59275 M-109.671875 -37.59375 C-23.306472769862935 -37.593356255467825, 63.05892946027413 -37.59296251093564, 109.671875 -37.59275" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-109.671875 88.78125 C-37.47992722267361 88.78157912698802, 34.71202055465278 88.78190825397603, 109.671875 88.78225 M-109.671875 88.78125 C-63.51213254181231 88.78146044475832, -17.352390083624627 88.78167088951665, 109.671875 88.78225" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g><g id="mermaid-334-classId-DataNode-6" data-look="classic" transform="translate(637.31640625, 1049.84375)"><g><path d="M-106.982421875 -112.78125 L106.982421875 -112.78125 L106.982421875 112.78125 L-106.982421875 112.78125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-106.982421875 -112.78125 C-28.043260716007183 -112.78125, 50.895900442985635 -112.78125, 106.982421875 -112.78125 M-106.982421875 -112.78125 C-29.274968401637523 -112.78125, 48.432485071724955 -112.78125, 106.982421875 -112.78125 M106.982421875 -112.78125 C106.982421875 -34.8307383025961, 106.982421875 43.119773394807794, 106.982421875 112.78125 M106.982421875 -112.78125 C106.982421875 -37.604379892267374, 106.982421875 37.57249021546525, 106.982421875 112.78125 M106.982421875 112.78125 C53.344065339269854 112.78125, -0.2942911964602928 112.78125, -106.982421875 112.78125 M106.982421875 112.78125 C39.93574903064062 112.78125, -27.110923813718756 112.78125, -106.982421875 112.78125 M-106.982421875 112.78125 C-106.982421875 37.563600166262006, -106.982421875 -37.65404966747599, -106.982421875 -112.78125 M-106.982421875 112.78125 C-106.982421875 37.208639995369595, -106.982421875 -38.36397000926081, -106.982421875 -112.78125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-43.28515625, -88.78125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="86.5703125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 135px; text-align: center;"><span style=""><p>«Data 节点»</p></span></div></foreignObject></g></g><g transform="translate(-35.9453125, -63.1875)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="71.890625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 124px; text-align: center;"><span style=""><p>DataNode</p></span></div></foreignObject></g></g><g transform="translate(-94.982421875, -13.59375)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="104.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 164px; text-align: center;"><span style=""><p>+存储分片数据</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="121.1015625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 182px; text-align: center;"><span style=""><p>+执行 CRUD 操作</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="120.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 180px; text-align: center;"><span style=""><p>+执行搜索和聚合</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="146.6796875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 212px; text-align: center;"><span style=""><p>+Translog + Segment</p></span></div></foreignObject></g></g><g transform="translate(-94.982421875, 112.78125)"></g><g style=""><path d="M-106.982421875 -37.59375 C-56.44311708173423 -37.59351379622041, -5.903812288468458 -37.59327759244082, 106.982421875 -37.59275 M-106.982421875 -37.59375 C-32.90194319171603 -37.5934037726601, 41.17853549156794 -37.59305754532021, 106.982421875 -37.59275" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-106.982421875 88.78125 C-37.232228216835026 88.78157598903837, 32.51796544132995 88.78190197807673, 106.982421875 88.78225 M-106.982421875 88.78125 C-41.66121367731897 88.78155528944407, 23.659994520362062 88.78186057888813, 106.982421875 88.78225" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g><g id="mermaid-334-classId-CoordinatingNode-7" data-look="classic" transform="translate(406.80078125, 434.734375)"><g><path d="M-129.45703125 -112.78125 L129.45703125 -112.78125 L129.45703125 112.78125 L-129.45703125 112.78125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-129.45703125 -112.78125 C-65.09630690545703 -112.78125, -0.7355825609140538 -112.78125, 129.45703125 -112.78125 M-129.45703125 -112.78125 C-77.54183201077137 -112.78125, -25.62663277154273 -112.78125, 129.45703125 -112.78125 M129.45703125 -112.78125 C129.45703125 -47.850773006869034, 129.45703125 17.079703986261933, 129.45703125 112.78125 M129.45703125 -112.78125 C129.45703125 -27.663907712316956, 129.45703125 57.45343457536609, 129.45703125 112.78125 M129.45703125 112.78125 C76.00709293136941 112.78125, 22.55715461273884 112.78125, -129.45703125 112.78125 M129.45703125 112.78125 C68.16782971251651 112.78125, 6.878628175033015 112.78125, -129.45703125 112.78125 M-129.45703125 112.78125 C-129.45703125 54.38707632752644, -129.45703125 -4.007097344947127, -129.45703125 -112.78125 M-129.45703125 112.78125 C-129.45703125 53.68182056208458, -129.45703125 -5.417608875830837, -129.45703125 -112.78125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-40.390625, -88.78125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="80.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 130px; text-align: center;"><span style=""><p>«协调节点»</p></span></div></foreignObject></g></g><g transform="translate(-66.7890625, -63.1875)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="133.578125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 188px; text-align: center;"><span style=""><p>CoordinatingNode</p></span></div></foreignObject></g></g><g transform="translate(-117.45703125, -13.59375)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="120.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 180px; text-align: center;"><span style=""><p>+接收客户端请求</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="104.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 164px; text-align: center;"><span style=""><p>+请求路由分发</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="104.390625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 164px; text-align: center;"><span style=""><p>+结果归并排序</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="168.125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 225px; text-align: center;"><span style=""><p>+Query/Fetch 阶段协调</p></span></div></foreignObject></g></g><g transform="translate(-117.45703125, 112.78125)"></g><g style=""><path d="M-129.45703125 -37.59375 C-28.554235022571802 -37.59336028460466, 72.3485612048564 -37.592970569209314, 129.45703125 -37.59275 M-129.45703125 -37.59375 C-76.49024099361647 -37.593545427140015, -23.523450737232935 -37.59334085428003, 129.45703125 -37.59275" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-129.45703125 88.78125 C-76.68980919447736 88.78145380207064, -23.92258713895474 88.78165760414129, 129.45703125 88.78225 M-129.45703125 88.78125 C-72.11965154016178 88.78147145332376, -14.782271830323552 88.78169290664754, 129.45703125 88.78225" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g><g id="mermaid-334-classId-Index-8" data-look="classic" transform="translate(703.375, 434.734375)"><g><path d="M-97.1171875 -125.578125 L97.1171875 -125.578125 L97.1171875 125.578125 L-97.1171875 125.578125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-97.1171875 -125.578125 C-44.2405521904169 -125.578125, 8.636083119166202 -125.578125, 97.1171875 -125.578125 M-97.1171875 -125.578125 C-57.52941986745296 -125.578125, -17.941652234905916 -125.578125, 97.1171875 -125.578125 M97.1171875 -125.578125 C97.1171875 -59.30887274634932, 97.1171875 6.960379507301354, 97.1171875 125.578125 M97.1171875 -125.578125 C97.1171875 -40.798764447524036, 97.1171875 43.98059610495193, 97.1171875 125.578125 M97.1171875 125.578125 C54.369495103699975 125.578125, 11.62180270739995 125.578125, -97.1171875 125.578125 M97.1171875 125.578125 C23.33622266121023 125.578125, -50.44474217757954 125.578125, -97.1171875 125.578125 M-97.1171875 125.578125 C-97.1171875 74.82010509979905, -97.1171875 24.0620851995981, -97.1171875 -125.578125 M-97.1171875 125.578125 C-97.1171875 40.28094688000742, -97.1171875 -45.01623123998516, -97.1171875 -125.578125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-24.390625, -101.578125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="48.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 98px; text-align: center;"><span style=""><p>«索引»</p></span></div></foreignObject></g></g><g transform="translate(-20.609375, -75.984375)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="41.21875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 91px; text-align: center;"><span style=""><p>Index</p></span></div></foreignObject></g></g><g transform="translate(-85.1171875, -26.390625)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="87.9765625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 150px; text-align: center;"><span style=""><p>+indexName</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="145.84375" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 214px; text-align: center;"><span style=""><p>+primaryShardCount</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="99.84375" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 161px; text-align: center;"><span style=""><p>+replicaCount</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="65.375" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 126px; text-align: center;"><span style=""><p>+shards[]</p></span></div></foreignObject></g><g style="" transform="translate(0,89.578125)"><foreignObject width="69.2421875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 132px; text-align: center;"><span style=""><p>+mapping</p></span></div></foreignObject></g></g><g transform="translate(-85.1171875, 125.578125)"></g><g style=""><path d="M-97.1171875 -50.390625 C-33.183078227753654 -50.39029584039953, 30.75103104449269 -50.38996668079906, 97.1171875 -50.389625 M-97.1171875 -50.390625 C-19.975652236193355 -50.390227843032996, 57.16588302761329 -50.389830686065984, 97.1171875 -50.389625" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-97.1171875 101.578125 C-56.614861009584686 101.57833352295836, -16.112534519169373 101.57854204591672, 97.1171875 101.579125 M-97.1171875 101.578125 C-21.906687692660554 101.57851221518685, 53.30381211467889 101.5788994303737, 97.1171875 101.579125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g><g id="mermaid-334-classId-Shard-9" data-look="classic" transform="translate(703.375, 748.6875)"><g><path d="M-86.28515625 -112.78125 L86.28515625 -112.78125 L86.28515625 112.78125 L-86.28515625 112.78125" stroke="none" stroke-width="0" fill="#4abf8a" style=""></path><path d="M-86.28515625 -112.78125 C-45.95532561267003 -112.78125, -5.625494975340061 -112.78125, 86.28515625 -112.78125 M-86.28515625 -112.78125 C-19.319377839181314 -112.78125, 47.64640057163737 -112.78125, 86.28515625 -112.78125 M86.28515625 -112.78125 C86.28515625 -54.54174200415445, 86.28515625 3.6977659916910994, 86.28515625 112.78125 M86.28515625 -112.78125 C86.28515625 -53.11861072915929, 86.28515625 6.5440285416814135, 86.28515625 112.78125 M86.28515625 112.78125 C47.16677664931589 112.78125, 8.048397048631784 112.78125, -86.28515625 112.78125 M86.28515625 112.78125 C48.014643699252154 112.78125, 9.744131148504309 112.78125, -86.28515625 112.78125 M-86.28515625 112.78125 C-86.28515625 44.13631445528445, -86.28515625 -24.508621089431102, -86.28515625 -112.78125 M-86.28515625 112.78125 C-86.28515625 58.73554712942617, -86.28515625 4.689844258852347, -86.28515625 -112.78125" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g transform="translate(-24.390625, -88.78125)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="48.78125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 98px; text-align: center;"><span style=""><p>«分片»</p></span></div></foreignObject></g></g><g transform="translate(-21.15625, -63.1875)"><g style="font-weight: bolder" transform="translate(0,-12.796875)"><foreignObject width="42.3125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 94px; text-align: center;"><span style=""><p>Shard</p></span></div></foreignObject></g></g><g transform="translate(-74.28515625, -13.59375)"><g style="" transform="translate(0,-12.796875)"><foreignObject width="60.515625" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 122px; text-align: center;"><span style=""><p>+shardId</p></span></div></foreignObject></g><g style="" transform="translate(0,12.796875)"><foreignObject width="74.1796875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 137px; text-align: center;"><span style=""><p>+isPrimary</p></span></div></foreignObject></g><g style="" transform="translate(0,38.390625)"><foreignObject width="94.8203125" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 156px; text-align: center;"><span style=""><p>+luceneIndex</p></span></div></foreignObject></g><g style="" transform="translate(0,63.984375)"><foreignObject width="124.1796875" height="25.59375"><div xmlns="http://www.w3.org/1999/xhtml" style="display: table-cell; white-space: nowrap; line-height: 1.5; max-width: 191px; text-align: center;"><span style=""><p>+nodeAssignment</p></span></div></foreignObject></g></g><g transform="translate(-74.28515625, 112.78125)"></g><g style=""><path d="M-86.28515625 -37.59375 C-19.06052492834101 -37.59336045077599, 48.16410639331798 -37.59297090155198, 86.28515625 -37.59275 M-86.28515625 -37.59375 C-21.12540187772231 -37.593372416199934, 44.03435249455538 -37.59299483239987, 86.28515625 -37.59275" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g><g style=""><path d="M-86.28515625 88.78125 C-22.772929567208266 88.78161803680635, 40.73929711558347 88.7819860736127, 86.28515625 88.78225 M-86.28515625 88.78125 C-35.59591053543035 88.7815437309725, 15.093335179139302 88.78183746194499, 86.28515625 88.78225" stroke="#4abf8a" stroke-width="1.3" fill="none" stroke-dasharray="0 0" style=""></path></g></g></g></g></g><defs></defs><defs></defs><linearGradient id="mermaid-334-gradient" gradientUnits="objectBoundingBox" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" stop-color="#4abf8a" stop-opacity="1"></stop><stop offset="100%" stop-color="#000" stop-opacity="1"></stop></linearGradient></svg>

### Elasticsearch 分布式架构设计

Elasticsearch 的架构是一个 **分布式、去中心化** 的集群。

- **集群协调：** 集群中的节点会选举产生一个 **Master Node** 。Master Node 负责管理集群范围的元数据和状态，如创建/删除索引、增加/删除节点、分片分配等。 **Master Node 不参与具体的文档索引和搜索请求的路由和处理（除了将请求转发给 Coordinating Node）。**
- **数据分布：** 一个索引创建时，会确定其 Primary Shard 的数量。Elasticsearch 会将每个 Primary Shard 和其 Replica Shard **尽可能地分散** 到不同的 Node 上，保证数据的高可用。
- **Document 路由到 Shard：** 当索引一个文档时，Elasticsearch 会根据文档的 ID 和 Primary Shard 的数量，通过一个 **路由算法** （默认是 `hash(document_id) % primary_shard_count` ）计算出该文档应该被存储在哪个 Primary Shard 上。这保证了具有相同 ID 的文档总是被发送到同一个 Primary Shard。
- **基于 Lucene：** Elasticsearch 底层使用 Lucene 作为其核心的 **全文索引库** 。Elasticsearch 在 Lucene 的基础上构建了分布式层、RESTful API、高可用、可伸缩、实时分析等功能。每个 Shard 本质上就是一个独立的 Lucene 索引。

### Elasticsearch 索引与搜索流程

### Elasticsearch 工作流程

理解 Elasticsearch 的工作原理，关键在于理解索引 (Indexing) 和搜索 (Searching) 的分布式流程。

#### 索引流程 (Indexing Workflow)

- **客户端发送索引请求：** 客户端向集群中的 **任一节点** (该节点充当 Coordinating Node) 发送索引文档的请求 (PUT/POST /index\_name/\_doc/document\_id)。
- **Coordinating Node 路由请求：** Coordinating Node 根据文档 ID 和路由算法，计算出该文档应该存储在哪个 **Primary Shard** 上。然后将请求转发到该 Primary Shard 所在的节点。
- **Primary Shard 处理索引：** Primary Shard 接收请求，将文档写入到本地 Lucene 索引的事务日志 (Translog)，并添加到内存缓冲区。然后刷新到 Lucene 段 (Segment) 中（近乎实时）。
- **Primary Shard 复制到 Replica Shards：** Primary Shard 并行地将文档 **复制** 到其所有 Replica Shard 所在的节点。
- **Replica Shard 确认：** Replica Shard 接收复制的数据，写入自己的 Translog，并刷新到自己的 Lucene 段。完成后向 Primary Shard 发送确认。
- **Primary Shard 确认：** Primary Shard 收到所有（或配置的最小数量）Replica Shard 的确认后，向 Coordinating Node 发送成功响应。
- **Coordinating Node 确认：** Coordinating Node 向客户端发送成功响应。

> **💡 核心提示** ：Elasticsearch 的"近实时"特性来自 Refresh 机制——数据写入内存 Buffer 后，默认每 1 秒刷新到 Segment 才可被搜索。如果需要严格实时写入，可以手动调用 `_refresh` API，但会牺牲写入性能。

#### 搜索流程 (Search Workflow)

- **客户端发送搜索请求：** 客户端向集群中的 **任一节点** (该节点充当 Coordinating Node) 发送搜索请求。
- **Coordinating Node 分发请求 (Query Phase - Scatter)：** Coordinating Node 将搜索请求 **分发** 到目标索引的所有 **Primary Shard 和 Replica Shard** (优先选择负载较低的副本)。
- **Shards 执行搜索并返回本地结果：** 每个 Shard 接收请求，在本地 Lucene 索引上执行搜索查询。它们计算匹配文档的相关度分数，并返回 **本地的、排序后的匹配文档 ID 列表** （包含分数和必要的排序信息）给 Coordinating Node。
- **Coordinating Node 归集、排序与合并：** Coordinating Node 接收来自所有 Shard 的本地结果。它将这些结果 **归集** 起来，根据相关度分数或其他排序规则进行 **全局排序** 和合并。此时，Coordinating Node 只知道哪些文档匹配，以及它们的排序，但还没有文档的完整内容。
- **Coordinating Node 获取原始 Document (Fetch Phase - Gather)：** Coordinating Node 确定了最终需要返回给客户端的文档的顺序和 ID 列表后，它会根据文档 ID，向这些文档所在的 **Primary Shard 或其对应的 Replica Shard** 发送请求， **获取原始的 Document 内容** 。
- **返回最终结果：** Coordinating Node 收集所有需要的原始 Document 内容，构建最终的搜索结果集，并返回给客户端。
- **Query Phase (Scatter)：** 散射阶段，将请求分发到所有相关 Shard，Shards 执行本地查询并返回轻量级结果。
- **Fetch Phase (Gather)：** 收集阶段，根据 Query Phase 结果，从相关 Shard 获取完整的 Document 内容。

#### 更新/删除流程 (简述)

更新和删除操作也是先路由到 Primary Shard，Primary Shard 执行操作并复制到 Replica Shard。Elasticsearch 中的更新和删除是"伪"操作，实际上是标记旧文档为删除并在新的 Lucene 段中写入新版本文档。

> **💡 核心提示** ：ES 中的删除是"标记删除"——旧文档不会被立即物理删除，只是在 `.del` 文件中标记为已删除。随着 Segment Merge 的进行，被标记删除的文档才会被真正清理。这也是为什么频繁更新/删除会导致磁盘空间膨胀的原因之一。

### Elasticsearch 核心功能回顾

- **分片 (Sharding)：** 实现数据水平扩展和并行处理。
- **副本 (Replication)：** 实现数据高可用和读请求负载分担。
- **索引 (Indexing)：** 将 Document 写入到 Lucene 索引，支持全文检索和结构化查询。
- **搜索 (Searching)：** 基于 Lucene 的强大搜索能力，支持多种查询类型。
- **聚合 (Aggregations)：** 对搜索结果或全量数据进行实时统计分析。
- **REST API：** 提供统一、方便的 HTTP 接口进行所有操作。
- **可伸缩性 (Scalability)：** 通过增加或减少节点，Elasticsearch 自动进行分片迁移和重新分配。
- **弹性与容错 (Resilience & Fault Tolerance)：** Master 选举、Replica 提升、分片重分配等机制保证节点故障时服务可用。

### Elasticsearch 常见应用场景

- **日志和事件数据分析 (ELK Stack)：** Elasticsearch 是 ELK Stack (Elasticsearch, Logstash, Kibana) 的核心，用于存储、索引和搜索日志和事件数据，进行实时监控和分析。
- **全文检索：** 网站搜索、文档管理系统、电商商品搜索等。
- **应用性能监控 (APM)：** 存储和分析应用产生的 Trace、Metrics 等数据。
- **指标分析：** 存储时序数据进行监控和报警。
- **业务分析：** 对业务数据进行多维度聚合分析。

### Elasticsearch vs 关系型数据库 对比分析

| 特性 | Elasticsearch | 关系型数据库 (RDBMS) | 推荐指数 |
| --- | --- | --- | --- |
| **核心模型** | 面向文档 (Document Oriented) | 面向关系 (Relational) | \- |
| **数据结构** | JSON 文档 (灵活，支持嵌套) | 表 (严格的行和列) | \- |
| **底层索引** | 倒排索引 (Inverted Index) | B-Tree / B+Tree | \- |
| **查询语言** | RESTful API + Query DSL (JSON) | SQL | \- |
| **主要用途** | 全文检索、实时分析、日志处理 | 事务处理 (ACID)、结构化数据管理 | \- |
| **Schema** | 默认动态 Mapping，可显式定义 | 严格的 Schema (建表时定义) | \- |
| **事务** | 仅支持单文档原子操作 | 支持跨多行、多表的 ACID 事务 | \- |
| **扩展性** | 天然分布式，易于水平扩展 | 垂直扩展为主，水平扩展复杂 | \- |
| **数据一致性** | 最终一致性 (副本异步复制) | 强一致性 | \- |
| **适用场景** | 搜索、日志分析、聚合统计 | 核心业务数据、交易、权限管理 | \- |
| **推荐方案** | 作为搜索/分析引擎 | 作为事务数据存储 | 两者配合使用 |

### 理解 Elasticsearch 架构与使用方式的价值

- **掌握分布式搜索/分析原理：** 深入理解分片、副本、倒排索引等核心概念在分布式环境下的应用。
- **构建高吞吐/低延迟应用：** 知道如何利用 ES 实现快速的数据写入和查询。
- **理解分布式系统设计：** 学习 ES 如何通过 Master 选举、分片复制、请求路由等机制实现高可用和可伸缩性。
- **排查 ES 集群问题：** 根据架构和工作流程，定位索引慢、查询慢、集群不稳定等问题。
- **对比选型：** 能够清晰地对比 ES 和 RDBMS 的优劣，在实际项目中做出合理的选型。
- **应对面试：** Elasticsearch 是当前数据处理领域的热点，其架构和核心概念是高频考点。

### Elasticsearch 为何是面试热点

- **数据处理基础设施：** 在日志、搜索、分析领域的广泛应用使其成为必备知识。
- **分布式系统代表：** 其架构涉及分布式、高可用、一致性等核心原理。
- **原理独特：** 倒排索引、分片、副本、Query/Fetch 阶段等概念与 RDBMS 截然不同。
- **与 RDBMS 对比：** 这是最常见的面试问题，考察候选人对不同数据库类型及其适用场景的理解。
- **ELK Stack 核心：** 许多公司使用 ELK 进行日志分析，理解 ES 是基础。

### 面试问题示例与深度解析

- **什么是 Elasticsearch？它解决了关系型数据库在哪些方面的不足？** (定义分布式搜索/分析引擎，解决 RDBMS 在全文检索、实时分析、非结构化数据、大规模扩展性上的不足)
- **请描述一下 Elasticsearch 的核心概念：Index, Document, Shard, Replica, Node, Cluster。它们之间的关系是什么？** (**核心！** 分别定义并说明关系：Cluster 由 Node 组成，Index 是逻辑集合，Index 分为 Shard (Primary)，Primary 有 Replica，Document 存储在 Shard 中)
- **请解释一下 Shard (分片) 和 Replica (副本) 的作用。为什么要分片和设置副本？** (**核心！** Shard：水平扩展，并行处理；Replica：高可用，读扩展。为什么需要：解决单机容量/性能瓶颈，提高可用性和读吞吐量)
- **请详细描述一个文档在 Elasticsearch 中被索引 (Indexing) 的流程。** (**核心！** 必考题。Client -> Coordinating Node -> 路由到 Primary Shard -> Primary 写入 Translog/内存/Segment -> 复制到所有 Replica -> Replica 确认 -> Primary 确认 -> Coordinating Node 确认 -> Client 成功响应)
- **请详细描述一个搜索请求在 Elasticsearch 中的处理流程。** (**核心！** 必考题。Client -> Coordinating Node -> 分发到所有 Shard (Query Phase) -> Shards 执行搜索并返回本地结果 (ID, 分数) -> Coordinating Node 归集排序 -> 从相关 Shard 获取原始 Document (Fetch Phase) -> 返回最终结果)
- **请解释一下 Query Phase 和 Fetch Phase 在搜索流程中的作用。** (Query: 散射/查询，找到匹配文档 ID 并排序；Fetch: 收集，根据 Query 结果获取原始文档内容)
- **Elasticsearch 的 Master Node 主要负责什么？它是否参与文档的索引和搜索？** (负责集群元数据管理，如索引创建/删除，分片分配。不直接参与文档的索引和搜索请求的处理)
- **什么是 Mapping？它的作用是什么？动态 Mapping 有什么优缺点？** (定义字段类型和索引方式。作用：控制数据存储/索引/搜索行为。动态 Mapping：方便但可能类型推断错误)
- **请对比一下 Elasticsearch 和关系型数据库在数据模型、索引、查询方式、适用场景等方面的区别。** (**核心！** 必考题。对比 Document vs 行/列，倒排索引 vs B+Tree，Query DSL vs SQL，搜索/分析 vs 事务/结构化)
- **什么是倒排索引 (Inverted Index)？它为什么适合全文检索？** (定义：词条到文档列表的映射。适合原因：能快速找到包含某个词条的所有文档)
- **你了解哪些 Elasticsearch 的常见应用场景？** (ELK Stack, 全文搜索, APM, 业务分析)

### 生产环境避坑指南

| 坑点 | 症状 | 解决方案 |
| --- | --- | --- |
| **分片数过多** | 集群状态膨胀，Master 节点压力大，查询变慢 | 单个 Index 的 Shard 数控制在几百以内，使用 Index Lifecycle Management (ILM) 自动滚动 |
| **分片数过少** | 无法充分利用多节点并行处理能力 | 根据数据量和节点数合理规划，一般单个 Shard 大小在 20-50GB |
| **动态 Mapping 失控** | 字段类型推断错误（如数字被推断为 `long` ），导致聚合失败 | 预先定义显式 Mapping，禁用动态 Mapping 或设置为 `strict` |
| **深分页 (Deep Pagination)** | `from + size` 超过 10000 时性能急剧下降 | 使用 `search_after` 或 `scroll` API 替代深分页 |
| **脑裂问题 (Split Brain)** | 集群出现两个 Master，数据不一致 | 设置 `discovery.zen.minimum_master_nodes = N/2 + 1` (ES 6.x)，ES 7+ 内置 Zen 2 已解决 |
| **Refresh 间隔不当** | 过于频繁影响写入，过于延迟影响实时性 | 保持默认 1s，大批量导入时可临时调大到 30s 提升吞吐 |
| **JVM Heap 配置错误** | OOM 或 GC 停顿导致节点假死 | Heap 设为物理内存的 50% 但不超过 31GB（避免压缩普通指针 OOPs 失效） |
| **Translog 持久化策略** | 默认 `request` 级别保证数据安全但影响写入性能 | 批量导入场景可临时改为 `async` ，导入完成后恢复 |
| **忽略 Segment Merge** | 磁盘空间暴涨，搜索性能下降 | 监控 `_segments` API，配置合理的 merge policy |

### 行动清单

1. **检查点** ：确认生产环境的 JVM Heap 配置不超过 31GB，并且设置了 `-XX:+UseG1GC` 和 `-XX:MaxGCPauseMillis=200` 。
2. **优化建议** ：为所有索引预先定义显式 Mapping，避免动态 Mapping 导致的类型推断错误。对于不需要全文检索的字段使用 `keyword` 而非 `text` 。
3. **架构审查** ：检查分片策略——Primary Shard 数是否在创建 Index 时就已规划好？单个 Shard 大小是否控制在 20-50GB 范围内？
4. **扩展阅读** ：推荐阅读 Elastic 官方博客文章 "How Sharding Works" 和 "Tuning for Throughput vs Latency"。
5. **实践建议** ：搭建一个 3 节点的本地 ES 集群，完整走通索引写入、搜索查询、聚合分析的完整流程，并使用 Kibana Dev Tools 进行实操练习。

### 总结

Elasticsearch 是一个功能强大的分布式搜索与分析引擎，它凭借基于 Lucene 构建的倒排索引、分片、副本、以及高效的分布式架构，解决了关系型数据库在全文检索、实时分析和大规模数据处理方面的不足。理解 Index, Document, Shard, Replica 等核心概念，掌握索引和搜索的分布式工作流程，以及它与 RDBMS 的关键区别，是掌握分布式搜索和分析技术栈的关键。