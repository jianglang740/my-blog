---
title: "突破趋势策略"
description: "价格突破关键阻力位后追多的量价突破策略"
date: 2026-09-20
lastmod: 2026-10-16
weight: 6
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 突破趋势策略

> **策略文件**：`BreakoutTrendStrategy.py`（366 行）
> **来源**：自己开发

## 策略思路

这是典型的「突破阻力位追多」策略，跑在 1 小时 K 线上、只做多。指标层先计算两种阻力位：基于滚动窗口高点的 `resistance`（过去 20 根最高价，已 `shift(1)` 避免用当根数据），以及唐奇安通道上轨；突破信号取「真实向上突破」（`close` 超过阻力位且有 `breakout_threshold`% 的幅度）与「唐奇安上轨突破」的并集。开仓要求突破信号成立，同时成交量相对均量放大到 `volume_factor` 倍、EMA 快线在慢线之上且价格在快线之上、RSI 在阈值之上但 <85、ADX>25 且 DI+>DI-、突破幅度 >1%、当根涨跌幅在 0.5%–8% 之间、MACD 金叉且柱状图为正、布林带宽度 >2%、价格位置 <0.95。平仓用「或」逻辑：向下突破支撑、EMA 转向、跌破快 EMA、RSI 超买、ADX 弱化、DI 转向、MACD 转弱、价格贴近布林带下轨。`custom_stoploss` 用支撑位与 ATR 取较严者做止损，`custom_exit` 处理突破失败、假突破、动能衰减和时间止损。

## 适用场景

适合**放量突破、趋势启动的行情**，尤其是价格有效站上前期高点/区间上沿、且有成交量配合的单边上涨；1 小时周期过滤掉了大部分日内噪声。**不适合假突破频发的震荡市**：在箱体里价格反复「刺破」上沿又回落，是最容易连续止损的场景，代码里的成交量确认（`volume_factor` 默认 2.5 倍，门槛较高）、`price_change_abs < 0.08` 和 `price_position < 0.95` 都是为了压制这类假信号，但无法消除。**也不适合无量阴跌**：突破必须放量，没有量能的行情不会触发。

## 核心代码

```python
# populate_indicators：阻力/支撑 + 真实突破 + 唐奇安通道
dataframe['highest'] = dataframe['high'].rolling(window=self.lookback_period.value).max()
dataframe['resistance'] = dataframe['highest'].shift(1)
dataframe['true_breakout_up'] = (
    (dataframe['close'] > dataframe['resistance']) &
    (dataframe['close'] > dataframe['resistance'] * (1 + self.breakout_threshold.value / 100))
)
dataframe['donchian_upper'] = dataframe['high'].rolling(window=20).max()
dataframe['donchian_breakout_up'] = dataframe['close'] > dataframe['donchian_upper'].shift(1)

# populate_entry_trend：突破 + 放量 + 趋势/动量过滤
dataframe.loc[
    (
        conditions[0] &   # 突破信号
        conditions[1] &   # 成交量
        # ... 中间省略 ...
        conditions[13] &  # 布林带宽度
        conditions[14]    # 价格位置上限
    ),
    'enter_long'
] = 1
# conditions[0] = dataframe['true_breakout_up'] | dataframe['donchian_breakout_up']

# populate_exit_trend：任一条件成立即平仓
dataframe.loc[
    (
        conditions[0] |   # 向下突破
        conditions[1] |   # 趋势转向
        # ... 中间省略 ...
        conditions[7]     # 布林带下轨
    ),
    'exit_long'
] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `lookback_period` | 20 | 阻力/支撑的最高最低点回看窗口 |
| `breakout_threshold` | 1.5 | 有效突破所需的最小幅度（%） |
| `volume_factor` | 2.5 | 突破时的成交量放大倍数 |
| `volume_lookback` | 15 | 成交量均线窗口 |
| `bb_period` / `bb_std` | 20 / 2.0 | 布林带参数 |
| `atr_period` / `atr_multiplier` | 14 / 3.0 | ATR 与动态止损倍数 |
| `ema_fast` / `ema_slow` | 20 / 60 | 趋势过滤均线 |
| `rsi_period` / `rsi_threshold` | 14 / 50 | RSI 周期与下限阈值 |
| `stoploss` | -0.06 | 固定止损 6% |
| `timeframe` | 1h | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **假突破是核心风险**：突破策略的收益完全建立在「突破后能延续」的假设上，一旦市场进入震荡，价格反复刺破阻力又回落，`custom_exit` 里的 `false_breakout` / `breakout_failure` 会频繁触发，产生大量小亏单；成交量 2.5 倍的门槛虽能过滤一部分，但在放量假突破时反而可能成为陷阱。
- **开仓条件多且部分高度相关**：突破、EMA 排列、MACD、ADX、布林带宽度等十多个条件全为 AND，信号会较稀疏；`breakout_strength > 0.01` 与 `breakout_threshold = 1.5%`、`close > ema_fast` 与 `ema_trend` 之间存在明显重叠。
- **`custom_stoploss` 的支撑位逻辑有边界问题**：盈利 >10% 时返回 `max(min(-support_distance * 0.8, -atr_stop_distance * 0.5), -0.02)`，若价格已远离支撑（`support_distance` 很大），止损会被 `-0.02` 兜底成固定 2%，实际行为与「跟随支撑」的意图不完全一致，需实测确认。
- **`stoploss = -0.06` 与止损函数并存**：`use_custom_stoploss = True` 时以动态止损为准，两套止损口径需要自行核对不会冲突。
- **只做多、无杠杆**：`can_short = False`，向下突破的 `true_breakdown` / `donchian_breakout_down` 只用于平仓、不会开空。
