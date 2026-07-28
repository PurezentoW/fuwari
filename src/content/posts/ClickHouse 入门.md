---
title: 'ClickHouse 入门'
published: 2026-07-28
description: '面向 OLAP 场景的开源列式数据库：ClickHouse'
image: 'https://cdn.wcxian.cc/img/20260728162339359.png'
tags: ['ClickHouse', 'OLAP', '数据库']
category: '技术分享'
draft: false
lang: zh-CN
---
# ClickHouse 入门

---

## 0. 写在前面

在无人值守商店 SaaS 中,MySQL 很适合处理订单创建、库存扣减、门店配置、设备绑定这类**单条或少量数据的事务操作**;但当我们要分析「最近 30 天每个租户、门店的进店人数、支付转化率、Top 商品、设备故障率」时,往往需要扫描几千万甚至上亿行明细事件。

如果继续让 MySQL 承担这类大范围聚合查询,常见结果是:

- 报表 SQL 执行几十秒甚至几分钟;
- 业务库 CPU、IO 被分析任务拖高,影响正常下单;
- 为了提速不断加索引、拆表、建汇总表,维护成本越来越高;
- 数据量继续增长后,原来的优化方案又会失效。

这类以「写入大量明细、按维度聚合分析、读多写多但很少更新」为特征的需求,正是 **ClickHouse** 的主战场。

**一句话总结:ClickHouse 是一个面向 OLAP 场景的开源列式数据库,擅长对海量明细数据进行实时分析。**

本文以无人值守商店 SaaS 为主线,介绍 ClickHouse 的核心原理、安装方式、表设计、常用 SQL、与 MySQL 的差异、Spring Boot 集成以及选型建议。读完后,读者应能把"门店经营异常、设备故障、支付转化下降"落到可查询的数据和可执行的处理动作上。

> Kafka 实时接入、物化视图预聚合、跳数索引/Projection 调优、分布式集群、运维监控等进阶内容,已单独整理到《ClickHouse 进阶实战》,新手先把本文读透即可。

---

## 1. ClickHouse 是什么

### 1.1 OLTP 和 OLAP

先理解两个概念:

- **OLTP(Online Transaction Processing)联机事务处理**:关注一笔数据是否准确地写入或修改,例如创建订单、支付确认、库存扣减、门店设备配置。
- **OLAP(Online Analytical Processing)联机分析处理**:关注从大量历史数据中快速统计规律,例如门店经营报表、进店到支付的转化分析、设备健康度和监控指标大盘。

| 对比项  | OLTP:MySQL / PostgreSQL | OLAP:ClickHouse  |
| ---- | ----------------------- | ---------------- |
| 典型操作 | 单条查询、插入、更新、删除           | 多维筛选、分组、聚合、TopN  |
| 数据规模 | 万~千万级单表较常见              | 亿~万亿级明细数据        |
| 查询返回 | 少量记录                    | 聚合结果、报表、明细钻取     |
| 事务能力 | 强,支持行级事务                | 不以高频事务更新为目标      |
| 存储方式 | 通常是行式存储                 | 列式存储             |
| 典型场景 | 订单、账户、商品、库存             | 日志、埋点、指标、画像、风控分析 |

举一个无人店顾客与设备事件的例子:

```text
顾客开门进店 → 浏览/拿取商品 → 发起结算 → 支付成功 → 离店
```

每天如果有 1 亿条门店、设备和交易事件,运营与运维人员可能会频繁查询:

```text
- 今天每个租户、门店的进店人数和支付人数是多少?
- 最近 7 天哪些商品"被浏览很多但结算很少"?
- 某城市支付失败是否集中在特定门店、设备型号或支付通道?
- 从进店到支付成功的漏斗转化率是否在某个门店突然下降?
```

这正是 ClickHouse 擅长的聚合分析场景。

### 1.2 ClickHouse 的核心特点

1. **列式存储**:只读取查询所需的列,减少磁盘 IO。
2. **向量化执行**:一次处理一批数据,而不是逐行解释执行,充分利用 CPU。
3. **高压缩比**:同一列数据类型相同、重复度高,天然适合压缩。
4. **并行计算**:一条 SQL 会使用多个 CPU 核心并行扫描和聚合。
5. **实时写入、近实时可查**:数据写入后通常很快就可以参与查询。
6. **分布式扩展**:可通过分片和副本扩展容量、吞吐和可用性。
7. **SQL 友好**:大部分分析需求直接使用 SQL 即可完成。

### 1.3 ClickHouse 不适合什么场景

ClickHouse 很快,但不是「可以替代所有数据库」的万能工具。

以下情况通常不建议优先使用 ClickHouse:

- **高频单行更新**:例如用户余额每秒更新多次。
- **强事务场景**:例如支付扣款,要求严格的多表事务一致性。
- **高频按主键随机查一条数据**:例如根据订单号查询订单详情。
- **频繁的小批量写入**:每次只写一两行,会产生大量小文件(Part),影响合并性能。
- **数据存在复杂关联且每次都需要多表 Join**:应优先考虑宽表、字典或预聚合设计。

> 这些「不适合」会在第 7 章《MySQL vs ClickHouse 概念对比》和第 10 章《选型对比与优劣势》里展开讲。

推荐的架构不是「用 ClickHouse 替换 MySQL」,而是:

```text
                    ┌───────────────┐
                    │ MySQL / PG     │  订单、库存、租户和门店等事务数据
                    │ (业务主库)     │
                    └───────┬───────┘
                            │ CDC / 定时同步
                            ▼
┌──────────┐      ┌──────────────────┐      ┌──────────────────┐
│ 门店/设备/服务│ ─► │ Kafka / Flink     │ ───► │ ClickHouse        │
│ 事件与日志    │    │ 清洗、关联、聚合  │      │ 明细 + 实时分析   │
└──────────┘      └──────────────────┘      └─────────┬────────┘
                                                        │
                                                        ▼
                                               ┌──────────────────┐
                                               │ BI 报表 / 数据大盘 │
                                               └──────────────────┘
```

### 1.4 三个可以直接落地的业务案例

先不急着看存储引擎和参数。下面用三个典型场景说明:业务团队究竟会拿 ClickHouse 查什么、查到结果后又能做什么。案例中的数据规模和数字为便于理解而简化,但处理方式可以直接迁移到实际项目。

#### 案例一:无人店交易下降,判断是「客流、商品、设备」还是「支付」的问题

**业务背景**:一家 SaaS 平台服务数百家无人店。单店每天会持续产生门禁、摄像头/识别、货柜、结算和订单事件;会员日或商圈活动期间,事件量还会明显增长。运营希望在营业中,而不是第二天复盘时,回答下面的问题:

- 是进店客流下降,还是进店后没有完成支付?
- 哪些商品"浏览或拿取很多、最终购买很少",是缺货、定价还是识别问题?
- 转化突然下跌,是某个城市、门店、设备型号、软件版本,还是支付通道出了问题?

**数据怎么进来**:门禁上报 `door_opened`、传感器或视觉服务上报 `entry_detected`、商品浏览/拿取上报 `product_view`,收银和支付服务上报 `payment_success`、`payment_failed`,订单和退款状态从业务库同步。流式程序补齐租户、门店、城市、设备型号、商品类目等维度后,写入 ClickHouse 明细表。数据通常在几秒到几十秒内可查询。

**运营大盘会看什么**:

| 维度 | 指标 | 能发现什么 |
| --- | --- | --- |
| 租户 / 门店 | 进店人数、支付人数、GMV、退款数 | 哪家门店客流正常但不成交 |
| 商品 | 浏览/拿取 → 支付转化率 | 缺货、定价、商品识别或陈列是否影响成交 |
| 城市 / 设备型号 / 软件版本 | 支付成功率、设备错误数、接口耗时 | 某批设备、网络或新版本是否异常 |
| 小时 | 每 5 分钟进店数、订单数、GMV、退款数 | 活动是否达到预期,以及异常何时开始 |

