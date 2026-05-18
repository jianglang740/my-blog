---
title: "动量➕均线多头策略回测及基于streamlit的交互式web页面构建"
description: "经典多头策略之“动量+均线”的代码模块化回测及基于streamlit搭建的交互式web页面全流程"
date: 2026-05-018
lastmod: 2026-05-018
weight: 4
categories:
    - Quant
    - Tech Stack
tags:
    - 代码实现
    - 回测
    - 交互式web页面
---

# 动量➕均线多头策略回测及基于streamlit的交互式web页面构建

---

- 作者：山财小蒋
- 联系方式：2018036661@qq,com
- 创作不易，转载请注明出处，欢迎批评探讨

---

- **在本篇文章中我将展示关于“动量➕均线”策略的完整代码展示，这也是量化领域最常见最基础的策略之一，并给予之前提到过的streamlit构建一个完整的交互式web页面，便于我们的回测以及展示。**

- **这也是我第一次写完整的回测项目，并做交互式web页面，有不足的地方也欢迎大家与我探讨。**

##绩效与web页面展示：

- 在直接进入code正题之前，我想先像大家展示一下该策略的绩效：

- **以太坊近一年的15m级别数据回测绩效图：**

![](https://img-reg-ab.imagency.cn/e/7a426b84b1a6a9a2c10d5c114ebd5c8f.png)

- **以太坊近3年的1h级别数据回测绩效图：**

![](https://img-reg-ab.imagency.cn/e/da9472d4fe876d1612d3f79d39be7ac1.png)

- **web页面：**

![](https://img-reg-ab.imagency.cn/e/f1a3b12c375e786712eb2a5dafa6212d.png)

![](https://img-reg-ab.imagency.cn/e/d924da183bf067612e0b5456bf9e5a00.png)

---

## 策略基类：

- 我们在写量化回测时不是简简单单跑一遍脚本就行了，也要有框架感很模块化管理的思想在里面，就像做一个完整的项目一样，每个模块负责一部分的功能，也方便我们后期的更改、查阅，或是策略的修改

- 接下来我将展示一个完整的策略基类：

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import yfinance as yf

# ==================== 1. 策略基类 ====================
class Strategy:
    """所有策略的基类"""
    def __init__(self, data, short_window=20, long_window=50, position=0, cash=300, use_cash_rate=1.0, commission_rate=0.001,return_window = 7,return_threshold = 0.01):
        self.data = data.copy()
        self.short_window = short_window
        self.long_window = long_window
        self.position = position
        self.cash = cash
        self.initial_cash = cash #初始资金，是不变的，用于最后的绩效计算
        self.use_cash_rate = use_cash_rate
        self.commission_rate = commission_rate  # 手续费率，例如 0.001 表示 0.1%
        self.return_window = return_window
        self.return_threshold = return_threshold
        self.trades = []

    def init(self):
        """在回测开始前调用，用于计算指标"""
        self.data['SMA_short'] = self.data['close'].rolling(self.short_window).mean()
        self.data['SMA_long']  = self.data['close'].rolling(self.long_window).mean()
        self.data["n_period_return"] = self.data["close"].pct_change(periods=self.return_window)
        self.data['n_period_return'].fillna(0, inplace=True)
    def next(self, i):
        raise NotImplementedError

    def buy(self, date, price, i, shares=None):
        """买入逻辑（含手续费）"""
        if shares is None:
            available_cash = self.cash * self.use_cash_rate
            shares = float(available_cash / price)

        if shares <= 0:   #如果计算出的买入股数为0或负数，代表资金不足，直接返回
            return

        cost = shares * price
        commission = cost * self.commission_rate   # 计算手续费
        total_cost = cost + commission

        if total_cost > self.cash:
            # 如果现金不足（考虑手续费后），调整买入股数
            max_shares = self.cash / (price * (1 + self.commission_rate))
            shares = float(max_shares)
            cost = shares * price
            commission = cost * self.commission_rate
            total_cost = cost + commission

        self.cash -= total_cost
        self.position += shares

        self.trades.append({
            'date': date,
            'price': price,
            'shares': shares,
            'type': 'BUY',
            'commission': commission,
            'cash_after': self.cash,
            'position_after': self.position
        })
        print(f"{date} 买入 {shares:.4f} 股 @ {price:.2f}，手续费 {commission:.2f}，"
              f"剩余现金 {self.cash:.2f}，短期均线 {self.data['SMA_short'].iloc[i]:.2f}，长期均线 {self.data['SMA_long'].iloc[i]:.2f}")

    def sell(self, date, price, i, shares=None):
        """卖出逻辑（含手续费）"""
        if shares is None:  #none是指调用时每输入shares参数时，是空值，那么默认卖出所有持仓
            shares = float(self.position)  #默认卖出所有持仓

        if shares <= 0:
            return

        revenue = shares * price  #卖出后获得的总收益
        commission = revenue * self.commission_rate   # 计算手续费
        net_revenue = revenue - commission #减去手续费之后的卖出操作净利润

        self.cash += net_revenue  #更新净值
        self.position -= shares  #更新持仓

        self.trades.append({
            'date': date,
            'price': price,
            'shares': shares,
            'type': 'SELL',
            'commission': commission,
            'cash_after': self.cash,
            'position_after': self.position
        })
        print(f"{date} 卖出 {shares:.4f} 股 @ {price:.2f}，手续费 {commission:.2f}，"
              f"现金变为 {self.cash:.2f}，短期均线 {self.data['SMA_short'].iloc[i]:.2f}，长期均线 {self.data['SMA_long'].iloc[i]:.2f}")
```

## 具体策略：

```python
# ==================== 2. 具体的动量+双均线策略 ====================
class DoubleMAStrategy(Strategy):  #并没有使用__int__,所以它只是策略基类的子类，可以直接调用基类
    """动量+双均线策略：金叉按比例买入，死叉全仓卖出"""
    def next(self, i):
        if i < self.long_window:# 1. 跳过前N根K线（未形成长期均线的阶段）
            return

        # 2. 获取当前和前一根K线的短/长期均线值
        sma_short_curr = self.data['SMA_short'].iloc[i]  # 当前短期均线
        sma_long_curr  = self.data['SMA_long'].iloc[i]   # 当前长期均线
        sma_short_prev = self.data['SMA_short'].iloc[i-1]# 前一根短期均线
        sma_long_prev  = self.data['SMA_long'].iloc[i-1] # 前一根长期均线
        # 3. 获取当前时间和收盘价（用于交易）
        date = self.data.index[i]  #以日期标签时，如i=100则找第一百行对应的日期，而非位置标签，
        price = self.data['close'].iloc[i]  #先取出close列，取位置标签，每一行即是一根新的k线数据
        current_return = self.data['n_period_return'].iloc[i] # 近n个周期收益率

        # 金叉：当前短期上穿长期，且前一根短期均线在长期均线下方
        golden_cross = (sma_short_prev <= sma_long_prev) and (sma_short_curr > sma_long_curr)
        return_condition = current_return >= self.return_threshold  # 收益率达标
        if golden_cross and return_condition and self.position == 0: #golden_cross，return_condition是布尔型，self.position == 0是判别持仓是否为0（==的结果也是布尔型），只有当三者均成立时才执行买入
            self.buy(date, price, i)
        
        # 死叉条件：短期均线下穿长期 + 有持仓（原有逻辑不变）
        death_cross = (sma_short_prev >= sma_long_prev) and (sma_short_curr < sma_long_curr)
        if death_cross and self.position > 0:
            self.sell(date, price, i)
```

## 回测引擎：

```python
# ==================== 3. 回测引擎 ====================
class Backtest:
    def __init__(self, strategy):
        self.strategy = strategy
        self.equity_curve = [] #账户权益列表，记录每根K线的权益变化，包含日期和权益值

    def run(self):
        self.strategy.init()
        # 遍历每一根K线，逐行执行next()
        for i in range(len(self.strategy.data)):  #遍历每一根k线
            self.strategy.next(i)  # 核心：每根K线（每一行）执行一次策略逻辑
            total_equity = self.strategy.cash + self.strategy.position * self.strategy.data['close'].iloc[i] #总权益（cash + position * price）
            self.equity_curve.append({
                'date': self.strategy.data.index[i],
                'equity': total_equity
            })

        # 如果回测结束后还有持仓，则强制平仓（同样扣除手续费）
        if self.strategy.position > 0:
            last_date = self.strategy.data.index[-1]
            last_price = self.strategy.data['close'].iloc[-1]
            self.strategy.sell(last_date, last_price, len(self.strategy.data)-1, shares=self.strategy.position)
            self.equity_curve[-1]['equity'] = self.strategy.cash

        self.equity_df = pd.DataFrame(self.equity_curve).set_index('date')
        return self.equity_df

    def plot_results(self):
        """画出价格、均线、净值曲线"""
        fig, (ax1, ax2, ax3) = plt.subplots(3, 1, figsize=(12, 12), sharex=True)

        ax1.plot(self.strategy.data.index, self.strategy.data['close'], label='Close', color='black', alpha=0.6)
        ax1.plot(self.strategy.data.index, self.strategy.data['SMA_short'], label=f'SMA {self.strategy.short_window}', linestyle='--')
        ax1.plot(self.strategy.data.index, self.strategy.data['SMA_long'], label=f'SMA {self.strategy.long_window}', linestyle='--')

        buy_dates = [t['date'] for t in self.strategy.trades if t['type'] == 'BUY']  #t是临时变量，代表每一笔交易记录，t['type']是交易类型，筛选出买入记录的日期和价格，这是字典的键值对取值法，而buy_date是一个列表，包含所有买入记录的日期
        buy_prices = [t['price'] for t in self.strategy.trades if t['type'] == 'BUY']  #同上，筛选出买入记录的价格
        sell_dates = [t['date'] for t in self.strategy.trades if t['type'] == 'SELL']
        sell_prices = [t['price'] for t in self.strategy.trades if t['type'] == 'SELL']
        ax1.scatter(buy_dates, buy_prices, marker='^', color='green', s=100, label='Buy')
        ax1.scatter(sell_dates, sell_prices, marker='v', color='red', s=100, label='Sell')
        ax1.set_title('Price & Moving Averages & Trade Signals')
        ax1.legend()
        ax1.grid(True)

        ax2.plot(self.equity_df.index, self.equity_df['equity'], label='Equity Curve', color='blue')
        ax2.set_title('Equity Curve')
        ax2.legend()
        ax2.grid(True)

        ax3.plot(self.strategy.data.index, self.strategy.data['n_period_return'], label=f'{self.strategy.return_window}-Period Return', color='orange')  #采样粒度是一样的，都是每一根k线基于n个周期算收益率，所以不应担心数据对齐问题
        ax3.axhline(y=self.strategy.return_threshold, color='red', linestyle='--', label=f'Return Threshold ({self.strategy.return_threshold:.1%})')  #画出动量阈值线
        ax3.set_title(f'{self.strategy.return_window}-Period Return')
        ax3.legend()
        ax3.grid(True)
        ax3.set_ylabel('Return')

        plt.tight_layout()
        plt.show()

    def calculate_win_rate(self):
        trades = self.strategy.trades  #调用交易记录列表
        if not trades:
            return 0.0, 0, 0.0, 0

        buy_list = [t for t in trades if t['type'] == 'BUY']  #列表里包含了所有买单的详细记录，是列表，不过列表里有字典，每个字典代表一笔交易记录，包括日期、价格、数量、手续费等信息
        sell_list = [t for t in trades if t['type'] == 'SELL']

        num_trades = min(len(buy_list), len(sell_list))  #统计完整的交易轮数
        if num_trades == 0:
            return 0.0, 0, 0.0, 0

        win_count = 0
        total_profit = 0.0

        for i in range(num_trades):
            buy = buy_list[i] #第i比买单的详细记录，包括日期、价格、数量、手续费等信息，是字典
            sell = sell_list[i]
            cost = buy['price'] * buy['shares'] + buy.get('commission', 0) # 买入成本（含手续费）
            revenue = sell['price'] * sell['shares'] - sell.get('commission', 0) # 卖出收入（扣除手续费），sell.get('commission', 0)是字典的get方法，如果sell字典里有'commission'键，则返回对应的值，否则返回0，确保即使某笔交易记录没有手续费信息也不会出错
            profit = revenue - cost #每笔交易的净利润（已扣除手续费）
            total_profit += profit #累计净利润
            if profit > 0:
                win_count += 1

        win_rate = (win_count / num_trades) * 100  #胜率（百分比）
        avg_profit_per_trade = total_profit / num_trades #平均每笔交易的净利润（已扣除手续费）
        return win_rate, num_trades, avg_profit_per_trade, win_count

    def stats(self):
        initial_cash = self.strategy.initial_cash  #初始资金
        final_equity = self.equity_df['equity'].iloc[-1]  #最终权益（已减去手续费）
        total_return = (final_equity - initial_cash) / initial_cash * 100 #总收益率（已减去手续费）
        #peak：用 cummax()（累积最大值）计算，代表从回测开始到当前时间点的最高账户权益（比如账户先涨到 1500，再跌到 1200，那么 1500 就是这个阶段的 peak）。
        peak = self.equity_df['equity'].cummax() #计算每个时间点的历史最高权益
        drawdown = (self.equity_df['equity'] - peak) / peak #计算回撤率（当前权益与历史最高权益的差值占历史最高权益的比例）(当前权益 - 峰值) / 峰值，永远为负数或零，越接近零代表回撤越小，越负代表回撤越大
        max_drawdown = drawdown.min() * 100 #找负数中最小的那个（即最大回撤），乘以100转成百分比

        num_trades = len(self.strategy.trades) #总交易记录次数（包含买卖）
        win_rate, complete_trades, avg_profit, win_count = self.calculate_win_rate()

        # 计算总手续费
        total_commission = sum(t.get('commission', 0) for t in self.strategy.trades)
        

        print("========== 回测绩效 ==========")
        print(f"初始资金: {initial_cash:.2f}")
        print(f"最终权益: {final_equity:.2f}(已减去手续费)")
        print(f"总收益率: {total_return:.2f}%")
        print(f"最大回撤: {max_drawdown:.2f}%")
        print(f"总交易记录次数: {num_trades} 次（包含买卖）")
        print(f"完整交易轮数: {complete_trades}")
        print(f"胜率: {win_rate:.2f}% ({win_count}/{complete_trades})")
        print(f"平均每笔盈利(已扣除手续费): {avg_profit:.2f}")
        print(f"总手续费: {total_commission:.2f}")
        print(f"动量窗口: {self.strategy.return_window} 根K线")
        print(f"动量阈值: {self.strategy.return_threshold:.2%}")

        return {
            'total_return': total_return, #总收益率
            'max_drawdown': max_drawdown, # 最大回撤
            'num_trades': num_trades, #买卖次数
            'win_rate': win_rate, #胜率
            'avg_profit_per_trade': avg_profit, #平均盈利
            'total_commission': total_commission #手续费
        }

```

## 数据获取：

```python
# ==================== 4. 数据获取 ====================
def get_data_from_yfinance(symbol='AAPL', start='2020-01-01', end='2023-12-31'):
    df = yf.download(symbol, start=start, end=end)
    df = df[['Close']].rename(columns={'Close': 'close'})
    return df

def get_simulated_data():
    dates = pd.date_range('2020-01-01', periods=10000, freq='1h')
    np.random.seed(42)
    returns = np.random.normal(0.001, 0.02, len(dates))
    prices = 100 * np.exp(np.cumsum(returns))
    df = pd.DataFrame({'close': prices}, index=dates)
    return df

def get_data_from_csv():
    df = pd.read_csv("/Users/clinking/开发/quant/data/ETH_USDT_USDT_15m.csv")
    df['datetime'] = pd.to_datetime(df['datetime'])
    df.set_index('datetime', inplace=True)  #显式设计，设置datetime为索引
    return df

```

## 主函数：

```python
# ========================================
if __name__ == "__main__":
    data = get_data_from_csv()
    # 设置手续费率，例如加密货币现货常见 0.1%（即 0.001）
    strategy = DoubleMAStrategy(data, short_window=7, long_window=14, cash=1000, commission_rate=0.001,return_window=1,return_threshold=0.005)
    bt = Backtest(strategy)
    equity_df = bt.run()
    bt.plot_results()
    bt.stats()
```

## streamlit交互式web页面：

```python
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt

# ---------- 导入策略模块 ----------
import sys
from pathlib import Path

# Add parent directory to path to allow imports
#__file__是当前脚本的文件路径；Path(__file__)将其转换为Path对象，方便后续链式操作；
# .resolve()获取绝对路径；.parent.parent获取上两级目录（即项目根目录），然后将其转换为字符串并插入到sys.path的开头，这样就可以直接导入strategy目录下的模块了
sys.path.insert(0, str(Path(__file__).resolve().parent.parent))

from strategy.double_ma_strategy import (
    DoubleMAStrategy,   
    get_data_from_yfinance,
    get_simulated_data,
    get_data_from_csv,
    Backtest
) #导入策略类和数据获取函数


# ---------- Streamlit 界面 ----------
st.set_page_config(page_title="双均线+动量策略回测", layout="wide") #浏览器标签页标题
st.title("📈 均值动量多头策略回测可视化面板") #居中大标题

# 侧边栏参数设置
with st.sidebar:
    st.header("⚙️ 策略参数") #侧边栏标题
    st.write("请选择数据源和回测参数") #侧边栏说明
    #data_source是一个普通的变量，用于存储用户选择的数据源
    data_source = st.selectbox("数据源", ["模拟数据", "Yahoo Finance", "上传 CSV"])
    #st.selectbox 是 Streamlit 提供的下拉选择框组件 ，用于在网页上创建一个交互式的下拉菜单。
    #第一个参数是下拉菜单的标题，第二个参数是一个列表，包含了用户可以选择的选项。
    if data_source == "Yahoo Finance":
        symbol = st.text_input("股票代码", "AAPL")
        start_date = st.date_input("开始日期", pd.to_datetime("2020-01-01"))
        end_date = st.date_input("结束日期", pd.to_datetime("2023-12-31"))
        #st.text_input 和 st.date_input 都是 Streamlit 的交互式输入组件 ，用于收集用户输入的数据。
        #第一个参数是输入框的标题，第二个参数是默认值。
        #st.text_input("股票代码", "AAPL") 是一个文本输入框，用于输入股票代码，默认值为 "AAPL"。
        #st.date_input("开始日期", pd.to_datetime("2020-01-01")) 是一个日期选择框，用于选择开始日期，默认值为 "2020-01-01"。
        #st.date_input("结束日期", pd.to_datetime("2023-12-31")) 是一个日期选择框，用于选择结束日期，默认值为 "2023-12-31"。
    elif data_source == "上传 CSV":
        uploaded_file = st.file_uploader("选择 CSV 文件", type="csv")
        #st.file_uploader("选择 CSV 文件", type="csv") 是 Streamlit 的文件上传组件 ，用于让用户从本地选择并上传 CSV 文件。
        #第一个参数是文件上传组件的标题，第二个参数是文件类型，这里指定为 CSV 文件。
        #type="csv" 表示只能上传 CSV 文件，其他文件类型会被拒绝。
    else:
        st.write("使用内置模拟数据")
    #st.number_input 是 Streamlit 的数字输入组件 ，用于让用户输入数值（整数或小数）value是默认值
    short_window = st.number_input("🧶短期移动均线窗口", min_value=2, value=20)
    long_window = st.number_input("🧵长期移动均线窗口", min_value=5, value=50)
    return_window = st.number_input("📊收益率窗口", min_value=1, value=7)
    return_threshold = st.number_input("📈动量阈值", min_value=0.0, value=0.01, format="%.4f") #显示格式（如 "%.4f" 表示保留4位小数）
    commission_rate = st.number_input("手续费",min_value=0.000,value=0.0001,format="%.4f")
    initial_cash = st.number_input("初始资金", min_value=100.0, value=10000.0, step=100.0)


    
    run_backtest = st.button("🚀 运行回测")
    #st.button 是 Streamlit 的按钮组件 ，用于创建一个交互式的按钮，用户点击后会触发相应的操作。布尔型
    #第一个参数是按钮的标题，这里设置为 "🚀 运行回测"。

# 主区域
if run_backtest:
    # 1. 获取数据，st.spinner 是 Streamlit 的加载状态提示组件 ，用于在执行耗时操作时显示一个加载动画，提升用户体验。
    with st.spinner("正在加载数据..."):
        if data_source == "模拟数据":
            data = get_simulated_data()
            st.success("使用模拟数据")#st.success 是 Streamlit 的成功消息组件 ，用于在页面上显示一条绿色的成功提示信息。
        elif data_source == "Yahoo Finance":
            try:
                data = get_data_from_yfinance(symbol, start_date.strftime("%Y-%m-%d"), end_date.strftime("%Y-%m-%d"))
                st.success(f"已加载 {symbol} 数据")
            except Exception as e:
                st.error(f"下载数据失败: {e}")#st.error 是 Streamlit 的错误消息组件 ，用于在页面上显示一条红色的错误提示信息，通常在操作失败或出现异常时使用。
                st.stop()#st.stop 是 Streamlit 的停止执行组件 ，用于在代码执行过程中停止当前操作，防止继续执行。
        else:  # 上传 CSV
            if uploaded_file is not None:
                try:
                    data = pd.read_csv(uploaded_file)
                    # 尝试自动解析日期列
                    if 'datetime' in data.columns:
                        data['datetime'] = pd.to_datetime(data['datetime'])
                        data.set_index('datetime', inplace=True)#set_index 是 pandas 的方法，用于将指定列设置为数据框的索引列。
                    elif 'Date' in data.columns:
                        data['Date'] = pd.to_datetime(data['Date'])
                        data.set_index('Date', inplace=True)
                    else:
                        data.index = pd.to_datetime(data.index)
                    # 确保有 close 列
                    if 'close' not in data.columns:
                        if 'Close' in data.columns:
                            data.rename(columns={'Close': 'close'}, inplace=True)
                        else:
                            st.error("CSV 文件必须包含 'close' 或 'Close' 列")
                            st.stop()
                    st.success("CSV 数据加载成功")
                except Exception as e:
                    st.error(f"读取 CSV 失败: {e}")
                    st.stop()
            else:
                st.warning("请上传 CSV 文件")#st.warning 是 Streamlit 的警告消息组件 ，用于在页面上显示一条黄色的警告提示信息，通常在用户操作不完整或需要注意某些事项时使用。
                st.stop()

    # 2. 创建策略与回测实例
    # 注意：原始策略类中的初始资金写死在 300，我们通过修改实例的 cash 属性来覆盖
    strategy = DoubleMAStrategy(data, short_window=short_window, long_window=long_window,commission_rate=commission_rate, return_window=return_window, return_threshold=return_threshold)
    strategy.cash = initial_cash #是会变的，因为会根据交易结果更新资金
    strategy.initial_cash = initial_cash  # 记录初始资金，用于统计，是不变的

    bt = Backtest(strategy)

    # 3. 运行回测
    with st.spinner("回测运行中..."):
        equity_df = bt.run()
    #equity_df = bt.run() 是 回测的核心执行语句 ，用于运行完整的策略回测并获取权益曲线结果。
    # 4. 显示结果
    st.subheader("📊 回测结果")
    #st.subheader 是 Streamlit 的子标题组件 ，用于在页面上显示一个次级标题，通常用于组织和分隔页面内容。
    
    col1, col2, col3, col4, col5, col6,col7 = st.columns(7)
    final_equity = equity_df['equity'].iloc[-1] # 最终权益，即策略在最后一天的权益值
    total_return = (final_equity - initial_cash) / initial_cash * 100 # 总收益率，即策略在最后一天的权益值与初始资金的百分比差
    
    # 计算最大回撤
    peak = equity_df['equity'].cummax()
    drawdown = (equity_df['equity'] - peak) / peak
    max_drawdown = drawdown.min() * 100
    
    win_rate, complete_trades, avg_profit, win_count = bt.calculate_win_rate()
    #计算总手续费
    stats = bt.stats()                     # 调用方法获得统计字典
    total_commission = stats['total_commission']
    #通过 st.columns() 创建列布局，然后在各列中使用 .metric() 方法：#st.columns 是 Streamlit 的布局组件 ，用于创建页面上的列布局。
    #.metric() 是 Streamlit 的指标组件 ，用于在页面上显示一个数值指标。
    col1.metric("最终权益", f"${final_equity:,.2f}")
    col2.metric("总收益率", f"{total_return:.2f}%")
    col3.metric("最大回撤", f"{max_drawdown:.2f}%")
    col4.metric("交易次数", f"{len(strategy.trades)} 次")
    col5.metric("盈利次数", f"{win_count} 次")
    col6.metric("胜率", f"{win_rate:.2f}%")
    col7.metric("总手续费",f"{total_commission}usdt")
    
    # 绘制图表
    st.subheader("📈 价格、均线与交易信号")
    fig1, ax1 = plt.subplots(figsize=(12, 5))
    ax1.plot(strategy.data.index, strategy.data['close'], label='Close', color='black', alpha=0.6)
    ax1.plot(strategy.data.index, strategy.data['SMA_short'], label=f'SMA {short_window}', linestyle='--')
    ax1.plot(strategy.data.index, strategy.data['SMA_long'], label=f'SMA {long_window}', linestyle='--')
    
    buy_dates = [t['date'] for t in strategy.trades if t['type'] == 'BUY']
    buy_prices = [t['price'] for t in strategy.trades if t['type'] == 'BUY']
    sell_dates = [t['date'] for t in strategy.trades if t['type'] == 'SELL']
    sell_prices = [t['price'] for t in strategy.trades if t['type'] == 'SELL']
    
    ax1.scatter(buy_dates, buy_prices, marker='^', color='green', s=100, label='Buy')
    ax1.scatter(sell_dates, sell_prices, marker='v', color='red', s=100, label='Sell')
    ax1.set_title('Price & Moving Averages & Trade Signals')
    ax1.legend()
    ax1.grid(True)
    st.pyplot(fig1)
    
    st.subheader("💰 净值曲线")
    fig2, ax2 = plt.subplots(figsize=(12, 4))
    ax2.plot(equity_df.index, equity_df['equity'], label='Equity Curve', color='blue')
    ax2.set_title('Equity Curve')
    ax2.legend()
    ax2.grid(True)
    st.pyplot(fig2)

    st.subheader("💁🏻 动量阈值")
    fig3,ax3 = plt.subplots(figsize = (12,4))
    ax3.plot(strategy.data.index, strategy.data['n_period_return'], label='return', color='orange')
    ax3.axhline(y=strategy.return_threshold, color='red', linestyle='--', label=f'Return Threshold ({strategy.return_threshold:.1%})')  #画出动量阈值线
    ax3.set_title(f'{strategy.return_window}-Period Return')
    ax3.legend()
    ax3.grid(True)
    ax3.set_ylabel('Return')
    st.pyplot(fig3)
```