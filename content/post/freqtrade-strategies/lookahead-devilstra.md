---
title: "反面教材：DevilStra"
description: "故意植入未来函数的反面教材：归一化用了全量数据"
date: 2026-10-11
lastmod: 2026-10-16
weight: 18
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 反面教材：DevilStra

> ⚠️ **这是一份反面教材**：本策略故意保留了未来函数（lookahead bias），回测曲线会漂亮得不真实。请勿用于实盘，也请勿直接抄进自己的策略。

> **策略文件**：`DevilStra.py`（722 行）
> **来源**：原作者 @Mablue (Masoud Azizi)，来自 freqtrade 官方策略仓库的 `lookahead_bias` 教学目录；许可证以源码实际声明为准（源码仅标注作者，未声明具体许可证）。

## 策略思路

DevilStra 是 GodStraNew 的「恶魔」版本，走的是「遗传法术」路线：它把一大堆 TA-Lib 指标、指标周期、比较算子、阈值预先组合成若干「法术（spell）」，存放在 `SPELLS` 字典里（`Zi` / `Gu` / `Lu` / `La` / `Si` / `Pa` / `De` / `Ra` / `Cu`）。每个交易对按顺序认领一个法术，用其中的 `indicator0/1/2` 与 `crossed_indicator0/1/2` 两两比较（算子如 `>R`、`<R`、`CA`、`CDT` 等），再用 `real_num` 做阈值，三条条件的**逻辑与（AND）**同时成立才开仓；卖出一侧同理。周期为 4h，且要求配置里使用**静态** pairlist。

它本来想做的事很朴素：把任意 TA-Lib 指标归一化到 [0, 1] 之后，就能用统一的「0.7 以上算高位、0.3 以下算低位」这类阈值来生成信号，于是可以批量超参优化找出「神之一手」。

**未来函数就藏在第 331 行**——`normalize()` 函数里的 `df.min()` / `df.max()`；`gene_calculator()` 在每个指标分支（第 361、369、379、389、400 行）调用它，把这些被「污染」的指标一路带进买卖条件。

## 适用场景

严格来说**没有可信任的适用场景**：只要 `normalize()` 还在用全量数据，任何回测结论都是赝品。设计者瞄准的是 4h 周期的山寨币趋势行情，但「它在趋势市/震荡市表现如何」这个问题在这里毫无意义——你看到的一切漂亮数字都来自未来信息。这个脚本唯一合适的用途，是拿来练「一眼找出未来函数」的眼力。

## 核心代码

```python
def normalize(df):
    df = (df-df.min())/(df.max()-df.min())   # 第 331 行：未来函数就在这里
    return df


def gene_calculator(dataframe, indicator):
    # ... 省略 CDL 名称规整（把周期位强制设成 0）...
    gene = indicator.split("-")
    gene_name, gene_len = gene[0], len(gene)

    if indicator in dataframe.keys():
        return dataframe[indicator]
    if gene_len == 1:
        result = getattr(ta, gene_name)(dataframe)
        return normalize(result)             # 第 361 行
    elif gene_len == 2:
        result = getattr(ta, gene_name)(dataframe, timeperiod=int(gene[1]))
        return normalize(result)             # 第 369 行：未来信息由此进入
    elif gene_len == 3:
        result = getattr(ta, gene_name)(
            dataframe, timeperiod=int(gene[2])).iloc[:, int(gene[1])]
        return normalize(result)             # 第 379 行
    # ... gene_len == 4 / 5 的「趋势算子」分支同理（第 389、400 行）...


def condition_generator(dataframe, operator, indicator, crossed_indicator, real_num):
    condition = (dataframe['volume'] > 10)
    dataframe[indicator] = gene_calculator(dataframe, indicator)
    dataframe[crossed_indicator] = gene_calculator(dataframe, crossed_indicator)

    if operator == "<R":
        condition = (dataframe[indicator] < real_num)
    elif operator == ">R":
        condition = (dataframe[indicator] > real_num)
    elif operator == "CA":
        condition = qtpylib.crossed_above(dataframe[indicator], dataframe[crossed_indicator])
    # ... 其余算子分支 ...
    return condition, dataframe


# populate_entry_trend 里：
if conditions:
    dataframe.loc[reduce(lambda x, y: x & y, conditions), 'enter_long'] = 1
```

