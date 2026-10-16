---
title: "GodStra 策略"
description: "TA 全特征库配超参「DNA」拼装条件的空框架策略"
date: 2026-10-05
lastmod: 2026-10-16
weight: 15
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# GodStra 策略

> **策略文件**：`GodStra.py`（178 行）
> **来源**：第三方作者。原作者 @Mablue（Masoud Azizi），GitHub: `https://github.com/mablue/`。**源码未声明许可证**。

## 策略思路

GodStra（God Strategy）走的是「先把所有指标都算出来，再让超参挑」的路线。`populate_indicators` 调用 `ta` 库的 `add_all_ta_features()` 一次性生成**上百个**技术特征列（趋势、动量、成交量、波动率、形态识别等）。随后开仓/平仓条件不是写死的，而是由一组「DNA」式超参拼装：每一组包含「一个指标列名 `indicator` + 一个对照列名 `cross` + 一个操作符 `oper` + 一个整数 `int` + 一个实数 `real`」，操作符支持 `> / < / = / CA(上穿) / CB(下穿) / >I / =I / <I / >R / =R / <R` 等，把指标和别的东西（另一列、常数、交叉事件）比较。开多（平多）就是这些条件全部取「与」。默认只配了 1 组 DNA。整套东西要真正发挥作用，必须先把指标名和对照项用 hyperopt 搜出来。

## 适用场景

这是一个**探索 / 自动化搜索框架**，而不是一个可以直接上实盘的成型策略。它适合用来快速试「某个 TA 特征配某个操作符是否有效」，但默认参数只是某一次 hyperopt 的产物，不具备普适性，**换品种、换周期后需要重新搜索**。作者在文件头给出了运行前置条件（务必遵守）：需要在 `config.json` 的 StaticPairList 下加 `AgeFilter`（`min_days_listed: 30`）过滤新币；必须 `pip install ta`；并建议把 `max_open_trades` 设得很小以取得较好结果。**不该用的场景**：不做超参搜索就用默认值；以及把它当「上帝策略」直接实盘——名字夸张但内核是对历史的高维拟合。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 一次性生成 `ta` 库的「全部」技术特征列
    dataframe = dropna(dataframe)
    dataframe = add_all_ta_features(
        dataframe, open="open", high="high", low="low", close="close", volume="volume",
        fillna=True)
    return dataframe

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = list()
    for i in range(self.dna_size(self.buy_params)):        # 每组 DNA 是一个条件
        OPR = self.buy_params[f'buy-oper-{i}']
        IND = self.buy_params[f'buy-indicator-{i}']
        CRS = self.buy_params[f'buy-cross-{i}']
        INT = self.buy_params[f'buy-int-{i}']
        REAL = self.buy_params[f'buy-real-{i}']
        DFIND = dataframe[IND]; DFCRS = dataframe[CRS]
        if   OPR == ">":  conditions.append(DFIND > DFCRS)
        elif OPR == "=":  conditions.append(np.isclose(DFIND, DFCRS))
        elif OPR == "<":  conditions.append(DFIND < DFCRS)
        elif OPR == "CA": conditions.append(qtpylib.crossed_above(DFIND, DFCRS))
        elif OPR == "CB": conditions.append(qtpylib.crossed_below(DFIND, DFCRS))
        elif OPR == ">R": conditions.append(DFIND > REAL)
        elif OPR == "=R": conditions.append(np.isclose(DFIND, REAL))
        elif OPR == "<R": conditions.append(DFIND < REAL)
        # ... 还有 >I / =I / <I 等
    dataframe.loc[reduce(lambda x, y: x & y, conditions), 'enter_long'] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy-indicator-0` | trend_ichimoku_base | 买向参与比较的指标列名（来自 `ta` 全特征库） |
| `buy-cross-0` | volatility_kcc | 买向对照列名（交叉/比较对象） |
| `buy-oper-0` | `<R` | 买向操作符：指标 < 常数 |
| `buy-int-0` | 42 | 买向整数常数（供 `>I/=I/<I` 使用） |
| `buy-real-0` | 0.06295 | 买向实数常数（供 `>R/=R/<R` 使用） |
| `sell-indicator-0` | trend_kst_diff | 卖向指标列名 |
| `sell-cross-0` | volume_mfi | 卖向对照列名 |
| `sell-oper-0` | `=R` | 卖向操作符：指标 ≈ 常数 |
| `sell-int-0` | 98 | 卖向整数常数 |
| `sell-real-0` | 0.8779 | 卖向实数常数 |
| `stoploss` | -0.34549 | 固定止损 34.5% |
| `timeframe` | 12h | K 线周期 |
| `minimal_roi` | 0.3556 / 0.21275 / 0.09024 / 0 | 阶梯止盈表 |

> 这些键名不是固定参数，而是 hyperopt 的搜索维度——真正的「参数」是「选哪个指标、用哪个操作符、配哪个常数」的无限组合。

## 风险与局限

- **不搜索就没法用（重度依赖 hyperopt）**：指标名、对照列、操作符、常数全是待搜索维度，默认值只是一次搜索的落点，不是设计意图；不开 hyperopt 直接用等于用一组随机规则交易。
- **默认规则已退化、几乎不触发**：卖向默认 `oper = "=R"`，即 `np.isclose(trend_kst_diff, 0.8779)`——用一个浮点指标去近似等于某个精确小数，现实中几乎永不成立，平仓信号形同虚设；买向 `<R`（`trend_ichimoku_base < 0.06295`）也缺乏经济含义。
- **高维过拟合风险极高**：上百个特征 × 多个操作符 × 连续/离散常数，搜索空间巨大，极易搜到只在历史里好看、样本外失效的组合。文件注释里的最佳结果是 In-Sample 数字，不能当业绩。
- **`populate_indicators` 里 `dropna` 删行**：先 `dropna(dataframe)` 再补特征，会让返回的 dataframe **行数少于输入**，偏离 freqtrade「指标只增列不删行」的约定，可能引起行对齐异常；配合上百列特征，计算开销也很大。
- **代码含 `print` 副作用**：类体在**导入时会打印** AgeFilter 配置提示，运行 `populate_entry_trend` 时还会 `print(conditions)`，生产环境里属噪声、会影响性能与日志。
- **额外依赖**：必须 `pip install ta`，且作者要求配 `AgeFilter(min_days_listed: 30)`；不满足前置条件会直接报错或结果失真。
- **未回测 / 需要自行验证**：源码注释只有 In-Sample hyperopt 输出，没有可信的样本外业绩，本页不给具体收益数字。
- **维护状态**：源码无维护信息；属个人贡献（社区另有 GodStraQuick / GodStraNew 等多个衍生版本），原作者是否持续维护无法确认。
- **未来函数**：`add_all_ta_features` 生成的特征以及 `crossed_above/below` 均为因果计算，**未发现未来函数**；唯一需警惕的是上面的 `dropna` 删行带来的行对齐问题。
