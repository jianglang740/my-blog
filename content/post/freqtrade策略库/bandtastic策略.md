---
title: "Bandtastic 策略"
description: "布林带多档逆势、可加 RSI/MFI 守卫的均值回归策略"
date: 2026-10-09
lastmod: 2026-10-16
weight: 17
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# Bandtastic 策略

> **策略文件**：`Bandtastic.py`（163 行）
> **来源**：第三方作者。原作者 Robert Roman（邮箱 `robertroman7@gmail.com`）。源码头部声明 **`__license__ = "MIT"`**，version 1.0，copyright "Free For Use"，并附 BTC 捐赠地址。

## 策略思路

Bandtastic 是**布林带逆势**策略，跑在 15m 周期、只做多。它把布林带拆成 1 / 2 / 3 / 4 倍标准差共四档，触发信号是「收盘价跌破/突破某一档的轨道」：买向触发为 `close < bb_lowerband{1..4}`，卖向触发为 `close > bb_upperband{1..4}`，具体用哪一档由 `buy_trigger` / `sell_trigger` 两个 categorical 参数决定。在三类可选的「守卫（guard）」——RSI 阈值、MFI 阈值、快慢 EMA 方向——之上再叠加触发条件。守卫各自有 `_enabled` 开关（categorical 的 True/False），默认买向三个守卫全关、卖向只开 MFI，因此**默认行为接近于「纯粹的布林带低买高卖」**。所有阈值、EMA 周期都是可 hyperopt 的 `IntParameter`。

## 适用场景

设计意图是**震荡市里的逆势高抛低吸**：价格偏离布林带下轨就买、上轨就卖。适合波动有均值回复特征、区间明确的品种与周期。**不适合单边趋势市**：单边下跌时收盘价会持续贴着下轨，策略不断逆势买入（接飞刀）；单边上涨则会反复触发上轨卖出、过早离场。守卫（RSI/MFI/EMA）本意是缓解这个问题，但默认几乎没开。周期按 15m 设计，且需要较长预热（见风险）。

## 核心代码

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['mfi'] = ta.MFI(dataframe)
    # 布林带 1/2/3/4 倍标准差（基于 typical price，窗口 20）
    bollinger1 = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=1)
    dataframe['bb_lowerband1'] = bollinger1['lower']
    dataframe['bb_upperband1'] = bollinger1['upper']
    # ... bollinger2/3/4 同理
    for period in set(list(self.buy_fastema.range) + list(self.buy_slowema.range)
                      + list(self.sell_fastema.range) + list(self.sell_slowema.range)):
        dataframe[f'EMA_{period}'] = ta.EMA(dataframe, timeperiod=period)
    return dataframe

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    conditions = []
    if self.buy_rsi_enabled.value:                       # 可选守卫
        conditions.append(dataframe['rsi'] < self.buy_rsi.value)
    if self.buy_mfi_enabled.value:
        conditions.append(dataframe['mfi'] < self.buy_mfi.value)
    if self.buy_ema_enabled.value:
        conditions.append(dataframe[f'EMA_{self.buy_fastema.value}'] > dataframe[f'EMA_{self.buy_slowema.value}'])
    if self.buy_trigger.value == 'bb_lower1':            # 触发：跌破某一档下轨
        conditions.append(dataframe["close"] < dataframe['bb_lowerband1'])
    # ... bb_lower2/3/4 同理
    conditions.append(dataframe['volume'] > 0)           # 成交量护栏
    if conditions:
        dataframe.loc[reduce(lambda x, y: x & y, conditions), 'enter_long'] = 1
    return dataframe
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy_trigger` | bb_lower1 | 买向触发档位（bb_lower1/2/3/4） |
| `buy_rsi` / `buy_mfi` | 52 / 30 | 买向 RSI / MFI 阈值 |
| `buy_fastema` / `buy_slowema` | 211 / 250 | 买向快/慢 EMA 周期（范围 1–236 / 1–250） |
| `buy_rsi_enabled` / `buy_mfi_enabled` / `buy_ema_enabled` | False / False / False | 买向守卫开关（默认全关） |
| `sell_trigger` | sell-bb_upper2 | 卖向触发档位 |
| `sell_rsi` / `sell_mfi` | 57 / 46 | 卖向 RSI / MFI 阈值 |
| `sell_fastema` / `sell_slowema` | 7 / 6 | 卖向快/慢 EMA 周期（范围 1–365） |
| `sell_rsi_enabled` / `sell_mfi_enabled` / `sell_ema_enabled` | False / True / False | 卖向守卫开关（默认仅 MFI 开） |
| `stoploss` | -0.345 | 固定止损 34.5% |
| `minimal_roi` | 0.162 / 0.097 / 0.061 / 0 | 阶梯止盈表 |
| `startup_candle_count` | 999 | 预热 K 线根数（很大） |
| `timeframe` | 15m | K 线周期 |

## 风险与局限

- **源码注释里的「业绩」是 1 年样本内 hyperopt 结果**：文件头写着「Optimized With Sharpe Ratio and 1 year data」并列出 `199/40000` 迭代、约 3 万笔交易、Total profit 119.93% 等数字——这是**在 1 年数据上跑 4 万次搜索得到的 In-Sample 结果，不是样本外业绩**。1 年数据里 3 万笔交易本身就强烈提示过拟合，绝不应当作可复现收益。
- **参数空间巨大、过拟合风险高**：4 个 EM 周期上限高达 250–365，加上触发档位、阈值、三个守卫开关，搜索维度很多；默认值都是 hyperopt 落点，换品种/周期需重新验证。
- **默认守卫几乎全关**：买向三个守卫默认 False，卖向仅 MFI 开，所以默认策略就是「跌破布林带下轨就买、突破上轨就卖」的裸逆势逻辑，趋势市里风险敞口没有任何保护。
- **逆势接飞刀**：单边下跌中持续买入、单边上涨中过早卖出（只做多、无做空）；-34.5% 止损很深，单边行情回撤可观。
- **预热很长**：`startup_candle_count = 999`，需要大量历史数据才能进入可交易状态，短样本回测会被截掉一大段。
- **未回测 / 需要自行验证**：除上文那段样本内 hyperopt 注释外，源码没有样本外收益/回撤数据，本页不提供可信业绩数字。
- **维护状态**：源码无维护信息，作者仅留邮箱与捐赠地址；该策略在社区合集里被标记为「修改版」并见归档迹象，原作者是否持续维护无法确认。
- **未来函数**：布林带、RSI、MFI、EMA 皆为因果指标，信号只依赖当根及更早数据，**未发现未来函数**。
