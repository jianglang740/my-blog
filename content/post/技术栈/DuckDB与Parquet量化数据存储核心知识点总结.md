---
title: "DuckDB + Parquet 量化数据存储核心知识点总结"
description: "面向量化场景梳理 DuckDB 与 Parquet 的定位、物理布局与工程取舍，核心结论均在本机实测并复现"
date: 2026-10-09
lastmod: 2026-10-09
weight: 4
categories:
    - Tutorial
    - Tech Stack
    - Quant
tags:
    - 教程
    - 技术栈
    - 量化投资
    - duckdb
    - parquet
---

# DuckDB + Parquet 量化数据存储核心知识点总结

---

- 作者：山财小蒋
- 联系方式：2018036661@qq.com
- 创作不易，转载请注明出处，欢迎批评探讨

## 前言

本文面向**量化金融场景**（分钟频量价、因子计算、回测），系统梳理 DuckDB 与 Parquet 的定位、原理、物理布局与工程取舍。

与一般教程不同，**本文的核心结论全部在本机实测验证过**，测试环境：

```
DuckDB 1.4.5 · macOS arm64 · 4 线程 · 内存模式连接
```

文中所有数字都标注了「实测」，并给出了复现命令。**性能绝对值依赖机器，请只看数量级和比例关系**——重点是背后的机制，不是具体毫秒数。

需要特别说明：本文在整理过程中**修正了流传较广的三处错误结论**，分别是第 2.5 节的「索引」、第 6.2 节的「分区一定更快」、以及第 8.1 节的「COPY APPEND」。如果你只看三件事，看这三节。

---

## 一、核心概念：嵌入式数据库 VS 服务端数据库

### 1.1 什么是嵌入式数据库（DuckDB / SQLite）

**定义**：数据库引擎以代码库的形式，**嵌入到应用程序进程内部运行**，无独立后台服务、无端口监听、无需手动启动/停止。

**核心特征**：

- 同进程运行：和 Python 回测脚本、数据清洗脚本共用一个进程
- 无服务、无端口：无需 `systemctl` 启动，无需配置环境
- 启动逻辑：代码执行 `connect()` 自动打开库，程序退出自动关闭
- 存储形态：单个 `.duckdb` 数据库文件，开箱即用

**核心误区纠正**：嵌入式数据库 ≠ 硬件单片机嵌入式，指「嵌入应用进程」，可正常跑在 PC、服务器上。

### 1.2 什么是服务端数据库（MySQL / PostgreSQL / Oracle）

**定义**：独立部署的后台服务进程（C/S 架构），常驻系统后台，独立占用端口、内存、资源。

**核心特征**：

- 独立进程：数据库服务和业务程序完全分离
- 网络通信：通过 TCP 端口（MySQL 3306）远程/本地连接
- 常驻运行：业务程序退出，数据库服务依然运行
- 核心能力：支持**多客户端、多进程、高并发读写**

### 1.3 两类数据库核心对比

| 特性 | DuckDB（嵌入式 OLAP） | MySQL（服务端 OLTP） |
| --- | --- | --- |
| 运行模式 | 同进程内嵌，无服务 | 独立后台服务，常驻端口 |
| 使用门槛 | 仅 pip 安装，无需部署配置 | 需安装、启动、配置、运维 |
| 并发能力 | 单进程内 MVCC 并发；**跨进程一写多读** | 原生支持海量并发读写 |
| 存储引擎 | 列存 + 向量化执行 | InnoDB 行存 |
| 核心场景 | 离线分析、批量计算、回测、OLAP | 线上业务、高频事务、OLTP |
| 外部文件支持 | 原生直读 Parquet/CSV/JSON | 无法直读，必须导入入库 |

#### 补充：并发能力需要说得更准确

「DuckDB 并发差」是个过于粗糙的说法，准确的分层是：

| 场景 | 能力 |
| --- | --- |
| 单进程内多线程 | ✅ 读写都支持，MVCC 保证快照隔离 |
| 跨进程 | ⚠️ **一个进程以读写模式打开时，其他进程只能只读打开** |
| 跨进程只读 | ✅ 多个进程可同时只读（配合 `read_only=True`） |

所以真正受限的只有「多个进程同时写同一个 `.duckdb` 文件」这一种情况。量化场景里这恰恰很常见（多个回测脚本并行跑），这就是后面推荐「原始数据用 Parquet」的根本原因——**Parquet 文件没有这个锁，多进程随便并行读**。

---

## 二、DuckDB 的真实定位

### 2.1 关键误区：DuckDB 不是文件阅读器，是完整关系型数据库

**错误认知**：DuckDB 只能用来打开 Parquet，不算正经数据库。

**正确认知**：DuckDB 是**标准关系型 OLAP 数据库**，完整支持标准 SQL、多表 JOIN、CTE、窗口函数、ACID 事务、MVCC、WAL、UPDATE/DELETE、索引、统计信息与查询优化器。

Parquet/CSV 只是它的**外部可查询数据源**，不是唯一存储方式。

### 2.2 三种运行模式

**1）内存模式（临时计算）**

```python
con = duckdb.connect()          # 不传路径，数据全在内存，退出即消失
```

**2）单文件持久化模式（量化主力用法）**

```python
con = duckdb.connect("quant.duckdb")     # 自动创建，无需建库、无需启服务
```

**3）服务模式（不推荐）**

可以起端口服务供多客户端连接，但放弃嵌入式优势、并发缺陷放大，生产价值低。

