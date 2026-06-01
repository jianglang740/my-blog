---
title: "用MySQL沉淀量化系统数据：K线、订单、成交与权益曲线建模"
description: "本文复盘量化框架数据库模块的设计，介绍如何用 MySQL 和 SQLAlchemy 存储行情、回测、订单、成交、权益曲线和实盘快照。"
date: 2026-06-02
lastmod: 2026-06-02
draft: true
weight: 4
categories:
    - Quant
    - Tech Stack
tags:
    - 量化投资
    - MySQL
    - SQLAlchemy
    - 数据库设计
---

# 用MySQL沉淀量化系统数据：K线、订单、成交与权益曲线建模

---

- 作者：山财小蒋
- 联系方式：2018036661@qq.com
- 项目地址：https://github.com/jianglang740/crypto_quant_framework.git
- 创作不易，转载请注明出处，欢迎批评探讨

---

- **这一篇文章讲数据库层。很多量化项目一开始都是 CSV + 脚本 + 图片，但如果想让项目可追踪、可复盘、可展示，就需要把行情、回测结果和实盘状态沉淀下来。**

- **我在这个项目中使用 MySQL 和 SQLAlchemy，并不是为了把架构做复杂，而是希望让每一次策略运行都有记录，后面可以查询、比较、展示和复盘。**

---

## 1. 为什么量化框架需要数据库

在最简单的回测脚本里，数据通常读自 CSV，结果直接打印在终端或保存成图片。但当项目逐渐变大后，我们会遇到几个问题：

1. 历史 K 线反复下载，效率低；
2. 每次回测结果无法结构化保存；
3. 订单、成交、权益曲线难以追踪；
4. 实盘运行状态没有长期记录；
5. 后续做仪表盘缺少统一数据源。

因此，这个项目引入了 MySQL 作为数据沉淀层，并使用 SQLAlchemy 做 ORM 封装。

数据库层的目标不是让所有模块都强依赖数据库，而是在需要保存和查询时，提供一套统一接口。

## 2. 数据库模块在框架中的位置

整体关系如下：

```mermaid
flowchart TD
    A[Binance K线] --> B[数据清洗]
    B --> C[MySQL Kline表]
    D[BacktestResult] --> E[订单/成交/权益曲线]
    E --> F[MySQL 回测结果表]
    G[LiveEngine / Recorder] --> H[账户/持仓/订单快照]
    H --> I[MySQL 实盘记录表]
    C --> J[Streamlit Dashboard]
    F --> J
    I --> J
```

数据库层连接了数据、回测、实盘和可视化，是框架从“临时脚本”走向“可追踪系统”的关键一步。

## 3. 为什么选择 MySQL + SQLAlchemy

项目中数据库方案是：

```text
MySQL + PyMySQL + SQLAlchemy
```

MySQL 适合保存结构化数据，部署和学习成本也比较低。SQLAlchemy 则负责把 Python 对象和数据库表映射起来，避免业务代码中到处手写 SQL。

这种设计有几个好处：

1. 表结构清晰；
2. 读写逻辑集中在 repository；
3. 后续迁移数据库时更容易；
4. dashboard 可以直接基于数据库查询；
5. 回测和实盘数据格式统一。

### 3.1 MySQL 在项目里具体保存什么

MySQL 在这个项目里不是只保存一个最终收益数字，而是保存整个量化系统运行过程中产生的结构化数据。

可以把它理解为三类表：

```text
行情类：klines
回测/交易类：strategy_runs、orders、trades、equity_curve
实盘快照类：account_snapshots、position_snapshots
```

这些表解决的问题不同：

1. `klines` 让历史行情可以重复使用，不需要每次都重新拉取；
2. `strategy_runs` 记录一次策略运行的基本信息；
3. `orders` 保存策略发出的订单请求和订单状态；
4. `trades` 保存真正成交的记录；
5. `equity_curve` 保存每个时间点账户权益；
6. `account_snapshots` 和 `position_snapshots` 用于记录实盘或 dry-run 过程中的账户和持仓状态。

