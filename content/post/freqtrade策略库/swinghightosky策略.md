---
title: "SwingHighToSky 策略"
description: "CCI 与 RSI 双超卖抄底的均值回归策略"
date: 2026-10-07
lastmod: 2026-10-16
weight: 16
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# SwingHighToSky 策略

> **策略文件**：`SwingHighToSky.py`（110 行）
> **来源**：第三方作者。原作者 Kevin Ossenbrück（credits：Bloom Trading, Mohsen Hassan）。源码头部声明 **license = "MIT"**，version 1.0，status "Live"。

## 策略思路

这是一个典型的**超卖抄底均值回归**策略，跑在 15m 周期、只做多。开多需要两个条件同时满足：CCI（周期 `buy_cciTime`）低于阈值 `buy_cci`，且 RSI（周期 `buy_rsiTime`）低于阈值 `buy_rsi`——即「CCI 和 RSI 双双处于低位」。平多在相反的对称条件下触发：CCI 高于 `sell_cci` 且 RSI 高于 `sell_rsi`。指标周期和阈值全部由 `IntParameter` 暴露，可交给 hyperopt 优化。买入和卖出各用独立的 CCI、RSI（列名分别加了 `-sell-` 前缀），互不干扰。

## 适用场景

适合**区间震荡、且在区间下沿有支撑**的行情——目标是在超卖处进场、反弹后离场。**最不适合单边下跌**：均值回归在趋势里会不断「接下落的刀」，价格一路新低时 CCI/RSI 长期停在超卖区，策略会反复给出买入信号并被止损（默认止损深达 -34%）。同样不适合单边上涨（会过早平仓、错过趋势）。周期上按 15m 设计，换更小周期时超卖信号噪声更大、假信号更多。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    for val in self.buy_cciTime.range:                 # 按可调周期逐个生成 CCI 列
        dataframe[f'cci-{val}'] = ta.CCI(dataframe, timeperiod=val)
    for val in self.buy_rsiTime.range:                 # 逐个生成 RSI 列
        dataframe[f'rsi-{val}'] = ta.RSI(dataframe, timeperiod=val)
    # sell 侧同理生成 cci-sell-* / rsi-sell-*
    return dataframe

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe[f'cci-{self.buy_cciTime.value}'] < self.buy_cci.value) &   # CCI 超卖
            (dataframe[f'rsi-{self.buy_rsiTime.value}'] < self.buy_rsi.value)     # RSI 超卖
        ),
        'enter_long'] = 1
    return dataframe

def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe[f'cci-sell-{self.sell_cciTime.value}'] > self.sell_cci.value) &  # CCI 超买
            (dataframe[f'rsi-sell-{self.sell_rsiTime.value}'] > self.sell_rsi.value)    # RSI 超买
        ),
        'exit_long'] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy_cci` | -175 | 买向 CCI 阈值（参数声明范围 -200–200） |
| `buy_cciTime` | 72 | 买向 CCI 周期（范围 10–80） |
| `buy_rsi` | 90 | 买向 RSI 阈值（范围 10–90） |
| `buy_rsiTime` | 36 | 买向 RSI 周期（范围 10–80） |
| `sell_cci` | -106 | 卖向 CCI 阈值 |
| `sell_cciTime` | 66 | 卖向 CCI 周期 |
| `sell_rsi` | 88 | 卖向 RSI 阈值 |
| `sell_rsiTime` | 45 | 卖向 RSI 周期 |
| `stoploss` | -0.34338 | 固定止损 34.3% |
| `minimal_roi` | 0.27058 / 0.0853 / 0.04093 / 0 | 阶梯止盈表 |
| `timeframe` | 15m | K 线周期 |

## 风险与局限

- **默认条件近似退化为单条件**：买向 `RSI < 90` 在绝大多数时间里恒为真（RSI 本来就多在 30–70），实际开仓几乎只由 `CCI < -175` 决定；卖向 `RSI > 88` 同理几乎恒真，平仓实际由 `CCI > -106` 主导。所谓「双确认」名不副实。
- **阈值极端、疑似过度拟合**：CCI -175、RSI 90 这类极端阈值更像是为了在历史数据里凑信号而调出来的，样本外要么几乎不触发、要么一次性涌入大量信号。参数是 `IntParameter` 全开 `optimize=True`，搜索空间大、过拟合风险高。
- **死代码**：模块顶部定义了一组常量 `cciBuyTP / cciBuyVal / cciSellTP / cciSellVal / rsiBuyTP / rsiBuyVal / rsiSellTP / rsiSellVal`，但类里从头到尾没有引用它们，只是摆设，容易误导读者以为阈值来自这些常量。
- **均值回归的固有风险**：单边下跌中持续抄底、-34% 的深止损，单边行情里回撤会很可观；只做多、无做空。
- **未回测 / 需要自行验证**：源码注释没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **维护状态**：文件头部 `status = "Live"` 是作者自述，源码本身无法证明其仍在维护；原作者是否持续更新无法确认。
- **未来函数**：CCI、RSI 都是标准因果指标，信号只依赖当根及更早数据，**未发现未来函数**。