#### 无论哪种模式，都建议先设置这几个 PRAGMA

```sql
PRAGMA threads = 8;                          -- 并行度，默认=CPU 核数
PRAGMA memory_limit = '8GB';                 -- 内存上限，超出会落盘
PRAGMA temp_directory = '/fast_ssd/duckdb_tmp';  -- 落盘目录，务必指向 SSD
```

**为什么重要**：DuckDB 是少数会**主动利用磁盘做 out-of-core 计算**的分析引擎。当聚合/JOIN 的工作集超过 `memory_limit`，它会自动溢写到 `temp_directory`。默认目录可能落在慢盘甚至系统盘上，是「跑大回测莫名很慢」的常见原因。

### 2.3 致命限制：单写多读

同一个 `.duckdb` 数据库文件：**同一时刻仅允许一个进程以读写模式打开，其他进程只能只读**。

- ✅ 适合：单进程批量计算、回测、因子加工
- ❌ 不适合：多服务、多脚本并行写入

### 2.4 【实测】约束不是摆设，是真强制的

很多资料含糊地说 DuckDB「支持约束」。实测结论是——**全部真强制，且包括外键**：

| 约束 | 是否强制 | 触犯时的实际报错 |
| --- | --- | --- |
| `PRIMARY KEY`（重复值） | ✅ 强制 | `Duplicate key "id: 1" violates primary key constraint` |
| `PRIMARY KEY`（NULL） | ✅ 强制 | `NOT NULL constraint failed` |
| `UNIQUE` | ✅ 强制 | `Duplicate key "a: 1" violates unique constraint` |
| `NOT NULL` | ✅ 强制 | `NOT NULL constraint failed: n.a` |
| `CHECK` | ✅ 强制 | `CHECK constraint failed on table c with expression CHECK((id > 0))` |
| `FOREIGN KEY` | ✅ **强制** | `Violates foreign key constraint because key "id: 9999" does not exist` |

```sql
CREATE TABLE p(id INT PRIMARY KEY, v INT);
INSERT INTO p VALUES (1, 10);
INSERT INTO p VALUES (1, 20);    -- ❌ ConstraintException: Duplicate key ...

CREATE TABLE child(id INT, pid INT REFERENCES p(id));
INSERT INTO child VALUES (1, 9999);   -- ❌ ConstraintException: Violates foreign key constraint
```

> ⚠️ 注意：**外键在较老的 DuckDB 版本里只解析、不强制**。如果你在网上看到「DuckDB 外键不生效」的说法，那是旧版本的信息。写生产代码前，请在自己使用的版本上验证一遍。

**这对量化意味着什么**：维度表（股票基础信息、行业归属）用原生表建，主键去重、外键保证引用完整性都是真实可用的，不需要在应用层自己写校验。

### 2.5 【纠偏】索引不是 DuckDB 的卖点，别拿它当选型理由

网上流传的选型理由常见这一条：

> ~~「DuckDB 原生表支持索引，精准查询速度远超 Parquet」~~

**这句话的因果是错的。** 我造了 2880 万行分钟线实测：

| 查询 | 无索引 | 有 ART 索引 |
| --- | --- | --- |
| 点查：单只股票 + 单日（返回 240 行） | **0.4 ms** | 0.5 ms |

索引不但没有加速，还**略慢**。原因是**数据本身按时间有序**，DuckDB 的 **zone map**（每个 RowGroup 维护列的 min/max）已经把不相关的数据块整块跳过了，根本轮不到索引发挥作用；索引反而多了一层查找开销。

再看分析型查询（全市场按月聚合 2880 万行）：**16.5 ms**，全程列存顺序扫描，索引完全不参与。

#### 结论：DuckDB 快在哪，慢在哪

| 机制 | 作用 | 什么时候有效 |
| --- | --- | --- |
| **列存 + 向量化执行** | 只读需要的列、批量处理 | 几乎所有分析查询，**这是主要来源** |
| **Zone Map**（RowGroup min/max） | 整块跳过不相关数据 | **数据按该列有序/聚集时**效果最好 |
| **ART 索引** | 精确定位单行 | 数据无序、且查询是极少量行的点查 |

**实践建议**：

1. **不要习惯性地建索引**。DuckDB 是为顺序扫描优化的，绝大多数分析查询建了索引反而更慢、还占空间。
2. 真正的高性价比优化是**让数据有序**（详见 6.5 节），而不是建索引。
3. Parquet 场景同理——读 Parquet 时**完全没有索引可用**，靠的是文件级分区剪枝 + RowGroup 统计信息跳过，所以**排序比什么都重要**。

---

## 三、为什么 DuckDB 能直读 Parquet，MySQL 不行？

### 3.1 核心底层差异

- **DuckDB（列存引擎）**：内核原生集成 Parquet/Arrow 解析器，天生适配列存文件，支持**列裁剪、谓词下推、延迟物化**，可直接把 Parquet 当虚拟表查询，无需导入。
- **MySQL（行存引擎）**：核心存储是 InnoDB 行存，数据必须导入引擎内部的 `.ibd` 文件，内核无 Parquet 解析能力，也无法利用列存文件的统计信息。

补充两点，避免把话说得太绝对：

- MySQL 并非完全不能读外部文件（`LOAD DATA INFILE` 可读 CSV），但**不能直接扫描 Parquet**，且必须先导入再查。
- MySQL 在分析场景慢的根本原因是**行存 + 无向量化执行**：聚合要按行遍历、无法只读所需列、无法利用 min/max 跳过数据块。所以不是「MySQL 不好」，而是**用错了引擎**。