这样保存之后，每一次实验都不是一次性结果，而是可以被追踪、查询、比较和复盘的运行记录。

### 3.2 SQLAlchemy 在项目里怎么用

SQLAlchemy 在项目中主要做 ORM 映射。也就是说，我不用在业务代码里手写：

```sql
INSERT INTO trades ...
SELECT * FROM equity_curve WHERE run_id = ...
```

而是先定义 Python 类，让类和数据库表对应起来。例如：

```text
StrategyRun -> strategy_runs 表
Kline -> klines 表
OrderRecord -> orders 表
TradeRecord -> trades 表
EquityCurve -> equity_curve 表
```

这样做的好处是，业务代码可以围绕 Python 对象组织逻辑，而不是到处拼 SQL 字符串。

不过 SQLAlchemy 并不是为了完全隐藏数据库。像唯一索引、字段精度、查询条件、事务提交这些数据库概念仍然要理解。项目中使用 ORM，是为了让数据库操作更集中、更可维护，而不是让开发者完全不用关心表结构。

### 3.3 PyMySQL 在连接链路中的位置

PyMySQL 是 SQLAlchemy 连接 MySQL 时使用的底层驱动。项目中的连接 URL 类似：

```text
mysql+pymysql://username:password@host:port/database?charset=utf8mb4
```

这里可以拆开理解：

| 部分 | 含义 |
| --- | --- |
| `mysql` | 数据库类型是 MySQL |
| `pymysql` | 使用 PyMySQL 作为 Python 驱动 |
| `username/password` | 数据库账号密码 |
| `host/port` | 数据库地址和端口 |
| `database` | 数据库名 |
| `charset=utf8mb4` | 使用 utf8mb4 字符集 |

项目中把这些信息封装在 `MySQLConfig` 里，再由 `session.py` 创建 SQLAlchemy engine 和 session。这样其他模块不需要关心连接字符串怎么拼，只要拿到 repository 使用即可。

## 4. 需要保存哪些数据

量化框架中值得保存的数据大致分成三类。

第一类是行情数据：

- exchange；
- symbol；
- timeframe；
- open_time；
- open/high/low/close；
- volume。

第二类是回测结果：

- strategy run；
- orders；
- trades；
- equity curve。

第三类是实盘状态：

- account snapshots；
- position snapshots；
- order status；
- trade records。

这些数据如果只存在内存里，程序结束就消失了。保存到数据库后，才能做横向比较、历史追踪和可视化复盘。

## 5. Repository 模式

项目中没有让业务代码直接操作 SQLAlchemy model，而是使用 `TradingRepository` 封装读写方法。

这种 repository 模式的好处是：

```text
业务代码
  -> repository 方法
  -> ORM model
  -> MySQL
```

业务代码只需要关心“保存一组 K 线”“保存一次回测结果”“查询某次运行的权益曲线”，不需要关心具体 SQL 怎么写。

这也方便后续维护。如果表结构调整，只需要集中修改 repository，而不是全项目搜索 SQL。

## 6. K 线数据的 upsert 思路

行情数据经常需要反复拉取。如果每次都直接插入，容易产生重复 K 线。因此 K 线表通常需要唯一约束，例如：

```text
exchange + symbol + timeframe + open_time
```

当同一根 K 线已经存在时，可以选择更新；不存在时才插入。这就是常见的 upsert 思路。

对量化项目来说，这个设计很实用：

1. 可以定期增量更新行情；
2. 可以避免重复数据污染回测；
3. 可以保证同一周期同一交易对的时间序列唯一。

## 7. 回测结果如何入库

一次回测不仅有最终收益，还包含大量过程数据：

- 本次运行的策略名称、参数、交易对、周期；
- 每一笔订单；
- 每一笔成交；
- 每个时间点的权益；
- 最终账户状态。

将这些数据保存下来后，就可以回答很多问题：

1. 哪个策略参数组合表现更稳？
2. 某次回测最大亏损发生在哪里？
3. 策略交易是否过于频繁？
4. 收益主要来自哪几笔交易？
5. 权益曲线和价格走势如何对应？

