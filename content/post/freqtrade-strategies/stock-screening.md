---
title: "股票筛选策略"
description: "为做 T 筛选高波动 A 股并按评分择时入场"
date: 2026-09-30
lastmod: 2026-10-16
weight: 12
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
    - A股
---

# 股票筛选策略

> **策略文件**：`StockScreeningStrategy.py`（349 行）
> **来源**：自己开发

## 策略思路

这是上一套「股票布林带 RSI 策略」的**筛选前置版**——先给标的打分，只交易「适合做 T」的高波动个股。策略在 5 分钟主周期上维护一个 0 到 11 分的**筛选评分**，评分项包括：平均日内波动率是否落在合适区间（越高越加分）、过去 20 根 K 线里高波动出现的次数、放量上涨是否多于放量下跌、60 分钟 MACD 与均线是否多头、ATR 是否在合理区间、布林带是否够宽。只有评分达到 7 分以上的个股才允许入场，入场时机则从「低吸、突破、趋势跟随」三种里任选其一（低吸 = RSI 超卖 + 触及下轨 + 放量；突破 = MACD 金叉 + 站上 MA20 + 强放量；趋势跟随 = 均线多头 + RSI 健康 + MACD 多头 + 适度放量）。出场条件是高抛、趋势转弱，或标的评分跌破 5 分（不再符合筛选标准就清仓）。仓位也随评分走，评分越高仓位越大。

## 适用场景

**这是针对 A 股个股的策略，不是加密货币策略**。它建立在 A 股市场规则和 sh600629 这类「高波动、成交活跃、适合日内做 T」的个股特征之上（代码注释明确以此为模板）。适合：日内波动 2% 到 6%、有明显趋势、放量活跃的个股，用 5 分钟周期捕捉日内价差。**不该**用在：低波动蓝筹（达不到评分门槛）、单边下跌股、以及流动性不足的标的。

需要特别说明一点：**这个「筛选」是逐标的打分过滤，并不是横截面的全市场选股**。Freqtrade 按交易对逐一下发数据，策略只能给白名单里的每个标的独立算分，无法在全部 A 股里排序挑出最高的几只；真正的选股工作仍需在策略之外完成（例如先跑一遍脚本筛出候选池再放进白名单）。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # ... RSI / 布林带 / MACD / 均线 / ATR 等基础指标省略
    dataframe['screening_score'] = 0
    # 评分1: 波动率合适 (0-3 分)
    dataframe.loc[(dataframe['avg_volatility_20'] >= self.min_volatility) &
                  (dataframe['avg_volatility_20'] <= self.max_volatility), 'screening_score'] += 1
    dataframe.loc[(dataframe['avg_volatility_20'] >= self.ideal_volatility) &
                  (dataframe['avg_volatility_20'] <= 4), 'screening_score'] += 2
    # 评分2: 频繁高波动 (0-2 分)
    dataframe.loc[dataframe['high_volatility_count'] > 10, 'screening_score'] += 2
    # 评分3: 放量上涨多于下跌 (0-2 分)
    dataframe.loc[dataframe['volume_up_count'] > dataframe['volume_down_count'], 'screening_score'] += 2
    # 评分4: 60 分钟趋势向上 (0-2 分)
    dataframe.loc[(dataframe['macd_1h'] > dataframe['macdsignal_1h']) &
                  (dataframe['ma5_1h'] > dataframe['ma20_1h']), 'screening_score'] += 2
    # 评分5/6: ATR 区间、布林带宽度 各 1 分
    return dataframe

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    stock_qualified = (dataframe['screening_score'] >= 7)      # 只做合格股
    buy_dip = ((dataframe['rsi'] < 35) & (dataframe['close'] <= dataframe['bb_lower']) &
               (dataframe['volume_ratio'] > self.min_volume_ratio))
    breakout = ((dataframe['macd_cross_up']) & (dataframe['close'] > dataframe['ma20']) &
                (dataframe['volume_ratio'] > 1.5) & (dataframe['price_change'] > 0.5))
    trend_follow = ((dataframe['ma5'] > dataframe['ma10']) & (dataframe['ma10'] > dataframe['ma20']) &
                    (dataframe['rsi'] > 50) & (dataframe['rsi'] < 65) &
                    (dataframe['macd'] > dataframe['macdsignal']) & (dataframe['volume_ratio'] > 1.2))
    dataframe.loc[stock_qualified & (buy_dip | breakout | trend_follow), 'enter_long'] = 1
    return dataframe
```

## 关键参数

无可调参数（`IntParameter` / `DecimalParameter`），阈值全部硬编码为类属性：

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `min_volatility` | 1.5 | 最小日内波动幅度（%） |
| `ideal_volatility` | 2.0 | 理想波动幅度（%），额外加分 |
| `max_volatility` | 6.0 | 最大波动幅度（%），避免过度投机 |
| `min_volume_ratio` | 1.3 | 最小放量比例 |
| `rsi_low` / `rsi_high` | 25 / 75 | 可接受的 RSI 区间 |
| `atr_min` / `atr_max` | 0.1 / 1.0 | 可接受的 ATR 区间 |
| `timeframe` | 5m | 主时间框架（硬编码） |
| `stoploss` | -0.03 | 止损（硬编码） |
| `trailing_stop_positive` / `trailing_stop_positive_offset` | 0.015 / 0.025 | 追踪止损参数（硬编码） |

评分口径：评分 1 最多 3 分、评分 2 到 4 各 2 分、评分 5 和 6 各 1 分，理论满分 11 分。入场门槛 7 分，出场触发线 5 分。

## 风险与局限

- **未回测 / 需要自行验证**：源码没有提供任何回测收益、胜率或回撤数据，评分权重与 7 分门槛也没有经过统计验证。
- **不是真正的选股**：评分为逐标的独立计算，无法跨全市场排序，需要配合外部脚本构造候选白名单，「筛选」能力被 Freqtrade 的逐对数据模型限制。
- **评分权重靠经验拍定**：「波动率占 3 分、趋势占 2 分」这类权重缺乏依据，某只股票可能长期卡在 6 到 7 分之间反复触发 / 取消信号，造成频繁开平仓。
- **没有可调参数**：全部硬编码，无法 hyperopt，稳健性只能靠手工改代码去验证。
- **A 股制度与成本未建模**：T+1、涨跌停、印花税与佣金都会影响实际可执行性和收益，做 T 策略对交易成本尤其敏感。
