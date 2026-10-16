---
title: "反面教材：wtc"
description: "反面教材：MinMaxScaler 全量拟合造成未来函数"
date: 2026-10-16
lastmod: 2026-10-16
weight: 21
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 反面教材：wtc

> ⚠️ **这是一份反面教材**：本策略故意保留了未来函数（lookahead bias），回测曲线会漂亮得不真实。请勿用于实盘，也请勿直接抄进自己的策略。

> **策略文件**：`wtc.py`（158 行）
> **来源**：原作者 @Mablue (Masoud Azizi)，来自 freqtrade 官方策略仓库的 `lookahead_bias` 教学目录；许可证以源码实际声明为准（源码仅标注作者，未声明具体许可证）。

## 策略思路

wtc（World Trade Center Tabriz，也是 Wave Trend with Crosses 的缩写）复刻了 TradingView 上 LazyBear 的 **WaveTrend** 指标，并用 **Stoch-RSI** 做辅助来降低信号噪声：

- 用 `(high + low + close) / 3` 作为典型价 `ap`，两条 EMA（周期 10、21）构造出 `wt1` 与它的 4 周期 SMA `wt2`；
- 另算一个 `slowk`（`ta.STOCH` 的慢速线）；
- 定义 `def = slowk − wt1`；
- **开仓**：`wt1` 上穿 `wt2`，且 `wt1`、`slowk`、`def` 三者分别落在各自的 `[min, max]` 区间内；
- **平仓**：`wt1` 下穿 `wt2`，且三个量分别落在卖出区间内。

周期 30m，可调参数是六组区间上下界（`buy_min/max`、`buy_min0/max0`、`buy_min1/max1` 及其 sell 对应项）。

它本来想做的事：把这些量统一缩放到相近尺度，方便「用一组 [min, max] 区间」同时约束三个指标。**未来函数就藏在第 121~122 行**的 `MinMaxScaler` —— 它 `fit_transform` 时用了整条序列的绝对极值。

## 适用场景

**没有可信任的适用场景**。设计者想做的是 30m 高频的波峰波谷择时（WaveTrend 经典用法），但 `MinMaxScaler` 一旦在全量数据上拟合，回测里的区间边界就带上了未来色彩。无论趋势市还是震荡市，绩效都不可信，请勿据此实盘。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    try:
        ap = (dataframe['high'] + dataframe['low'] + dataframe['close']) / 3
        esa = ta.EMA(ap, 10)
        d = ta.EMA((ap - esa).abs(), 10)
        ci = (ap - esa).div(0.0015 * d)
        tci = ta.EMA(ci, 21)

        wt1 = tci
        wt2 = ta.SMA(np.nan_to_num(wt1), 4)
        dataframe['wt1'], dataframe['wt2'] = wt1, wt2

        stoch = ta.STOCH(dataframe, 14)
        dataframe['slowk'] = stoch['slowk']

        x = dataframe.iloc[:, 6:].values                  # 取出 wt1/wt2/slowk 等
        min_max_scaler = preprocessing.MinMaxScaler()     # 第 121 行
        x_scaled = min_max_scaler.fit_transform(x)        # 第 122 行：用全序列 min/max 拟合
        dataframe.iloc[:, 6:] = pd.DataFrame(x_scaled)

        dataframe['def'] = dataframe['slowk'] - dataframe['wt1']
    except:
        dataframe['wt1'], dataframe['wt2'], dataframe['def'], dataframe['slowk'] = 0, 10, 100, 1000
    return dataframe


def populate_entry_trend(self, dataframe, metadata):
    dataframe.loc[
        (qtpylib.crossed_above(dataframe['wt1'], dataframe['wt2']))
        & (dataframe['wt1'].between(self.buy_min0.value, self.buy_max0.value))
        & (dataframe['slowk'].between(self.buy_min1.value, self.buy_max1.value))
        & (dataframe['def'].between(self.buy_min.value, self.buy_max.value)),
        'enter_long'] = 1
    return dataframe
