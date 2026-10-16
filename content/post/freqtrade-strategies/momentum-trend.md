---
title: "动量趋势策略"
description: "多动量指标共振、只做多头的强趋势跟随策略"
date: 2026-09-14
lastmod: 2026-10-16
weight: 3
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# 动量趋势策略

> **策略文件**：`MomentumTrendStrategy.py`（468 行）
> **来源**：自己开发

## 策略思路

这个策略把市面上常见的动量指标几乎全用上了，跑在 1 小时 K 线上、只做多。开仓要求 21 个条件**同时成立**（全部用 `&` 连接）：`momentum_up`（RSI>50、ROC>0、MACD 金叉、MOM>0 四者共振）、RSI 在 55–80 之间、ROC 超过阈值、MACD 柱状图为正且环比增强、CCI>100、威廉指标、随机指标 K>50 且 K>D、EMA 短>长、价格在短 EMA 之上、ADX>25 且 DI+>DI-、成交量放大、VPT 与 RVI 信号为正、短期/中期价格动量为正，最后还要求一个自制的 `momentum_score`（把 RSI/ROC/MACD/CCI/威廉/随机加权合成 0–100）大于 65。平仓则是把上面这些指标转弱的条件用 `|` 全部罗列——**只要任意一条触发就平仓**。另有 `custom_stoploss` 按动量评分动态收紧止损、`custom_exit` 处理动量衰竭/反转/背离/缩量。

## 适用场景

适合**动能充沛、放量突破的强趋势市**，追求的是「确认趋势正在加速」后再进场，因此在趋势中后段的加速上涨里信号质量较高。由于开仓条件极多且必须全满足，信号**非常稀疏**，可能长时间不出手。**不适合震荡市和量能不足的行情**：没有成交量配合、ADX 上不去时几乎不会开仓；而一旦趋势掉头，平仓条件是宽泛的「或」逻辑，会很快离场（这既是保护也是缺点，容易在回踩中被过早震出）。

## 核心代码

```python
# populate_indicators：自制综合动量评分（0-100）
dataframe['momentum_score'] = (
    rsi_norm * 0.2 +
    roc_norm * 0.2 +
    macd_norm * 0.15 +
    cci_norm * 0.15 +
    williams_norm * 0.15 +
    stoch_norm * 0.15
)

# populate_entry_trend：21 个条件全部为真才开仓
dataframe.loc[
    (
        conditions[0] &   # 动量向上
        conditions[1] &   # RSI下限
        conditions[2] &   # RSI上限
        conditions[3] &   # ROC
        conditions[4] &   # MACD金叉
        # ... 中间省略 ...
        conditions[19] &  # 中期价格动量
        conditions[20]    # 综合动量评分
    ),
    'enter_long'
] = 1

# populate_exit_trend：条件之间全部用 | （任一触发即平仓）
dataframe.loc[
    (
        conditions[0] |   # 动量转弱
        conditions[1] |   # RSI超买
        # ... 中间省略 ...
        conditions[15]    # 动量评分下降
    ),
    'exit_long'
] = 1
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `rsi_period` / `rsi_buy_threshold` / `rsi_sell_threshold` | 14 / 55 / 75 | RSI 周期与买/卖阈值 |
| `roc_period` / `roc_threshold` | 12 / 2.5 | ROC 周期与开仓阈值 |
| `macd_fast` / `macd_slow` / `macd_signal` | 12 / 26 / 9 | MACD 参数 |
| `cci_period` / `cci_buy_threshold` / `cci_sell_threshold` | 20 / 100 / -100 | CCI 参数 |
| `williams_period` / `williams_buy_threshold` / `williams_sell_threshold` | 14 / -70 / -30 | 威廉指标参数 |
| `ema_short` / `ema_long` | 20 / 50 | 趋势确认均线 |
| `adx_period` / `adx_threshold` | 14 / 25 | ADX 周期与强度阈值 |
| `volume_factor` | 1.8 | 成交量放大倍数 |
| `atr_period` / `atr_multiplier` | 14 / 2.5 | ATR 与动态止损倍数 |
| `stoploss` | -0.07 | 固定止损 7% |
| `timeframe` | 1h | K 线周期 |

## 风险与局限

- **未回测 / 需要自行验证**：源码与注释中没有任何收益率、胜率或回撤数据，本页不提供此类数字。
- **开仓条件过多（21 个全 AND）**：任何一条不满足就完全不开仓，实际信号频率可能低到样本不足，回测统计意义弱；同时这么多阈值意味着极高的参数维度，过拟合风险很大。
- **平仓是宽泛 OR 逻辑**：十几个「转弱」条件只要命中一个就平仓，容易在趋势正常回踩时被过早止盈，吃到趋势的能力被削弱；配合 `custom_exit` 的多重退出，实盘退出原因会很零碎。
- **指标高度共线**：RSI、ROC、MACD、CCI、威廉、随机指标本质都在描述同一件事（价格动能），21 个条件里大部分信息是重复的，看似「多重确认」，实际并未真正增加独立信息量。
- **自制的 `momentum_score` 权重无依据**：6 个指标按 0.2/0.2/0.15… 手工加权、各自用不同的线性变换压到 0–100，这些系数和映射没有理论或回测支撑，属于主观设定，需要自行验证稳健性。
- **只做多、无杠杆**：`can_short = False`，空头行情只能空仓等待。
