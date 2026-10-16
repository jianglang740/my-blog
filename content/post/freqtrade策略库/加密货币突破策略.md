---
title: "加密货币突破策略"
description: "30 分钟突破阻力 + 成交量确认的趋势跟随"
date: 2026-09-21
lastmod: 2026-10-16
weight: 7
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 加密货币突破策略

> **策略文件**：`CryptoBreakoutStrategy.py`（429 行）
> **来源**：自己开发

## 策略思路

在 30 分钟周期上，用滚动窗口（默认 30 根 K 线）计算动态的支撑与阻力位。当收盘价向上突破前一根 K 线记录的阻力位时触发做多信号，同时要求：成交量放大到均量的 2 倍以上、RSI 落在 40 到 70 之间（既不超买也不在下跌途中抄底）、突破幅度超过 1%、且前一根 K 线本身也站在阻力之上（连续确认）来过滤假突破。趋势方向用 EMA（30 周期）过滤，只做价格在 EMA 上方的多头。平仓条件比较宽松，跌破支撑、跌破 EMA、RSI 超过 85、MACD 死叉、成交量萎缩等任一条成立即离场。另有 ATR 与支撑位结合的动态止损，以及「突破失败 / 假突破 / 动能衰减 / 结构破坏 / 时间止损」五类自定义离场。代码里保留了做空分支，但 `can_short = False`，当前只做多。

## 适用场景

适合**趋势市**中的加密货币，尤其是放量突破关键位后能延续的行情，30 分钟周期在响应速度和噪音之间取平衡。适合流动性好、成交量数据可信的主流币对。**不该**用在：横盘震荡（假突破频繁、频繁被止损）、单边下跌（只做多等于逆势）、以及流动性差、成交量容易被操纵的小币种。注意代码注释把它叫「30 日高低点」，但实际窗口是 30 根 30 分钟 K 线（约 15 小时），并不是 30 天，参数含义与注释不符。

## 核心代码

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    long_breakout = (
        (dataframe['close'] > dataframe['resistance'].shift(1)) &          # 突破阻力
        (dataframe['volume'] > dataframe['volume_ma'] * self.volume_multiplier.value) &
        (dataframe['rsi'] < self.rsi_buy_upper.value) &                    # RSI < 70
        (dataframe['rsi'] > self.rsi_buy_lower.value) &                    # RSI > 40
        (dataframe['breakout_distance'] > self.breakout_threshold.value / 100) &
        (dataframe['close'].shift(1) > dataframe['resistance'].shift(2)) & # 连续确认
        (dataframe['price_above_ema']) &                                   # 趋势过滤
        ~(dataframe['false_breakout_up']) &                                # 排除假突破
        (dataframe['breakout_strength'] > 0.5)
    )
    dataframe.loc[long_breakout, 'enter_long'] = 1
    # ... 做空分支需 can_short=True 才会生效
    return dataframe

def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    exit_long_condition = (
        (dataframe['close'] < dataframe['support'].shift(1)) |
        (dataframe['close'] < dataframe['ema_trend']) |
        (dataframe['rsi'] > 85) |
        (dataframe['macd'] < dataframe['macd_signal']) |
        (dataframe['false_breakout_up'] & (dataframe['close'] < dataframe['resistance'] * 0.98)) |
        ~(dataframe['uptrend_structure']) |          # 结构破坏
        (dataframe['volume_ratio'] < 0.5)            # 成交量萎缩
    )
    dataframe.loc[exit_long_condition, 'exit_long'] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `support_resistance_period` | 30 | 支撑阻力回看窗口（根 K 线） |
| `breakout_threshold` | 1.0 | 突破幅度要求（百分比，代码里再除以 100） |
| `volume_multiplier` | 2.0 | 做多所需的成交量放量倍数 |
| `volume_short_multiplier` | 1.8 | 做空所需的放量倍数（当前未启用） |
| `rsi_period` | 14 | RSI 周期 |
| `rsi_buy_lower` / `rsi_buy_upper` | 40 / 70 | 做多时 RSI 的允许区间 |
| `rsi_sell_lower` / `rsi_sell_upper` | 30 / 70 | 做空时 RSI 的允许区间 |
| `volume_ma_period` | 30 | 成交量均线周期 |
| `atr_period` | 14 | ATR 周期（用于动态止损） |
| `atr_multiplier` | 2.5 | ATR 止损倍数 |
| `ema_trend_period` / `ema_fast_period` | 30 / 15 | 趋势与快速 EMA 周期 |

另有一组硬编码的基础配置：`timeframe = '30m'`、`stoploss = -0.08`、`startup_candle_count = 150`，以及 `minimal_roi` 阶梯（30 分钟内 30% 止盈，逐级放宽到 8 小时后 3%）。

## 风险与局限

- **未回测 / 需要自行验证**：源码里没有任何回测收益、胜率或回撤数据，这里也不编造。
- **平仓条件过于宽松**：`~(dataframe['uptrend_structure'])` 要求「更高的高点且更高的低点」同时成立，这个结构在实盘中很难持续满足，很可能导致刚开仓就被结构破坏条件平掉，需要回测确认信号频率。
- **参数较多、存在过拟合风险**：可调参数十几个（突破幅度、成交量倍数、RSI 上下限、EMA/ATR 周期），在有限历史数据上容易调出「好看但不可复现」的结果。
- **只做多**：代码里的做空逻辑形同虚设，在单边下跌行情中只能被动离场，无法获利。
- **注释与实现不一致**：「30 日高低点」实际是 30 根 K 线，容易误导参数调优。