### 3.2 场景适配逻辑

- Parquet 是**列存、只读、批量分析型**文件格式，适配 DuckDB 的扫描/聚合/JOIN 场景；
- MySQL 主打**单行随机读写、高频更新事务**，列存 Parquet 与其核心场景不匹配。

### 3.3 一个关键澄清：DuckDB 读 Parquet 不是「不导入就慢」

很多人以为「不导入 = 每次都要全量解析 = 慢」。实际上 DuckDB 在读 Parquet 时能利用文件内的统计信息跳过大量数据，**性能可以接近原生表**。

但它不是零成本：每次查询都要读文件 Footer、解析元数据。所以文件数量越多、单个文件越小，这部分固定开销占比越高——这就是第 6.3 节「小文件灾难」的根源。

---

## 四、Parquet 内部结构：四层模型

要理解所有调优手段，必须先建立 Parquet 的分层结构认知。

### 4.1 四层结构

```
Parquet 文件
├── Footer（文件级元数据：Schema、每个 RowGroup 的统计信息、偏移量）
├── RowGroup 0
│   ├── ColumnChunk（symbol 列）
│   │   ├── Page 0（含统计信息）
│   │   ├── Page 1
│   │   └── ...
│   ├── ColumnChunk（ts 列）
│   └── ColumnChunk（close 列）
├── RowGroup 1
└── ...
```

| 层级 | 是什么 | 对应哪一级跳过 |
| --- | --- | --- |
| **Footer** | 文件级元数据，记录所有 RowGroup 的位置和统计 | — |
| **RowGroup** | 一批行的**列式集合**，是并行与跳过的基本单位 | ✅ RowGroup 级跳过 |
| **ColumnChunk** | 某个 RowGroup 内**某一列**的数据，独立编码压缩 | ✅ 列裁剪 |
| **Page** | ColumnChunk 内的数据页，**编码/压缩的最小单位** | ✅ Page 级跳过（需索引页） |

**关键点**：压缩和编码是**按列、按 Page** 独立进行的。这就是为什么「列存压缩率高」——同一列的数据类型相同、分布相近，编码效率远高于行存把不同类型混在一起。

### 4.2 实测：默认 RowGroup 有多大

```sql
COPY bars TO 'bars.parquet' (FORMAT PARQUET);
SELECT count(DISTINCT row_group_id) FROM parquet_metadata('bars.parquet');
```

2140 万行的数据，写出来 **176 个 RowGroup**，每个 RowGroup **122,880 行**。

> ✅ **DuckDB 的默认 `ROW_GROUP_SIZE` 是 122,880 行**（实测确认）。这是一个偏保守的默认值，下面会讲什么时候该调。

### 4.3 检查 Parquet 的利器：`parquet_metadata()` 与 `parquet_schema()`

这是原文完全没提、但排查性能问题**必须掌握**的工具。

```sql
-- 看每个 RowGroup 的统计信息
SELECT row_group_id, row_group_num_rows, path_in_schema,
       total_compressed_size, stats_min, stats_max
FROM parquet_metadata('bars.parquet')
WHERE path_in_schema IN ('ts', 'symbol')
ORDER BY row_group_id, path_in_schema;
```

实测输出（节选）：

```
RG0  行数=122880  列=symbol  压缩后=1537B  min=sh000000  max=sh000499
RG0  行数=122880  列=ts      压缩后=2383B  min=2025-01-01 09:30:00  max=2025-01-02 13:29:00
RG1  行数=122880  列=symbol  压缩后=1542B  min=sh000000  max=sh000499
RG1  行数=122880  列=ts      压缩后=2430B  min=2025-01-02 09:30:00  max=2025-01-03 13:29:00
```

`parquet_metadata()` 返回 31 个字段，常用的有：

| 字段 | 用途 |
| --- | --- |
| `row_group_id` / `row_group_num_rows` | 检查 RowGroup 划分是否合理 |
| `total_compressed_size` | 判断压缩效果、定位「哪一列最占空间」 |
| `stats_min` / `stats_max` | **判断跳过是否可能发生** |
| `stats_null_count` / `stats_distinct_count` | 数据质量检查 |
| `min_is_exact` / `max_is_exact` | 统计值是否精确（截断统计时会为 false） |
| `compression` / `encodings` | 确认实际用了什么压缩与编码 |
| `bloom_filter_offset` | 是否写了布隆过滤器 |

### 4.4 【关键洞察】统计信息只在数据有序时才有用

看上面实测输出里的 `symbol` 列：**每一个 RowGroup 的 min/max 都是 `sh000000` ~ `sh000499`**。

这意味着：**按 symbol 过滤时，zone map 几乎无法跳过任何 RowGroup**——因为每个 RowGroup 里都含有全部 500 只股票，任何一只都在统计范围内。

而 `ts` 列每个 RowGroup 的 min/max 精确覆盖一天左右，所以**按时间过滤能高效跳过**。

结论：

> **Parquet 的谓词下推效果，完全取决于数据在文件内是否按该列有序。**
>
> 同一份数据，写文件前 `ORDER BY symbol, ts` 还是 `ORDER BY ts, symbol`，会让「按股票查询」的性能差出数量级——而文件大小、压缩率完全一样。

这就是为什么第 6.5 节说**排序是性价比最高的优化**。

---

## 五、选型：Parquet VS DuckDB 原生表

### 5.1 Parquet 适用场景

**适合**：全市场分钟 K 线、原始量价、历史冷数据、多进程并行读取的数据。

