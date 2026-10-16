---
title: "移动平均线交叉策略"
description: "双均线金叉开多、死叉平仓的经典趋势跟随策略"
date: 2026-09-11
lastmod: 2026-10-16
weight: 1
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 移动平均线交叉策略

> **策略文件**：`MovingAverageCrossStrategy.py`（275 行）
> **来源**：自己开发

## 策略思路

这是最经典的「双均线交叉 + 趋势过滤」趋势跟随策略，跑在 1 小时 K 线上、只做多。开仓主信号是快均线上穿慢均线（金叉，由 `qtpylib.crossed_above` 判定），同时要求收盘价站在长期趋势 EMA 之上、`DI+ > DI-`（趋势方向确认）、`RSI < 80`（避免在极度超买处追高）。均线本身用 `talib` 的 `MA` 泛型接口计算，快慢线周期和均线类型（SMA/EMA/WMA 等）都可调。平仓用三组条件取并集：死叉且 ADX 走弱、快线斜率转负且 MACD 转弱、跌破趋势 EMA 且 DI 转向。止盈靠 `minimal_roi` 阶梯，另有 `custom_stoploss` / `confirm_trade_entry` 做动态止损和开仓前的最后过滤。

## 适用场景

适合**有明确方向、单边推进的趋势市**，尤其是主流币种在 1 小时级别走出持续趋势的行情；均线策略在趋势中段能吃到主要涨幅。**不适合震荡市**：价格反复上下穿越均线时会频繁发出金叉/死叉，来回被止损，这正是均线类策略最典型的亏损来源。趋势强度过滤（ADX、DI）能缓解但不能根除这个问题。周期上代码按 1h 设计，换到更小周期噪声会明显放大。

## 核心代码

```python
# populate_indicators 关键部分
dataframe['ma_fast'] = ta.MA(dataframe, timeperiod=self.fast_ma_period.value, matype=self.ma_type_fast.value)
dataframe['ma_slow'] = ta.MA(dataframe, timeperiod=self.slow_ma_period.value, matype=self.ma_type_slow.value)
dataframe['trend_ema'] = ta.EMA(dataframe, timeperiod=self.trend_ema_period.value)
dataframe['adx'] = ta.ADX(dataframe, timeperiod=self.adx_period.value)
dataframe['di_plus'] = ta.PLUS_DI(dataframe, timeperiod=self.adx_period.value)
dataframe['di_minus'] = ta.MINUS_DI(dataframe, timeperiod=self.adx_period.value)
# ...
dataframe['ma_cross_up'] = qtpylib.crossed_above(dataframe['ma_fast'], dataframe['ma_slow'])
dataframe['ma_cross_down'] = qtpylib.crossed_below(dataframe['ma_fast'], dataframe['ma_slow'])

# populate_entry_trend：金叉 + 趋势确认 + DI 确认
dataframe.loc[
    (
        dataframe['ma_cross_up'] &                              # 金叉信号
        (dataframe['close'] > dataframe['trend_ema']) &         # 价格在长期趋势之上
        (dataframe['di_plus'] > dataframe['di_minus']) &        # DI确认上升趋势
        (dataframe['rsi'] < 80)                                  # RSI不能过度超买
    ),
    'enter_long'
] = 1

# populate_exit_trend：三组条件取并集
dataframe.loc[
    (
        (conditions[0] & conditions[1]) |   # 死叉 + ADX弱化
        (conditions[2] & conditions[4]) |   # 快线下降 + MACD转弱
        (conditions[6] & conditions[3])     # 跌破趋势线 + DI转向
    ),
    'exit_long'
] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `fast_ma_period` | 10 | 快均线周期 |
| `slow_ma_period` | 30 | 慢均线周期 |
| `ma_type_fast` / `ma_type_slow` | 1 | 均线类型，1 = EMA |
| `volume_factor` | 1.5 | 成交量确认系数（声明了但开仓逻辑中未实际使用） |
| `adx_period` | 14 | ADX 周期 |
| `adx_threshold` | 18 | ADX 强度阈值，用于 `confirm_trade_entry` |
| `trend_ema_period` | 75 | 长期趋势 EMA 周期 |
| `atr_period` / `atr_multiplier` | 14 / 2.0 | ATR 止损参数（当前未生效，见风险） |
| `stoploss` | -0.08 | 固定止损 8% |
| `timeframe` | 1h | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **`custom_stoploss` 是死代码**：`use_custom_stoploss = False`，但类里仍定义了基于 ATR 的动态止损函数，实盘不会调用；真正生效的是固定 `stoploss = -0.08` 与 `minimal_roi`。止损行为与代码看起来的不一致，容易误判。
- **参数过多，过拟合风险高**：十几个 `IntParameter`/`DecimalParameter` 叠加在一起，若用 hyperopt 全量搜索，很容易拟合出只在历史数据里好看、样本外失效的组合。
- **震荡市是主要亏损来源**：金叉/死叉在横盘里会频繁触发，均线交叉策略的老问题；ADX/DI 过滤能减少次数但代价是错过趋势启动的早期段。
- **只做多、无杠杆**：`can_short = False`，下跌趋势中策略只能空仓，收益完全依赖多头行情。
- **`volume_factor` 声明未用**：参数面板里可调但实际不参与任何判断，优化时会得到无效的搜索维度。