## 为什么这是未来函数

看这一行：

```python
df = (df-df.min())/(df.max()-df.min())
```

`df.min()` 和 `df.max()` 是**在整条序列上求的最小值和最大值**。在回测里，「整条序列」包含了你站在今天这一刻**还不可能知道**的未来 bar。于是：

- 第 `t` 根 K 线归一化后的数值 = `(x_t - 全历史(含未来)最小值) / (全历史(含未来)最大值 - 全历史(含未来)最小值)`。
- 分母和分子里的极值，都是「事后诸葛亮」才能拿到的数。
- 举个直观的例子：某个指标真正的高点在 2025 年 12 月才出现，但用全量 min/max 归一化后，早在 2024 年的某一根 bar 上，程序就已经「知道」了 2025 年的极值在哪，从而把 2024 年的数值压缩到了一个只有预知未来才可能的区间。

这就是数据泄漏（data leakage）：**用未来才知道的最高点和最低点，去缩放今天的数据**。它让归一化后的指标带着一丝未来的味道，而这丝味道刚好足以让 `real_num` 阈值在回测里「卡」得异常精准——顺带一提，`DevilStra` 那种动辄几十上百个指标的组合，本质上是在海量维度里反复抽取「碰巧与未来对齐」的巧妙组合，属于典型的过拟合放大器。

`normalize()` 出现在每条指标计算路径上，因此泄漏是系统性的，不是某一根 bar 的偶然。

## 怎么改才对

核心原则：**任何归一化 / 标准化，都只能使用「截至当前 bar」的数据**。最直接的做法是把全量 `.min()/.max()` 换成滚动窗口：

```python
def normalize_rolling(df, window=200, min_periods=1):
    roll_min = df.rolling(window, min_periods=min_periods).min()
    roll_max = df.rolling(window, min_periods=min_periods).max()
    denom = (roll_max - roll_min).replace(0, 1e-10)   # 防止除零
    return (df - roll_min) / denom
```

要点：

1. `rolling(200)` 在第 `t` 根 bar 上只看过去 200 根（含当前这根**已收盘**的 bar），不再「偷看」未来。
2. 想要更省事的等价做法是 `expanding()`（只往回头看，永不看未来）：
   ```python
   roll_min = df.expanding().min()
   roll_max = df.expanding().max()
   ```
   `expanding()` 只用历史累积的极值，同样不含未来。
3. **改完必须重新超参优化**。归一化尺度一变，原来那批 `real_num`（比如 `0.0509`、`0.6891`）就全部失效了，不能直接复用。
4. 对本身就有界、有物理意义的振荡类指标（如 RSI、WILLR、STOCH），更干净的做法是**干脆不归一化**，直接用原始值配合理性的阈值，避免「为了统一阈值而引入缩放」这个动机本身。
5. 修好之后，务必用 `freqtrade backtesting-analysis` 或直接对比「改了归一化前后的信号分布」来验证——正常情况下，修复后回测的收益率会**大幅下降**，这才是真实水平。

## 风险与局限

- **回测结果完全不可信**：未来函数让回测在使用未来信息，任何收益率、夏普、胜率数字都是赝品，请当作「未回测 / 需要自行验证」，绝不能拿去展示。
- **实盘会立刻打回原形**：实盘没有未来数据可偷，归一化尺度会跳变，回测里「精准」的阈值在实盘会大面积失效。
- **极度过拟合**：`SPELLS` 是几十上百个指标的随机组合，配合被泄漏污染的阈值，几乎必然能在历史数据上「碰」出漂亮曲线。
- **可复现性差**：`spell_pot` 用 `random.choices(...)` 且未固定随机种子，每次运行生成的候选法术串都可能不同。
- **工程上的脆弱约定**：要求静态 pairlist、要手动把超参粘回第 535~564 行、`PAIR_LIST_LENGHT` 与实际白名单数量必须精确对齐，否则直接 `exit()`——这些都是回测脚本的坏习惯，不适合工程化。
