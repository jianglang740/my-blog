---
title: "Supertrend 策略"
description: "三组 Supertrend 共振确认的趋势跟随策略"
date: 2026-10-02
lastmod: 2026-10-16
weight: 13
categories:
    - Quant
tags:
    - 量化投资
    - freqtrade
    - 策略
---

# Supertrend 策略

> **策略文件**：`Supertrend.py`（177 行）
> **来源**：第三方作者。原作者 @juankysoriano（Juan Carlos Soriano），GitHub: `https://github.com/juankysoriano/`；该文件亦收录于社区仓库 `freqtrade/freqtrade-strategies`。**源码未声明许可证**。

## 策略思路

策略一次性计算六条 Supertrend 指标——三条用于「买」、三条用于「卖」，每条由两个参数决定：ATR 倍数 `multiplier`（1–7）和周期 `period`（7–21）。开仓条件是三条「买」Supertrend 在同一根 K 线上**全部**判定为 `up`，并且该根 K 线 `volume > 0`；平仓条件是三条「卖」Supertrend 全部为 `down` 且 `volume > 0`。也就是说，它用多组不同敏感度的 Supertrend 做一次「趋势共振」确认，而不是单条指标翻多就进场。指标本身是作者按社区讨论（freqtrade-strategies issue #30）自行实现的，ATR 用 `SMA(TR)` 近似而非 Wilder 平滑。

## 适用场景

适合**方向明确、单边推进的趋势市**，官方示例周期为 1h、只做多，主要面向加密主流币种。三条指标共振能过滤掉一部分单条指标的假信号，因此比单 Supertrend 更能避开横盘里的反复翻面。**不适合震荡市**：Supertrend 本质是趋势跟随，价格在区间里来回穿越时，共振条件被打破又会重建，容易高买低卖、反复被止损。也不适合极小周期——噪声会显著放大。

## 核心代码

```python
# populate_entry_trend：三条「买」Supertrend 全部 up + 有成交量
dataframe.loc[
    (
       (dataframe[f'supertrend_1_buy_{self.buy_m1.value}_{self.buy_p1.value}'] == 'up') &
       (dataframe[f'supertrend_2_buy_{self.buy_m2.value}_{self.buy_p2.value}'] == 'up') &
       (dataframe[f'supertrend_3_buy_{self.buy_m3.value}_{self.buy_p3.value}'] == 'up') &
       (dataframe['volume'] > 0)
    ),
    'enter_long'] = 1

# supertrend() 关键部分：ATR 用 SMA(TR) 近似，逐根循环推导最终上下轨
df['TR'] = ta.TRANGE(df)
df['ATR'] = ta.SMA(df['TR'], period)
df['basic_ub'] = (df['high'] + df['low']) / 2 + multiplier * df['ATR']
df['basic_lb'] = (df['high'] + df['low']) / 2 - multiplier * df['ATR']
for i in range(period, len(df)):
    df['final_ub'].iat[i] = df['basic_ub'].iat[i] if df['basic_ub'].iat[i] < df['final_ub'].iat[i - 1] \
        or df['close'].iat[i - 1] > df['final_ub'].iat[i - 1] else df['final_ub'].iat[i - 1]
    df['final_lb'].iat[i] = df['basic_lb'].iat[i] if df['basic_lb'].iat[i] > df['final_lb'].iat[i - 1] \
        or df['close'].iat[i - 1] < df['final_lb'].iat[i - 1] else df['final_lb'].iat[i - 1]
# 方向标记：收盘价在 Supertrend 之上为 up，反之为 down
df[stx] = np.where((df[st] > 0.00), np.where((df['close'] < df[st]), 'down', 'up'), np.NaN)
```

## 关键参数

| 参数 | 默认值 | 说明 |
| :--- | :--- | :--- |
| `buy_m1` / `buy_m2` / `buy_m3` | 4 / 7 / 1 | 三条「买」Supertrend 的 ATR 倍数（范围 1–7） |
| `buy_p1` / `buy_p2` / `buy_p3` | 8 / 9 / 8 | 三条「买」Supertrend 的周期（范围 7–21） |
| `sell_m1` / `sell_m2` / `sell_m3` | 1 / 3 / 6 | 三条「卖」Supertrend 的 ATR 倍数 |
| `sell_p1` / `sell_p2` / `sell_p3` | 16 / 18 / 18 | 三条「卖」Supertrend 的周期 |
| `stoploss` | -0.265 | 固定止损 26.5% |
| `minimal_roi` | 0.087 / 0.058 / 0.029 / 0 | 阶梯止盈表 |
| `trailing_stop` | True | 回撤止盈开启，止盈 5%、触发偏移 14.4% |
| `timeframe` | 1h | K 线周期 |
| `startup_candle_count` | 199 | 预热 K 线根数 |

## 风险与局限

- **源码自带的警告必须读**：文件头明确写着该 Supertrend 实现「未经验证」（not validated），不保证能复现原始论文或任何可信学术资料的结论，并声明「use at your own risk」。ATR 用 `SMA(TR)` 而非标准 Wilder ATR，数值与主流平台/文献的 Supertrend 不会完全一致。
- **参数是 hyperopt 生成、存在样本内过拟合**：代码注释说明 `buy_params` / `sell_params` / ROI / 止损均来自一次 `freqtrade hyperopt ... --timerange=20210101- --timeframe=1h --spaces all` 搜索，属于**样本内**结果，不能当作样本外业绩。
- **未回测 / 需要自行验证**：源码注释中没有给出可复现的收益率、胜率或最大回撤，本页不提供此类数字。
- **只做多、无做空**：下跌行情中只能空仓，收益完全依赖多头趋势。
- **参数空间耦合**：六个 ATR 倍数 + 六个周期共 12 个可调参数，共振条件对参数较敏感，若全量 hyperopt 很容易拟合出只在历史里好看的组合。
- **维护状态**：源码本身没有维护信息；该文件是社区贡献，原作者是否持续维护无法从代码确认。
- **未来函数**：`supertrend()` 用 `for i in range(period, len(df))` 逐根推导，第 i 根只用到第 i 与 i-1 根数据，方向标记也基于当根收盘价，属于因果计算，**未发现未来函数**。仅需注意文件末尾用 `df.fillna(0, inplace=True)` 把指标预热期的 `NaN` 改写成 `0`，`0` 不等于 `'up'`，不会误触发信号，但会掩盖「指标还没准备好」这一事实。