这些都是策略研究、问题排查和项目复盘中很有价值的内容。

## 8. 实盘快照为什么重要

实盘系统不能只记录成交，还应该记录账户和持仓快照。因为很多风险不是成交后才出现，而是在持仓过程中逐渐积累。

项目中的实盘数据库记录器会保存：

- 账户快照；
- 持仓快照；
- 订单状态；
- 成交记录。

这样即使程序中途停止，也能通过数据库回看策略运行过程。对于后续做监控、告警和复盘都很重要。

## 9. 数据库层的设计边界

当前数据库层采用主动调用方式，不强制嵌入每一次回测或实盘。这种设计比较灵活：

- 不需要数据库时，可以只跑本地回测；
- 需要沉淀结果时，再调用 repository；
- dashboard 只读数据库，不执行下单；
- 实盘记录器可以独立运行。

这种松耦合设计减少了模块之间的相互影响，更适合学习型项目逐步迭代。

## 10. 总结

数据库层的价值，不是简单地“把数据存起来”，而是让量化研究具备可追踪、可复盘、可展示的基础设施。

在这个项目中，MySQL 承担了三类核心数据的沉淀：

1. 行情数据；
2. 回测结果；
3. 实盘状态。

有了这些数据，后续才能自然接入 Bokeh 报告、Streamlit 仪表盘和更系统化的策略复盘。

## 11. 数据库源码阅读路线

数据库模块主要在：

```text
crypto_quant/database/models.py
crypto_quant/database/session.py
crypto_quant/database/repository.py
crypto_quant/database/recorder.py
```

推荐阅读顺序：

1. `models.py`：先看有哪些表；
2. `session.py`：看 MySQL engine 和 session 如何创建；
3. `repository.py`：看 K 线、运行记录、订单、成交、权益曲线如何写入和查询；
4. `recorder.py`：看实盘账户、持仓、订单状态如何记录。

## 12. 表之间的关系

项目中最重要的关联字段是 `run_id`。一次回测、一次 dry-run 或一次测试网运行，都可以有一个唯一的 run_id。其他表通过 run_id 归属于这次运行：

```mermaid
flowchart TD
    A[strategy_runs] --> B[orders]
    A --> C[trades]
    A --> D[equity_curve]
    A --> E[account_snapshots]
    A --> F[position_snapshots]
    G[klines] --> D
```

这样设计后，我们可以根据一次运行 ID 查询：

- 这次运行的策略参数；
- 产生了哪些订单；
- 成交了哪些交易；
- 权益曲线如何变化；
- 实盘过程中的账户和持仓快照。

## 13. 为什么金额字段使用 Numeric

数据库中的价格、数量、手续费、盈亏都使用 `Numeric(36, 18)` 这一类高精度字段，而不是普通 float。

原因和 Python 中使用 `Decimal` 类似：金融数据对小数精度敏感。如果用浮点数存储，可能出现精度误差，长期累计后会影响账户、手续费和收益统计。

所以项目在 Python 层使用 `Decimal`，在数据库层使用 `Numeric`，二者保持一致。

## 14. repository 的好处

如果没有 repository，业务代码可能到处写：

```python
session.add(...)
session.query(...)
session.commit()
```

这样会让数据库操作分散在各个脚本中。项目中把数据库读写封装到 `TradingRepository`，使得业务代码可以更语义化：

```text
保存 K 线
创建策略运行记录
保存订单
保存成交
保存权益曲线
查询某次运行结果
```

这种封装让数据库层更像框架的一部分，而不是临时 SQL 脚本。

## 15. MarketDataRepository：行情数据如何进入回测框架

数据库层不是只负责保存结果，它也负责把行情数据重新变成框架可用的数据源。

`MarketDataRepository` 的链路可以理解为：

```text
BarData 列表
  -> upsert_klines()
  -> MySQL klines 表
  -> get_klines()
  -> Kline ORM 对象
  -> get_data_feed()
  -> DataFeed
```

