---
title: "A股单因子回测框架拆解：从两份 parquet 到分层回测"
date: 2026-10-18
lastmod: 2026-10-18
draft: false
group: "engineering"
status: "done"
summary: "把一个 390 行的单因子回测框架拆成数据对齐、因子预处理、分层检验与绩效统计四段，并标出它在实盘化之前还缺什么。"
tags: ["多因子", "回测框架", "Python", "研究工程化"]
math: true
comments: false
license: false
---

这是我在做 A 股单因子研究时写的一个最小回测框架。它不接数据库、不做撮合模拟，只回答一个很具体的问题：

> 给定一份价格面板和一份已经算好的因子面板，这个因子在未来 1 / 5 / 10 / 20 个交易日的横截面上，有没有区分度？

框架刻意保持扁平：390 行、10 个函数、两份 parquet 输入。它适合快速验证一个因子想法，不适合直接拿来当实盘策略的收益证明。下面按数据流把代码拆开，最后单独讨论它的假设和偏差。

## 一、输入约定

框架默认读取两个文件：

```text
code/price_data.parquet   # 行情面板
code/fct_df.parquet       # 因子面板
```

两者都使用 `MultiIndex(trade_date, code)`，即每一行是「某只股票在某一天」的观测。行情面板至少包含：

| 字段 | 含义 | 在框架中的用途 |
| --- | --- | --- |
| `close_adj` | 复权收盘价 | 计算未来收益 |
| `is_st` | 是否 ST | 股票池过滤 |
| `trade_days` | 上市以来交易天数 | 过滤次新股 |

因子面板的每一列是一个因子，索引必须与行情面板一致。主程序用 `concat(..., axis=1, join='inner')` 合并两份数据，因此只有两边都存在的「日期 × 股票」才会进入回测。

这里有两个隐含前提，代码没有校验：

1. `close_adj` 必须是**复权**价格。如果传入未复权收盘价，除权除息日会被当成真实下跌，因子收益会被污染。
2. 因子列名不能与行情列重名，否则 `concat` 后会出现重复列，后续按列取值的行为不再唯一。

## 二、主流程与函数分工

整个框架可以画成一条直线：

```text
price_data ─┐
            ├─ inner join ─ MAD 去极值 ─ z-score ─ 股票池过滤
fct_df ─────┘                                      │
                                                   ▼
              factor_analysis ─ 未来收益 + 分层 ─ analyze_factor_performance
                                                   │
                                                   ▼
                              generate_summary_statistics / 两张图
```

对应到函数：

| 函数 | 作用 |
| --- | --- |
| `extreme_MAD` | 用中位数与 MAD 做去极值 |
| `standardize_z` | 对因子做 z-score |
| `preprocess_data` | 剔除 ST 与上市不足约一年的股票 |
| `factor_analysis` | 对齐未来收益，按日做分位数分组 |
| `analyze_factor_performance` | 计算分层累计收益、多空收益、IC 与 ICIR |
| `calculate_portfolio_metrics` | 计算年化、回撤、夏普、索提诺、卡玛 |
| `determine_factor_direction` | 根据多空收益符号判断因子方向 |
| `generate_summary_statistics` | 汇总并写出 Excel |
| `plot_quantile_returns_separate` / `plot_factor_performance` | 分层曲线与四联图 |

主程序最后只回测 `fct_1` 这一列。实际使用时，把 `price_data['fct_1']` 换成自己的因子列名即可。

## 三、收益对齐：刻意留出一天执行滞后

`factor_analysis` 先把 `close_adj` 透视为「日期 × 股票」的价格矩阵，然后：

```python
future_prices = price_aligned.shift(-period - 1)
current_prices = price_aligned.shift(-1)
future_returns = future_prices / current_prices - 1
```

设因子在 $t$ 日收盘后可得，$P_{i,t}$ 为复权收盘价，则第 $i$ 只股票的未来 $h$ 日收益是：

$$
r_{i,t}^{(h)} = \frac{P_{i,t+1+h}}{P_{i,t+1}} - 1
$$

也就是说，因子在 $t$ 日计算，$t+1$ 日收盘才建仓，持有 $h$ 个交易日。这多留出的一天是必要的：如果直接用 $P_{t+h}/P_t$，等于假设在 $t$ 日收盘就能完成交易，而因子本身往往就是用 $t$ 日收盘数据算出来的。

之后：

