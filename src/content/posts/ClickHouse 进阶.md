---
title: ClickHouse 进阶
published: 2026-08-26
description: 'ClickHouse X Kafka'
image: 'https://cdn.wcxian.cc/img/20260826222239093.png'
tags: ['ClickHouse', 'OLAP', '数据库']
category: '技术分享'
draft: false 
lang: zh-CN
---
# ClickHouse 进阶实战

> 本文承接《[ClickHouse 入门](ClickHouse%20入门.md)》。基础篇已经介绍了 `store_events`、MergeTree 和常用分析 SQL，本文继续使用无人店 SaaS 场景，重点讲 Kafka 接入、物化视图、预聚合、性能优化、集群和运维，并完成一个可以直接展示的实时经营看板案例。

基础篇解决的是“数据能不能查”，本文要完成的是：

```text
事件进入 → 明细落地 → 小时聚合 → 看板发现异常 → 下钻到设备和版本
```

**一句话总结：进阶篇解决的是「数据怎么进来、查询怎么变快、结果怎么可信、故障怎么处理」四类生产问题，核心是把无人店 SaaS 的「Kafka 接入 → 物化视图预聚合 → 性能调优 → 集群高可用 → 运维监控 → 实时看板」串成一条可验证的闭环。**

读完后，你应该能：

- 说清 Kafka Engine、物化视图、`AggregatingMergeTree` 中间状态、跳数索引与 Projection 的原理；
- 独立搭建「明细表 + 小时聚合表」的预聚合链路，并用 `system.query_log` 对比两种查询的扫描量；
- 按「查询路径 → 分区/排序键 → SQL 改写 → 索引/投影 → 扩容」的顺序定位和优化慢 SQL；
- 判断何时需要集群，完成一次「发现异常门店 → 下钻设备版本 → 定位原因」的完整演示。

---

## 0. 从“能查”到“可生产”

生产环境里的 ClickHouse，通常同时面对四类问题：

| 问题         | 典型表现                     | 对应能力                     |
| ------------ | ---------------------------- | ---------------------------- |
| 数据怎么进来 | Kafka 重复、坏消息、消费延迟 | Schema、批量写入、隔离和重放 |
| 查询怎么变快 | 看板反复扫描数月明细         | 排序键、预聚合、查询验证     |
| 结果怎么可信 | 支付、退款和订单状态对不上   | 事实模型、幂等和时间口径     |
| 故障怎么处理 | 小 Part、慢 SQL、磁盘将满    | 系统表、TTL、备份和恢复      |

四类问题与本文章节的对应关系：

| 章节          | 对应问题     | 核心交付                      |
| ------------- | ------------ | ----------------------------- |
| 1. 实时接入   | 数据怎么进来 | Kafka Engine + 物化视图链路   |
| 2. 预聚合     | 查询怎么变快 | 小时聚合表与中间状态          |
| 3. 性能优化   | 查询怎么变快 | 排序键、SQL、索引、Projection |
| 4. 分布式集群 | 容量与可用性 | 分片、副本决策                |
| 5. 运维监控   | 故障怎么处理 | 慢 SQL、小 Part、TTL、备份    |
| 6. 实战看板   | 全部         | 可演示的完整案例              |

典型链路如下：