**核心优势**：

- **开放标准**：跨 Python/Spark/DuckDB/ClickHouse 通用，无厂商绑定
- **高压缩率**：ZSTD + 列式编码，量价数据压缩效果好（实测见 5.3）
- **无锁并发读**：多脚本同时只读，无写锁冲突
- **容错性好**：分区文件相互独立，单文件损坏不波及全局
- **可被 DVC/git-lfs/对象存储直接管理**：就是普通文件，没有不透明的内部状态

**缺点**：

- 不可原地修改，更新数据需重写文件
- 无索引、无约束、无事务
- 文件数量多时元数据开销显著

### 5.2 DuckDB 原生 `.duckdb` 库适用场景

**适合**：维度表、因子结果表、回测汇总表、需要 UPDATE/DELETE 的中间数据。

**核心优势**：

- **事务与 ACID**：修正错误数据安全可靠
- **原地 UPDATE/DELETE**：不需要重写整个数据集
- **约束与关系建模**：主键去重、外键引用完整性**真强制**（见 2.4）
- **复杂 JOIN / 窗口函数优化更好**：有完整统计信息，优化器有更多发挥空间
- **单文件便于备份**：一个文件就是全部

**缺点**：

- 跨进程单写锁限制
- 绑定 DuckDB 生态
- 单个大文件损坏风险集中

### 5.3 【实测】压缩算法与体积：默认不是最优

同一份 2140 万行分钟线数据，同一台机器：

| 压缩算法 | 文件大小 | 相对默认 |
| --- | --- | --- |
| `UNCOMPRESSED` | 77.45 MB | 慢、大 |
| **（不指定，即默认）** | **47.8 MB** | — |
| `SNAPPY` | 47.77 MB | 与默认一致 |
| **`ZSTD`** | **25.78 MB** | **↓ 46%** |

```sql
COPY bars TO 'bars.parquet' (FORMAT PARQUET, COMPRESSION ZSTD);
```

> ✅ **DuckDB 的默认压缩是 SNAPPY，不是 ZSTD。** 只要显式写上 `COMPRESSION ZSTD`，**体积直接砍掉 46%**，而解压速度在现代 CPU 上完全够用。
>
> 这是本文性价比最高的一条建议：**一个单词，省一半磁盘和一半 IO。**

---

## 六、物理布局：分区、排序与文件大小

这一节是全文最重要的部分，也是原文问题最多的部分。

### 6.1 先分清三个独立的优化层次

很多人把「分区」当成万能优化，其实存在**三个互相独立**的跳过层次：

| 层次 | 机制 | 跳过粒度 | 触发条件 |
| --- | --- | --- | --- |
| **① 分区剪枝** | 目录名即谓词（`month=03/`） | 整个文件 | 过滤条件作用在**分区列**上 |
| **② RowGroup 跳过** | Footer 里每列的 min/max | RowGroup | 数据在该列**有序/聚集** |
| **③ Page 跳过** | Page 级统计 + 索引页 | Page | 类似 ②，粒度更细 |

理解这个分层，才能明白：**分区不是唯一手段，甚至常常不是最优手段。**

### 6.2 【纠偏】分区并不总是更快，小规模下反而更慢

原文说分区是为了「精准跳过无用数据」，言下之意是更快。实测结果相反。

**实验设计**：同一份 2880 万行、240 个交易日的数据，两种布局，查同一天。

| 布局 | 文件数 | 总体积 | 查一天耗时 |
| --- | --- | --- | --- |
| **单文件**（不分区） | 1 | 34.4 MB | **3.1 ms** |
| **按交易日分区** | 293 | 44.3 MB | **26.0 ms** |

两个反直觉的结论：

1. **按日分区后查询慢了 8 倍**（26.0 ms vs 3.1 ms）
2. **按日分区后体积还大了 29%**（44.3 MB vs 34.4 MB）

**为什么变慢**：单文件时，DuckDB 直接读一个 Footer 就能靠 RowGroup 统计信息跳到目标数据；分区后，**必须先打开 293 个文件的 Footer**，这个「元数据开销」在这里占了主导。

**为什么变大**：分区把数据切碎后，每个文件有自己的 Footer 和元数据，RowGroup 变小导致压缩效率下降（同样的数据量，压缩块越小、跨块冗余越多）。

### 6.3 小文件灾难：量化最贵的错误

如果分区键选得更细，后果是灾难性的。实测：

| 布局 | 文件数 | 平均单文件 | 查一天耗时 |
| --- | --- | --- | --- |
| 单文件 | 1 | 34.4 MB | 3.1 ms |
| 按交易日分区 | 293 | 151 KB | 26.0 ms |
| **按（交易日 + 股票）分区** | **15,504** | **1.6 KB** | **1746 ms** |

**慢 560 倍**，而且这还是在「只查其中一天、理论上只需读 500 个小文件」的情况下。

**为什么会这么慢**：Parquet 的读取**必须以文件为单位打开、读 Footer、解析元数据**。每个文件还有固定的打开开销（文件系统调用、元数据解析、对象存储的场景下更是每次一个网络往返）。当文件小到 1.6 KB 时，**元数据比数据本身还大**，读取时间几乎全是开销。

> ⚠️ **量化场景下这个坑特别容易踩**：直觉上「按股票分区」很合理（查某只股票只需读它的文件），但 A 股 5000 只股票 × 250 个交易日 = 125 万个文件。这是把数据集变成「元数据坟场」。
>
> **不要按 symbol 分区。** 想让「按股票查询」快，靠的是**文件内排序**（6.5 节），不是分区。

