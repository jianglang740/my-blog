---
title: "股票布林带 RSI 策略"
description: "A 股个股的布林带 RSI 高抛低吸（做 T）策略"
date: 2026-09-28
lastmod: 2026-10-16
weight: 11
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
    - A股
---

# 股票布林带 RSI 策略

> **策略文件**：`StockBollingerRSIStrategy.py`（330 行）
> **来源**：自己开发

## 策略思路

这是一套针对**A 股个股**（代码注释里以 sh600629 为分析对象）设计的日内「做 T」策略，主周期 5 分钟，辅以 15 分钟和 1 小时两个信息性周期做趋势确认。建仓分四种情形，只要满足其一即可：一是「主入场」——1 小时 MACD 多头且均线多头（大趋势向上）、15 分钟 RSI 回调到 40 以下或触及布林下轨（中期回调）、5 分钟 RSI 低于 35 且触及布林下轨或远离 MA5 且放量（短期精确买点），三者同时成立；二是「强势入场」——5 分钟 MACD 金叉、均线多头排列、放量 1.5 倍以上、当日上涨；三是「超卖反弹」——前一根 RSI 低于 30 且触及下轨、当前反弹回来；四是「突破 MA20」——放量上穿 20 日均线。卖出则是反向的高抛逻辑：RSI 超买且触及上轨、MACD 死叉、15 分钟止盈信号、或短期均线死叉配合放量下跌。此外还实现了 `adjust_trade_position` 做 T 加仓（仅在盈利、且 RSI 低位触及下轨时加仓，最多加两次）。

## 适用场景

**这是针对 A 股个股的策略，不是加密货币策略**，请务必注意这个差异——它依赖 A 股市场的交易规则和个股波动特征（T+1、涨跌停、以「600 开头」的沪市个股为标的）。代码注释明确写出适用条件：日内波动幅度大于 2%、有明确趋势、成交量活跃的个股。5 分钟周期配合 5%/3%/1.5% 的阶梯止盈，本质是赚日内的波动价差。**不该**用在：波动太小（做 T 空间不足）、长期单边下跌（低吸变接刀）、停牌或流动性枯竭的个股；也**不适合**直接套用到加密货币上。

## 核心代码

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 条件1: 60 分钟趋势向上
    trend_up_1h = (dataframe['macd_1h'] > dataframe['macdsignal_1h']) & \
                  (dataframe['ma5_1h'] > dataframe['ma20_1h'])
    # 条件2: 15 分钟确认回调
    pullback_15m = (dataframe['rsi_15m'] < 40) | (dataframe['close'] < dataframe['bb_lower_15m'])
    # 条件3: 5 分钟精确入场
    entry_signal_5m = (
        (dataframe['rsi'] < self.buy_rsi_threshold) &                  # RSI < 35
        ((dataframe['close'] <= dataframe['bb_lower']) |
         (dataframe['close'] < dataframe['ma5'] * 0.98)) &
        (dataframe['volume_ratio'] > self.volume_factor)               # 放量 1.3 倍
    )
    main_entry = trend_up_1h & pullback_15m & entry_signal_5m

    # 另有 strong_entry / oversold_bounce / ma20_breakout 三个信号
    dataframe.loc[main_entry | strong_entry | oversold_bounce | ma20_breakout, 'enter_long'] = 1
    return dataframe

def adjust_trade_position(self, trade, current_time, current_rate,
                          current_profit, min_stake, max_stake, **kwargs):
    dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
    last_candle = dataframe.iloc[-1]
    if current_profit > 0.01:                      # 只在盈利时做 T
        if (last_candle['rsi'] < 35 and last_candle['close'] < last_candle['bb_lower']
                and trade.nr_of_successful_entries < 3):
            return min_stake * 0.3                 # 加仓 30%
    return None
```

## 关键参数

无可调参数（`IntParameter` / `DecimalParameter`），阈值全部硬编码为类属性：

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy_rsi_threshold` | 35 | 买入 RSI 超卖阈值 |
| `sell_rsi_threshold` | 65 | 卖出 RSI 超买阈值 |
| `bb_period` / `bb_std` | 20 / 2 | 布林带周期与标准差 |
| `volume_factor` | 1.3 | 放量倍数 |
| `ma_short` / `ma_medium` / `ma_long` | 5 / 10 / 20 | 短中长均线周期 |
| `max_entry_position_adjustment` | 2 | 允许加仓次数（做 T 用） |
| `timeframe` | 5m | 主时间框架（硬编码） |
| `stoploss` | -0.035 | 止损（硬编码） |
| `trailing_stop_positive` / `trailing_stop_positive_offset` | 0.02 / 0.03 | 追踪止损参数（硬编码） |

## 风险与局限

- **未回测 / 需要自行验证**：源码没有给出任何回测收益、胜率或回撤数据；注释里提到「基于 ATR 平均值 0.479 设置 -3.5% 止损」，但这是对单一标的的观察，不是普适结论。
- **针对单一标的设计、泛化性存疑**：所有阈值都围绕 sh600629 一只股票观察得出，直接搬到其他个股或板块很可能失效。
- **没有可调参数、无法 hyperopt**：全部硬编码，既降低了过拟合风险，也意味着无法通过参数寻优验证稳健性，改动只能手工改代码。
- **跨周期合并依赖 `@informative`**：15 分钟 / 1 小时的 `rsi_15m`、`macd_1h` 等列由装饰器生成，若数据结构或合并出错，入场条件会静默失效（需要回测确认信号确实产生）。
- **A 股规则与成本**：T+1 限制下「日内做 T」实际靠的是先有底仓再高抛低吸，且未考虑印花税、佣金、涨跌停导致的无法成交——这些都会显著侵蚀收益。