```python
future_returns = future_returns.iloc[::period].dropna(axis=0, how='all')
```

`iloc[::period]` 每隔 $h$ 行取一个调仓日，使同一持有期内的收益窗口不重叠。例如 $h=5$ 时，样本日约为第 0、5、10……行，对应区间 $[t+1, t+6]$、$[t+6, t+11]$……。这样做的好处是年化时可以直接按 $252/h$ 换算；代价是采样起点固定在数据第一行，换一份起始日期不同的数据，调仓日会整体平移。

## 四、因子预处理

### 4.1 MAD 去极值

```python
def extreme_MAD(dt, n=5.2):
    median = dt.quantile(0.5)
    new_median = (abs((dt - median)).quantile(0.5))
    dt_up = median + n*new_median
    dt_down = median - n*new_median
    return dt.clip(dt_down, dt_up, axis=1)
```

对每个因子列，先取中位数 $\mathrm{med}$，再取绝对偏差的中位数：

$$
\mathrm{MAD} = \mathrm{median}\left(\left|x - \mathrm{med}\right|\right)
$$

把超出 $[\mathrm{med} - 5.2\mathrm{MAD},\ \mathrm{med} + 5.2\mathrm{MAD}]$ 的值截断。$5.2$ 是常用的经验倍数；在正态分布下 $\mathrm{MAD} \approx 0.6745\sigma$，所以这个区间大约对应 $\pm 3.5\sigma$，比直接按 $3\sigma$ 截断更保守。

### 4.2 z-score 与缺失值

```python
price_data[factor_names] = standardize_z(price_data[factor_names].fillna(0))
```

标准化公式是：

$$
z_{i,t} = \frac{x_{i,t} - \bar{x}}{\sigma_x}
$$

这里有两个细节值得说清楚：

- `dt.quantile(0.5)`、`dt.mean()`、`dt.std()` 都**没有按 `trade_date` 分组**，算的是整张面板上每个因子列的统计量，而不是每日横截面统计量。
- `fillna(0)` 在标准化之前执行，缺失值被当成原始值 0 参与均值和标准差的计算，标准化后变成一个固定的伪观测。

对当前的**排序型**回测来说，第一点的影响没有看起来那么大：全局 z-score 是单调仿射变换，不改变同一日内股票的相对排序，因此 Spearman IC 和分位数分组通常不变。但如果后续要把因子送进回归、多因子合成或机器学习模型，全局统计量就会把不同日期的量纲混在一起，应该改成按日横截面处理：

```python
# 示意：按日做截面标准化
z = factor.groupby(level='trade_date').transform(
    lambda x: (x - x.mean()) / x.std()
)
```

第二点的影响更直接：缺失值被填成 0 之后，再减去全局均值，会得到一个与其他股票可比较的数值，相当于把「数据缺失」编码成了一种因子暴露。更稳妥的做法是保留 `NaN`，在分组或排序时排除，或者在截面内用中位数填充。

### 4.3 股票池过滤

```python
mask = (~df['is_st']) & (df['trade_days'] >= 365)
```

剔除 ST 股票和上市不足约一年的次新股，目的是避开退市风险高、价格限制特殊、以及上市初期波动异常的样本。这个过滤发生在去极值和标准化之后，所以 ST 与次新股仍然参与了全局中位数、MAD 和标准差的估计。样本量很大时影响有限，但严格来说，预处理和股票池的顺序应该反过来。

## 五、分层与 IC

### 5.1 分位数组合

把因子面板 `reset_index` 成 `trade_date / code / factor` 三列，和未来收益按日期、代码合并，再按日分组切分：

```python
merged_df['quantile'] = merged_df.groupby('trade_date')['factor'].transform(
    lambda x: pd.qcut(x, quantiles, labels=False, duplicates='drop') + 1
)
```

默认切 10 组，`Q1` 是因子值最小的一组，`Q10` 是最大的一组。每个日期、每个分组的收益取组内等权平均，然后按调仓日复利：

$$
R_q = \prod_{t \in T}\left(1 + \bar{r}_{q,t}^{(h)}\right) - 1
$$

`analyze_factor_performance` 里的多空收益是两组**累计收益之差**：

```python
long_short_return = quantile_returns.iloc[-1] - quantile_returns.iloc[0]
```

即 `Q10 累计收益 - Q1 累计收益`。它不是每日再平衡的多空组合净值，而是一个便于快速比较的近似值；两者在波动较大时会有差异。

