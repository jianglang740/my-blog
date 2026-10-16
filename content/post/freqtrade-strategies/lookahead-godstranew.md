---
title: "反面教材：GodStraNew"
description: "反面教材：全量数据归一化引发未来函数"
date: 2026-10-12
lastmod: 2026-10-16
weight: 19
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 反面教材：GodStraNew

> ⚠️ **这是一份反面教材**：本策略故意保留了未来函数（lookahead bias），回测曲线会漂亮得不真实。请勿用于实盘，也请勿直接抄进自己的策略。

> **策略文件**：`GodStraNew.py`（645 行）
> **来源**：原作者 @Mablue (Masoud Azizi)，来自 freqtrade 官方策略仓库的 `lookahead_bias` 教学目录；许可证以源码实际声明为准（源码仅标注作者，未声明具体许可证）。

## 策略思路

GodStraNew 是「上帝版遗传策略」：它先攒出一个庞大的 TA-Lib 指标库 `god_genes`（重叠研究、动量、成交量、波动率、价格变换、周期、K 线形态、统计函数八大类，上百个指标），再与一组周期 `timeperiods = [5, 6, 12, 15, 50, 55, 100, 110]` 做笛卡尔积，得到候选「基因」。这些基因被包进 `CategoricalParameter`，连同算子（`>R`、`<R`、`CA`、`CDT` …）和阈值 `real_num` 一起交给 hyperopt 去搜。

开仓逻辑是：把 `indicator0/1/2` 分别与其配对指标 `crossed_indicator0/1/2` 做比较，三条条件**全部成立（AND）**才 `enter_long = 1`；平仓用同样结构的三条卖条件。周期 4h。

它本来想做的事是：让「指标 + 周期 + 算子 + 阈值」全空间自动化搜索，把归一化到 [0, 1] 的指标统一用 0~1 的实数阈值来卡点。

**未来函数就藏在第 256 行**——`normalize()` 里的 `df.min()` / `df.max()`；`gene_calculator()` 在每个指标分支（第 286、294、304、314、325 行）都调用了它。

## 适用场景

**没有可信任的适用场景**。这个策略的口号是「让 hyperopt 找到最优组合」，但它的评价函数（回测结果）本身已经被未来函数污染，于是 hyperopt 搜索的方向就是「往更能偷看未来的组合上跑」。无论趋势市还是震荡市，你得到的都是过拟合的假象。它唯一的正当用途，是和 DevilStra 一起当作「识别未来函数」的练习题。

## 核心代码

```python
def normalize(df):
    df = (df-df.min())/(df.max()-df.min())   # 第 256 行：未来函数就在这里
    return df


def gene_calculator(dataframe, indicator):
    # ... 省略 CDL 名称规整 ...
    gene = indicator.split("-")
    gene_name, gene_len = gene[0], len(gene)

    if indicator in dataframe.keys():
        return dataframe[indicator]
    if gene_len == 1:
        result = getattr(ta, gene_name)(dataframe)
        return normalize(result)             # 第 286 行
    elif gene_len == 2:
        result = getattr(ta, gene_name)(dataframe, timeperiod=int(gene[1]))
        return normalize(result)             # 第 294 行：未来信息由此进入
    elif gene_len == 3:
        result = getattr(ta, gene_name)(
            dataframe, timeperiod=int(gene[2])).iloc[:, int(gene[1])]
        return normalize(result)             # 第 304 行
    # ... gene_len == 4 / 5 的「趋势算子」分支同理（第 314、325 行）...


# 可优化空间：指标 × 周期 组成候选「基因」
buy_indicator0 = CategoricalParameter(
    god_genes_with_timeperiod, default="SMA-100", space='buy')
buy_operator0 = CategoricalParameter(operators, default="/<R", space='buy')
buy_real_num0 = DecimalParameter(0, 1, decimals=DECIMALS, default=0.89009, space='buy')

# populate_entry_trend 里：
if conditions:
    dataframe.loc[reduce(lambda x, y: x & y, conditions), 'enter_long'] = 1
```

## 为什么这是未来函数

`normalize()` 与 DevilStra 完全同源，问题也完全一样：

```python
df = (df-df.min())/(df.max()-df.min())
```

`df.min()` / `df.max()` 计算的是**整条序列**（整个回测区间）的极值，而这个区间里包含了决策时点**之后**才出现的 bar。等价于：

- 站在第 `t` 根 bar 上，程序却已经知道整段行情（含未来）的最高点和最低点；
- 用它来把 `x_t` 缩放到 [0, 1]，得到的是一个「只有预知未来才可能算出」的数；
- 因为所有指标都被这套未来极值缩放过，`real_num` 阈值（0~1 的实数）就能在回测里异常精准地卡中「低位」，制造出虚假的择时能力。

这就是**用未来才知道的最高最低点去缩放今天的数据**——数据泄漏（data leakage）。GodStraNew 把它放大到了极致：上百个指标 × 8 个周期，每个都自带未来极值，hyperopt 在这么大的空间里搜索，几乎必然挑出一堆「与未来对齐」的组合，换来一条漂亮但不可能复现的曲线。

顺带一提，`populate_indicators()` 里几乎什么都没算（直接 `return dataframe`），所有指标都在入场/离场时刻现算，这也让 `normalize()` 的泄漏无处藏身地被摊开在每一根 bar 上。

## 怎么改才对

原则不变：**归一化只能用截至当前 bar 的数据**。把全量 `.min()/.max()` 改成滚动或扩展窗口即可：

```python
def normalize_rolling(df, window=200, min_periods=1):
    roll_min = df.rolling(window, min_periods=min_periods).min()
    roll_max = df.rolling(window, min_periods=min_periods).max()
    denom = (roll_max - roll_min).replace(0, 1e-10)
    return (df - roll_min) / denom
```

或者用只向历史看的扩展窗口：

```python
roll_min = df.expanding().min()
roll_max = df.expanding().max()
```

要点：

1. `rolling(200)` / `expanding()` 都只看过去（含当前**已收盘** bar），不含任何未来 bar。
2. **改完必须重新 hyperopt**：归一化尺度变了，那批 0~1 的 `real_num` 全部作废，不能沿用。
3. 有界振荡类指标（RSI、WILLR、STOCHRSI 等）建议直接不归一化，用原始值加合理阈值，绕开「为了统一阈值而缩放」的动机。
4. 对本策略还有一层工程建议：`default` 里出现了 `ADD-20`、`ASIN-6` 这类**不在 `god_genes` 基因池**里的指标（池里没有 `ADD`、`ASIN`），说明默认配置本身就不可靠、不可复现——即使修好未来函数，也建议把参数空间和默认值都重新梳理干净。
5. 验证方法：修复后回测收益应当**显著下降**，且信号分布应在样本内外保持一致（不再「回测无敌、实盘全废」）。

## 风险与局限

- **回测结果完全不可信**：未来函数让整条评价链路失真，任何收益/夏普/胜率都是赝品，只能标为「未回测 / 需要自行验证」。
- **实盘立刻失效**：实盘拿不到未来极值，归一化尺度会漂移，依赖 0~1 阈值的信号大面积失效。
- **极端过拟合**：上百指标 × 8 周期 × 多算子的搜索空间，配合泄漏污染的目标函数，必然搜出「历史最优、未来无用」的组合。
- **默认配置可疑**：`default="ADD-20"`、`default="ASIN-6"` 等指标不在基因池中，说明这份配置未必可复现，参考价值低。
- **工程坏味道**：`populate_indicators()` 空实现、全部指标在入场/离场时现算，既慢又掩盖了泄漏点，不适合直接工程化。
