---
title: "Hyperliquid 动量策略"
description: "多重指标共振 + 置信度打分的动量策略"
date: 2026-09-27
lastmod: 2026-10-16
weight: 10
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# Hyperliquid 动量策略

> **策略文件**：`HyperliquidMomentumStrategy.py`（572 行）
> **来源**：自己开发

## 策略思路

在 15 分钟周期上做多空双向的动量突破。开仓要求一系列条件同时成立：三条 EMA（20 / 50 / 150）呈多头排列或空头排列、MACD 出现金叉（或死叉）且柱状图方向一致、RSI 处于 40 到 65 的中性区间且方向正确、成交量放大到强放量水平（2.5 倍）、价格突破 20 周期阻力位且突破强度足够、并且当前不是高波动环境。整个策略的核心是一个自研的**置信度打分**（0 到 1）：把趋势一致性、MACD 动量、RSI 位置、成交量确认、突破强度五项加权求和再归一化，只有分数超过 0.65 才允许开仓。这个分数还被用来动态调仓和调杠杆——分数越高，仓位越大（最高 2 倍基础仓、上限 10%），杠杆最高 3 倍。平仓条件包括趋势反转、动量衰竭、RSI 过热 / 超卖、破位、以及置信度跌破 0.35。

## 适用场景

适合**波动活跃、趋势连续的加密货币**（策略名里的 Hyperliquid 是目标交易所，但代码本身是通用的，没有交易所专有逻辑），15 分钟周期配合 150 周期 EMA 做长趋势锚。多空双向让它既能吃上涨也能吃下跌。**不该**用在：低波动、窄幅震荡的标的（置信度很难过线，或突破反复失败）、流动性差的市场（高杠杆 + 市价单滑点大），以及缺乏足够历史数据的新币（150 周期 EMA 需要较长的预热）。

## 核心代码

```python
def _calculate_confidence_score(self, dataframe: DataFrame) -> DataFrame:
    # 趋势一致性 (0-0.3) / 动量 (0-0.25) / RSI (0-0.2) / 成交量 (0-0.15) / 突破 (0-0.1)
    trend_score = np.where(dataframe['ema_bullish_alignment'], 0.3,
                  np.where(dataframe['ema_bearish_alignment'], -0.3, 0))
    momentum_score = np.where(dataframe['macd_bullish'] & dataframe['macd_hist_positive'], 0.25,
                     np.where(dataframe['macd_bearish'] & ~dataframe['macd_hist_positive'], -0.25, 0))
    # ... rsi_score / volume_score / breakout_score 省略
    total_score = trend_score + momentum_score + rsi_score + volume_score + breakout_score
    return np.clip((total_score + 1) / 2, 0, 1)

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    long_conditions = (
        dataframe['price_above_ema_trend'] &
        dataframe['ema_bullish_alignment'] &
        dataframe['macd_cross_above'] &
        (dataframe['rsi'] > self.rsi_neutral_low.value) &
        (dataframe['rsi'] < self.rsi_neutral_high.value) &
        dataframe['volume_high'] &
        (dataframe['breakout_strength'] > self.breakout_threshold.value) &
        ~dataframe['volatility_high'] &
        (dataframe['confidence_score'] > 0.65)      # 置信度门槛
    )
    dataframe.loc[long_conditions, 'enter_long'] = 1
    # ... short_conditions 逻辑对称，设置 enter_short
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `ema_fast` / `ema_slow` / `ema_trend` | 20 / 50 / 150 | 三条 EMA 周期 |
| `macd_fast` / `macd_slow` / `macd_signal` | 12 / 26 / 9 | MACD 参数 |
| `rsi_period` | 14 | RSI 周期 |
| `rsi_neutral_low` / `rsi_neutral_high` | 40 / 65 | 中性区间（开仓允许范围） |
| `rsi_oversold` / `rsi_overbought` | 25 / 75 | 出场用的超卖 / 超买阈值 |
| `atr_period` | 14 | ATR 周期 |
| `atr_multiplier_stoploss` | 3.0 | 止损 ATR 倍数 |
| `volume_ma_period` | 20 | 成交量均线周期 |
| `volume_multiplier_min` / `volume_multiplier_strong` | 1.5 / 2.5 | 放量的普通 / 强阈值 |
| `breakout_threshold` | 0.8 | 突破强度门槛 |
| `position_size_base` | 0.02 | 基础仓位 2% |
| `position_size_multiplier` | 2.0 | 高置信度仓位倍数 |
| `max_position_size` | 0.10 | 单仓上限 10% |

硬编码的基础配置：`timeframe = '15m'`、`stoploss = -0.10`、`can_short = True`、`startup_candle_count = 200`，订单全部用市价单（`order_types` 里 entry/exit/stoploss 均为 market，且未开启交易所侧止损）。

## 风险与局限

- **未回测 / 需要自行验证**：源码没有给出任何回测业绩数字，置信度阈值的有效性也未经验证。
- **置信度打分是「人工权重」**：0.3 / 0.25 / 0.2 / 0.15 / 0.1 这几个权重来自经验设定，缺乏统计依据，0.65 的开仓门槛同样需要 hyperopt 或回测来确认，否则容易过拟合到某段行情。
- **入场条件过窄**：要求 MACD 恰好在当根 K 线发生金叉（单根事件），再叠加置信度 0.65 以上，两者同时满足的样本可能很少，信号频率低。
- **双向 + 高杠杆放大风险**：最高 3 倍杠杆、单仓上限 10%、市价单成交，滑点和短时反向波动都被放大；做空在加密货币的逼空行情里尤其危险。
- **命名与实现不符**：策略名带「Hyperliquid」，但代码中没有任何交易所专有逻辑，本质仍是通用现货/合约策略，取名可能会造成误解。