### 5.2 IC 与 ICIR

对每个调仓日，计算因子值与未来收益的 Spearman 秩相关：

$$
IC_t = \mathrm{Spearman}\left(f_{i,t},\ r_{i,t}^{(h)}\right)
$$

代码要求当日至少有 5 只股票才计算，然后取时间序列均值与信息比率：

$$
\overline{IC} = \frac{1}{T}\sum_t IC_t,\qquad
ICIR = \frac{\overline{IC}}{\mathrm{std}(IC_t)}
$$

这里的 ICIR 没有做年化，只是「均值 / 标准差」，适合在同一套采样频率下横向比较因子，不适合直接拿去和年化 ICIR 的口径对比。

### 5.3 因子方向

`determine_factor_direction` 看多空收益的符号：大于 0 认为是正向因子，取 `Q10` 计算绩效；小于 0 认为是负向因子，取 `Q1`。这一步用到了**全样本**的多空结果，等于先看了答案再决定持有哪一端，存在明显的事后选择偏差。更严谨的做法是：

- 同时报告 `Q1` 和 `Q10` 两端的绩效；
- 或者只用训练集确定方向，再在样本外验证；
- 或者直接以 IC 的符号作为方向，并报告 IC 的显著性。

## 六、绩效指标

`calculate_portfolio_metrics` 接收目标组的逐期收益序列，计算：

| 指标 | 代码口径 |
| --- | --- |
| 累计收益率 | $\prod_t(1+r_t)-1$ |
| 年化收益率 | $(1+R)^{252/(N\cdot h)}-1$ |
| 最大回撤 | 净值从历史高点的最大跌幅 |
| 年化波动率 | $\mathrm{std}(r_t)\sqrt{252/h}$ |
| 夏普比率 | 年化收益率 / 年化波动率 |
| 索提诺比率 | 年化收益率 / 下行波动率 |
| 卡玛比率 | 年化收益率 / 最大回撤 |

其中 $N$ 是调仓期数，$h$ 是持有天数。因为收益窗口不重叠，$N\cdot h$ 就是实际覆盖的交易天数。

有三点需要注意：

1. 夏普比率没有减无风险利率，且分子用的是几何年化收益，不是「平均超额收益 / 波动率」，与教科书口径略有差异。
2. 索提诺比率的下行波动用的是负收益样本的标准差，而不是相对目标收益的下行偏差，是一个简化版本。
3. 最大回撤在汇总表里以负数输出（代码返回 `-max_drawdown`），阅读时注意符号。

## 七、输出

`generate_summary_statistics` 会写出一个 Excel，包含三张表：

- `汇总统计`：每个持有期的方向、IC、ICIR、累计收益、年化、最大回撤、夏普；
- `分位数收益`：每个持有期 Q1–Q10 的区间累计收益；
- `IC时间序列`：每个持有期逐日的 IC 值。

绘图函数输出两类图：

- `plot_factor_performance`：多空收益、IC、分位数收益、IC 时间序列的四联图；
- `plot_quantile_returns_separate`：每个持有期一张 Q1–Q10 的累计净值曲线，Q1 和 Q10 加粗。

## 八、这套框架的边界

这套代码的定位是「快速筛因子」，不是「证明策略能赚钱」。下面这些问题在把它用于更严肃的结论之前必须处理。

### 8.1 收益是毛收益

没有手续费、印花税、滑点和冲击成本，也没有换手率统计。分位数组合每期等权重建，换手可能很高；对高换手因子，毛收益和净收益可能差一个量级。

### 8.2 没有可交易性过滤

停牌、涨跌停、流动性不足的股票仍然可能进入组合。框架假设所有股票都能在调仓日按收盘价成交。若行情数据对停牌做了前向填充，停牌期间的收益会被记成 0，结果偏乐观。

### 8.3 退市与缺失值

未来收益为 `NaN` 的样本会被直接丢弃。如果退市股票的末段收益缺失，组合实际上只保留了「活下来」的样本，存在幸存者偏差。需要根据数据源确认退市收益是否被正确纳入。另外，目标组收益序列 `dropna()` 之后，年化按剩余期数折算；如果缺失并非均匀分布，年化口径会与真实覆盖区间不一致。

### 8.4 没有行业与市值中性化

