---
title: "多时间框架趋势策略"
description: "日线 / 4 小时 / 1 小时三重趋势共振的顺势策略"
date: 2026-09-25
lastmod: 2026-10-16
weight: 9
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 多时间框架趋势策略

> **策略文件**：`MultiTimeframeTrendStrategy.py`（467 行）
> **来源**：自己开发

## 策略思路

主周期是 1 小时，但开仓要求三个时间框架方向一致：日线判主趋势（EMA 多头、MACD 多头、RSI 大于 50、ADX 够强、+DI 大于 -DI），4 小时判中期趋势（同样是 EMA/MACD/RSI/ADX/DI 五件事），1 小时找入场点（EMA 多头、价格站上快线、RSI 在阈值与 80 之间、MACD 金叉且柱状图走强、ADX 够强、成交量放大到 1.8 倍以上）。三套时间框架的指标各算一遍，再把 4 小时和日线的结果前向填充合并进 1 小时序列，只有全部条件同时满足才开多。平仓则相反——**任一时间框架**出现趋势转向（日线或 4 小时的 EMA/MACD/DI 转空、1 小时跌破快线、RSI 转弱、ADX 弱化）就会离场，属于「严格入场、宽松出场」。此外还有 ATR 与跨周期 EMA 结合的分层动态止损，以及日线 / 4 小时趋势反转、多周期 RSI 同时转弱、时间止损等自定义离场。

## 适用场景

适合**方向明确的中长期趋势市**，用日线定方向、小时级找买点的思路天然过滤掉了大量噪音，适合主流币对或趋势性强的标的。**不该**用在：震荡市（三个周期很难同时同向，要么不开仓、要么刚开仓就被某一周期的转向信号平掉，交易成本高）、以及数据历史不足的新币（日线指标需要足够长的历史才能算稳）。信号出现频率低是这套策略的固有特征——只有「顺大势 + 顺中势 + 顺小势」的窗口才开仓。

## 核心代码

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = [
        # 日线趋势确认（主趋势）
        dataframe['trend_1d'].fillna(False),
        dataframe['macd_bullish_1d'].fillna(False),
        dataframe['rsi_bullish_1d'].fillna(False),
        dataframe['adx_1d'].fillna(0) > self.adx_daily_threshold.value,
        dataframe['di_plus_1d'].fillna(0) > dataframe['di_minus_1d'].fillna(0),
        # 4 小时趋势确认（中期趋势）
        dataframe['trend_4h'].fillna(False),
        dataframe['macd_bullish_4h'].fillna(False),
        dataframe['rsi_4h'].fillna(50) > 50,
        dataframe['adx_4h'].fillna(0) > self.adx_h4_threshold.value,
        # 1 小时入场信号（短期时机）
        dataframe['trend_h1'],
        dataframe['price_above_ema_fast_h1'],
        dataframe['rsi_h1'] > self.h1_rsi_threshold.value,
        dataframe['volume_ratio_h1'] > self.volume_factor.value,
        # ... 其余 1 小时 MACD/ADX/DI 条件省略
    ]
    dataframe.loc[
        (conditions[0] & conditions[1] & conditions[2] & conditions[3] & conditions[4] &
         conditions[5] & conditions[6] & conditions[7] & conditions[8] &
         conditions[9] & conditions[10] & conditions[11] & conditions[12]),
        'enter_long'
    ] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `daily_ema_fast` / `daily_ema_slow` | 12 / 30 | 日线 EMA 周期 |
| `daily_rsi_period` | 14 | 日线 RSI 周期 |
| `h4_ema_fast` / `h4_ema_slow` | 18 / 48 | 4 小时 EMA 周期 |
| `h4_macd_fast` / `h4_macd_slow` | 12 / 26 | 4 小时 MACD 参数 |
| `h1_ema_fast` / `h1_ema_slow` | 14 / 32 | 1 小时 EMA 周期 |
| `h1_rsi_period` / `h1_rsi_threshold` | 14 / 50 | 1 小时 RSI 周期与阈值 |
| `adx_daily_threshold` / `adx_h4_threshold` / `adx_h1_threshold` | 25 / 25 / 25 | 各周期 ADX 强度门槛 |
| `volume_factor` | 1.8 | 1 小时成交量放量倍数 |
| `atr_period` / `atr_multiplier` | 14 / 3.0 | ATR 止损参数 |

硬编码的基础配置：`timeframe = '1h'`、`stoploss = -0.08`、`startup_candle_count = 100`，并在 `informative_pairs` 里注册了 4 小时和日线两个信息性周期。

## 风险与局限

- **未回测 / 需要自行验证**：源码中没有给出任何回测收益、胜率或回撤数字。
- **跨周期合并方式存在隐患**：把 4 小时 / 日线数据合并进 1 小时序列时用的是 `left_index=True, right_index=True`，但右侧先做了 `set_index('date')`（索引是时间戳），左侧 1 小时序列的索引是整数序号，两者很可能对不上，导致合并后 4 小时 / 日线指标整列变成 NaN；而入场条件里又用了 `.fillna(False)`，一旦如此，策略将永远不开仓。**这一点必须先用回测验证**。
- **`fillna(method='ffill')` 已过时**：新版 pandas 已弃用该写法，环境升级后可能报错，需要改成 `.ffill()`。
- **入场条件极多且为「与」关系**：三个周期累计 20 多个条件同时成立，信号频率会非常低，样本量少时统计意义有限。
- **出场比入场宽松得多**：任一周期的转向信号都能平仓，容易出现「入场难、出场早」的不对称，实际持仓时间可能偏短。