![165d42ba-4ea6-4040-88af-2d247bb17b86](https://cdn.wcxian.cc/img/20260826204855473.png)

设计指标前，先明确四件事：事实是什么、使用 `event_time` 还是 `ingest_time`、如何去重、迟到数据何时回补。否则查询再快，结果也未必能用于经营和对账。

---

## 1. 实时接入：Kafka + Materialized View

Kafka 负责传输、缓冲和重放；ClickHouse 负责落地和分析。常见链路是：

![c7e5c2ba-4f83-4a2c-9070-0e3a83719183](https://cdn.wcxian.cc/img/20260826205645548.png)

### 1.1 关键概念

| 概念           | 含义                  | 生产关注点             |
| -------------- | --------------------- | ---------------------- |
| Topic          | `store-events` 事件流 | 是否保留以便重放       |
| Partition      | 消息并行单元          | 决定消费并行上限       |
| Consumer Group | ClickHouse 消费组     | 测试、生产不能共用     |
| Offset         | 消费位置              | 故障恢复和重放依据     |
| Consumer Lag   | 尚未消费的消息量      | 持续增长说明下游跟不上 |

例如 Topic 有 6 个 Partition，但消费者只有 2 个，实际并行度最多为 2。Lag 增长时，不能只增加消费者，还要检查小 Part、后台合并和物化视图计算。

#### Kafka Engine 的工作机制

`ods_store_events_kafka` 这张表本身**不存数据**，它只负责「消费」：ClickHouse 从 Kafka 拉取消息，按 `kafka_format` 解析后，把每一批数据交给挂在这张表上的物化视图，由物化视图写入目标 MergeTree 表。因此：

- 直接查询 `ods_store_events_kafka` 通常只能看到刚消费、还没来得及转存的少量数据；
- 真正可查询的是目标表 `store_events`；
- 消费确认发生在数据成功写入目标表之后。Kafka 是「至少一次」语义，重复消费时物化视图会重复计算，所以目标表需要用 `event_id` 做幂等键（见 1.3 节）。

这也解释了为什么物化视图适合字段映射、默认值和简单 JSON 提取：逻辑越重，消费越慢，Lag 越高；复杂清洗应前移到 Kafka 或 Flink 预处理。

### 1.2 创建接入表

基础篇已经创建了 `analytics.store_events`，这里只展示核心结构：

```sql
-- ============================================================
-- 1. 创建 Kafka 引擎表（接入表）
--    作用：作为数据管道，从 Kafka 拉取消息，本身不存储数据。
--    注意：该表仅用于消费，查询它只能看到极少量未转存的缓存数据。
-- ============================================================
CREATE TABLE analytics.ods_store_events_kafka
(
    event_id UUID,
    tenant_id String,
    store_id String,
    member_id Nullable(UInt64),
    device_id Nullable(String),
    city String,
    event_name String,
    order_id Nullable(UInt64),
    product_id Nullable(UInt64),
    page_code String,
    event_time DateTime64(3, 'Asia/Shanghai'),
    properties String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'localhost:9092',          -- Kafka 集群地址
    kafka_topic_list = 'test-topic',               -- 订阅的 Topic
    kafka_group_name = 'clickhouse-local-consumer',-- 消费者组名（保证偏移量管理）
    kafka_format = 'JSONEachRow',                  -- 消息格式（每行一个 JSON）
    kafka_num_consumers = 1,                       -- 消费者线程数（可根据分区数调整）
    kafka_handle_error_mode = 'stream'; 

-- ============================================================
-- 2. 创建物化视图（数据管道核心）
--    作用：自动将 Kafka 接入表的数据转换并写入目标 MergeTree 表。
--    特点：视图本身不存储数据，每次 Kafka 新消息到达时触发写入。
-- ============================================================
CREATE MATERIALIZED VIEW analytics.mv_kafka_to_store_events
TO analytics.store_events
AS SELECT
    event_id, tenant_id, store_id, member_id, device_id, city,
    event_name, order_id, product_id, page_code, event_time, properties
FROM analytics.ods_store_events_kafka;   -- 数据来源为 Kafka 引擎表
```

验证时查询落地后的 MergeTree 表：

```sql
-- ============================================================
-- 3. 验证查询（查询目标 MergeTree 表，而非 Kafka 表）
--    目的：确认数据已成功写入，并查看最近事件。
-- ============================================================
SELECT event_time, store_id, event_name, order_id, ingest_time
FROM analytics.store_events
WHERE tenant_id = 'tenant-a'
ORDER BY ingest_time DESC
LIMIT 20;
```

执行后应能看到最近写入的事件，`ingest_time` 由写入时间自动填充，例如：

```sql
   ┌──────────────event_time─┬─store_id─────┬─event_name──────┬─order_id─┬─────────ingest_time─┐
1. │ 2026-08-25 12:30:00.000 │ store-hz-001 │ page_view       │     ᴺᵁᴸᴸ │ 2026-08-25 13:51:38 │
2. │ 2026-07-21 10:00:01.123 │ store-hz-001 │ entry_detected  │     ᴺᵁᴸᴸ │ 2026-07-24 11:34:59 │
3. │ 2026-07-21 10:01:03.456 │ store-hz-001 │ product_view    │     ᴺᵁᴸᴸ │ 2026-07-24 11:34:59 │
4. │ 2026-07-21 10:02:20.000 │ store-hz-001 │ payment_success │   200001 │ 2026-07-24 11:34:59 │
   └─────────────────────────┴──────────────┴─────────────────┴──────────┴─────────────────────┘
```

### 1.3 重复、坏消息与重放

Kafka 常见的是至少一次消费，重试可能产生重复事件。因此：

- 不可变事件用 `event_id` 做幂等键；
- 订单状态用 `order_id + version` 表示版本；
- 非法 JSON、字段类型错误进入死信 Topic 或隔离表；
- 重放前确认 offset、时间范围和去重策略。

例如同一条 `order_id` 的支付回调被重复投递时，只有 `version` 更大的记录才被接受；重复的 `event_id` 直接跳过。这样即使 Kafka 重复投递，明细表也不会出现双倍计数。

#### 配置一：防止“重复消息”（幂等表配置）

在 ClickHouse 中，用 **`ReplacingMergeTree`** + **版本号** 来处理 Kafka 重复投递。不需要改任何代码，只需调整表结构。

**1. 修改明细表引擎（或新建表）**
将 `store_events` 的引擎改为 `ReplacingMergeTree`，并把 `event_id` 放在排序键末尾，指定 `ingest_time` 作为版本（保留最新的一条）。

```sql
-- 如果表已存在，用此语句修改（会阻塞写入，建议停写或低峰期操作）
ALTER TABLE analytics.store_events 
MODIFY ENGINE = ReplacingMergeTree(ingest_time)
PARTITION BY toYYYYMM(event_date)
ORDER BY (tenant_id, store_id, event_date, event_id);  -- event_id 必须放最后
```

**2. 确保插入时带版本号**
在物化视图里，用 `now()` 作为版本，这样相同 `event_id` 重复到达时，后到的 `ingest_time` 更大，后台合并时就会覆盖旧数据。

```sql
-- 修改物化视图，主动填充 ingest_time
CREATE MATERIALIZED VIEW analytics.mv_kafka_to_store_events
TO analytics.store_events
AS SELECT
    event_id, tenant_id, store_id, member_id, device_id, city,
    event_name, order_id, product_id, page_code, event_time, properties,
    now() AS ingest_time   -- 显式写入当前时间作为版本号
FROM analytics.ods_store_events_kafka;
```

#### 配置二：隔离“坏消息”（死信管道配置）

通过修改 Kafka 引擎表的错误处理模式，自动捕获解析失败的消息。

**1. 开启 Kafka 错误流模式（核心开关）**

这边其实没办法直接修改，只能重新建表。但是因为是引擎表，所以数据也不对丢失，直接重建就好了。

```sql
ALTER TABLE analytics.ods_store_events_kafka
MODIFY SETTING kafka_handle_error_mode = 'stream';
```

**2. 创建死信隔离表（存放坏消息）**

```sql
CREATE TABLE analytics.ods_kafka_dead_letter
(
    topic String,
    partition UInt32,
    offset UInt64,
    raw_payload String,
    error_type String,
    error_message String,
    retry_count UInt32 DEFAULT 0,
    failed_at DateTime DEFAULT now()
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(failed_at)
ORDER BY (topic, failed_at);
```

**3. 创建物化视图，自动将坏消息写入死信表**

```sql
CREATE MATERIALIZED VIEW analytics.mv_kafka_errors_to_dlq
TO analytics.ods_kafka_dead_letter
AS SELECT
    _topic AS topic,
    _partition AS partition,
    _offset AS offset,
    _raw_message AS raw_payload,
    _error AS error_type,
    _error AS error_message,   -- 这里按需细化，你也可以只取 _error
    now() AS failed_at
FROM analytics.ods_store_events_kafka
WHERE length(_error) > 0;      -- 只要包含错误信息，就自动分流
```

配置完成后，正常消息照常进入 `store_events`，坏消息自动进入 `ods_kafka_dead_letter`，互不影响。

<iframe width="100%" height="468" src="https://cdn.wcxian.cc/img/20260825163012320.mp4" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

物化视图适合字段映射、默认值和简单 JSON 提取，不适合复杂 Join 或完整 ETL。流量突增时，应先看 Partition、写入批次、Part、Merge 和磁盘 IO，再决定扩消费者、扩分片还是前移清洗。

---

## 2. 物化视图与预聚合：让指标更快

看板每 5 秒刷新一次时，如果每次都扫描 90 天明细，即使 ClickHouse 很快也会浪费资源。常见分层是：

| 数据层     | 用途                         |
| ---------- | ---------------------------- |
| 原始事件层 | 明细追溯、临时分析、问题定位 |
| 小时聚合层 | 看板、趋势图和告警           |
| 最终快照层 | 财务日报和结算对账           |

```text
store_events 明细表 → 物化视图 → store_event_1h 小时聚合表 → 看板
```

#### 物化视图为什么快

普通查询是「查询时计算」：每次请求都扫描明细并现场 `GROUP BY`；物化视图是「写入时计算」：明细写入 `store_events` 的同时，后台把聚合结果**增量**写入 `store_event_1h`，查询时直接读结果：

```text
普通查询：明细表 → 每次查询现场聚合 → 返回
物化视图：明细写入 → 增量聚合 → 结果表 → 查询直接读结果
```

### 2.1 创建小时聚合表

`AggregatingMergeTree` 保存聚合函数的中间状态，查询时使用对应的 `Merge` 函数：

```sql
CREATE TABLE analytics.store_event_1h
(
    hour DateTime,
    tenant_id LowCardinality(String),
    store_id LowCardinality(String),
    event_name LowCardinality(String),
    event_count AggregateFunction(count),
    member_uv AggregateFunction(uniqCombined64, Nullable(UInt64))
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(hour)
ORDER BY (tenant_id, store_id, hour, event_name);

CREATE MATERIALIZED VIEW analytics.mv_store_events_to_1h
TO analytics.store_event_1h
AS SELECT
    toStartOfHour(event_time) AS hour,
    tenant_id, store_id, event_name,
    countState() AS event_count,
    uniqCombined64State(member_id) AS member_uv
FROM analytics.store_events
GROUP BY hour, tenant_id, store_id, event_name;
```

#### 中间状态：State 与 Merge

`AggregateFunction(count)` 这一列保存的不是数字，而是聚合函数的**中间状态**（Intermediate State）。写入时用 `countState()` 把状态存进去，查询时用 `countMerge()` 取出结果：

```text
写入：countState()            → 每个分组存一个「计数状态」
查询：countMerge(event_count) → 把状态还原成最终数字
```

`uniqCombined64State(member_id)` 同理，保存去重算法的哈希状态。这样设计有两个好处：

1. **增量合并**：新数据写入后，状态可以直接叠加，不用重扫历史明细；
2. **可继续聚合**：多个小时的 UV 状态合并后仍能算准确，这是 2.2 节「UV 不能简单相加」的技术前提。

查询小时数据：

```sql
SELECT hour,
       countMerge(event_count) AS event_count,
       uniqCombined64Merge(member_uv) AS member_uv
FROM analytics.store_event_1h
WHERE tenant_id = 'tenant-a'
  AND store_id = 'store-hz-001'
  AND event_name = 'entry_detected'
  AND hour >= '2026-07-21 00:00:00'
  AND hour < '2026-07-22 00:00:00'
GROUP BY hour
ORDER BY hour;
```

| hour                | event_count | member_uv |
| :------------------ | ----------- | --------- |
| 2026-07-21 10:00:00 | 3           | 3         |

`store-hz-001` 的 3 次进店都发生在 10 点，所以只有一行。真实环境数据量越大，聚合表的优势越明显。

UV 不能把每小时 UV 简单相加得到全天 UV，应继续合并聚合状态，或单独建设日级 UV 表。

### 2.2 明细表与聚合表对比

这边插入十万条记录：

```sql
INSERT INTO analytics.store_events
    (event_id, tenant_id, store_id, member_id, device_id, city, event_name,
     order_id, product_id, page_code, event_time, properties)
SELECT
    generateUUIDv4(), 
    'tenant-a', 
    concat('store-', toString(number % 100)),  -- 生成 100 个不同的门店
    toUInt64(number % 100000), 
    concat('pos-', toString(number % 20)), 
    '杭州',
    if(number % 10 = 0, 'payment_success',
       if(number % 10 = 1, 'payment_failed', 'entry_detected')),
    if(number % 10 IN (0, 1), toUInt64(number), NULL),
    if(number % 10 IN (0, 1), toUInt64(number % 1000), NULL),
    'checkout',
    toDateTime64('2026-07-21 00:00:00', 3) + toIntervalSecond(number % 86400),
    '{}'
FROM numbers(100000);
```

直接扫描明细表：

```sql
SELECT toStartOfHour(event_time) AS hour,
       countIf(event_name = 'entry_detected') AS entry_count
FROM analytics.store_events
WHERE tenant_id = 'tenant-a'
  AND event_date >= '2026-07-01'
  AND event_date < '2026-08-01'
GROUP BY hour;

-- 24 行已获取 - 0.013s, 2026-08-25 17:55:10
```

查询聚合表：

```sql
SELECT hour,
       countMerge(event_count) AS entry_count
FROM analytics.store_event_1h
WHERE tenant_id = 'tenant-a'
  AND event_name = 'entry_detected'
  AND hour >= '2026-07-01 00:00:00'
  AND hour < '2026-08-01 00:00:00'
GROUP BY hour;

-- 24 行已获取 - 0.01s, 2026-08-25 17:56:47
```

这边可能数据太少了，看出差距，通过EXPLAIN看一下

| 表                      | Parts（数据片段） | Granules（索引粒度） | 含义                                                  |
| :---------------------- | :---------------- | :------------------- | :---------------------------------------------------- |
| 明细表 `store_events`   | 2                 | **13**               | 为了计算 24 小时的进店人数，扫描了 **13 个** 数据粒度 |
| 聚合表 `store_event_1h` | 2                 | **2**                | 只扫描了 **3 个** 数据粒度                            |

聚合表扫描的 Granule 数量从 **13 降到 2**，减少了约 **80%** 的数据扫描量。虽然没能做到极致的 1 个 Granule（因为你的 10 万条数据分布在 24 个小时内，且 `entry_detected` 事件在多个时段都有分布，聚合表需要读取这些匹配的小时数据块），但这**已经直观展示了“写入时计算”的加速效果**。

在生产环境（千万级数据）中，这个差距会被放大到**数百倍甚至上千倍**（明细表扫几万个 Granule，聚合表只扫十几个）。

![image-20260825180003865](https://cdn.wcxian.cc/img/20260825180005306.png)

### 2.3 迟到数据与指标口径

```text
10:00  顾客支付成功
10:08  设备恢复网络，消息进入 ClickHouse
10:20  顾客退款
次日    按最终订单快照生成财务日报
```

- **迟到数据**：事件发生时间 `event_time` 远早于实际写入时间 `ingest_time`（例如网络断线后补传）。
- **实时口径**：按数据进入系统的时间（`ingest_time`）累计，适用于看板，允许延迟。
- **最终口径**：按业务截止时间（如 T+1）重新计算，剔除退款、重复等，适用于财务对账。

物化视图在 `INSERT` 明细时**自动增量更新**聚合表，因此即使 `event_time` 是过去的时间，只要数据插入，聚合表就会重新计算对应小时的聚合结果，保证近实时看板能“回填”历史小时数据。

两种口径的适用范围：

| 口径               | 典型使用方     | 说明                                   |
| ------------------ | -------------- | -------------------------------------- |
| 实时口径（近实时） | 经营看板、告警 | 按事件进入时间累计，允许一定延迟和重复 |
| 最终口径（结算）   | 财务日报、对账 | 按截止时间和退款规则生成，事后校正     |

### 2.4 聚合表与 Projection

| 场景                 | 方案              |
| -------------------- | ----------------- |
| 指标和粒度固定       | 聚合表 + 物化视图 |
| 稳定的查询路径       | Projection        |
| 复杂清洗或跨表补维度 | Flink / ETL       |
| 临时探索             | 明细表            |

- **Projection**：在同一个 MergeTree 表中存储的“预聚合投影”，类似物化视图，但**与明细表绑定**，查询优化器可自动选择使用投影（如果查询模式匹配）。
- 优势：无需维护独立的聚合表，查询自动路由；劣势：写入时额外存储和计算开销，且投影定义不能轻易修改。
- 适用场景：固定查询路径，且不想额外管理聚合表。

#### 为 `store_events` 添加一个日聚合投影

```sql
ALTER TABLE analytics.store_events
ADD PROJECTION p_store_daily
(
    SELECT store_id, event_date,
           countIf(event_name = 'entry_detected') AS entry_count,
           countIf(event_name = 'payment_success') AS payment_count
    GROUP BY store_id, event_date
);
```



这个投影是“逻辑定义”，此时尚未实际构建数据。

#### 手动构建投影数据（物化）

```sql
ALTER TABLE analytics.store_events
MATERIALIZE PROJECTION p_store_daily;
```

#### 验证 Projection 是否被查询优化器使用

执行一条与投影定义匹配的查询，并带上 `EXPLAIN` 查看执行计划：

```sql
EXPLAIN indexes = 1
SELECT store_id, event_date,
       countIf(event_name = 'entry_detected') AS entry_count,
       countIf(event_name = 'payment_success') AS payment_count
FROM analytics.store_events
WHERE tenant_id = 'tenant-a'      -- 注意：投影定义中没有 tenant_id，但查询可以带过滤条件
  AND event_date = '2026-07-21'
GROUP BY store_id, event_date;
```

![image-20260825202556549](https://cdn.wcxian.cc/img/20260825202558164.png)

---

## 3. 数据建模与性能优化：先减少扫描

优化顺序应是：**查询路径 → 分区和排序键 → SQL 改写 → EXPLAIN 验证 → 聚合表/索引/Projection → 扩容。**

### 3.1 准备性能实验数据

只有几条数据时看不出优化效果，可以生成 500 万条测试事件：

```sql
INSERT INTO analytics.store_events
    (event_id, tenant_id, store_id, member_id, device_id, city, event_name,
     order_id, product_id, page_code, event_time, properties)
SELECT
    generateUUIDv4(),
    concat('tenant-', toString(number % 3)),                          -- 3 个租户
    concat('store-', toString(number % 200)),                         -- 200 个门店
    toUInt64(number % 100000),                                       -- 会员 ID 重复出现
    concat('pos-', toString(number % 50)),                           -- 50 个设备
    if(number % 10 < 3, '杭州', if(number % 10 < 6, '上海', '北京')), -- 3 个城市
    if(number % 10 = 0, 'payment_success',
       if(number % 10 = 1, 'payment_failed', 'entry_detected')),     -- 事件类型
    if(number % 10 IN (0, 1), toUInt64(number), NULL),               -- order_id 仅支付事件有值
    if(number % 10 IN (0, 1), toUInt64(number % 2000), NULL),        -- product_id
    'checkout',
    toDateTime64('2026-07-01 00:00:00', 3) + toIntervalSecond(number % 259200), -- 分散在 3 天（72 小时）
    '{}'
FROM numbers(5000000);  -- ← 改这个数字调整行数，例如 5000000 为 500 万
```

### 3.2 宽表、类型与排序键

OLAP 通常使用宽表，减少大表 Join。租户、门店、设备版本、商品类目、支付通道和金额等高频字段，可以在写入时补齐。

| 不推荐             | 推荐                     |
| ------------------ | ------------------------ |
| `String` 存数值 ID | `UInt64`                 |
| `String` 存金额    | `Decimal(18, 2)`         |
| `String` 存时间    | `Date` / `DateTime64`    |
| 普通字符串枚举     | `LowCardinality(String)` |

`PARTITION BY` 负责分区管理和粗粒度裁剪；`ORDER BY` 决定 Part 内的物理顺序。常见排序键是：

```sql
ORDER BY (tenant_id, store_id, event_date, event_time, event_name)
```

**业务闭环**：运营按「租户 → 门店 → 日期 → 时段」下钻时，分区先按月份裁剪，排序键再按租户、门店和日期跳过无关数据块；`event_time` 放在 `event_name` 前面，是因为时段范围查询比事件名等值查询更常见。类型和枚举也影响压缩率：`LowCardinality(String)` 的城市、事件名列，压缩后占用通常远小于普通 `String`。

### 3.3 一条慢 SQL 的优化

不推荐：

```sql
SELECT *
FROM analytics.store_events
WHERE toDate(event_time) >= '2026-07-01'
  AND toDate(event_time) < '2026-08-01'
  AND event_name = 'payment_success';
```

![image-20260825223511650](https://cdn.wcxian.cc/img/20260825223539842.png)

![image-20260825223949146](https://cdn.wcxian.cc/img/20260825223950925.png)

改为只取需要的列，直接过滤日期、租户和排序键前缀：

```sql
SELECT store_id, count() AS payment_count
FROM analytics.store_events
WHERE tenant_id = 'tenant-a'
  AND event_date >= '2026-07-01'
  AND event_date < '2026-08-01'
  AND event_name = 'payment_success'
GROUP BY store_id
ORDER BY payment_count DESC;
```

![image-20260825223532213](https://cdn.wcxian.cc/img/20260825223543508.png)

![image-20260825223902241](https://cdn.wcxian.cc/img/20260825224031790.png)

用 `EXPLAIN indexes = 1` 验证分区、Part 和 Granule 是否被裁剪，输出大致是（数字为示例，取决于数据量和分区状态）：

| 查询类型                                                     | Parts | Granules | 扫描数据量         | 性能表现 |
| :----------------------------------------------------------- | :---- | :------- | :----------------- | :------- |
| **低效查询**（`SELECT *`, `toDate(event_time)`, 无 `tenant_id`） | 1     | **623**  | 几乎全表扫描       | ❌ 慢     |
| **优化查询**（只取 `store_id`, `event_date`, 带 `tenant_id`） | 1     | **13**   | 仅扫描 2% 的数据块 | ✅ 极快   |

`Parts: 1/80` 表示 80 个 Part 只读取了 1 个，`Granules: 12/1873` 表示 1873 个索引粒度只读了 12 个，说明分区和排序键裁剪生效；如果显示扫描了全部 Part，就要回到排序键和查询条件上找原因。再从 `system.query_log` 对比 `read_rows`、`read_bytes` 和耗时。看板只需要小时趋势时，应直接查询聚合表，而不是反复扫描明细。

### 3.4 跳数索引、Projection 与 JSON

随机查询 `product_id` 或 `device_id` 是常见场景，但这两个字段不在排序键中。我们可以加一个 `Bloom Filter` 跳数索引来加速。

```sql
-- 1. 添加索引定义
ALTER TABLE analytics.store_events
ADD INDEX idx_product_id product_id TYPE bloom_filter(0.01) GRANULARITY 4;

-- 2. 物化索引（构建数据，这一步可能会消耗一点时间，但你的 10 万行数据几秒内完成）
ALTER TABLE analytics.store_events
MATERIALIZE INDEX idx_product_id;
```

跳数索引不是 B+Tree，它按 Granule 粒度记录每块数据的特征（例如 Bloom Filter 判断该块是否可能包含某个值），查询时跳过不满足条件的块。是否有效取决于数据分布：`product_id` 这类选择性好、分布随机的列效果明显；取值很少或几乎全表出现的列建了也没用。因此建完后必须用真实 SQL 验证，不能只确认索引存在。

验证步骤：先执行 `ALTER TABLE analytics.store_events MATERIALIZE INDEX idx_product_id`（或等待后台构建），再用原 SQL 重查，对比 `system.query_log` 中 `read_rows` 是否下降；也可以用下面的 SQL 查看索引是否已构建、占用多大空间：

```sql
SELECT 
    name, 
    type, 
    granularity, 
    formatReadableSize(index_size) AS index_size
FROM system.data_skipping_indices
WHERE table = 'store_events';
```

---

## 4. 分布式集群与高可用：什么时候需要集群

单机容量不足、峰值读写无法满足 SLA，或核心看板不能接受单点故障时，再考虑集群：

| 问题             | 优先方案                     |
| ---------------- | ---------------------------- |
| 容量和计算不足   | 分片                         |
| 节点故障不可接受 | 副本和故障转移               |
| SQL 扫描量过大   | 先改 SQL、聚合和排序键       |
| Kafka Lag 上升   | 先查分区、批次、Merge 和磁盘 |

```text
Distributed 表
      ↓
Shard 1（Replica 1 / 2）
Shard 2（Replica 1 / 2）
      ↓
ClickHouse Keeper 负责副本协调
```

三个组件各司其职，不要混为一谈：

| 组件              | 解决什么   | 说明                                             |
| ----------------- | ---------- | ------------------------------------------------ |
| 分片（Shard）     | 容量与并行 | 数据按分片键分散到多台机器，SQL 并行处理         |
| 副本（Replica）   | 可用性     | 同一分片的数据存多份，节点故障自动切换           |
| ClickHouse Keeper | 协调       | 管理副本元数据和分布式 DDL（取代早期 ZooKeeper） |

统一查询入口使用 `Distributed`：

```sql
CREATE TABLE analytics.store_events_all ON CLUSTER ck_cluster
AS analytics.store_events_local
ENGINE = Distributed(
    'ck_cluster', 'analytics', 'store_events_local',
    cityHash64(tenant_id, store_id)
);
```

有真实集群时，可以用 `hostName()` 观察各节点参与查询的本地行数：

```sql
SELECT hostName() AS node, store_id, count() AS cnt
FROM analytics.store_events_all
WHERE tenant_id = 'tenant-a'
  AND event_date = '2026-07-21'
GROUP BY node, store_id
ORDER BY cnt DESC;
```

如果某个分片完全没有结果，检查分片键和该节点上的本地表数据。分片键要均匀，也要考虑查询路径；副本解决可用性，不等于备份，也不能让低效 SQL 自动变快。

---

## 5. 运维监控与问题排查

运维重点关注四类指标：

| 类别     | 指标                   | 能发现什么         |
| -------- | ---------------------- | ------------------ |
| 接入     | Consumer Lag、失败消息 | 数据延迟或中断     |
| 写入     | Part、Merge、磁盘      | 小批写入和存储压力 |
| 查询     | P95/P99、`read_rows`   | 看板变慢和无界查询 |
| 数据质量 | 延迟、重复率、对账差异 | 指标失真           |

排查前先认识三张系统表：`system.query_log` 记录每次查询的耗时、扫描行数和读取字节；`system.parts` 展示每个表的 Part 数量与大小；`system.merges` 展示正在进行的后台合并。排查顺序通常是：先用 `query_log` 找慢 SQL，再看 `parts` 找小 Part，最后用 `merges` 确认合并是否被卡住。

### 5.1 用 query_log 定位慢 SQL

```sql
SELECT
    query_duration_ms, read_rows,
    formatReadableSize(read_bytes) AS read_size, query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY read_rows DESC
LIMIT 10;
```

结果大致是（`query` 字段过长时可用 `substring(query, 1, 80)` 截断展示）：

![image-20260826203340446](https://cdn.wcxian.cc/img/20260826203342354.png)

发现扫描量大的 SQL 后，按“补时间条件、补租户条件、减少列、检查排序键、改查聚合表”的顺序处理，并对比明细表和聚合表的 `read_rows`、`read_bytes` 与耗时。

### 5.2 用 system.parts 发现小 Part

```sql
SELECT
    partition,
    count() AS active_parts,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size
FROM system.parts
WHERE database = 'analytics'
  AND table = 'store_events'
  AND active
GROUP BY partition
ORDER BY active_parts DESC;
```

结果大致是：

![image-20260826203412644](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20260826203412644.png)

如果行数不多却有大量 Part，通常是写入批次过小。应改成按消息数或时间窗口攒批，再观察 `system.merges`、写入延迟和 Consumer Lag 是否恢复。

### 5.3 TTL 与备份

日志类数据可以使用 TTL：

```sql
CREATE TABLE analytics.api_access_log
(
    request_id UUID,
    request_time DateTime,
    service_name LowCardinality(String),
    status_code UInt16,
    cost_ms UInt32,
    message String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(request_time)
ORDER BY (service_name, request_time)
TTL request_time + INTERVAL 90 DAY DELETE;
```

TTL 通常随后台合并异步执行，不是立即删除：可以查询 `system.parts` 观察数据是否消失，或用 `SELECT * FROM system.ttl_merges` 查看 TTL 合并任务，磁盘空间释放也需要时间。副本不是备份，因为误写、误删和结构变更也会同步到副本。重要数据应备份到独立存储，并定期恢复一个测试分区，核对行数、订单数和金额。

---

## 6. 实践案例：搭建无人店实时经营看板

这个案例用少量固定数据展示完整效果：统计门店进店 UV、支付订单、支付失败和转化率，发现异常门店后继续下钻设备和软件版本。演示时直接 INSERT，生产环境则由 Kafka 链路写入同一张明细表。

![2d5475b4-04d7-401d-ad64-12ffa53830f3](https://cdn.wcxian.cc/img/20260826210122254.png)

### 6.1 准备演示数据

```sql
INSERT INTO analytics.store_events
    (event_id, tenant_id, store_id, member_id, device_id, city, event_name,
     order_id, product_id, page_code, event_time, properties)
VALUES
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10001, 'pos-01', '杭州', 'entry_detected', NULL, NULL, 'store_entry', '2026-07-21 10:00:01', '{}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10001, 'pos-01', '杭州', 'payment_success', 200001, 1001, 'checkout', '2026-07-21 10:02:20', '{"software_version":"2.4.1","cost_ms":680}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10002, 'pos-01', '杭州', 'entry_detected', NULL, NULL, 'store_entry', '2026-07-21 10:05:01', '{}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10002, 'pos-01', '杭州', 'payment_success', 200002, 1002, 'checkout', '2026-07-21 10:06:20', '{"software_version":"2.4.1","cost_ms":720}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10003, 'pos-02', '杭州', 'entry_detected', NULL, NULL, 'store_entry', '2026-07-21 10:10:01', '{}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10003, 'pos-02', '杭州', 'payment_failed', 200003, 1003, 'checkout', '2026-07-21 10:11:20', '{"software_version":"2.4.2","error_code":"PAY_TIMEOUT","cost_ms":3200}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-002', 10011, 'pos-03', '宁波', 'entry_detected', NULL, NULL, 'store_entry', '2026-07-21 10:03:01', '{}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-002', 10011, 'pos-03', '宁波', 'payment_failed', 300001, 1001, 'checkout', '2026-07-21 10:04:20', '{"software_version":"2.4.2","error_code":"PAY_TIMEOUT","cost_ms":4100}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-002', 10012, 'pos-03', '宁波', 'entry_detected', NULL, NULL, 'store_entry', '2026-07-21 10:08:01', '{}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-002', 10012, 'pos-03', '宁波', 'payment_failed', 300002, 1002, 'checkout', '2026-07-21 10:09:20', '{"software_version":"2.4.2","error_code":"PAY_TIMEOUT","cost_ms":3800}');
```

### 6.2 生成门店经营指标

```sql
-- ============================================================
-- 查询各门店在指定日期（2026-07-21）的转化漏斗核心指标
-- 数据来源：analytics.store_events（门店事件明细表）
-- 租户过滤：仅统计 tenant-a 的数据
-- 聚合粒度：按门店（store_id）分组
-- 输出排序：按转化率（conversion_rate_pct）降序，即转化表现好的门店排在前面
-- ============================================================

SELECT
    store_id,
    -- 进入门店的独立会员数（去重 member_id）
    -- 事件条件：event_name = 'entry_detected'（检测到进入）
    uniqExactIf(member_id, event_name = 'entry_detected') AS entry_uv,
    -- 支付成功的订单数（去重 order_id）
    -- 事件条件：event_name = 'payment_success'（支付成功）
    uniqExactIf(order_id, event_name = 'payment_success') AS payment_orders,
    -- 支付失败的事件次数（非去重，每次失败计一条）
    -- 事件条件：event_name = 'payment_failed'（支付失败）
    countIf(event_name = 'payment_failed') AS payment_failed,
    -- 计算转化率（支付订单数 / 进入UV * 100），保留两位小数
    -- 使用 nullIf(entry_uv, 0) 防止除零错误，当 entry_uv 为 0 时返回 NULL，使结果为 NULL
    round(payment_orders / nullIf(entry_uv, 0) * 100, 2) AS conversion_rate_pct
FROM analytics.store_events
WHERE
    tenant_id = 'tenant-a'
    AND event_date = '2026-07-21'
GROUP BY store_id          -- 按门店聚合，每个门店输出一行汇总结果
ORDER BY conversion_rate_pct DESC;   -- 按转化率从高到低排序

```

结果大致是：

| 门店           | 进店 UV | 支付订单 | 支付失败 | 转化率 |
| -------------- | ------: | -------: | -------: | -----: |
| `store-hz-001` |       3 |        2 |        1 | 66.67% |
| `store-hz-002` |       2 |        0 |        2 |     0% |

![image-20260826203821525](https://cdn.wcxian.cc/img/20260826203824376.png)

### 6.3 下钻异常设备

```sql
-- ============================================================
-- 查询指定日期（2026-07-21）各门店各设备上支付失败事件的详细分布
-- 目的：定位高频失败组合（门店 + 设备 + 软件版本 + 错误码），辅助排查系统问题
-- 数据来源：analytics.store_events（门店事件明细表）
-- 租户过滤：仅统计 tenant-a 的数据
-- 聚合粒度：按门店、设备、软件版本、错误码 四个维度组合
-- 输出排序：先按失败次数降序（问题最严重的排前面），再按最大耗时降序（性能最差的排前面）
-- ============================================================

SELECT
    store_id,
    device_id,
    -- 从 properties 字段中解析出软件版本号（字符串类型）
    -- properties 为 JSON 格式，示例：{"software_version":"v2.3.1","error_code":"E001","cost_ms":150}
    JSONExtractString(properties, 'software_version') AS software_version,
    -- 从 properties 字段中解析出错误码（字符串类型）
    -- 用于标识具体的失败原因，如超时、签名错误、库存不足等
    JSONExtractString(properties, 'error_code') AS error_code,
    -- 统计该组合下支付失败事件的次数（count() 计数所有行，无需去重，因为每条事件即一次失败）
    count() AS failed_count,
    -- 从 properties 字段中解析出耗时（毫秒，无符号整数），并取该组合下的最大值
    -- 用于观察最慢的那次失败请求耗时，辅助判断是否因性能问题导致超时失败
    max(JSONExtractUInt(properties, 'cost_ms')) AS max_cost_ms
FROM analytics.store_events
WHERE
    tenant_id = 'tenant-a'
    AND event_date = '2026-07-21'
    AND event_name = 'payment_failed'
GROUP BY
    store_id,               -- 按门店分组，看问题是否集中在某个门店
    device_id,              -- 按设备分组，看是否某台设备硬件故障或网络差
    software_version,       -- 按软件版本分组，看是否新版本引入的缺陷
    error_code              -- 按错误码分组，看具体失败原因类型
ORDER BY
    failed_count DESC,      -- 失败次数最多的组合排在最前，优先关注高发问题
    max_cost_ms DESC;       -- 若失败次数相同，耗时最大的排前面，优先解决性能瓶颈
```

![image-20260826204121513](https://cdn.wcxian.cc/img/20260826204335907.png)

结果会显示 `store-hz-002` 的 `pos-03`、版本 `2.4.2` 和 `PAY_TIMEOUT` 集中出现。后续可以暂停版本灰度、检查设备网络、回滚支付 SDK，并继续观察转化率是否恢复。

### 6.4 用聚合表支撑看板

固定的小时趋势查询聚合表，异常下钻再查明细：

```sql
-- ============================================================
-- 查询特定门店（store-hz-002）在指定日期（2026-07-21）内，
-- 按小时统计各核心事件（进店、支付成功、支付失败）的发生趋势。
-- 数据来源：analytics.store_event_1h（按小时预聚合的中间表 / 物化视图）
-- 作用：用于观察一天中不同时段的流量和支付转化波动，辅助运营排班或系统压测分析
-- ============================================================

SELECT
    hour,
    event_name,

    -- 【重要】合并预聚合的事件计数
    -- 该表使用 AggregatingMergeTree 引擎，原数据已通过 countState() 预先聚合。
    -- countMerge() 用于将底层多个数据块中的中间状态合并为最终的实际计数值。
    countMerge(event_count) AS event_count,

    -- 【重要】合并预聚合的独立访客数（UV）
    -- 原数据已通过 uniqCombined64State() 预先进行了近似去重。
    -- uniqCombined64Merge() 用于将底层存储的中间状态合并为最终的近似 UV 值。
    -- 注意：这是近似去重算法（HyperLogLog 变种），性能极高，适合大数据量场景。
    uniqCombined64Merge(member_uv) AS member_uv

FROM analytics.store_event_1h

WHERE
    tenant_id = 'tenant-a'
    AND store_id = 'store-hz-002'
    AND event_name IN ('entry_detected', 'payment_success', 'payment_failed')
    AND hour >= '2026-07-21 00:00:00'
    AND hour < '2026-07-22 00:00:00'
GROUP BY hour, event_name
ORDER BY hour, event_name;
```

以 `store-hz-002` 当天的演示数据为例，结果大致是：

| hour                | event_name     | event_count | member_uv |
| ------------------- | -------------- | ----------: | --------: |
| 2026-07-21 10:00:00 | entry_detected |           2 |         2 |
| 2026-07-21 10:00:00 | payment_failed |           2 |         2 |

看板每分钟查询聚合表，用户点击异常点时再查询 `store_events` 的原始事件和 `properties`。这就是：**聚合表负责快，明细表负责查。**

页面上的指标需要标注口径：看板展示的是近实时值（数据进入即累计），财务对账使用最终快照；两者不一致时，先核对事件是否补写、是否重复，再判断是不是口径差异。

![image-20260826204204264](https://cdn.wcxian.cc/img/20260826204332703.png)



---

## 7. 常见误区

### 7.1 把物化视图当完整 ETL

错误方式：在物化视图里做复杂 Join、跨表关联、长窗口计算。

正确方式：物化视图只做字段映射、默认值和简单 JSON 提取；复杂清洗前移到 Kafka 或 Flink 预处理。

原因：物化视图逻辑越重，消费越慢，Lag 越高，最终拖慢整个接入链路。

### 7.2 以为跳数索引是 B+Tree

错误方式：给所有列加 `bloom_filter`，指望任何查询都变快。

正确方式：只在排序键覆盖不到、且选择性好的列（如 `product_id`、`device_id`）上评估，建完用真实 SQL 对比 `read_rows`。

原因：跳数索引按 Granule 跳过数据，分布差时可能完全无效，建了反而增加写入成本。

### 7.3 每小时 UV 相加得全天 UV

错误方式：`sum(每小时 member_uv)` 当作全天 UV。

正确方式：继续合并 `uniqCombined64` 的中间状态，或单独建设日级 UV 表。

原因：同一会员可能出现在多个小时，简单相加会重复计数（原理见 2.1 节）。

### 7.4 把副本当备份

错误方式：有了副本就不再备份。

正确方式：副本同步所有写入，包括误写、误删和结构变更；重要数据独立备份并定期演练恢复。

原因：副本解决可用性，不解决逻辑错误。

### 7.5 全量 FINAL 兜底

错误方式：日常大范围查询一律 `SELECT ... FINAL`。

正确方式：上游幂等 + 聚合表/快照表保证结果；`FINAL` 只用于小范围主键查询。

原因：`FINAL` 在查询时强制合并，扫描成本很高，不能成为报表的默认写法。

### 7.6 无限加消费者解 Lag

错误方式：Lag 上涨就加 `kafka_num_consumers`。

正确方式：先看 Partition 数量、写入批次、Part 和 Merge、物化视图计算，再决定扩消费者还是前移清洗。

原因：并行度受 Partition 数量限制，瓶颈常常在写入和计算侧。

### 7.7 TTL 建了不验证

错误方式：建好 TTL 就以为数据会自动按时删除。

正确方式：观察 `system.parts` 和 `system.ttl_merges`，确认删除生效、磁盘空间释放。

原因：TTL 随后台合并异步执行，不保证立即删除，也不保证磁盘立即回收。

---

## 8. 总结

ClickHouse 进阶的重点不是记住更多参数，而是把下面这条链路做成可验证的闭环：

```text
数据进入 → 指标聚合 → 看板发现异常 → 明细下钻 → 定位原因 → 修复验证
```

读完本文，你应该能独立完成：

- **接入**：用 Kafka Engine + 物化视图把事件实时落地，并处理重复、坏消息和重放；
- **加速**：建小时聚合表，理解 `State`/`Merge` 中间状态，并用 `query_log` 验证扫描量下降；
- **调优**：按「查询路径 → 分区/排序键 → SQL 改写 → 索引/投影 → 扩容」的顺序优化慢 SQL；
- **兜底**：定位慢 SQL 和小 Part，设计 TTL 与备份，并按检查清单上线；
- **演示**：搭出「明细 + 聚合 + 下钻」的实时经营看板。

核心结论不变：

- Kafka 解决缓冲和接入，MergeTree 保存事实；
- 物化视图和聚合表减少高频看板的扫描；
- 排序键、分区和 SQL 改写决定明细查询效率；
- `query_log`、`system.parts` 帮助定位慢 SQL 和小 Part；
- 分片解决容量，副本解决可用性，但副本不能替代备份；
- 真正的业务价值，是从“门店转化率下降”一路定位到“设备、版本和错误码异常”。

---

## 参考资料

- [ClickHouse 官方文档](https://clickhouse.com/docs)
- [ClickHouse SQL Reference](https://clickhouse.com/docs/sql-reference)
- [ClickHouse MergeTree 引擎](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Materialized View](https://clickhouse.com/docs/materialized-view)
- [ClickHouse Kafka 表引擎](https://clickhouse.com/docs/engines/table-engines/integrations/kafka)
- Kafka 基础篇：《[Kafka 入门](Kafka%20入门.md)》
- ClickHouse 基础篇：《[ClickHouse 入门](ClickHouse%20入门.md)》