因子可能只是行业或市值的代理。框架没有做行业中性、市值中性，也没有在分层前回归掉风格暴露。对于一个新因子，至少应该检查它的行业分布和市值相关性。

### 8.5 方向选择与样本内检验

如前所述，方向由全样本多空收益决定，目标组的绩效也是在同一段样本上计算的。这不是严格的样本外检验。要做更可信的结论，需要切分训练/验证区间，或者做滚动窗口。

### 8.6 预处理不是截面处理

全局 MAD 和全局 z-score 不改变同日排序，因此对当前的 IC / 分层结果影响有限；但它们不是截面统计量，不能直接复用到回归或机器学习流程。`fillna(0)` 则会改变缺失样本的相对位置，需要单独处理。

### 8.7 采样与分组的技术细节

- `iloc[::period]` 的调仓日依赖数据起始行，换一份数据可能得到不同的采样网格；
- `pd.qcut(..., duplicates='drop')` 在某日因子值大量相同时可能只剩少数分组，绘图函数默认按 10 组处理，极端情况下会缺组；
- `warnings.filterwarnings('ignore')` 会连 pandas 的 future warning 和数据对齐警告一起吞掉，不利于排查问题；
- `plt.show()` 与 `SimHei` 字体适合本地交互，不适合无图形界面的服务器；工程化时应改为 `Agg` 后端加 `savefig`。

### 8.8 缺少显著性检验

框架报告 IC 均值和 ICIR，但没有 IC 的 t 统计量、Newey-West 调整或 Bootstrap 区间。因子数量一多，还需要考虑多重检验问题。

## 九、继续工程化的顺序

如果要把这套代码从「研究脚本」推进到「可复用的研究框架」，我会按下面的顺序改：

1. **配置化**：把文件路径、持有期、分组数、因子列名收进 `dataclass` 或 `argparse`，并在输出里记录运行参数、数据区间和样本量。
2. **截面预处理**：按 `trade_date` 做去极值、标准化和缺失值处理，避免全局统计量。
3. **可交易性与成本**：加入停牌、涨跌停、成交额过滤，按换手率扣减成本，输出净值与换手率两条曲线。
4. **样本外检验**：训练/验证切分或滚动回测，方向只在训练集确定。
5. **中性化**：行业与市值中性化，或者至少输出因子的行业/市值暴露。
6. **统计检验**：IC 的 t 值、Newey-West、分层收益的显著性。
7. **可复现产物**：固定随机种子（如有）、输出版本化的结果目录，图保存为文件而不是 `show()`。
8. **单元测试**：至少覆盖「未来收益没有用到当日之后不可得的数据」「日期与代码对齐正确」「分组边界正确」三件事。

## 十、如何运行

依赖：

```bash
pip install pandas numpy scipy matplotlib openpyxl pyarrow
```

把两份数据放到 `code/` 下，修改主程序里的因子列名，然后：

```bash
python fct_backtest无冗余版.py
```

默认会测试 1、5、10、20 日四个持有期，切 10 组，并把汇总写到 `code/因子回测统计结果.xlsx`。

## 附：完整源码

