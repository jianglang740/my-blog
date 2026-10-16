---
title: "反面教材：Zeus"
description: "反面教材：Ichimoku/KST 归一化用了未来数据"
date: 2026-10-14
lastmod: 2026-10-16
weight: 20
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 反面教材：Zeus

> ⚠️ **这是一份反面教材**：本策略故意保留了未来函数（lookahead bias），回测曲线会漂亮得不真实。请勿用于实盘，也请勿直接抄进自己的策略。

> **策略文件**：`Zeus.py`（149 行）
> **来源**：原作者 @Mablue (Masoud Azizi)，来自 freqtrade 官方策略仓库的 `lookahead_bias` 教学目录；许可证以源码实际声明为准（源码仅标注作者，未声明具体许可证）。

## 策略思路

Zeus 是「GodStra 家族」的第一代简化版，放弃了大杂烩式的指标组合，只留下两个指标：

- 入场信号用**一目均衡表的基准线**（`trend_ichimoku_base`，由 `ta.trend.ichimoku_base_line` 计算，`window1=9, window2=26`）；
- 离场信号用 **KST 指标的差值**（`trend_kst_diff`）。

它本来想做的事：把这两个量都**归一化到 [0, 1]**，然后用一个 0~1 之间的实数阈值 `real_num` 加算子（`>R` / `=R` / `<R`）来判断「基准线是不是足够低（超卖）该买」「KST 差值是不是足够高该卖」。周期 4h，超参 `buy_real`、`sell_real`、`buy_cat`、`sell_cat`。

**未来函数就藏在第 101~104 行**：对 `trend_ichimoku_base` 和 `trend_kst_diff` 做的归一化，用的是全量 `.min()` / `.max()`。

## 适用场景

**没有可信任的适用场景**。Zeus 的设计初衷是「用一条经过缩放的基准线做均值回归择时」，但归一化一旦引入未来极值，回测里的阈值就失去了真实含义。趋势市、震荡市都别指望它——你看到的绩效全部来自未来信息，参考价值为零。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 一目均衡表基准线
    dataframe['trend_ichimoku_base'] = ta.trend.ichimoku_base_line(
        dataframe['high'], dataframe['low'], window1=9, window2=26,
        visual=False, fillna=False)

    # KST 差值
    KST = ta.trend.KSTIndicator(
        close=dataframe['close'], roc1=10, roc2=15, roc3=20, roc4=30,
        window1=10, window2=10, window3=10, window4=15, nsig=9, fillna=False)
    dataframe['trend_kst_diff'] = KST.kst_diff()

    # 归一化 —— 问题就出在这四行
    tib = dataframe['trend_ichimoku_base']
    dataframe['trend_ichimoku_base'] = (
        tib-tib.min())/(tib.max()-tib.min())        # 第 101-102 行
    tkd = dataframe['trend_kst_diff']
    dataframe['trend_kst_diff'] = (
        tkd-tkd.min())/(tkd.max()-tkd.min())        # 第 103-104 行
    return dataframe


def populate_entry_trend(self, dataframe, metadata):
    IND, REAL, OPR = 'trend_ichimoku_base', self.buy_real.value, self.buy_cat.value
    DFIND = dataframe[IND]
    if OPR == ">R":
        conditions.append(DFIND > REAL)
    elif OPR == "=R":
        conditions.append(np.isclose(DFIND, REAL))
    elif OPR == "<R":
        conditions.append(DFIND < REAL)             # 默认：基准线偏低才买
    # ...
```

## 为什么这是未来函数

慢动作看第 101~102 行（第 103~104 行同理）：

```python
tib = dataframe['trend_ichimoku_base']
dataframe['trend_ichimoku_base'] = (tib - tib.min()) / (tib.max() - tib.min())
```

`tib.min()` 和 `tib.max()` 是在**整条序列**上取的极值——而这条序列覆盖了整个回测区间，包含了当前 bar **之后**的数据。于是：

- 第 `t` 根 bar 的归一化值 = `(x_t − 全历史(含未来)最小值) / (全历史(含未来)最大值 − 全历史(含未来)最小值)`；
- 站在第 `t` 根 bar 上，程序其实**提前知道了整段行情的最高点和最低点**；
- 用它去缩放「今天的基准线」，得到的数值只有预知未来才算得出来。

这正是**用未来才知道的最高最低点去缩放今天的数据**的典型泄漏（data leakage）。后果是：`buy_real` / `sell_real` 这两个 0~1 的阈值，在回测里能异常精准地卡在「归一化后的低位/高位」，制造出虚假的择时能力。实盘时未来极值不存在，同一根基准线的归一化值会完全不同，阈值随之失效。

## 怎么改才对

把全量 `.min()/.max()` 换成**只看历史**的滚动窗口或扩展窗口：

```python
def normalize_rolling(df, window=200, min_periods=1):
    roll_min = df.rolling(window, min_periods=min_periods).min()
    roll_max = df.rolling(window, min_periods=min_periods).max()
    denom = (roll_max - roll_min).replace(0, 1e-10)
    return (df - roll_min) / denom


tib = dataframe['trend_ichimoku_base']
dataframe['trend_ichimoku_base'] = normalize_rolling(tib)
tkd = dataframe['trend_kst_diff']
dataframe['trend_kst_diff'] = normalize_rolling(tkd)
```

或者：

```python
roll_min = tib.expanding().min()
roll_max = tib.expanding().max()
dataframe['trend_ichimoku_base'] = (tib - roll_min) / (roll_max - roll_min)
```

要点：

1. `rolling(200)` 在第 `t` 根 bar 上只看过去 200 根（含当前**已收盘** bar），`expanding()` 只看历史累积，两者都不含未来，是安全的。
2. **改完必须重新 hyperopt**：归一化尺度变了，`buy_real`（默认 0.11908）、`sell_real`（默认 0.59608）这批阈值不再适用。
3. 更省事的思路是**放弃归一化**：一目均衡表基准线本身是价格量级，KST 差值也有明确的零点含义，直接用原始值配合理性的价格/零点阈值，比强行缩放到 [0, 1] 干净得多，也彻底断了这条泄漏路径。
4. 验证：修复后收益应显著下降；同时观察样本内外的信号频率是否稳定，避免「回测好看、实盘失灵」。

## 风险与局限

- **回测结果完全不可信**：归一化使用了未来信息，所有绩效数字都是赝品，只能标为「未回测 / 需要自行验证」。
- **实盘立即失效**：未来极值在实盘不存在，归一化值漂移，依赖 0~1 阈值的买卖条件会大面积失真。
- **信号稀疏、样本敏感**：只看两个指标、单一阈值砍信号，交易次数通常很少，统计意义弱，回测易被少数几笔交易主导。
- **接口依赖**：依赖第三方 `ta` 库（`pip install ta`），与 freqtrade 常用的 talib 路线不同，版本变动可能影响结果复现。
- **易被误当模板**：代码短、结构清爽，最容易被新手当成「简洁策略模板」抄走——正因如此，这份未来函数才格外值得记住。