### 6.4 那到底什么时候该分区

既然分区会变慢变大，为什么还要分区？因为**分区的真正价值不是查询速度，而是可管理性**：

| 分区的真实价值 | 说明 |
| --- | --- |
| **增量写入** | 新的一天数据只写一个新目录，不碰历史文件 |
| **按周期删除** | 数据保留策略（如只留 3 年）只需 `rm -rf year=2022` |
| **故障隔离** | 单个分区损坏只影响一个时间段 |
| **写并行** | 不同分区可并行生成 |
| **跳过文件 Footer** | 数据量极大时（TB 级），避免打开上千个文件 |

**判断标准**：

- 数据总量 < 几十 GB、且不常追加 → **优先单文件/少文件**，别分区
- 数据需要按周期追加/删除、或总量到 TB 级 → **分区**
- 无论哪种，都要保证**单个文件不小于 ~64–128 MB**，否则元数据开销会吃掉收益

### 6.5 【重点】排序：性价比最高的优化

`parquet_metadata()` 那节已经说明：**谓词下推的效果取决于数据在文件内是否有序**。

**实测对比**（同一份 2160 万行数据，同一查询，**均无索引**）：

| 数据物理顺序 | 查询耗时 |
| --- | --- |
| 按 `ts` 有序（聚集） | **34.3 ms** |
| 按 `random()` 打散 | **65.3 ms** |
| 打散 + 建 ART 索引 | 62.9 ms（几乎无改善） |

**两个结论**：

1. **写入前排序，收益立竿见影**（这里约 2 倍；过滤条件越窄，差距越大，因为有序时能整块跳过）
2. **数据打散后，索引救不回来**（62.9 vs 65.3 ms）——再次印证 2.5 节的结论

**写法**：

```sql
COPY (SELECT * FROM bars ORDER BY symbol, ts)
  TO 'bars.parquet' (FORMAT PARQUET, COMPRESSION ZSTD);
```

**排序键怎么选**：按你的**典型查询模式**排序，把最常用于过滤的列放最前面。

| 典型查询 | 推荐排序键 |
| --- | --- |
| 按股票回测（`WHERE symbol=? AND ts BETWEEN ?`） | `ORDER BY symbol, ts` |
| 全市场时序聚合（`WHERE ts BETWEEN ?` GROUP BY symbol） | `ORDER BY ts, symbol` |

> ⚠️ 两者不可兼得。选型依据是「绝大多数查询长什么样」。**量化里最常见的是按股票做回测，所以 `ORDER BY symbol, ts` 通常更优。**

### 6.6 落地布局建议

综合以上实测，推荐如下（**与原文的建议有实质差别**）：

| 参数 | 建议值 | 依据 |
| --- | --- | --- |
| 分区粒度 | 分钟线**按月或按年**；数据量小则**不分区** | 保证单文件 ≥ 64–128 MB（6.2 / 6.3） |
| **绝对不要** | 按 symbol 分区 | 文件数爆炸（6.3） |
| 写入前排序 | `ORDER BY symbol, ts` | 决定下推效率（6.5） |
| 压缩 | **显式 `COMPRESSION ZSTD`** | 比默认省 46%（5.3） |
| RowGroup | 默认 122,880 行；大数据集可调到 50 万–100 万行 | 太大降低跳过精度，太小增加元数据开销 |
| 单文件 | ≥ 64 MB，理想 128 MB–1 GB | 低于此元数据开销占比过高 |

**关于原文「单文件 1–4 GB」的说法**：这个范围不算错，但**上限没有意义**——DuckDB 完全可以高效扫描几十 GB 的单文件，真正有害的是**过小**而不是过大。原文强调「禁止单超大文件」时给的三个理由（查询低效、更新灾难、容错差）里，只有**「更新灾难」站得住**（改一行要重写整个文件）；「查询低效」在实测中是相反的（6.2 节：单文件比分区更快）。

---

## 七、类型与精度：容易埋雷的地方

### 7.1 时间戳：三种类型在 Parquet 里都是 INT64

实测：

| DuckDB 类型 | Parquet 物理类型 | 逻辑类型 | `converted_type` |
| --- | --- | --- | --- |
| `TIMESTAMP`（无时区） | INT64 | `isAdjustedToUTC=0`, MICROS | `TIMESTAMP_MICROS` |
| `TIMESTAMPTZ`（有时区） | INT64 | `isAdjustedToUTC=1`, MICROS | `TIMESTAMP_MICROS` |
| `TIMESTAMP_NS`（纳秒） | INT64 | `isAdjustedToUTC=0`, NANOS | **`None`** |

**三个要点**：

1. **物理上都是 INT64**，区别只在逻辑类型注解。所以「换成 INT64 存时间更省」是没意义的——它本来就是。
2. **`isAdjustedToUTC` 是时区语义的关键**：0 表示「这是本地墙上时间」，1 表示「这是 UTC 绝对时刻」。混用会导致同一份数据在不同系统里解释成不同时间。
3. ⚠️ **纳秒精度没有 `converted_type`**。老版本 Parquet 读取器（只认 `converted_type` 而不认 `logical_type`）会把纳秒时间戳当成裸 INT64，读出天文数字。**跨系统交换数据时，用微秒（`TIMESTAMP`，DuckDB 默认精度）最安全。**

**量化实践建议**：

