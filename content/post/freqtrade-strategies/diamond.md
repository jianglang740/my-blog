---
title: "Diamond 策略"
description: "用 OHLCV 交叉、靠超参定义规则的空框架策略"
date: 2026-10-04
lastmod: 2026-10-16
weight: 14
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# Diamond 策略

> **策略文件**：`Diamond.py`（155 行）
> **来源**：第三方作者。原作者 @Mablue（Masoud Azizi），GitHub: `https://github.com/mablue/`；文件注释致谢 @Kroissan、@drakes00、@xmatthias。**源码未声明许可证**。

## 策略思路

Diamond 的设计理念是「纯」——它的 `populate_indicators` **不计算任何技术指标**，直接拿原始 OHLCV 列做计算（作者注释原话：designed to be pure）。规则的骨架是一次交叉：开多 = 快「键」向后位移若干根后**上穿**慢「键」乘以一个放大倍数；平多 = 快键位移后**下穿**慢键乘以放大倍数。所谓「键」不是固定的某个指标，而是用 `CategoricalParameter` 从 `open / high / low / close / volume` 里挑一列；位移（`horizontal_push`，0–10）和放大倍数（`vertical_push`，0.5–1.5）也都是可搜索参数。换句话说，这套策略本身几乎不含人工经验，把「用什么、错位多少、放大多少」全部交给 hyperopt。作者在注释里也明确说了：想用别的指标，就自己先 `populate` 好，再把列名加进 categorical 参数即可。

## 适用场景

它的适用性完全取决于 hyperopt 搜索出来的那组「键 + 位移 + 倍数」——没有固定的行情假设，谈不上「适合趋势市」还是「适合震荡市」。默认参数把慢键设成了 `volume`（买向）和 `low`（卖向），属于搜索结果而非理论推导，因此**换品种、换周期后大概率失效**。这个文件更适合被当作**模板 / 教学骨架**（怎么在纯 dataframe 上用 categorical 超参），而不是拿默认参数直接上实盘。**不该用的场景**：任何未经自己重新 hyperopt 与样本外验证的实盘；以及低流动性品种——策略没有任何成交量或流动性过滤。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 不计算任何指标；想加就自己 populate 并启用对应的 categorical 键
    return dataframe

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = []
    conditions.append(
        qtpylib.crossed_above(
            dataframe[self.buy_fast_key.value].shift(self.buy_horizontal_push.value),
            dataframe[self.buy_slow_key.value] * self.buy_vertical_push.value
        )
    )
    if conditions:
        dataframe.loc[reduce(lambda x, y: x & y, conditions), 'enter_long'] = 1
    return dataframe

def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = []
    conditions.append(
        qtpylib.crossed_below(
            dataframe[self.sell_fast_key.value].shift(self.sell_horizontal_push.value),
            dataframe[self.sell_slow_key.value] * self.sell_vertical_push.value
        )
    )
    if conditions:
        dataframe.loc[reduce(lambda x, y: x & y, conditions), 'exit_long'] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy_fast_key` | high | 买向快键，取 OHLCV 之一（可选 open/high/low/close/volume） |
| `buy_slow_key` | volume | 买向慢键，取 OHLCV 之一 |
| `buy_horizontal_push` | 7 | 快键向后位移根数（0–10） |
| `buy_vertical_push` | 0.942 | 慢键放大倍数（0.5–1.5） |
| `sell_fast_key` | high | 卖向快键 |
| `sell_slow_key` | low | 卖向慢键 |
| `sell_horizontal_push` | 10 | 卖向位移根数 |
| `sell_vertical_push` | 1.184 | 卖向放大倍数 |
| `stoploss` | -0.271 | 固定止损 27.1% |
| `minimal_roi` | 0.242 / 0.044 / 0.02 / 0 | 阶梯止盈表 |
| `timeframe` | 5m | K 线周期 |

## 风险与局限

- **强依赖 hyperopt，否则默认参数无意义**：快/慢键（categorical）、位移、倍数全部是搜索维度，不开 hyperopt 就得手工试参；默认值是某一次搜索结果，没有理论支撑。
- **注释里的「业绩」是样本内 hyperopt 结果**：文件头列了多组 `hyperopt --hyperopt-loss ...` 的输出（如某个 loss 下 297 trades、Total profit 约 33.96%），这些都是**在历史数据上搜出来的 In-Sample 数字，不是样本外业绩**，不可直接引用。真实表现需要自行下载数据、划分样本外区间验证。
- **无成交量 / 流动性过滤**：相较其它策略这里没有 `volume > 0` 之类的护栏，低流动性或数据异常的 K 线也可能触发信号。
- **可解释性弱**：交叉的两边是「OHLCV 任意列 × 任意倍数、任意位移」的组合，很难给出经济学解释，本质上是在拟合价量关系。
- **只做多、无做空**：默认只写 `enter_long` / `exit_long`。
- **维护状态**：源码无维护信息；作者后续有 GodStra 等其它版本，本文件属个人贡献，是否维护无法确认。
- **未来函数**：`shift(n)` 是**向后取历史值**（正数向过去），`crossed_above / crossed_below` 均为因果判定，**未发现未来函数**。
