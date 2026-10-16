---
title: "均值回归策略"
description: "布林带下轨抄底、上轨附近止盈的震荡市反转策略"
date: 2026-09-13
lastmod: 2026-10-16
weight: 2
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 均值回归策略

> **策略文件**：`MeanReversionStrategy.py`（386 行）
> **来源**：自己开发

## 策略思路

这是基于布林带的低买高卖均值回归策略，跑在 15 分钟 K 线上、只做多。开仓要求价格触及或接近布林带下轨（`close <= bb_lower * 1.005` 或最低价跌破下轨），同时满足 RSI 低于阈值、威廉指标 `WILLR < -50`、价格偏离 20 周期均线足够远、成交量高于均量、以及 `ema_12 > ema_26 * 0.95`（不过滤下跌趋势，避免在瀑布中逆势）。平仓是价格回到布林带上轨附近且 RSI 偏高、威廉指标偏高，或者用「MACD 死叉 + EMA 下穿」这组趋势转弱信号兜底。策略还实现了 `custom_stoploss`（ATR 动态止损）和 `custom_entry_price`（尝试用限价单买得更便宜），并进行 1 小时级别的多周期数据声明。

## 适用场景

适合**区间震荡、均值回复明显的市场**，尤其是波动收敛、价格在布林带上下轨之间来回摆动的行情；15 分钟周期让它能在日内反复抓小的回归波动。**不适合单边趋势市**：价格贴着下轨一路阴跌时，任何「抄底」信号都会不断被套，均值回归策略在趋势行情里最危险。因此代码用了 RSI、成交量、EMA 排列等多重条件来压制逆势入场，但这类过滤只能降低概率、无法消除。

## 核心代码

```python
# populate_indicators 关键部分
bollinger = qtpylib.bollinger_bands(
    dataframe['close'],
    window=self.bb_period.value,
    stds=self.bb_std.value
)
dataframe['bb_lower'] = bollinger['lower']
dataframe['bb_middle'] = bollinger['mid']
dataframe['bb_upper'] = bollinger['upper']
dataframe['bb_width'] = (dataframe['bb_upper'] - dataframe['bb_lower']) / dataframe['bb_middle']
dataframe['rsi'] = ta.RSI(dataframe, timeperiod=self.rsi_period.value)
dataframe['williams_r'] = ta.WILLR(dataframe, timeperiod=14)
dataframe['price_deviation'] = abs(dataframe['close'] - dataframe['sma_20']) / dataframe['sma_20']

# populate_entry_trend：布林带下轨 + RSI + 威廉 + 偏离度 + 成交量
dataframe.loc[
    (
        conditions[0] &  # 布林带下轨
        conditions[1] &  # RSI相对较低
        conditions[2] &  # 威廉指标相对较低
        conditions[3] &  # 价格偏离
        conditions[4] &  # 成交量
        conditions[5]    # 趋势条件
    ),
    'enter_long'
] = 1
# 其中 conditions[0] 为：
#   (dataframe['close'] <= dataframe['bb_lower'] * 1.005) | (dataframe['low'] <= dataframe['bb_lower'])

# populate_exit_trend：回归上轨 或 趋势转弱
dataframe.loc[
    (
        (conditions[0] & conditions[1] & conditions[2]) |
        (trend_weak_conditions[0] & trend_weak_conditions[1])
    ),
    'exit_long'
] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `bb_period` / `bb_std` | 20 / 2.0 | 布林带窗口与标准差倍数 |
| `rsi_period` | 14 | RSI 周期 |
| `rsi_oversold` | 45 | 开仓的 RSI 上限（注释说明已放宽，45 并不算严格超卖） |
| `rsi_overbought` | 65 | 平仓的 RSI 下限 |
| `price_deviation_threshold` | 0.005 | 价格偏离 20 均线的最小比例 |
| `volume_threshold` | 1.0 | 成交量相对均量的下限 |
| `atr_period` / `atr_multiplier` | 14 / 2.0 | ATR 与动态止损倍数 |
| `stoploss` | -0.05 | 固定止损 5% |
| `timeframe` | 15m | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **注释与代码不一致**：类文档说「RSI 处于超卖状态」，但 `rsi_oversold` 默认值 45 其实偏中性；代码里多处 `# 放宽` / `# 大幅放宽` 的注释说明参数被反复调松以增加信号数量，这种「为了开仓率而松阈值」的做法会显著提高假信号比例，也是最容易过拟合的地方。
- **`custom_stoploss` 当前不生效**：`use_custom_stoploss = False`，定义了 ATR 动态止损但不会调用，实际止损是固定 `-0.05`。
- **`custom_entry_price` 注释与实现不符**：注释写「最多降低 0.1%」，但代码用 `min(target_price, proposed_rate * 0.999)`，实际最多可能把限价挂到接近布林带下轨，降幅可远超 0.1%，且在快速下跌中可能挂不上单、错失成交。
- **`informative_pairs` 声明了 1h 数据却未使用**：注册了额外时间框架但没有对应的 `@informative` 处理逻辑，属于无用开销（会白白拉取数据）。
- **趋势市里逆势风险大**：只做多、无杠杆，一旦进入单边下跌，下轨抄底会连续触发并连环止损，是这类策略的典型回撤来源。