- A 股行情时间是**交易所本地时间**（Asia/Shanghai），语义上是「墙上时间」，用 **`TIMESTAMP`（无时区）** 最不容易出错
- 如果系统跨时区部署（如服务器在 UTC），用 `TIMESTAMPTZ` 并在写入时明确时区
- **绝对不要**同一个字段一会儿存 TIMESTAMP 一会儿存 TIMESTAMPTZ

### 7.2 价格用什么类型：float32 是真的不够

**实测精度问题**：

```python
0.1 + 0.2 存成 FLOAT          = 0.30000001192092896
0.1 + 0.2 存成 DOUBLE         = 0.3
0.1 + 0.2 存成 DECIMAL(18,6)  = 0.300000
```

更贴近量化的例子——**把 0.1 累加一万次**：

```
FLOAT  (float32): 1000.0000149011612     ← 已经偏了
DOUBLE (float64): 1000.0                 ← 精确
```

对价格和成交量来说，float32 的 7 位有效数字**不够用**：价格累加、权重计算、因子标准化都会把误差放大，而且这类误差是**静默的**——不会报错，只会让回测结果和实盘对不上。

**实测体积代价**（100 万行随机数，ZSTD，每个 RowGroup 压缩后）：

| 类型 | 单 RowGroup 大小 | 相对 |
| --- | --- | --- |
| `FLOAT`（float32） | ~426 KB | 1.0× |
| `DOUBLE`（float64） | ~902 KB | 2.1× |
| `DECIMAL(18,6)` | ~496 KB | 1.2× |

**建议**：

| 数据 | 推荐类型 | 理由 |
| --- | --- | --- |
| 价格、复权因子、因子值 | **`DOUBLE`** 或 `DECIMAL(18,6)` | 精度优先；DECIMAL 体积只比 float32 大 20%，但有精确小数语义 |
| 成交量、成交额 | `BIGINT` / `DOUBLE` | 整数用 BIGINT 无损 |
| 收益率 | `DOUBLE` | 小数值必需 |
| 股票代码 | `VARCHAR` | 有前导零，别用整数 |

> 注意：随机数是最坏情况（几乎不可压缩）。**真实量价数据有序性更强，DECIMAL 和 DOUBLE 的差距会缩小**，但精度差异不会。

---

## 八、数据更新：Parquet 不可原地改怎么破

### 8.1 【实测】`COPY ... APPEND` 的静默陷阱

原文说「更新数据需重写对应文件」，方向对，但漏掉了一个**会造成数据丢失的坑**。

**实测**（DuckDB 1.4.5）：先写 5 行，再用三种写法各追加 3 行：

| 写法 | 结果 |
| --- | --- |
| `(FORMAT PARQUET, APPEND)` | **3 行（原 5 行被覆盖！）** |
| `(FORMAT PARQUET, APPEND true)` | **3 行（同样被覆盖）** |
| `(FORMAT PARQUET)`（不写） | 3 行（覆盖，符合预期） |

**`APPEND` 被接受了、没有报错、然后什么都没做，原文件被销毁。**

**机制**（查证 DuckDB 文档）：`APPEND` 和 `OVERWRITE` **只在配合 `PARTITION_BY` 时才有意义**。不分区时，`COPY TO` 就是写单个文件、不检查是否已存在，`APPEND` 标志被忽略。

- `APPEND`（配 `PARTITION_BY`）：生成的文件名若已存在则重新生成路径，**避免覆盖已有文件**
- `OVERWRITE`（配 `PARTITION_BY`）：清空目标目录再写

> ⚠️ **这是本文最危险的一个发现**：一个被静默忽略的参数 + 一个不报错的覆盖行为 = 数据丢失。写追加逻辑时，**不要相信 `APPEND`**。

### 8.2 三种正确的更新模式

#### 模式一：整分区重写（适合修正历史数据）

```python
# 改 2025-03 的错误数据 → 只重写 2025-03 这一个分区
con.execute("""
    COPY (SELECT * FROM fixed_data ORDER BY symbol, ts)
    TO 'data/minute/year=2025/month=03' (FORMAT PARQUET, COMPRESSION ZSTD)
""")
```

代价是重写整个分区，但分区不大时完全可以接受，而且**保证一致性**。这是最简单可靠的方案。

#### 模式二：追加新文件 + 读取时去重（适合流式增量）

Parquet 不支持原地改，但**支持新增文件**。把所有版本都留着，读的时候取最新：

```python
# 每次增量写一个新文件，文件名带批次号
con.execute("""
    COPY (SELECT *, 'batch_002' AS _batch FROM new_data)
    TO 'data/minute/year=2025/month=03/batch_002.parquet' (FORMAT PARQUET)
""")
```

读取时用 `QUALIFY` 按主键取最新版本（实测有效）：

```sql
SELECT * FROM read_parquet('data/minute/year=2025/month=03/*.parquet')
QUALIFY row_number() OVER (
    PARTITION BY symbol, ts
    ORDER BY _batch DESC
) = 1;
```

**优点**：写入零冲突、天然保留历史版本。
**缺点**：数据会持续膨胀，需要定期做 compaction（把多个小文件合并重写）。

#### 模式三：上表格式（Iceberg / Delta Lake / DuckLake）

如果更新非常频繁、又不想自己维护 compaction，就该引入表格式层。它们在 Parquet 之上维护元数据与事务日志，提供 ACID、时间旅行、schema 演进。

| 方案 | 定位 |
| --- | --- |
| **Delta Lake / Iceberg** | 工业标准，生态广，适合大数据栈 |
| **DuckLake** | DuckDB 官方方向，SQL 型元数据目录，单机友好 |

