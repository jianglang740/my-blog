---
title: "网格交易策略"
description: "区间震荡中的分层网格买卖，配合趋势过滤"
date: 2026-09-23
lastmod: 2026-10-16
weight: 8
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 网格交易策略

> **策略文件**：`GridTradingStrategy.py`（536 行）
> **来源**：自己开发

## 策略思路

在 15 分钟周期上，用一段回看窗口（默认 30 根）的最高价和最低价划定价格区间，并在区间内均分出多层网格线。当价格处于区间下半部分、且靠近某条网格线（距离小于半个网格步长）时买入，本质是「跌下来分批低吸」。为了让网格只在震荡市有效，代码加了多重过滤：波动率要适中（太低没机会、太高是单边行情）、ATR 占比小于 3%、价格不能偏离长期趋势线（80 周期 SMA）超过 15%、MACD 不能过度看跌、RSI 在 30 到 50 之间且动量开始企稳、成交量确认但不过度放大。卖出条件是价格进入区间上半部分、接近网格线上方、RSI 超买或回落、MACD 死叉等。另有基于 ATR 与网格步长的动态止损，以及杠杆上限 2 倍。

## 适用场景

适合**区间震荡市**——价格在一个相对稳定的箱体内反复波动，网格低吸高抛才能赚到价差。适合流动性好、日内波动适中的币对，15 分钟周期够用。**不该**用在：明显的单边上涨（早早卖飞，错过主升浪）或单边下跌（网格越买越亏，容易击穿止损）。网格策略天生是「赚震荡、亏趋势」的，趋势市里必须靠趋势过滤和止损及时退出，否则回撤会很深。

## 核心代码

```python
def _calculate_grid_lines(self, dataframe: DataFrame):
    grid_bottom = dataframe['price_min'] * (1 + self.grid_step_pct.value)
    grid_top = dataframe['price_max'] * (1 - self.grid_step_pct.value)
    grid_range = grid_top - grid_bottom
    for i in range(self.grid_levels.value):
        level_pct = i / (self.grid_levels.value - 1)
        grid_price = grid_bottom + (grid_range * level_pct)
        dataframe[f'grid_level_{i}'] = grid_price
    # ... 用 argmin 找到当前价格最近的网格线，算出 distance_to_grid_pct

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['price_position'] < 0.6) &          # 价格在区间下半部分
            (dataframe['distance_to_grid_pct'] < self.grid_step_pct.value * 0.5) &
            (dataframe['volatility'] < self.range_volatility_threshold.value) &  # 波动率适中
            (dataframe['atr_pct'] < 0.03) &                # 波动不过大
            (dataframe['close'] > dataframe['trend_sma'] * 0.95) &  # 非强跌趋势
            (dataframe['macd'] > dataframe['macd'].rolling(10).min() * 1.2) &
            (dataframe['rsi'] > self.rsi_oversold.value) &  # RSI > 30
            (dataframe['rsi'] < 50) &                       # RSI < 50
            (dataframe['momentum'] > dataframe['momentum'].shift(1)) &  # 动量企稳
            (dataframe['volume_ratio'] > self.volume_threshold.value) &  # 放量
            (dataframe['volume_ratio'] < 3.0) &             # 但非恐慌抛售
            (dataframe['price_position'] < 0.4)
        ),
        'enter_long'
    ] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `grid_levels` | 5 | 网格层数 |
| `grid_range_pct` | 0.10 | 网格区间宽度（百分比） |
| `grid_step_pct` | 0.02 | 网格步长（百分比） |
| `trend_sma_period` | 80 | 趋势过滤用的 SMA 周期 |
| `range_lookback` | 30 | 价格区间回看窗口 |
| `range_volatility_threshold` | 0.25 | 允许的最大波动率 |
| `volume_sma_period` | 20 | 成交量均线周期 |
| `volume_threshold` | 1.0 | 成交量确认阈值 |
| `rsi_period` | 14 | RSI 周期 |
| `rsi_oversold` / `rsi_overbought` | 30 / 70 | RSI 超卖 / 超买阈值 |
| `atr_period` | 14 | ATR 周期 |

硬编码的基础配置：`timeframe = '15m'`、`stoploss = -0.08`、`startup_candle_count = 100`，杠杆上限写死 2 倍。文件底部还有一个 `test_strategy()` 自测函数，用随机模拟数据跑一遍指标和信号统计。

## 风险与局限

- **未回测 / 需要自行验证**：源码没有提供任何回测业绩数据，本节不编造。
- **`informative_pairs` 名存实亡**：代码里声明了 1 小时和 4 小时的信息性时间框架（`informative_pairs` 方法），但并没有用 `@informative` 装饰器去真正计算和合并这些周期的指标，所以这段配置目前是空跑，跨周期趋势判断其实没生效。
- **网格只是「近似」**：入场要求价格距离最近网格线在一个步长内、且位于区间下半部分，但底层是逐根 K 线的阈值判断，并不是真正的挂单网格，交易次数与实际网格挂单模型会有偏差。
- **震荡转趋势的切换风险**：一旦行情从震荡转成单边，网格会连续接刀或过早卖飞；虽然有趋势过滤和 8% 止损兜底，但趋势来得越急，损失越集中在同一段。
- **入场条件叠加很多**：波动率、ATR、趋势偏移、MACD、RSI、动量、成交量全部为「与」关系，信号可能非常稀疏，需要回测确认样本量是否足够。