```python
# -*- coding: utf-8 -*-
"""
Created on Tue Dec  2 13:40:27 2025

@author: andrew
"""

###############################################################################
#### import ####

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats
import warnings
warnings.filterwarnings('ignore')

plt.rcParams['font.sans-serif'] = ['SimHei']
plt.rcParams['axes.unicode_minus'] = False


###############################################################################
#### func ####
def extreme_MAD(dt, n=5.2):
    median = dt.quantile(0.5)
    new_median = (abs((dt - median)).quantile(0.5))
    dt_up = median + n*new_median
    dt_down = median - n*new_median
    return dt.clip(dt_down, dt_up, axis=1)

def standardize_z(dt):
    mean = dt.mean()
    std = dt.std()
    return (dt - mean)/std

def preprocess_data(data):
    df = data.copy()
    mask = (~df['is_st']) & (df['trade_days'] >= 365)
    df = df[mask].copy()
    df.drop(['is_st','trade_days'], axis=1, inplace=True)
    return df

def factor_analysis(factor_series, price_df, periods=(1, 5, 10), quantiles=10):
    results = {}
    price_aligned = pd.pivot_table(price_df['close_adj'].reset_index(),index='trade_date', columns='code', values='close_adj')
    
    for period in periods:
        future_prices = price_aligned.shift(-period-1)
        current_prices = price_aligned.shift(-1)
        future_returns = (future_prices / current_prices - 1)
        factor_df = factor_series.reset_index()
        factor_df.columns = ['trade_date', 'code', 'factor']
        
        future_returns = future_returns.iloc[::period].dropna(axis=0,how='all')
        
        returns_series = future_returns.stack()
        returns_series.name = f'return_{period}d'
        merged_df = pd.merge(factor_df, returns_series, 
                           left_on=['trade_date', 'code'], 
                           right_index=True)
        
        merged_df = merged_df.dropna(subset=['factor', f'return_{period}d'])
        merged_df = merged_df.sort_values('trade_date')
        
        if len(merged_df) > 0:
            merged_df['quantile'] = merged_df.groupby('trade_date')['factor'].transform(
                lambda x: pd.qcut(x, quantiles, labels=False, duplicates='drop') + 1
            )
            results[period] = merged_df
    
    return results

def analyze_factor_performance(res_):
    perform_ = {}
    
    for period, df in res_.items():
        quantile_returns = df.groupby(['trade_date','quantile'])[f'return_{period}d'].mean()
        quantile_returns = quantile_returns.reset_index()
        quantile_returns = pd.pivot_table(quantile_returns,index='trade_date',columns='quantile',values=f'return_{period}d')
        quantile_returns = quantile_returns.add(1).cumprod().sub(1).iloc[-1]
        
        long_short_return = quantile_returns.iloc[-1] - quantile_returns.iloc[0]
        
        ic_series = df.groupby('trade_date').apply(
            lambda x: stats.spearmanr(x['factor'], x[f'return_{period}d']).correlation 
            if len(x) > 5 else np.nan
        ).dropna()
        
        if len(ic_series) > 0:
            ic = ic_series.mean()
            ic_ir = ic_series.mean() / ic_series.std() if ic_series.std() > 0 else 0
            
            perform_[period] = {
                'quantile_returns': quantile_returns,
                'long_short_return': long_short_return,
                'ic': ic,
                'ic_ir': ic_ir,
                'ic_series': ic_series,
                'data': df
            }
    
    return perform_

def calculate_portfolio_metrics(returns_series, period_days=1):
    if len(returns_series) == 0:
        return {}
    
    cumulative_curve = (1 + returns_series).cumprod()
    cumulative_return = cumulative_curve.iloc[-1] - 1 if len(cumulative_curve) > 0 else 0
    
    if len(returns_series) > 0:
        total_days = len(returns_series) * period_days
        annual_return = (1 + cumulative_return) ** (252 / total_days) - 1
    else:
        annual_return = 0
    
    running_max = cumulative_curve.expanding().max()
    drawdown = (cumulative_curve - running_max) / running_max
    max_drawdown = abs(drawdown.min())
    
    if len(returns_series) > 1:
        annual_volatility = returns_series.std() * np.sqrt(252 / period_days)
    else:
        annual_volatility = 0
    
    sharpe_ratio = annual_return / annual_volatility if annual_volatility > 0 else 0
    
    downside_returns = returns_series[returns_series < 0]
    if len(downside_returns) > 1:
        downside_volatility = downside_returns.std() * np.sqrt(252 / period_days)
    else:
        downside_volatility = 0
    sortino_ratio = annual_return / downside_volatility if downside_volatility > 0 else 0
    
    calmar_ratio = annual_return / max_drawdown if max_drawdown > 0 else 0
    
    return {
        '累计收益率': cumulative_return,
        '年化收益率': annual_return,
        '最大回撤': -max_drawdown,
        '夏普比率': sharpe_ratio,
        '索提诺比率': sortino_ratio,
        '卡玛比率': calmar_ratio,
       
    }

def determine_factor_direction(perfo_):
    direction_info = {}
    
    for period, perf in perfo_.items():
        long_short_return = perf['long_short_return']
        if long_short_return > 0:
            direction = '正向因子'
            target_group = 'Top组(Q10)'
            target_quantile = 10
        else:
            direction = '负向因子'
            target_group = 'Bottom组(Q1)'
            target_quantile = 1
        
        direction_info[period] = {
            'direction': direction,
            'target_group': target_group,
            'target_quantile': target_quantile,
            'long_short_return': long_short_return
        }
    
    return direction_info

def generate_summary_statistics(perform_, output_path='因子回测统计结果.xlsx'):
    direction_info = determine_factor_direction(perform_)    
    summary_data = []
    
    for period in perform_.keys():
        df = perform_[period]['data']
        direction_data = direction_info[period]        
        daily_portfolio_returns = df.groupby(['trade_date', 'quantile'])[f'return_{period}d'].mean().reset_index()
        returns_pivot = daily_portfolio_returns.pivot(index='trade_date', columns='quantile', values=f'return_{period}d')
        returns_pivot = returns_pivot.sort_index()
        target_quantile = direction_data['target_quantile']
        
        if target_quantile in returns_pivot.columns:
            target_returns = returns_pivot[target_quantile].dropna()
        else:
            print(f"警告: 持有期{period}天没有找到分位数{target_quantile}")
            continue
        
        metrics = calculate_portfolio_metrics(target_returns, period)
        
        period_summary = {
            '持有期(天)': period,
            '因子方向': direction_data['direction'],
            'IC': perform_[period]['ic'],
            'ICIR': perform_[period]['ic_ir'],
            '累计收益率': metrics['累计收益率'],
            '年化收益率': metrics['年化收益率'],
            '最大回撤': metrics['最大回撤'],
            '夏普比率': metrics['夏普比率'],
        }
        
        summary_data.append(period_summary)
    
    if not summary_data:
        return pd.DataFrame()
        
    summary_df = pd.DataFrame(summary_data)
    
    try:
        with pd.ExcelWriter(output_path, engine='openpyxl') as writer:
            summary_df.to_excel(writer, sheet_name='汇总统计', index=False)
            
            quantile_returns_data = []
            for period in perform_.keys():
                quantile_returns = perform_[period]['quantile_returns']
                row = {'持有期': period}
                for q, ret in quantile_returns.items():
                    row[f'Q{q}'] = ret
                quantile_returns_data.append(row)
            
            quantile_df = pd.DataFrame(quantile_returns_data)
            quantile_df.to_excel(writer, sheet_name='分位数收益', index=False)
            
            ic_data = []
            for period in perform_.keys():
                if 'ic_series' in perform_[period]:
                    ic_series = perform_[period]['ic_series']
                    for date, ic_value in ic_series.items():
                        ic_data.append({
                            '持有期': period,
                            '日期': date,
                            'IC值': ic_value
                        })
            
            if ic_data:
                ic_df = pd.DataFrame(ic_data)
                ic_df.to_excel(writer, sheet_name='IC时间序列', index=False)
        
        print(f"统计结果已保存到: {output_path}")
    except Exception as e:
        print(f"保存Excel文件时出错: {e}")
    
    return summary_df

def plot_quantile_returns_separate(perform_):
    periods = list(perform_.keys())
    
    for period in periods:
        df = perform_[period]['data']
        daily_portfolio_returns = df.groupby(['trade_date', 'quantile'])[f'return_{period}d'].mean().reset_index()
        returns_pivot = daily_portfolio_returns.pivot(index='trade_date', columns='quantile', values=f'return_{period}d')
        returns_pivot = returns_pivot.sort_index()
        cumulative_returns = returns_pivot.add(1).cumprod()
        
        plt.figure(figsize=(14, 8))
        
        colors = {}
        colors[10] = '#FF3333'  
        colors[1] = '#3333FF'   
        
        middle_group_colors = [
            '#FF6B6B', '#4ECDC4', '#FFD166', '#06D6A0',
            '#118AB2', '#EF476F', '#7B68EE', '#20B2AA'
        ]
        
        for idx, q in enumerate(range(2, 10)):
            if q in cumulative_returns.columns:
                colors[q] = middle_group_colors[idx]
        
        quantile_order = sorted(cumulative_returns.columns)
        
        for quantile in quantile_order:
            if quantile not in cumulative_returns.columns:
                continue
            if quantile not in [1, 10]:
                plt.plot(cumulative_returns.index, cumulative_returns[quantile], 
                         label=f'Q{quantile}', color=colors[quantile], 
                         linewidth=1.5, alpha=0.8, linestyle='-')
        
        for quantile in [1, 10]:
            if quantile in cumulative_returns.columns:
                label = f'Top组(Q{quantile})' if quantile == 10 else f'Bottom组(Q{quantile})'
                plt.plot(cumulative_returns.index, cumulative_returns[quantile], 
                         label=label, color=colors[quantile], 
                         linewidth=3.0, alpha=1.0, linestyle='-')
        
        plt.title(f'{period}天 - 分层收益', fontsize=16, fontweight='bold')
        plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left', fontsize=10, framealpha=0.9)
        plt.grid(True, alpha=0.3)
        
        if len(cumulative_returns) > 10:
            n_ticks = min(8, len(cumulative_returns))
            tick_indices = np.linspace(0, len(cumulative_returns)-1, n_ticks, dtype=int)
            plt.xticks(cumulative_returns.index[tick_indices], rotation=45, fontsize=10)
        
        plt.tight_layout()
        plt.show()

def plot_factor_performance(perfor_):
    periods = list(perfor_.keys())
    
    fig, axes = plt.subplots(2, 2, figsize=(15, 10))
    axes = axes.flatten()
    
    ls_returns = [perfor_[p]['long_short_return'] for p in periods]
    axes[0].bar(range(len(periods)), ls_returns, color='skyblue', alpha=0.8)
    axes[0].set_xticks(range(len(periods)))
    axes[0].set_xticklabels([f'{p}天' for p in periods])
    axes[0].set_title('多空组合收益', fontweight='bold')
    axes[0].set_ylabel('收益')
    for i, v in enumerate(ls_returns):
        axes[0].text(i, v, f'{v:.3f}', ha='center', va='bottom')
    
    ics = [perfor_[p]['ic'] for p in periods]
    axes[1].bar(range(len(periods)), ics, color='lightcoral', alpha=0.8)
    axes[1].set_xticks(range(len(periods)))
    axes[1].set_xticklabels([f'{p}天' for p in periods])
    axes[1].set_title('信息系数(IC)', fontweight='bold')
    axes[1].set_ylabel('IC')
    for i, v in enumerate(ics):
        axes[1].text(i, v, f'{v:.3f}', ha='center', va='bottom')
    
    if periods:
        first_period = periods[0]
        quantile_returns = perfor_[first_period]['quantile_returns']
        axes[2].plot(quantile_returns.index, quantile_returns.values, 
                   marker='o', linewidth=2, markersize=8, color='blue')
        axes[2].set_title(f'{first_period}天期分位数收益', fontweight='bold')
        axes[2].set_xlabel('分位数')
        axes[2].set_ylabel('平均收益')
        axes[2].grid(True, alpha=0.6)
        
        for i, v in enumerate(quantile_returns.values):
            axes[2].text(quantile_returns.index[i], v, f'{v:.3f}', 
                        ha='center', va='bottom')
    
    if periods and 'ic_series' in perfor_[periods[0]]:
        ic_series = perfor_[periods[0]]['ic_series']
        axes[3].plot(ic_series.index, ic_series.values, 
                   linewidth=1, color='purple', alpha=0.7)
        axes[3].axhline(y=ic_series.mean(), color='red', linestyle='--', 
                      label=f'均值: {ic_series.mean():.3f}')
        axes[3].set_title(f'{periods[0]}天期IC时间序列', fontweight='bold')
        axes[3].set_xlabel('日期')
        axes[3].set_ylabel('IC')
        axes[3].legend()
        axes[3].grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.show()


###############################################################################
#### data & fct ####

#### 读取数据 ####
price_data = pd.read_parquet('code/price_data.parquet')

#### 构建因子 ####


#### 添加因子 ####
fct_df = pd.read_parquet('code/fct_df.parquet')
factor_names = fct_df.columns
price_data = pd.concat([price_data,fct_df],axis=1,join='inner')


#### 数据处理 ####
price_data[factor_names] = extreme_MAD(price_data[factor_names])
price_data[factor_names] = standardize_z(price_data[factor_names].fillna(0))
price_data = preprocess_data(price_data)

#### 因子回测 ####
# 测试 fct_1
factor_df = price_data['fct_1']


###############################################################################
#### 单因子回测 ####
print('*******************')
print(price_data)
results = factor_analysis(factor_df, price_data, periods=(1, 5, 10, 20), quantiles=10)
performance = analyze_factor_performance(results)
summary_df = generate_summary_statistics(performance, 'code/因子回测统计结果.xlsx')

print("\n=== 详细统计结果汇总 ===")
print(summary_df.to_string(index=False, float_format='%.4f'))

# 绘制传统因子表现图
plot_factor_performance(performance)
plot_quantile_returns_separate(performance)
```
