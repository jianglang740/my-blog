---
title: "ADX 趋势策略"
description: "以 ADX/DI 为核心的强趋势跟随策略"
date: 2026-09-16
lastmod: 2026-10-16
weight: 4
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# ADX 趋势策略

> **策略文件**：`ADXTrendStrategy.py`（423 行）
> **来源**：自己开发

## 策略思路

这个策略把趋势强度的判断集中到 ADX/DI 这一族指标上，跑在 1 小时 K 线上、只做多。开仓要求 `DI+ > DI-` 且差值超过阈值（`bullish_trend`）、ADX 高于强趋势阈值、ADX 正在上升（斜率 > `adx_min_slope`），再叠加 EMA 排列向上、价格在快 EMA 之上、RSI 在阈值区间、MACD 金叉且柱状图增强、SAR 多头、成交量放大、1% 以上的价格动量、Aroon 振荡器 > 20、布林带位置不贴边等过滤，最后要求一个自制的 `trend_score`（把上述趋势条件加权计分，满分约 11）达到 6 分以上。平仓用「或」逻辑：熊市趋势、ADX 弱化/下降、EMA 转向、跌破快 EMA、RSI 超买或转弱、MACD 死叉、SAR 转向、Aroon 转负、趋势评分跌破 3、DI 差值收窄到 2 以内。另有基于 ADX 强度的 `custom_stoploss` 和 `custom_exit`。

## 适用场景

适合**单边强趋势市场**，尤其是 ADX 持续走高、方向明确的行情；因为入场把「ADX 高 + 正在上升」作为硬条件，能较好避开趋势启动前的无序阶段，追求趋势中段的确定性。**不适合震荡市**：横盘时 ADX 通常低于阈值，策略会空仓甚至不产生信号，这是它相对稳健的一面；但反过来，一旦 ADX 冲高后迅速回落（如消息面急涨急跌），容易在假突破处入场后立刻被平仓条件扫出。周期按 1h 设计。

## 核心代码

```python
# populate_indicators：ADX 组 + 方向 + 斜率 + 趋势评分
dataframe['adx'] = ta.ADX(dataframe, timeperiod=self.adx_period.value)
dataframe['di_plus'] = ta.PLUS_DI(dataframe, timeperiod=self.adx_period.value)
dataframe['di_minus'] = ta.MINUS_DI(dataframe, timeperiod=self.adx_period.value)
dataframe['di_diff'] = dataframe['di_plus'] - dataframe['di_minus']
dataframe['adx_slope'] = (dataframe['adx'] - dataframe['adx'].shift(self.adx_slope_periods.value)) / self.adx_slope_periods.value
dataframe['adx_rising'] = dataframe['adx_slope'] > self.adx_min_slope.value
dataframe['bullish_trend'] = (dataframe['di_plus'] > dataframe['di_minus']) & (dataframe['di_diff'] > self.di_diff_threshold.value)
dataframe['bearish_trend'] = (dataframe['di_minus'] > dataframe['di_plus']) & (dataframe['di_diff'] < -self.di_diff_threshold.value)

trend_score = 0
trend_score += np.where(dataframe['bullish_trend'], 2, 0)                          # ADX方向
trend_score += np.where(dataframe['adx_strong'] | dataframe['adx_very_strong'], 2, 0)  # ADX强度
trend_score += np.where(dataframe['adx_rising'], 1, 0)
# ... 其余条件各计 1 分 ...
dataframe['trend_score'] = trend_score

# populate_exit_trend：任一条件成立即平仓
dataframe.loc[
    (
        conditions[0] |   # 熊市趋势
        conditions[1] |   # ADX弱化
        conditions[2] |   # ADX下降
        # ... 中间省略 ...
        conditions[12]    # DI差值收窄
    ),
    'exit_long'
] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `adx_period` | 14 | ADX/DI 周期 |
| `adx_threshold_strong` | 30 | 强趋势 ADX 阈值 |
| `adx_threshold_weak` | 20 | 趋势弱化阈值（平仓用） |
| `di_diff_threshold` | 5.0 | DI+ 与 DI- 的最小差值 |
| `adx_slope_periods` / `adx_min_slope` | 5 / 1.5 | ADX 斜率窗口与上升阈值 |
| `ema_fast` / `ema_slow` | 12 / 30 | 趋势确认均线 |
| `rsi_period` / `rsi_buy_threshold` / `rsi_sell_threshold` | 14 / 50 / 75 | RSI 参数 |
| `volume_factor` | 1.6 | 成交量放大倍数 |
| `atr_period` / `atr_multiplier` | 14 / 2.5 | ATR 与动态止损倍数 |
| `sar_acceleration` / `sar_maximum` | 0.02 / 0.2 | 抛物线 SAR 参数 |
| `stoploss` | -0.06 | 固定止损 6% |
| `timeframe` | 1h | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **`trend_score` 权重仍旧是手工设定**：各条件按 2/1 分累加、阈值取 6 分，满分约 11，这些分值没有回测依据，阈值 6 也只是主观选择，属于需要重点验证的过拟合来源。
- **开仓条件同样偏多且全为 AND**：在 ADX 强 + 上升的基础上再叠加 EMA、MACD、SAR、Aroon、布林带位置等十几条，信号会非常稀疏；其中 `bb_percent` 限在 0.3–0.9、`price_momentum > 0.01` 这类条件与「ADX 上升」在逻辑上有重叠，进一步压低了开仓频率。
- **平仓过于灵敏**：13 条退出条件取并集，`abs(di_diff) < 2`（DI 差值收窄）这种轻微变化就会触发离场，趋势尚在时可能被过早止盈。
- **硬编码常量参差**：`adx_very_strong` 用固定的 50、`adx_moderate` 用固定的 25，而强/弱阈值是可调参数，二者混用会让「趋势强弱」的分档口径不一致。
- **只做多、无杠杆**：`can_short = False`。