**但注意**：对单机量化研究而言，引入表格式往往是**过度工程**。绝大多数情况下模式一（整分区重写）+ 模式二（追加去重）就够了。

### 8.3 schema 演进：加了一列怎么办

Parquet 的 schema 是**写在每个文件里**的，所以不同批次的文件列可能不一致。DuckDB 用 `union_by_name` 按列名合并：

```sql
SELECT * FROM read_parquet('data/**/*.parquet', union_by_name = true);
```

不加这个参数时，列不一致会直接报错。**这是加字段后最常踩的坑。**

---

## 九、性能调优清单

### 9.1 写出端

```sql
COPY (SELECT * FROM bars ORDER BY symbol, ts)      -- ① 排序：影响最大
TO 'data/bars.parquet' (
    FORMAT PARQUET,
    COMPRESSION ZSTD,                              -- ② 压缩：省 46%
    ROW_GROUP_SIZE 500000                          -- ③ 大数据集可调大
);
```

分区写出：

```sql
COPY (SELECT * FROM bars ORDER BY symbol, ts)
TO 'data/minute'
(FORMAT PARQUET, COMPRESSION ZSTD, PARTITION_BY (year, month));
```

### 9.2 读取端

```sql
-- 基础：glob 读取
SELECT * FROM read_parquet('data/minute/**/*.parquet');

-- 推荐：显式声明，行为可预测
SELECT * FROM read_parquet(
    'data/minute/**/*.parquet',
    hive_partitioning = true,     -- 把目录名解析成列
    union_by_name     = true,     -- schema 不一致时按列名合并
    filename          = true      -- 附带来源文件名（排查数据来源很有用）
);
```

**建立视图层**（重要工程实践）：把物理路径封装成视图，上层查询不关心文件怎么组织：

```sql
CREATE VIEW minute_bars AS
SELECT * FROM read_parquet('data/minute/**/*.parquet', hive_partitioning = true);

-- 之后所有查询都面向视图
SELECT symbol, avg(close) FROM minute_bars WHERE ts >= '2025-01-01' GROUP BY symbol;
```

好处：**数据重新分区、重新排序后，上层代码一行都不用改**。文件格式变了也只在视图定义里改一处。

### 9.3 验证优化是否真的生效

**不要凭感觉，用 `EXPLAIN ANALYZE`**：

```sql
EXPLAIN ANALYZE
SELECT count(*) FROM read_parquet('data/minute/**/*.parquet')
WHERE year = 2025 AND month = 3;
```

关注：

- 计划里是否有 `TABLE_SCAN` + `Filters`（下推成功）
- **实际扫描的行数**（`rows`）远小于总行数 → 跳过生效了
- `Total Time` 是否符合预期

**结合 `parquet_metadata()` 交叉验证**：如果某个过滤列的 min/max 在每个 RowGroup 里都一样（如 4.4 节的 `symbol`），那么无论怎么写，跳过都不会发生——这时该改的是**排序**，不是查询。

### 9.4 Arrow 接口：别用 `fetchall()`

DuckDB 与 Arrow 内存格式互通，**列式传递**，避免逐行构造 Python 对象：

```python
con.sql("SELECT * FROM minute_bars WHERE symbol='sh600000'").arrow()
#   → RecordBatchReader（惰性，实测返回类型）

con.sql("SELECT * FROM minute_bars WHERE symbol='sh600000'").fetch_arrow_table()
#   → pyarrow.Table（已物化的表）

con.sql("SELECT * FROM minute_bars").fetch_arrow_reader(batch_size=100000)
#   → 流式批次，适合超内存数据集

con.sql("SELECT * FROM minute_bars WHERE symbol='sh600000'").df()
#   → pandas DataFrame
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `.fetchall()` | Python list/tuple | **最慢**：逐行构造 Python 对象，大数据集上开销极高 |
| `.arrow()` | `RecordBatchReader` | 惰性，列式 |
| `.fetch_arrow_table()` | `pyarrow.Table` | 已物化 |
| `.fetch_arrow_reader(batch_size=N)` | 流式 reader | **分块处理，可处理远超内存的数据集** |
| `.df()` | pandas DataFrame | pandas 2.x 下可由 Arrow 直接支撑 |

**对量化的意义**：大回测里反复在数据库和 DataFrame 之间搬运数据。用 `.fetchall()` 会把结果逐行变成 Python 对象，几百万行就能吃掉几十秒和数倍内存；换成 Arrow 接口是列式传递，开销小得多。

> ⚠️ **依赖提示**：`.arrow()` / `.fetch_arrow_table()` / `.fetch_arrow_reader()` 需要安装 **`pyarrow`**；`.df()` 还需要 **pandas**（连带 **numpy**）。DuckDB 本身不自动装这些。

---

## 十、混合架构（最终方案）

```
┌─────────────────────────────────────────────────────┐
│  原始行情（只读、海量、多进程并发读）                  │
│  Parquet 分区文件                                     │
│  · 按月分区（不按 symbol！）                          │
│  · ORDER BY symbol, ts 排序                          │
│  · COMPRESSION ZSTD                                  │
│  · 单文件 ≥ 64MB                                      │
└─────────────────────────────────────────────────────┘
                        ↓ 视图层封装