```

## 为什么这是未来函数

关键在于 `sklearn` 的 `MinMaxScaler` 到底做了什么：

```python
min_max_scaler = preprocessing.MinMaxScaler()
x_scaled = min_max_scaler.fit_transform(x)
```

`fit` 这一步会**扫描整列 `x` 求绝对最小值和最大值**，`transform` 再用这对极值把每个点缩放到 [0, 1]。问题在于 `x` 是**整段回测区间**的序列——它包含了当前决策时点**之后**的 bar。`fit_transform` 相当于把「未来才会出现的最高点和最低点」偷来当作缩放基准：

- 第 `t` 根 bar 的缩放值 = `(x_t − 全序列(含未来)最小值) / (全序列(含未来)最大值 − 全序列(含未来)最小值)`；
- 于是每一根 bar 上的 `wt1`、`slowk`、`def` 都被「事后极值」重新拉伸过；
- 那些 `[min, max]` 区间参数（`buy_min/max` 等）就能在回测里异常精准地圈中「低点买、高点卖」，制造虚假的择时能力。

这正是**用未来才知道的最高最低点去缩放今天的数据**——数据泄漏（data leakage），只不过这次不是手写 `.min()/.max()`，而是被封装进了 `MinMaxScaler.fit_transform` 里，更隐蔽。`fit_transform` 这个「拟合 + 转换」一体的 API 在离线机器学习里很正常，但**在时间序列回测里，任何「拟合」都只能用过去的数据**，用了全量就等于作弊。

## 怎么改才对

思路是：把「全量拟合」换成「只用截至当前 bar 的极值」。两种常见写法：

```python
# 方案一：滚动窗口缩放（只看过去 window 根）
x = dataframe.iloc[:, 6:]
window = 200
roll_min = x.rolling(window, min_periods=1).min()
roll_max = x.rolling(window, min_periods=1).max()
denom = (roll_max - roll_min).replace(0, 1e-10)
dataframe.iloc[:, 6:] = (x - roll_min) / denom
```

```python
# 方案二：扩展窗口缩放（只看历史累积极值）
x = dataframe.iloc[:, 6:]
exp_min = x.expanding().min()
exp_max = x.expanding().max()
dataframe.iloc[:, 6:] = (x - exp_min) / (exp_max - exp_min)
```

要点：

1. `rolling` / `expanding` 都只向过去看（含当前**已收盘** bar），不含未来 bar，从根上断了泄漏。
2. 如果非要用 `sklearn`，必须**逐 bar 重拟合**：对第 `t` 根 bar，只用 `x[:t]` 调 `fit`，再对 `x_t` 调 `transform`——但这在 freqtrade 的向量化回测里很慢，通常不值得，优先用上面的 pandas 方案。
3. **改完必须重新 hyperopt**：缩放尺度变了，那六组 `[min, max]` 区间参数（默认值如 `0.8633`、`0.0102`）全部作废。
4. 更干净的选择是**不缩放**：WaveTrend 的 `wt1/wt2` 本身围绕 0 波动，`def` 也有明确零点，直接用这些有物理意义的原始值设阈值，比先缩放再卡区间更稳，也彻底消除这条泄漏路径。
5. 验证：修复后回测收益应显著下降；再检查样本内外的信号频率、区间命中率是否一致。

## 风险与局限

- **回测结果完全不可信**：`MinMaxScaler` 在全量数据上拟合，所有绩效数字都是赝品，只能标为「未回测 / 需要自行验证」。
- **实盘立即失效**：实盘没有未来极值，缩放基准会随新数据漂移，`[min, max]` 区间大面积打空或乱触发。
- **隐蔽性极强**：泄漏被藏在 `fit_transform` 这个「看起来没问题」的 API 里，比手写 `.min()/.max()` 更难一眼看穿，容易误导新手照抄。
- **区间参数过拟合**：六组 `[min, max]` 是在被污染的数据上超优化出来的，位数精确到 4 位小数，明显是拟合噪声的产物。
- **异常兜底掩盖问题**：整个指标计算被包在 `try/except` 里，一旦出错就静默把 `wt1/wt2/def/slowk` 填成 `0/10/100/1000`，会让错误悄悄变成「假信号」而不报错，调试时需格外当心。
