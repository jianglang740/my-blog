---
title: "ADX 动量策略"
description: "面向加密市场优化的短周期 ADX 趋势突破策略"
date: 2026-09-18
lastmod: 2026-10-16
weight: 5
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# ADX 动量策略

> **策略文件**：`ADXMomentumStrategy.py`（295 行）
> **来源**：自己开发

## 策略思路

这是「ADX 趋势策略」的加密市场优化版，跑在更短的 15 分钟 K 线上、只做多。相对上一版，它用**更短的 ADX 周期（10）**来加快对趋势变化的响应，同时用**更高的 ADX 阈值（35）**来过滤假信号。开仓条件比趋势版精简得多：ADX 超过阈值、`DI+ > DI-` 且 `DI+` 本身高于 `di_threshold`（确保多头动能足够强，而不只是「略高于」）、RSI 落在 40–80 之间（避开极端超买超卖）、成交量相对均量显著放大。平仓用「或」逻辑：DI 明确反转（DI- > DI+ 且 DI- 够强）、ADX 跌破退出阈值、ADX 从强趋势快速下降、或 RSI 极度超买（>85）。`confirm_trade_entry` 里额外检查单根 K 线波动和价格动量是否过大，`custom_exit` 处理 ADX 崩盘、DI 快速反转、12 小时无收益的时间止损以及极端波动止损。做空分支写在 `if self.can_short` 里，当前 `can_short = False` 不会执行。

## 适用场景

适合**加密货币这类高波动、快节奏的强趋势/突破行情**，15 分钟周期让它能较快跟上趋势切换；放量突破时 `ADX 高 + DI 强 + 放量` 的组合正是它设计的目标场景。**不适合低波动、无量震荡的行情**：`ADX > 35` 的门槛在横盘期几乎不会满足，策略会长时间空仓；而 10 周期的短 ADX 对噪声更敏感，在假突破频发的时段可能刚入场 ADX 就回落触发平仓。**不该在流动性差、单根 K 线动辄 8%+ 的小币**上使用（`confirm_trade_entry` 会拒绝这类入场，但过快的波动本身就不适合）。

## 核心代码

```python
# populate_entry_trend：ADX强 + DI强 + RSI区间 + 放量
long_condition = (
    # ADX强趋势
    (dataframe['adx'] > self.adx_threshold.value) &
    # DI+上穿DI-且强度足够
    (dataframe['plus_di'] > dataframe['minus_di']) &
    (dataframe['plus_di'] > self.di_threshold.value) &
    # RSI在合理区间
    (dataframe['rsi'] > self.rsi_buy_lower.value) &
    (dataframe['rsi'] < self.rsi_buy_upper.value) &
    # 成交量显著放大
    (dataframe['volume_ratio'] > self.volume_multiplier.value)
)
dataframe.loc[long_condition, 'enter_long'] = 1

# populate_exit_trend：四项任一成立即平仓
exit_long_condition = (
    # 条件1: DI明确反转
    ((dataframe['minus_di'] > dataframe['plus_di']) &
     (dataframe['minus_di'] > self.di_threshold.value)) |
    # 条件2: ADX趋势减弱
    (dataframe['adx'] < self.adx_exit_threshold.value) |
    # 条件3: ADX从强趋势快速下降
    (dataframe['adx_strong'] & dataframe['adx_falling']) |
    # 条件4: RSI极度超买
    (dataframe['rsi'] > 85)
)
dataframe.loc[exit_long_condition, 'exit_long'] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `adx_period` | 10 | ADX/DI 周期（加密优化，较短） |
| `adx_threshold` | 35 | 开仓的 ADX 强度阈值（较短周期 + 较高门槛） |
| `di_threshold` | 30 | DI 强度阈值，确保趋势明确 |
| `rsi_buy_lower` / `rsi_buy_upper` | 40 / 80 | 开仓 RSI 区间 |
| `rsi_sell_lower` / `rsi_sell_upper` | 20 / 60 | 做空分支用的 RSI 区间（当前不生效） |
| `volume_ma_period` / `volume_multiplier` | 20 / 1.2 | 成交量均线窗口与放量倍数 |
| `adx_exit_threshold` | 25 | ADX 退出阈值 |
| `adx_strong_threshold` | 50 | 「强趋势」阈值（用于退出条件3） |
| `ema_fast` / `ema_slow` | 12 / 26 | EMA 参数（指标已算，但开仓逻辑未采用） |
| `stoploss` | -0.10 | 固定止损 10% |
| `timeframe` | 15m | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **源码存在一处乱码**：`populate_exit_trend` 做空分支的注释里有一个损坏字符（显示为 `# 条件1: DI明��反转`），应为「DI 明确反转」。不影响运行，但说明这段代码可能是复制/编辑过程中损坏的，维护性存疑。
- **stoploss 与 ROI 偏激进**：`stoploss = -0.10`（单笔最多亏 10%），`minimal_roi` 起步 25%，这套「大止损 + 高止盈」的组合在加密高波动下会让单笔回撤和收益都很大，资金曲线波动剧烈，需要自行验证是否可接受。
- **`confirm_trade_entry` 的硬编码阈值**：单根 K 线波动 > 8%、价格动量 > 15% 就拒绝入场，这两个数字写死在函数里、不在可调参数表中，换标的或换周期时需要手动改代码。
- **EMA 指标算了但没用**：`ema_fast`/`ema_slow`/`price_above_ema_fast` 在 `populate_indicators` 中计算，但开仓/平仓条件里完全没有引用，属于无效计算。
- **只做多、短期依赖放量**：`can_short = False`；开仓强依赖 `volume_ratio > 1.2`，若数据源的成交量质量不佳（部分交易所成交量失真），信号会明显受影响。