这里最关键的是 `get_data_feed()`。它不是简单查询数据库，而是把数据库里的 `Kline` 对象重新映射成 `BarData`，再包装成 `DataFeed`。这样回测引擎就不需要关心数据来自 CSV 还是 MySQL。

也就是说，数据库层和数据层之间通过 `BarData` / `DataFeed` 完成了解耦。

## 16. upsert K 线为什么比普通 insert 更适合行情数据

行情数据经常会遇到重复导入。例如第一次拉取 1000 根 K 线，第二次为了补数据又拉取了重叠区间。如果使用普通 insert，就可能因为唯一索引冲突报错。

项目中使用 MySQL 的 `on_duplicate_key_update`，逻辑是：

```text
如果这根 K 线不存在：插入
如果这根 K 线已经存在：更新 open/high/low/close/volume
```

这和前面 `klines` 表的唯一索引配合使用：

```text
exchange + symbol + timeframe + open_time
```

这四个字段共同确定一根 K 线。这样既能防止重复数据，又允许后续修正已有行情记录。

## 17. TradingRepository：运行结果如何被结构化保存

`TradingRepository` 负责交易和回测结果相关的数据读写。它围绕 `run_id` 组织所有记录。

一次完整回测保存到数据库时，通常会经历：

```text
create_run()
  -> 保存 strategy_runs 主记录
save_backtest_result()
  -> 保存 orders
  -> 保存 trades
  -> 保存 equity_curve
finish_run()
  -> 更新 final_equity 和 status
```

这样做之后，数据库中不是一堆散乱记录，而是可以通过 `run_id` 串起来的一次完整策略运行。

例如要复盘某次回测，可以查询：

1. `strategy_runs` 找到这次运行的基本信息；
2. `orders` 查看策略发出了哪些订单；
3. `trades` 查看实际成交；
4. `equity_curve` 查看账户权益变化；
5. `klines` 叠加行情走势。

这就是数据库沉淀的价值：让每一次实验都可以被追踪、比较和复盘。

## 18. run_id 是整套数据库设计的主线

在这个项目里，`run_id` 比数据库自增主键更重要。自增主键主要给数据库内部使用，而 `run_id` 是业务层理解一次运行的关键。

可以把它理解为：

```text
一次回测 / 一次 dry_run / 一次 live 记录周期 = 一个 run_id
```

其他表都通过 `run_id` 归属到这次运行：

```text
strategy_runs.run_id
orders.run_id
trades.run_id
equity_curve.run_id
account_snapshots.run_id
position_snapshots.run_id
```

有了这个主线，dashboard 才能在用户选择某一次运行后，把订单、成交、权益曲线、账户快照和持仓快照全部加载出来。

## 19. 数据库层的边界也要清楚

数据库层应该负责：

1. ORM 模型定义；
2. 数据插入、更新、查询；
3. Python 对象和数据库记录之间的转换；
4. 根据 `run_id` 组织运行结果；
5. 为 dashboard 提供稳定查询接口。

数据库层不应该负责：

1. 生成交易信号；
2. 判断是否开仓；
3. 计算撮合价格；
4. 执行 Binance 下单；
5. 在页面里直接组织 UI。

这种边界能避免项目越写越乱。策略、引擎、数据库、可视化各做各的事情，整个系统才容易维护。

## 20. 后续数据库模块可以继续扩展什么

如果继续完善数据库模块，可以考虑：

1. 增加参数实验表，专门保存不同参数组合；
2. 增加策略版本字段，避免代码变化后结果无法对比；
3. 增加数据导入批次表，记录行情数据来源和时间范围；
4. 增加异常日志表，保存实盘运行中的错误；
5. 给 dashboard 增加更多索引支持，提高查询速度；
6. 增加迁移工具，例如 Alembic，管理表结构变更。

这些内容会让项目从“能保存结果”进一步走向“能长期管理研究实验”。

## 免责声明

本文仅为个人项目学习复盘，不构成任何投资建议。数据库记录可以帮助复盘和管理风险，但不能消除市场风险。