┌─────────────────────────────────────────────────────┐
│  DuckDB 查询引擎                                      │
│  · CREATE VIEW 统一入口                               │
│  · PRAGMA 调好 threads / memory_limit / temp_directory│
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  维度与结果（需更新、需事务、需约束）                  │
│  DuckDB 原生 .duckdb 表                               │
│  · 股票基础信息、行业分类（主键/外键真强制）           │
│  · 因子结果表、回测汇总表（可 UPDATE/DELETE）          │
└─────────────────────────────────────────────────────┘
```

**核心原则**：

1. **原始数据用 Parquet**——只读、海量、多进程并发，这三条正好命中 Parquet 的强项和 DuckDB 的写锁短板
2. **需要改的数据用原生表**——理由是真的需要事务和原地更新，**不是因为「索引快」**
3. **用视图做解耦**——上层不感知物理布局
4. **排序和压缩是最高杠杆的两个动作**——成本几乎为零，收益立竿见影

---

## 十一、速查表

### 11.1 DuckDB VS MySQL

- 离线回测、因子计算、批量分析 → **DuckDB**
- 线上业务、用户系统、高频事务 → **MySQL**

### 11.2 Parquet VS DuckDB 原生表

- 只读、海量、归档、原始行情、多进程读 → **Parquet 分区文件**
- 需更新、需事务、需约束、维度表、计算结果 → **DuckDB 原生表**

### 11.3 一条命令记住的优化

```sql
COPY (SELECT * FROM t ORDER BY symbol, ts)   -- ① 排序（最重要）
TO 'out.parquet' (
    FORMAT PARQUET,
    COMPRESSION ZSTD,                        -- ② 压缩，省 46%
    ROW_GROUP_SIZE 500000
);
```

### 11.4 排查性能问题的顺序

```
1. parquet_metadata() 看统计信息 → 判断跳过是否可能
2. 数据在该列是否有序？ → 无序就 ORDER BY 重写
3. 文件数是否过多 / 单文件是否过小？ → 合并
4. EXPLAIN ANALYZE 看实际扫描行数
5. 最后才考虑调 RowGroup 大小、并行度
```

---

## 十二、避坑清单

按危险程度排序。

| # | 坑 | 后果 | 正确做法 |
| --- | --- | --- | --- |
| 1 | **相信 `COPY ... APPEND` 会追加** | **静默数据丢失** | 写新文件，或配合 `PARTITION_BY`；见 8.1 |
| 2 | **按 symbol 分区** | 文件数爆炸，慢 560 倍 | 按时间分区，靠排序解决按股票查询 |
| 3 | 数据写入前不排序 | 谓词下推失效，慢 2 倍以上 | 写入前 `ORDER BY` |
| 4 | 不指定 `COMPRESSION ZSTD` | 体积白白大一倍 | 显式写 ZSTD |
| 5 | 拿 float32 存价格 | 精度静默损失，回测失真 | 用 DOUBLE 或 DECIMAL(18,6) |
| 6 | 习惯性建索引 | 不加速，还占空间 | 靠排序和 zone map |
| 7 | 无脑分区 | 小规模下更慢更大 | 单文件 < 几十 GB 就别分区 |
| 8 | 多个 `UNION` 不同 schema 的文件不加 `union_by_name` | 直接报错 | 加参数 |
| 9 | 把多年分钟行情灌进 DuckDB 原生大表 | 文件臃肿、容错差 | 原始数据放 Parquet |
| 10 | 多脚本并行写同一个 `.duckdb` | 写锁冲突 | 原始数据用 Parquet（无锁并发读） |
| 11 | 用 DuckDB Server 模式做生产 | 并发缺陷 | 嵌入式单写进程 |
| 12 | 跨系统交换用纳秒时间戳 | 老读取器解析错误 | 用微秒精度 |
| 13 | 拿 MySQL 存量化分析数据 | 行存 + 无向量化，聚合极慢 | 用 DuckDB |

---

## 附：本文结论的复现方法

所有结论都可以用下面的方式自行验证：

```python
import duckdb
con = duckdb.connect()
con.execute("PRAGMA threads=4")

# 造合成分钟线
con.execute("""
CREATE TABLE bars AS
SELECT 'sh'||lpad(((i//240)%500)::VARCHAR,6,'0') AS symbol,
       TIMESTAMP '2025-01-01 09:30:00'
           + INTERVAL (i//(240*500)) DAY + INTERVAL (i%240) MINUTE AS ts,
       (10+((i//240)%500)*0.37)::DOUBLE AS close,
       (100000+(i%7919))::BIGINT AS volume
FROM range(0, 240*500*240) t(i)
""")

# 看 RowGroup 划分与统计信息
con.sql("""
SELECT row_group_id, row_group_num_rows, path_in_schema, stats_min, stats_max
FROM parquet_metadata('bars.parquet') LIMIT 8
""").show()

# 看查询是否真的下推
con.sql("EXPLAIN ANALYZE SELECT count(*) FROM 'bars.parquet' WHERE ts >= '2025-06-01'").show()
```

> **验证环境**：DuckDB 1.4.5 · macOS arm64 · 4 线程。数据库版本迭代很快，**关键结论请在你的版本上复验一遍**，尤其是第 2.4 节（外键强制）和第 8.1 节（APPEND 行为）这两处版本敏感的行为。

---

> **文档说明**：本文初稿整理自与豆包 AI 的对话，后经大幅扩充与修正：补全了 Parquet 四层结构、类型精度、更新模式、调优清单等章节；并基于 DuckDB 1.4.5 实测，修正了原稿中关于「索引」「分区收益」「COPY APPEND」三处不准确的结论。文中所有性能数字均为本机实测，非引用。