**一次真实的使用动作(示例)**:10:30 后,大盘显示某商圈门店的进店人数正常,但"进店 → 支付成功"转化率只有同类门店的三分之一。运营按 `store_id`、`device_id`、`product_id` 和支付通道下钻,发现一批收银设备升级后频繁报支付超时。于是先将设备切回稳定版本、引导顾客使用备用支付通道;研发同时用同一份事件数据确认修复后转化率是否恢复。这里 ClickHouse 的价值不只是出报表,而是让业务能在损失扩大的过程中定位和验证问题。

本文第 4 节的 `store_events` 表,以及第 6 节的进店/支付统计、商品 TopN 和漏斗 SQL,正是为这一类问题准备的。

#### 案例二:结算变慢时,从「顾客投诉」到「定位范围」

**业务背景**:门店客服反馈"顾客已扫码但结算一直转圈",而平均响应时间仍然正常。因为少量超慢请求会被平均值掩盖,研发需要立刻知道:是门禁、结算、支付还是商品识别服务变慢,从什么时间开始,影响了哪些门店、设备和软件版本。

**数据怎么进来**:设备服务日志、网关访问日志、应用日志和链路事件持续写入 ClickHouse。每条记录除了时间、接口和状态码外,还保留 `tenant_id`、`store_id`、`device_id`、`cost_ms`、服务名、发布版本、机房和错误码等字段。

**排查过程**:

1. 先按分钟查看支付接口的 P50、P95、P99,确认问题从 14:05 开始出现;
2. 再按门店、设备型号、发布版本和机房拆分,发现只有新版本的一批设备 P99 明显升高;
3. 继续过滤错误码和下游服务,定位到支付网关连接池耗尽;
4. 回滚发布或扩容后,持续查看 P99 和错误率,确认指标恢复。

这种场景中,ClickHouse 适合保留高吞吐的原始日志,并支持按任意时间段、接口、版本和错误码进行聚合与下钻。第 6.5 节的分位数查询就对应这类"平均值看不出问题"的线上排障需求。

#### 案例三:无人店 SaaS 的多租户实时经营看板

**业务背景**:一个无人店 SaaS 平台服务多个品牌和加盟商,旗下有数百家门店。总部、区域运营和平台运维不需要逐笔查询订单,而是希望随时看到"今天卖了多少、哪家门店落后、退款是否异常、哪些品类缺货、哪些设备影响了成交"。

**典型做法**:订单创建、支付、退款、门店库存变化及设备健康事件通过 CDC 或消息队列汇入 ClickHouse;写入时把租户、门店、城市、商品类目、设备型号、会员类型等高频分析维度补充到订单宽表中。系统再按小时、租户、门店和类目预聚合,给经营大盘、区域运营和平台运维使用。

| 角色         | 常见问题                  | 对应业务动作           |
| ---------- | --------------------- | ---------------- |
| 品牌总部 / 加盟商 | 今日 GMV、进店转化与目标差多少?    | 调整商品、促销和门店经营策略   |
| 区域运营       | 哪些门店订单下滑、退款异常或客流转化偏低? | 核实库存、设备、网络和活动执行  |
| 商品运营       | 哪个品类浏览上涨但支付下降、库存不足?   | 调拨库存、补货或调整商品策略   |
| 平台运维       | 哪批设备、哪个版本的故障率或耗时异常?   | 灰度回滚、远程修复或现场巡检   |
| 财务         | 支付金额、退款金额和订单数是否对得上?   | 尽早发现数据、订单或支付链路问题 |

这里的关键不是把 ClickHouse 当订单主库,而是把它作为"面向分析的订单与事件事实层":MySQL 仍负责创建订单、改状态和事务一致性;ClickHouse 负责快速汇总、趋势比较和多维钻取。订单宽表设计和预聚合的具体做法,可参见《ClickHouse 进阶实战》。

**判断一个需求是否值得接入 ClickHouse,可以先问三个问题**:数据是否持续累积、查询是否经常按多个维度做统计、结果是否会直接驱动运营或研发动作?三个答案都接近「是」时,通常就是一个合适的候选场景。

---

## 2. 为什么 ClickHouse 查询这么快

### 2.1 列式存储

假设有一张无人店事件事实表:

```text
tenant_id | store_id | event_time          | event_name       | device_id | properties
----------|----------|---------------------|------------------|-----------|-----------
tenant-a  | store-01 | 2026-07-21 10:00:01 | entry_detected   | door-01   | ...
tenant-a  | store-01 | 2026-07-21 10:00:03 | payment_success | pos-01    | ...
```

现在要查询:

```sql
SELECT store_id, count()
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_time >= '2026-07-21 00:00:00'
GROUP BY store_id;
```

行式数据库通常需要按行读取数据页,一条记录中可能包含商品明细、设备原始日志、`properties` 等完全不需要的字段。

ClickHouse 是按列保存的,只需要扫描:

```text
tenant_id 列 + event_time 列 + store_id 列
```

因此,**列越多、单次查询实际使用的列越少,列式存储的优势越明显**。

### 2.2 向量化执行和并行计算

传统数据库可以理解为「一行一行处理数据」;ClickHouse 会把同类型数据按块读取,并用向量化方式批量执行过滤、聚合等操作。

同时,一张大表会被拆成多个数据分区和数据片段(Part),查询时多个 CPU 核心可以并行处理:

```text
查询请求
   │
   ├── CPU Core 1:扫描 Part 1、Part 2
   ├── CPU Core 2:扫描 Part 3、Part 4
   ├── CPU Core 3:扫描 Part 5、Part 6
   └── CPU Core 4:扫描 Part 7、Part 8
                    │
                    ▼
                汇总最终结果
```

### 2.3 压缩和稀疏索引

ClickHouse 会对不同列分别压缩。例如 `event_name` 只有 `entry_detected`、`payment_success`、`payment_failed` 等少量取值,压缩效果通常很好。

此外,MergeTree 表会按照 `ORDER BY` 排序键组织数据,并建立**稀疏索引**。它不是为每一行建立索引,而是每隔若干行保存一个索引标记:

```text
Part 内部(按 tenant_id, event_date, event_time 排序)

tenant-a, 2026-07-21, 10:00:00  ─┐
tenant-a, 2026-07-21, 10:13:00   │ 一个索引粒度
tenant-a, 2026-07-21, 10:28:00  ─┘
tenant-b, 2026-07-21, 10:00:00  ─┐
tenant-b, 2026-07-21, 10:15:00   │ 下一个索引粒度
```

当查询条件和排序键前缀匹配时,ClickHouse 可以跳过大量不相关的数据块。

> ClickHouse 的性能核心,不是「多建几个传统索引」,而是**建好表、选好排序键、让查询尽可能少扫描数据**。

**业务闭环**:当运营先按租户、门店和营业时段查看"进店 → 支付"转化时,列存只读取相应维度与事件列;排序键又能跳过其他租户和日期的数据块。这样异常门店可以在营业中下钻,而不是等待离线日报。

---

## 3. 安装与基础配置

### 3.1 Windows 安装

#### 方案一:WSL2 安装(推荐)

Windows 环境推荐使用 WSL2 安装 ClickHouse。WSL2 提供了接近 Linux 的运行环境,不需要额外安装 Docker Desktop,适合本地学习和开发。

##### 1. 安装 WSL

以管理员身份打开 PowerShell,执行:

```powershell
wsl --install
```

安装完成后重启电脑。重启后查看 WSL 是否安装成功:

```powershell
wsl -l -v
```

正常情况下会看到类似输出:

```text
  NAME            STATE           VERSION
* Ubuntu-24.04    Stopped         2
```

##### 2. 进入 Ubuntu

在 PowerShell 中执行:

```powershell
wsl
```

进入 Ubuntu 后更新系统:

```bash
sudo apt update
sudo apt upgrade -y
```

##### 3. 安装 ClickHouse

ClickHouse 官方提供一键安装方式。在 Ubuntu 中执行:

```bash
curl https://clickhouse.com/ | sh
```

安装脚本会在当前目录下载名为 `clickhouse` 的二进制文件。启动服务端:

```bash
./clickhouse server
```

![image-20260724111717800](https://cdn.wcxian.cc/img/20260724111724153.png)

保持当前窗口运行,再打开一个新的 WSL 窗口:

```powershell
wsl
```

进入 ClickHouse 二进制文件所在目录,启动客户端:

```bash
./clickhouse client
```

看到类似下面的提示,说明安装成功:

```text
ClickHouse client version xx.xx.xx
localhost :) 
```

##### 4. 测试

在 ClickHouse 客户端中执行:

```sql
SELECT version();
```

![image-20260724111823732](https://cdn.wcxian.cc/img/20260724111825157.png)

创建测试表:

```sql
CREATE TABLE test
(
    id UInt32,
    name String
)
ENGINE = MergeTree
ORDER BY id;
```

插入测试数据:

```sql
INSERT INTO test VALUES
    (1, 'Tom'),
    (2, 'Jerry');
```

查询数据:

```sql
SELECT * FROM test;
```

如果能够看到插入的两条记录,说明 ClickHouse 已经安装并可以正常使用。

```
KJSD.localdomain :) SELECT * FROM test;

SELECT *
FROM test

Query id: b7b11161-96d8-4fea-a791-3d651e5be21e

   ┌─id─┬─name──┐
1. │  1 │ Tom   │
2. │  2 │ Jerry │
   └────┴───────┘

2 rows in set. Elapsed: 0.002 sec.
```



> 如果 `curl` 命令不存在,可以先执行 `sudo apt install curl -y`。服务端窗口需要保持运行;关闭该窗口后,ClickHouse 服务也会停止。

### 3.2 Docker 快速启动

本地学习或开发环境可以直接通过 Docker 启动:

```bash
docker run -d \
  --name clickhouse-server \
  --ulimit nofile=262144:262144 \
  -p 8123:8123 \
  -p 9000:9000 \
  -e CLICKHOUSE_DB=analytics \
  -e CLICKHOUSE_USER=default \
  -e CLICKHOUSE_PASSWORD=123456 \
  clickhouse/clickhouse-server:latest
```

端口说明:

| 端口 | 协议 | 用途 |
| --- | --- | --- |
| `8123` | HTTP | 浏览器、curl、JDBC HTTP 连接 |
| `9000` | Native TCP | `clickhouse-client`、部分驱动连接 |
| `9004` | MySQL 协议(可选) | 兼容 MySQL 客户端连接 |

查看容器状态:

```bash
docker ps
docker logs -f clickhouse-server
```

进入客户端:

```bash
docker exec -it clickhouse-server clickhouse-client \
  --user default \
  --password 123456
```

### 3.3 使用 Docker Compose

团队开发中更推荐使用 `docker-compose.yml` 固定环境:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-server
    restart: unless-stopped
    ports:
      - "8123:8123"
      - "9000:9000"
      - "9004:9004"
    environment:
      CLICKHOUSE_DB: analytics
      CLICKHOUSE_USER: app_user
      CLICKHOUSE_PASSWORD: change_me_in_production
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: "1"
    volumes:
      - ./clickhouse-data:/var/lib/clickhouse
      - ./clickhouse-logs:/var/log/clickhouse-server
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
```

启动:

```bash
docker compose up -d
```

> 生产环境不要把密码直接提交到 Git。建议通过环境变量、Docker Secret 或配置中心管理账号密码。

### 3.4 验证连接

通过 HTTP 接口执行 SQL:

```bash
curl "http://localhost:8123/?user=default&password=123456" \
  --data-binary "SELECT version(), now() FORMAT Vertical"
```

返回类似:

```text
Row 1:
──────
version(): 25.x.x.x
now():     2026-07-21 17:00:00
```

ClickHouse 自带的 HTTP 接口非常实用,临时排查问题时可以直接执行:

```bash
curl "http://localhost:8123/?query=SHOW%20DATABASES"
```

---

## 4. 第一个业务分析表

下面以"租户、门店、顾客、设备与交易事件"为例,创建一张可用于实时分析的明细表。

### 4.1 创建数据库

```sql
CREATE DATABASE IF NOT EXISTS analytics;

USE analytics;
```

### 4.2 创建明细表

```sql
CREATE TABLE IF NOT EXISTS store_events
(
    -- 事件唯一标识,用于技术去重和问题追踪
    event_id UUID,

    -- SaaS 租户、门店、顾客与设备维度
    tenant_id LowCardinality(String),
    store_id LowCardinality(String),
    member_id Nullable(UInt64),
    device_id Nullable(String),
    city LowCardinality(String),

    -- 业务事件与关联对象
    event_name LowCardinality(String),
    order_id Nullable(UInt64),
    product_id Nullable(UInt64),
    page_code LowCardinality(String),

    -- 业务发生时间。DateTime64(3) 支持毫秒精度
    event_time DateTime64(3, 'Asia/Shanghai'),
    event_date Date MATERIALIZED toDate(event_time),

    -- 动态扩展属性,例如设备型号、支付通道、软件版本、错误码
    properties String,

    -- 数据写入时间,用于识别采集或消费延迟
    ingest_time DateTime DEFAULT now()
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (tenant_id, store_id, event_date, event_time, event_name, ifNull(member_id, toUInt64(0)))
SETTINGS index_granularity = 8192;
```

这个建表 SQL 中最重要的是 `MergeTree`、`PARTITION BY` 和 `ORDER BY`。

| 配置 | 作用 |
| --- | --- |
| `ENGINE = MergeTree` | ClickHouse 最常用的存储引擎,提供分区、排序、稀疏索引和后台合并能力。 |
| `PARTITION BY toYYYYMM(event_date)` | 按月分区,便于生命周期管理和按月删除历史数据。 |
| `ORDER BY (...)` | 决定数据在每个 Part 内的物理排序方式,是最重要的性能设计。 |
| `LowCardinality(String)` | 适合城市、渠道、事件名这类枚举值较少的字符串列,可减少字典和存储开销。 |
| `MATERIALIZED` | `event_date` 由 `event_time` 自动计算,写入时无需手动传值。 |

### 4.3 为什么这样设计排序键

无人店 SaaS 的高频查询通常会带上:

```text
tenant_id + store_id + 日期范围 + 时间范围 + 事件名称
```

所以排序键选择:

```sql
ORDER BY (tenant_id, store_id, event_date, event_time, event_name, member_id)
```

这样查询某个租户下某家门店在某一天、某个时间段内的事件时,能够利用排序键快速裁剪数据。

排序键的经验原则:

1. **将高频过滤字段放在前面**。
2. **等值条件通常优先于范围条件**。
3. **时间字段很常用,但不要机械地把时间放在第一位**;如果几乎每次都先按租户和门店过滤,`tenant_id`、`store_id` 放在前面会更好。
4. 不要把所有字段都塞进 `ORDER BY`,排序键越长,写入和合并成本也会增加。

### 4.4 插入测试数据

```sql
INSERT INTO store_events
    (event_id, tenant_id, store_id, member_id, device_id, city, event_name,
     order_id, product_id, page_code, event_time, properties)
VALUES
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10001, 'door-01', '杭州', 'entry_detected',
     NULL, NULL, 'store_entry', '2026-07-21 10:00:01.123', '{"device_model":"door-v2","software_version":"2.4.1"}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10001, 'shelf-03', '杭州', 'product_view',
     NULL, 1001, 'product_detail', '2026-07-21 10:01:03.456', '{"device_model":"shelf-v3"}'),
    (generateUUIDv4(), 'tenant-a', 'store-hz-001', 10001, 'pos-01', '杭州', 'payment_success',
     200001, 1001, 'checkout', '2026-07-21 10:02:20.000', '{"payment_channel":"wechat","cost_ms":680}');
```

查询数据:

```sql
SELECT
    event_time,
    member_id,
    store_id,
    city,
    event_name,
    device_id,
    order_id,
    product_id
FROM store_events
ORDER BY event_time;
```

![image-20260724113804938](https://cdn.wcxian.cc/img/20260724113806284.png)

---

## 5. MergeTree 存储引擎详解

### 5.1 MergeTree 的数据写入过程

向 ClickHouse 插入一批数据后,数据不会像传统数据库一样逐行修改一个文件,而是先形成一个新的 **Part(数据片段)**。

```text
INSERT 批次 1  ──► Part_1
INSERT 批次 2  ──► Part_2
INSERT 批次 3  ──► Part_3
                         │
                         ▼
                后台自动 Merge
                         │
                         ▼
                    大的有序 Part
```

这就是名字中 `Merge` 的来源。

后台合并会带来两个结果:

- 小 Part 合并为大 Part,减少文件数量,提高读取效率;
- 某些引擎(例如 `ReplacingMergeTree`)会在合并时完成去重。

因此,应用侧应该尽量使用**批量写入**,而不是每条数据发一次 `INSERT`。对于无人店,若门禁、货架、收银设备每上报一条事件就直接写入,营业高峰会产生大量小 Part,最终表现为经营大盘刷新延迟或抖动;应在 SDK 网关、Kafka 或应用侧攒批写入。

### 5.2 PARTITION BY、ORDER BY 和 PRIMARY KEY 的区别

这三个概念很容易混淆:

| 概念 | 作用 | 是否建议频繁使用 |
| --- | --- | --- |
| `PARTITION BY` | 将数据拆分到不同分区,常用于按月、按天管理数据 | 否,分区不宜过细 |
| `ORDER BY` | 决定分区内数据排序和主索引,是查询性能核心 | 必须仔细设计 |
| `PRIMARY KEY` | 稀疏主索引的表达式;默认与 `ORDER BY` 相同 | 一般保持默认即可 |

一个常见误区是「按天分区一定更快」。

如果每天只有几万行数据,按天分区问题不大;但如果按用户 ID、订单 ID 这类高基数字段分区,就会产生海量小分区,元数据和合并压力会非常大。

通常建议:

- 日志、埋点、交易明细:按**月**或按**天**分区;
- 高吞吐、数据量特别大:可按天分区;
- 不要按用户、城市、渠道等维度分区;
- 查询加上分区字段条件,才能有效分区裁剪。

### 5.3 最常用的 MergeTree 家族引擎

| 引擎 | 用途 | 说明 |
| --- | --- | --- |
| `MergeTree` | 普通明细数据 | 最常用的基础引擎。 |
| `ReplacingMergeTree` | 最终一致去重 | 相同排序键的数据,后台合并后保留一个版本。 |
| `AggregatingMergeTree` | 聚合状态存储 | 配合物化视图保存聚合中间状态。 |
| `Distributed` | 分布式查询入口 | 本身不存数据,将 SQL 路由到多个分片。 |

> 此外还有 `SummingMergeTree`、`CollapsingMergeTree`、`ReplicatedMergeTree` 等引擎,以及预聚合、跳数索引、Projection、分片副本等更深入的用法,见《ClickHouse 进阶实战》。

### 5.4 ReplacingMergeTree 去重

如果上游 Kafka、Flink 或网络重试可能造成重复数据,可以使用 `ReplacingMergeTree` 保存同一业务主键的最新版本:

```sql
CREATE TABLE order_snapshot
(
    order_id UInt64,
    member_id UInt64,
    order_status LowCardinality(String),
    amount Decimal(18, 2),
    update_time DateTime,
    version UInt64
)
ENGINE = ReplacingMergeTree(version)
PARTITION BY toYYYYMM(update_time)
ORDER BY order_id;
```

写入两条相同 `order_id` 的数据:

```sql
INSERT INTO order_snapshot VALUES
    (100001, 20001, 'CREATED', 199.00, '2026-07-21 10:00:00', 1),
    (100001, 20001, 'PAID',    199.00, '2026-07-21 10:05:00', 2);
```

需要注意:`ReplacingMergeTree` 的去重发生在**后台合并时**,不是写入后立刻完成。

临时需要查询最终结果时可以使用:

```sql
SELECT *
FROM order_snapshot FINAL
WHERE order_id = 100001;
```

```sql
Query id: 57b8981f-274c-4d6d-8a36-187e9b348534

   ┌─order_id─┬─member_id─┬─order_status─┬─amount─┬──────────update_time─┬─version─┐
1. │  100001  │     20001 │ PAID         │ 199.00 │ 2026-07-21 10:05:00  │       2 │
   └──────────┴───────────┴──────────────┴────────┴──────────────────────┴─────────┘

1 row in set. Elapsed: 0.008 sec.
```



但 `FINAL` 会触发额外合并计算,大表全量查询成本很高,**不能把它当作日常查询的默认写法**。

更推荐的做法是:

- 明确上游幂等逻辑;
- 在离线任务或物化视图中处理最终快照;
- 针对小范围主键查询才谨慎使用 `FINAL`。

**业务边界**:支付回调、订单 CDC 重试会产生同一订单的多版本记录。实时经营大盘应明确"近实时估算口径",财务对账则应读取已完成去重或结算的最终快照;不要让全量 `FINAL` 成为日常大盘的兜底方案。

---

## 6. 常用分析 SQL

### 6.1 进店、支付与事件统计

```sql
SELECT
    event_date,
    countIf(event_name = 'entry_detected') AS entry_count,
    uniqCombined64If(member_id, event_name = 'entry_detected') AS entry_uv,
    countIf(event_name = 'payment_success') AS payment_count,
    round(payment_count / nullIf(entry_count, 0), 4) AS entry_to_payment_rate
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_date BETWEEN '2026-07-01' AND '2026-07-21'
GROUP BY event_date
ORDER BY event_date;
```

```sql
Query id: df511fb3-bb28-47ef-9e90-573397be0c07

   ┌─event_date─┬─entry_count─┬─entry_uv─┬─payment_count─┬─entry_to_payment_rate─┐
1. │ 2026-07-21 │           1 │        1 │             1 │                     1 │
   └────────────┴─────────────┴──────────┴───────────────┴───────────────────────┘

1 row in set. Elapsed: 0.004 sec.
```

**业务怎么用**:进店人数没有下降、支付转化却下降时,运营不应先归因于客流;应继续按门店、设备、商品和支付通道下钻,判断是库存、识别、结算还是支付链路的问题。

| 函数 | 含义 | 特点 |
| --- | --- | --- |
| `count()` / `countIf()` | 统计行数或满足条件的事件数 | 用于进店数、订单数、支付数、设备错误数。 |
| `uniqExact()` | 精确去重 | 结果精确,但高基数场景内存消耗更高。 |
| `uniqCombined64()` | 近似去重 | 性能和精度平衡较好,常用于大规模到店会员数。 |
| `sum()` / `avg()` | 求和 / 平均值 | 常用于交易金额、设备耗时等指标。 |
| `quantile()` | 分位数 | 常用于 P95、P99 延迟指标。 |

### 6.2 租户 × 门店多维聚合

```sql
SELECT
    tenant_id,
    store_id,
    city,
    countIf(event_name = 'entry_detected') AS entry_count,
    countIf(event_name = 'payment_success') AS payment_count,
    round(payment_count / nullIf(entry_count, 0), 4) AS conversion_rate
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_date = '2026-07-21'
GROUP BY tenant_id, store_id, city
ORDER BY conversion_rate ASC
LIMIT 20;
```

```sql
Query id: 70b7aebb-ad81-45a1-ab5d-3164162faa23

   ┌─tenant_id─┬─store_id─────┬─city─┬─entry_count─┬─payment_count─┬─conversion_rate─┐
1. │ tenant-a  │ store-hz-001 │ 杭州 │           1 │             1 │               1 │
   └───────────┴──────────────┴──────┴─────────────┴───────────────┴─────────────────┘

1 row in set. Elapsed: 0.004 sec.
```

**业务怎么用**:只在少数门店异常时,优先检查当地网络、设备版本、库存和门店活动执行;不要因为局部门店问题直接修改全量商品或支付策略。

### 6.3 TopN 商品与购买转化

```sql
SELECT
    product_id,
    countIf(event_name = 'product_view') AS view_count,
    countIf(event_name = 'product_taken') AS taken_count,
    countIf(event_name = 'payment_success') AS payment_count,
    round(payment_count / nullIf(view_count, 0), 4) AS conversion_rate
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_date BETWEEN '2026-07-15' AND '2026-07-21'
  AND product_id IS NOT NULL
GROUP BY product_id
ORDER BY payment_count DESC
LIMIT 10;
```

```sql
Query id: 46c915ee-917d-4a11-9fbd-efd84c9e21db

   ┌─product_id─┬─view_count─┬─taken_count─┬─payment_count─┬─conversion_rate─┐
1. │       1001 │          1 │           0 │             1 │               1 │
   └────────────┴────────────┴─────────────┴───────────────┴─────────────────┘

1 row in set. Elapsed: 0.006 sec.
```

`countIf` 可以在一次扫描中完成多个条件指标统计,避免为每种事件写一条 SQL。**高浏览/拿取、低支付**的商品,应结合库存、价格、商品识别置信度和结算异常继续排查,而不是仅按销量下架或补货。

### 6.4 从进店到支付的漏斗

```sql
SELECT
    countIf(event_name = 'entry_detected') AS entry_uv,
    countIf(event_name = 'product_view') AS view_uv,
    countIf(event_name = 'checkout_started') AS checkout_uv,
    countIf(event_name = 'payment_success') AS payment_uv
FROM
(
    SELECT member_id, event_name
    FROM store_events
    WHERE tenant_id = 'tenant-a'
      AND store_id = 'store-hz-001'
      AND event_date = '2026-07-21'
      AND member_id IS NOT NULL
    GROUP BY member_id, event_name
);
```

```sql
Query id: eccf1816-edbb-4078-bc66-f705a2f725e1

   ┌─entry_uv─┬─view_uv─┬─checkout_uv─┬─payment_uv─┐
1. │        1 │       1 │           0 │           1 │
   └──────────┴─────────┴─────────────┴────────────┘

1 row in set. Elapsed: 0.004 sec.
```

上面的写法统计的是"当天至少发生过某个行为的会员数"。如果必须严格判断先后顺序,例如"先进店,再开始结算,最后支付",可以使用 ClickHouse 的 `windowFunnel`:

```sql
SELECT
    funnel_step,
    count() AS member_count
FROM
(
    SELECT
        member_id,
        windowFunnel(3600)(
            event_time,
            event_name = 'entry_detected',
            event_name = 'checkout_started',
            event_name = 'payment_success'
        ) AS funnel_step
    FROM store_events
    WHERE tenant_id = 'tenant-a'
      AND store_id = 'store-hz-001'
      AND event_date = '2026-07-21'
      AND member_id IS NOT NULL
    GROUP BY member_id
)
GROUP BY funnel_step
ORDER BY funnel_step;
```

其中 `3600` 表示会员必须在 1 小时内完成漏斗步骤。**业务怎么用**:每一层对应不同责任边界——进店后无浏览需检查门店体验或货架;开始结算后未支付则优先检查收银设备、支付通道和网络。

### 6.5 分位数:设备与支付接口 P95 / P99

```sql
SELECT
    event_date,
    device_id,
    quantile(0.50)(toFloat64(JSONExtractFloat(properties, 'cost_ms'))) AS p50_ms,
    quantile(0.95)(toFloat64(JSONExtractFloat(properties, 'cost_ms'))) AS p95_ms,
    quantile(0.99)(toFloat64(JSONExtractFloat(properties, 'cost_ms'))) AS p99_ms
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_name = 'api_request'
  AND event_date BETWEEN '2026-07-15' AND '2026-07-21'
GROUP BY event_date, device_id
ORDER BY event_date, p99_ms DESC;
```

均值正常但 P99 很高,仍意味着少量顾客会长时间无法完成结算,造成排队和转化损失。高频参与筛选和聚合的字段(如支付通道、设备型号、错误码)最好在写入时拆成独立列;不要长期依赖从 JSON 字符串中实时解析字段。

---

## 7. MySQL vs ClickHouse 概念对比

前面几章我们看到了 ClickHouse 怎么建表、怎么写 SQL,但作为主要用 MySQL 的开发者,最关心的问题其实是:**它和 MySQL 到底有什么不一样?什么时候该用它、什么时候不该用?**

这一章不写可运行的对照实验,只把两者在**存储、索引、写入、更新、场景**五个维度的根本差异讲清楚。

### 7.1 存储方式:行存 vs 列存

同一张 `store_events` 表,MySQL 和 ClickHouse 在磁盘上的组织方式完全不同。

**MySQL(行式存储)**:一行数据的所有字段紧挨在一起存。

```text
行存(按行连续保存)
┌──────────────────────────────────────────────────────────────────────────┐
│ 行1: [tenant-a, store-hz-001, 10001, door-01, 杭州, entry_detected, ...] │
│ 行2: [tenant-a, store-hz-001, 10001, shelf-03, 杭州, product_view,   ...] │
│ 行3: [tenant-a, store-hz-001, 10001, pos-01,   杭州, payment_success,...] │
└──────────────────────────────────────────────────────────────────────────┘
```

**ClickHouse(列式存储)**:同一列的所有值连续保存,不同列分开存。

```text
列存(按列连续保存)
tenant_id 列:  tenant-a, tenant-a, tenant-a, ...
store_id  列:  store-hz-001, store-hz-001, store-hz-001, ...
event_name列:  entry_detected, product_view, payment_success, ...
properties列:  {...}, {...}, {...}, ...(大字段,单独一列)
```

差异带来的结果:第 6.1 节那条 SQL 只用到 `tenant_id`、`event_date`、`event_name`、`member_id` 四列。

- **MySQL** 即使有索引,范围扫描时仍要把整行(包括体积很大的 `properties` 字段)从磁盘读出来,再丢弃不要的列;
- **ClickHouse** 只读这四列,`properties` 那一列根本不会被碰。

**结论:列越多、单次查询用到的列越少,ClickHouse 的优势越明显。** 这也是为什么 ClickHouse 特别适合「宽表 + 聚合」——宽表的字段动辄几十上百列,但单次分析通常只挑其中几列。

### 7.2 索引与查询:B+树 vs 稀疏索引 + 向量化

**MySQL 的 B+树索引**:为每个索引字段维护一棵 B+树,定位到具体的行。它最擅长的是「按主键或唯一索引精确找一条/少量数据」,点查几乎是常数级。但范围聚合查询(比如「最近 7 天所有门店的进店人数」)即使有索引,也要大量回表扫描,数据量一大就很慢。

**ClickHouse 的稀疏索引 + 向量化**:不为每一行建索引,而是按排序键每隔一段(默认 8192 行)保存一个标记;查询时先用稀疏索引跳过「完全不可能命中」的数据块,再对剩下的列做向量化批量计算。它不擅长「精确找一条」,但非常擅长「快速算一批」。

```text
MySQL:   "我要找到 id=100001 这条记录"       →  几乎瞬间
ClickHouse: "我要找到 id=100001 这条记录"     →  不一定快,这不是它的强项

MySQL:   "最近7天每个门店的进店人数"          →  扫大量行,几十秒~分钟级
ClickHouse: "最近7天每个门店的进店人数"       →  列裁剪+并行,百毫秒级
```

> 一句话总结:**MySQL 擅长「精确找一条」,ClickHouse 擅长「快速算一批」。**

### 7.3 写入:逐条事务 vs 批量追加

**MySQL**:逐条 `INSERT`,每条都是一个事务,有 redo log、undo log、MVCC 保证强一致性。写一条就能立刻查到,代价是单条写入的固定开销较高,但胜在**实时、一致、可频繁单条写**。

**ClickHouse**:写入的本质是「追加生成一个新的 Part」,后台再异步合并。它对**批量**写入非常友好——一次写几万、几十万行都很快;但对**逐条**写入极其不友好——每条都生成一个小 Part,合并跟不上,最终表现为查询变慢、磁盘碎片化。

```text
MySQL 的理想姿势:
  收到一条订单 → 立刻 INSERT → 立刻可查            ✓ 推荐

ClickHouse 的理想姿势:
  攒够一批(几千~几万条) → 一次 INSERT            ✓ 推荐
  收到一条事件 → 立刻 INSERT → 再收到一条 → INSERT ✗ 反模式,会产生大量小 Part
```

> 这就是为什么 ClickHouse 的数据通常通过 Kafka / Flink / 应用攒批写入,而不是业务代码直接逐条写。具体接入方式见《ClickHouse 进阶实战》第 1 章。

### 7.4 更新与删除:单行更新 vs Mutation 重写 ⭐

**这是 ClickHouse 和 MySQL 最大的差异,也是新手最容易踩坑的地方。**

**MySQL**:`UPDATE` / `DELETE` 是日常操作,按主键定位一行后原地修改,代价很低,可以频繁执行。

```sql
-- MySQL:轻量,毫秒级
UPDATE store_events SET store_id = 'store-hz-002' WHERE id = 100001;
```

**ClickHouse**:列式存储的数据是按 Part 批量压缩保存的,**没有「原地修改一行」的能力**。`ALTER TABLE ... UPDATE / DELETE`(称为 **Mutation**)的本质是:把命中的整个 Part 重新读出来、改完、再写回去。

```sql
-- ClickHouse:Mutation,可能重写整个 Part,代价极高
ALTER TABLE store_events UPDATE store_id = 'store-hz-002' WHERE event_id = '...';
```

一条 SQL 看起来差不多,但底层工作量天差地别:MySQL 改的是「一行」,ClickHouse 重写的是「一个 Part(几万到上百万行)」。在大表上频繁 Mutation 会把 IO 和合并队列打满,严重拖慢整个实例。

**正确姿势**:不要在 ClickHouse 上做频繁更新,而是用「**追加写 + 去重**」模拟更新:

- 数据有变更时,写一条新版本记录;
- 用 `ReplacingMergeTree`(见第 5.4 节)在后台合并时保留最新版本;
- 查询时用 `FINAL` 或在预聚合层处理最终口径。

> 记住一个原则:**ClickHouse 适合「写一次、读多次」的追加型数据,不适合「经常改」的数据。** 订单状态演进、库存这类会反复变更的数据,主库仍应是 MySQL,ClickHouse 只接收同步过来的快照。

### 7.5 适用场景对比表

| 对比项 | MySQL | ClickHouse |
| --- | --- | --- |
| 存储方式 | 行式存储 | 列式存储 |
| 索引类型 | B+树,精确点查极快 | 稀疏索引,范围聚合极快 |
| 典型操作 | 单条增删改查、事务 | 多维聚合、分组、TopN、漏斗 |
| 事务能力 | 强 ACID,多表事务 | 不以事务为目标 |
| 更新 / 删除 | 原地修改,轻量 | Mutation,重写 Part,代价高 |
| 点查(按主键查一条) | 强项 | 弱项 |
| 大表 Join | 强项,优化器成熟 | 弱项,建议宽表 / 字典规避 |
| 高频小批量写入 | 适合 | 不适合,要攒批 |
| 压缩率 | 一般 | 高(同列数据同类型) |
| 适合数据规模 | 千万级单表 | 亿~万亿级 |
| 典型场景 | 订单、库存、账户、配置 | 日志、埋点、指标、画像、报表 |
| SQL 兼容 | 标准 SQL | 高度兼容标准 SQL,但有自有函数 |

### 7.6 ClickHouse 的优势与劣势

**优势:**

1. **聚合查询极快**:亿级明细的多维聚合、分组、TopN 通常在百毫秒到秒级返回。
2. **压缩率高**:同列数据类型一致、重复度高,存储占用通常只有行式数据库的 1/5 ~ 1/10。
3. **SQL 友好**:用标准 SQL 就能完成绝大多数分析,学习成本低。
4. **实时写入、近实时可查**:数据写入后几秒内即可参与查询。
5. **并行计算**:一条 SQL 自动利用多核并行扫描,无需手动优化。

**劣势(必须讲透,避免误用):**

1. **不擅长更新 / 删除**:Mutation 代价极高,不适合频繁变更的数据。
2. **不擅长事务**:没有完整的 ACID 事务保证,不能当业务主库。
3. **不擅长点查**:按主键精确查一条记录不如 MySQL 快。
4. **不擅长 Join**:复杂多表 Join 性能和优化器都不如 MySQL,推荐用宽表或字典规避。
5. **不适合频繁小批量写入**:逐条 INSERT 会产生大量小 Part,拖垮合并。

**结论:ClickHouse 不是用来替代 MySQL 的,而是配合 MySQL 的。**

```text
┌───────────────┐                        ┌──────────────────┐
│   MySQL / PG   │  ──── CDC / Kafka ───► │   ClickHouse     │
│  订单、库存、    │      同步明细 / 快照       │  分析、报表、大盘   │
│  事务与一致性   │                        │  追加写、聚合读    │
└───────────────┘                        └──────────────────┘
   强事务、频繁更新                         海量数据、快速聚合
```

业务库还是 MySQL,分析库交给 ClickHouse,两者各司其职,这才是正确的用法。

### 7.7 实测:1000 万行数据下的性能对比

概念讲了这么多,到底差多少?下面用一组**真实压测**来回答。两端用**完全相同的一批 1000 万行数据**,MySQL 建了 4 个合理的复合索引,均预热后取稳定耗时。

#### 测试环境与数据

| 项 | ClickHouse | MySQL |
| -- | ---------- | ----- |
| 版本 | 26.7.1(MergeTree) | 5.7.44(InnoDB) |
| 数据量 | 10,000,000 行 `bench_store_events` | 同左(从 ClickHouse 导出的同一批数据) |
| MySQL 索引 | — | `idx_tenant_date`、`idx_tenant_store_date`、`idx_product`、`idx_city` |
| 数据分布 | 5 租户 / 500 门店 / 8 城市 / 7 种事件,跨 30 天 | 同左 |

#### 写入与存储对比

| 项 | ClickHouse | MySQL | 差距 |
| -- | ---------- | ----- | ---- |
| 写入 1000 万行耗时 | **10 秒**(批量 INSERT) | **342 秒**(LOAD DATA 批量导入) | ≈ 34 倍 |
| 磁盘占用 | **477 MiB** | **2818 MB(≈2.8 GB)** | ≈ 5.9 倍 |

> 写入端 ClickHouse 的批量追加远快于 MySQL;列存的高压缩比让同样数据只占 MySQL 的约 1/6 空间——**这部分差距是结构性的,跟配置无关。**

![示例数据结构](https://cdn.wcxian.cc/img/20260724170339084.png)

![ClickHouse数据大小](https://cdn.wcxian.cc/img/20260724170345301.png)

![Mysql数据大小](https://cdn.wcxian.cc/img/20260724170348431.png)

#### 四类典型查询 + 一个关键变量:`innodb_buffer_pool_size`

四条查询都是无人店 SaaS 最常见的分析 SQL,两端语义一致(MySQL 用 `SUM(条件)` 等价 `countIf`,`COUNT(DISTINCT)` 等价 `uniqCombined64`)。

> 📌 **关于测法**:下面的耗时都是**数据库内部执行时间**——ClickHouse 用 `clickhouse-client --time`(native 协议,排除 HTTP 开销),MySQL 用连接内的 `UNIX_TIMESTAMP(NOW(6))` 差值(排除进程启动开销)。两端都充分预热后取稳定值。

![ClickHouse查询](https://cdn.wcxian.cc/img/20260724174136097.png)

![Mysql命中索引查询](https://cdn.wcxian.cc/img/20260724174141921.png)

![ClickHouse查询耗时](https://cdn.wcxian.cc/img/20260724174145732.png)

![Mysql无索引查询耗时](https://cdn.wcxian.cc/img/20260724174149318.png)

测下来发现一个**决定 MySQL 命运的关键变量:`innodb_buffer_pool_size`**(MySQL 用来缓存数据页的内存)。下面分两种场景给出数据:

**场景 A:MySQL 内存不足(buffer pool = 128 MB,默认值,装不下整张表)**

| #    | 查询场景                                        | ClickHouse | MySQL    | 倍数      |
| ---- | ----------------------------------------------- | ---------- | -------- | --------- |
| Q1   | 租户最近 7 天每天进店 / 支付数(范围聚合 + 分组) | 17 ms      | 52100 ms | ≈ 3000 倍 |
| Q2   | 租户最近 7 天支付数 Top10 商品(TopN)            | 25 ms      | 81000 ms | ≈ 3200 倍 |
| Q3   | 租户最近 7 天到店会员 UV(去重)                  | 23 ms      | 53600 ms | ≈ 2300 倍 |
| Q4   | 按城市统计(全表扫描型 + 高基数去重)             | 110 ms     | 77500 ms | ≈ 700 倍  |

**场景 B:MySQL 内存充足(buffer pool = 4 GB,能装下整张表)**

| #    | 查询场景                                        | ClickHouse | MySQL    | 倍数     |
| ---- | ----------------------------------------------- | ---------- | -------- | -------- |
| Q1   | 租户最近 7 天每天进店 / 支付数(范围聚合 + 分组) | **17 ms**  | 520 ms   | ≈ 30 倍  |
| Q2   | 租户最近 7 天支付数 Top10 商品(TopN)            | **25 ms**  | 660 ms   | ≈ 26 倍  |
| Q3   | 租户最近 7 天到店会员 UV(去重)                  | **23 ms**  | 500 ms   | ≈ 22 倍  |
| Q4   | 按城市统计(全表扫描型 + 高基数去重)             | **110 ms** | 15900 ms | ≈ 145 倍 |

> ⚠️ **同一个 MySQL、同一份数据、同样的 SQL,只是 buffer pool 大小不同,Q1 就从 520 ms 涨到 52 秒。** 这个对比本身比「谁更快」更值得记住。



#### 怎么解读这两组数据

**1. 内存充足时(场景 A),ClickHouse 对 MySQL 是 22-30 倍优势(Q1-Q3)。** 这部分差距来自三个叠加效应:

- **列裁剪**:MySQL 聚合时要回表把整行(含 `properties`、`device_id` 等用不到的大字段)读出来;ClickHouse 只读 `tenant_id`、`event_date`、`event_name` 这几列,实际扫描的数据量少一个数量级;
- **向量化执行**:ClickHouse 的 `countIf` 在一批数据(默认 8192 行)上批量计算,而不是 MySQL 那样逐行判断;
- **稀疏索引 + 并行**:按排序键跳过其他租户、其他日期的数据块,多个 CPU 核心并行扫描不同 Part。

**2. Q4 拉开 145 倍差距,核心是「去重算法」。** Q4 要对 230 万行做 `COUNT(DISTINCT member_id)`,MySQL 在内存里建临时表精确去重,CPU 密集,要 15.9 秒;ClickHouse 用 `uniqCombined64`(HyperLogLog 近似算法,误差约 1%)只扫 `member_id` 一列就估算出结果,只要 110 ms。**如果业务能接受近似 UV,这个差距是算法层面的。**

**3. ClickHouse 几乎不受内存配置影响。** 两种场景下它的耗时几乎一样(17-110 ms),因为列存只读需要的几列(几百 MB 而非 2.8 GB),稀疏索引跳过无关数据块——**它天生就不需要把整张表塞进内存。**

**4. MySQL 的耗时高度依赖 buffer pool 能否装下热数据。** 场景 A 能装下时 Q1-Q3 是 500-660 ms;一旦装不下(场景 B),就要频繁读磁盘,直接退化到几十秒。**这是行式存储做聚合的固有特性:要扫大量行,数据不在内存里就得现读磁盘。**

#### 结论:为什么生产场景该选 ClickHouse

把两组数据放一起看,答案就清楚了:

- **内存充足时(已经是 MySQL 最理想的情况)ClickHouse 仍快 22-30 倍**(Q1-Q3),Q4 这种去重场景快 145 倍——优势来自列裁剪 + 向量化 + 近似算法的叠加;
- **真实生产里,MySQL 的内存通常很紧张**:它还要服务线上订单、支付、库存事务,不可能给一张分析大表独占几 GB buffer pool。一旦内存不够,同样的查询会退化成几十秒甚至更久(场景 B),**还会反过来拖慢线上事务**;
- **ClickHouse 把分析负载从 MySQL 摘出来**,既让分析查询稳定保持在百毫秒级(不抢业务库内存),也让 MySQL 专注做事务——这才是它真正的价值。

> ⚠️ 注意:以上对比只针对**分析型聚合查询**。按主键查一条订单、强事务写入更新,MySQL 依然是更合适的选择——这正是第 7.5 节场景对比表要表达的意思。

---

## 8. Spring Boot 集成 ClickHouse

### 8.1 引入依赖

Maven 中添加 ClickHouse JDBC 驱动:

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>${clickhouse-jdbc.version}</version>
</dependency>
```

具体版本请以 Maven Central 和项目兼容性为准。生产项目建议锁定并测试驱动版本,不要长期使用浮动版本。

### 8.2 配置数据源

多数据源场景下,`spring.datasource.clickhouse` 不会被 Spring Boot 自动注册成 `clickHouseDataSource` Bean;需要显式绑定配置并创建数据源。以下示例使用 Hikari:

`application.yml`:

```yaml
spring:
  datasource:
    clickhouse:
      jdbc-url: jdbc:clickhouse:http://127.0.0.1:8123/analytics
      username: app_user
      password: ${CLICKHOUSE_PASSWORD}
      driver-class-name: com.clickhouse.jdbc.ClickHouseDriver
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 5000
```

```java
@Configuration
public class ClickHouseDataSourceConfig {

    @Bean(name = "clickHouseDataSource")
    @ConfigurationProperties("spring.datasource.clickhouse")
    public DataSource clickHouseDataSource() {
        return new HikariDataSource();
    }
}
```

> `HikariDataSource` 使用 `jdbc-url` 属性;若改用 `DataSourceProperties` 绑定方式,则应使用 `url` 并在配置类中调用 `initializeDataSourceBuilder()`。两种方式任选其一,避免混用。

如果项目同时连接 MySQL 和 ClickHouse,建议配置多数据源,明确区分:

```text
MySQL DataSource       → 业务 CRUD、事务数据
ClickHouse DataSource  → 指标查询、报表分析、批量写入
```

### 8.3 查询示例

```java
/**
 * 无人店门店事件分析查询示例
 */
@Repository
public class StoreEventAnalyticsRepository {

    private final DataSource clickHouseDataSource;

    public StoreEventAnalyticsRepository(
            @Qualifier("clickHouseDataSource") DataSource clickHouseDataSource) {
        this.clickHouseDataSource = clickHouseDataSource;
    }

    public List<DailyEntryUv> queryDailyEntryUv(
            String tenantId, LocalDate startDate, LocalDate endDate) throws SQLException {

        String sql = """
                SELECT
                    event_date,
                    uniqCombined64If(member_id, event_name = 'entry_detected') AS entry_uv
                FROM store_events
                WHERE tenant_id = ?
                  AND event_date >= ?
                  AND event_date < ?
                GROUP BY event_date
                ORDER BY event_date
                """;

        List<DailyEntryUv> result = new ArrayList<>();
        try (Connection connection = clickHouseDataSource.getConnection();
             PreparedStatement statement = connection.prepareStatement(sql)) {

            statement.setString(1, tenantId);
            statement.setObject(2, startDate);
            statement.setObject(3, endDate);

            try (ResultSet rs = statement.executeQuery()) {
                while (rs.next()) {
                    result.add(new DailyEntryUv(
                            rs.getObject("event_date", LocalDate.class),
                            rs.getLong("entry_uv")
                    ));
                }
            }
        }
        return result;
    }

    public record DailyEntryUv(LocalDate eventDate, long entryUv) {}
}
```

### 8.4 批量写入示例

```java
public void batchInsert(List<StoreEvent> events) throws SQLException {
    String sql = """
            INSERT INTO store_events
            (event_id, tenant_id, store_id, member_id, device_id, city, event_name,
             order_id, product_id, page_code, event_time, properties)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """;

    try (Connection connection = clickHouseDataSource.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {

        for (StoreEvent event : events) {
            statement.setObject(1, event.eventId());
            statement.setString(2, event.tenantId());
            statement.setString(3, event.storeId());
            statement.setObject(4, event.memberId());
            statement.setString(5, event.deviceId());
            statement.setString(6, event.city());
            statement.setString(7, event.eventName());
            statement.setObject(8, event.orderId());
            statement.setObject(9, event.productId());
            statement.setString(10, event.pageCode());
            statement.setObject(11, event.eventTime());
            statement.setString(12, event.properties());
            statement.addBatch();
        }

        statement.executeBatch();
    }
}
```

建议:

- 优先通过 Kafka、Flink、Logstash、Vector 等通道批量导入;
- 如果业务服务直接写入,按一定条数或时间窗口**攒批**后写入;
- 大批量数据可使用 `JSONEachRow`、CSV、Parquet 等格式写入,提高吞吐;
- ClickHouse 不适合把每一条设备、门禁或交易事件都同步写入。

---

## 9. 常见误区

### 9.1 把 ClickHouse 当 MySQL 使用

错误方式:

```sql
UPDATE store_events
SET store_id = 'store-hz-001'
WHERE event_id = '550e8400-e29b-41d4-a716-446655440000';
```

ClickHouse 支持 Mutation,但底层往往需要重写相关数据 Part,成本远高于行式数据库的单行更新。(原因详见第 7.4 节)

正确思路:

- 明细事实数据尽量采用追加写(Append Only);
- 需要更正的数据可写入新版本,配合 `ReplacingMergeTree` 或下游聚合处理;
- 大规模历史修正应规划批处理窗口,避免业务高峰执行。

### 9.2 每条消息执行一次 INSERT

错误方式:

```text
收到一条 Kafka 消息 → HTTP INSERT 一次 → 产生一个小 Part
```

正确方式:

```text
Kafka 批量消费 / 应用内缓存攒批 → 每批数千~数万条写入 → 后台高效合并
```

### 9.3 不带时间条件查询大表

错误方式:

```sql
SELECT count()
FROM store_events
WHERE event_name = 'entry_detected';
```

正确方式:

```sql
SELECT count()
FROM store_events
WHERE tenant_id = 'tenant-a'
  AND event_date >= today() - 7
  AND event_name = 'entry_detected';
```

### 9.4 过度分区

错误方式:

```sql
PARTITION BY store_id
```

这样会造成海量分区,严重影响元数据管理和后台合并。

一般按时间做分区即可,租户、门店、设备、商品等查询维度通过 `ORDER BY`、预聚合或跳数索引解决。

### 9.5 盲目使用 FINAL

`FINAL` 能在查询时强制合并数据,对去重表有用,但代价不低。

在报表和大范围查询中,应该优先通过表模型、聚合表和数据处理流程保证数据正确性,而不是依赖 `FINAL` 兜底。

---

## 10. 选型对比与优劣势

### 10.1 ClickHouse、MySQL、Elasticsearch 如何选择

| 场景 | 推荐组件 | 原因 |
| --- | --- | --- |
| 订单创建、支付确认、库存扣减、门店配置 | MySQL / PostgreSQL | 强事务、行级更新、主键查询能力强。 |
| 门店事件、设备日志、经营与转化分析 | ClickHouse | 可按租户、门店、设备、商品和时间快速做多维聚合。 |
| 商品标题、文章内容搜索 | Elasticsearch / OpenSearch | 全文检索、相关性评分、分词能力强。 |
| 全文检索结果的统计分析 | Elasticsearch + ClickHouse | ES 负责搜索,ClickHouse 承担复杂离线/实时统计。 |
| 秒级实时指标大盘 | Kafka / Flink + ClickHouse | 流式接入、实时聚合、OLAP 查询。 |

### 10.2 ClickHouse 的优势

1. **聚合查询极快**:亿级明细的多维聚合、分组、TopN 通常在百毫秒到秒级返回,这是它最核心的价值。
2. **压缩率高**:列存 + 同类型数据,存储占用通常只有行式数据库的 1/5 ~ 1/10。
3. **SQL 友好**:高度兼容标准 SQL,熟悉 MySQL 的人上手很快。
4. **实时写入、近实时可查**:数据写入后几秒内即可查询。
5. **并行计算**:一条 SQL 自动利用多核,无需手动调优并发。

### 10.3 ClickHouse 的劣势(选型时必须正视)

> 新手最容易高估 ClickHouse 的能力,以为「快」就能替代一切。下面这些短板决定了它**只能做分析库,不能做业务主库**。

1. **不擅长更新 / 删除**:Mutation 要重写整个 Part,代价极高。订单状态、库存这类频繁变更的数据不能存在 ClickHouse。
2. **不擅长事务**:没有完整 ACID,多表事务、隔离级别都无法保证。
3. **不擅长点查**:按主键精确查一条记录,远不如 MySQL 的 B+树快。
4. **不擅长 Join**:复杂多表 Join 的性能和优化器都不成熟,推荐用宽表或字典规避。
5. **不适合频繁小批量写入**:逐条 INSERT 会产生大量小 Part,拖垮后台合并,影响查询。
6. **生态不如 MySQL 成熟**:运维工具、ORM、人才储备都不如 MySQL 普及,团队需要额外学习成本。

### 10.4 一句话选型原则

```text
要做事务、要频繁改数据、要点查一条       →  MySQL
要做海量数据的聚合分析、报表、大盘        →  ClickHouse
要做全文检索、相关性排序                  →  Elasticsearch
三者各司其职,组合使用才是常态
```

很多业务系统最终不是只选择一个数据库,而是根据数据访问模式进行组合:

```text
MySQL / PostgreSQL:订单、库存、租户和门店配置等事务事实
Kafka:门禁、设备、交易和服务事件的传输与削峰
Flink:实时清洗、关联、口径计算
ClickHouse:门店事件沉淀、设备分析和经营指标
Redis:热点门店状态与低延迟缓存
Elasticsearch:商品、工单等全文检索
```

---

## 11. 总结

ClickHouse 的价值不只是「SQL 跑得快」,更重要的是它改变了处理海量分析数据的方式:

1. **用列式存储和向量化计算,加速海量门店、设备与交易事件的扫描和聚合。**
2. **用 MergeTree 表模型、排序键和分区设计,让"租户 → 门店 → 时段"的下钻查询保持可控。**
3. **它和 MySQL 不是替代关系,而是分工关系**:MySQL 管事务和频繁变更,ClickHouse 管海量数据的聚合分析。
4. **它的短板同样明确**:不擅更新、不擅事务、不擅点查、不擅 Join、不适合频繁小批量写入——这些场景仍应交给 MySQL。
5. **正确的架构是 MySQL + ClickHouse 组合**:业务库负责订单、库存、事务,ClickHouse 负责报表、大盘、多维分析,中间用 CDC / Kafka 同步。

对无人值守商店 SaaS 而言,ClickHouse 的价值不只是"SQL 跑得快":它让团队能更快区分客流、商品、设备和支付问题,定位异常门店,并持续验证修复是否真正改善了经营结果。

> 进阶内容(Kafka 实时接入、物化视图预聚合、性能优化、集群高可用、运维监控)见《ClickHouse 进阶实战》。

---

## 参考资料

- [ClickHouse 官方文档](https://clickhouse.com/docs)
- [ClickHouse SQL Reference](https://clickhouse.com/docs/sql-reference)
- [ClickHouse MergeTree 引擎](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Materialized View](https://clickhouse.com/docs/materialized-view)
- [ClickHouse Kafka 表引擎](https://clickhouse.com/docs/engines/table-engines/integrations/kafka)
- 进阶篇:《ClickHouse 进阶实战》
