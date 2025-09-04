---
title: Freqtrade 策略分析示例
subject: Freqtrade 策略开发指南
subtitle: 如何分析和调试交易策略
short_title: 策略分析示例
description: 本文档介绍如何使用 Freqtrade 提供的工具来分析和调试交易策略,包括数据可视化、回测分析等功能。
---

# 策略分析示例

调试一个策略可能非常耗时。Freqtrade 提供了辅助函数来可视化原始数据。
以下假设你使用的是 SampleStrategy，数据为来自 Binance 的 5 分钟线，并已下载到默认位置的 data 目录。
更多细节请参见[官方文档](https://www.freqtrade.io/en/stable/data-download/)。

## 环境设置

### 将工作目录切换到仓库根目录

```python
import os
from pathlib import Path

# 切换目录
# 修改此单元格以确保输出显示正确的路径。
# 所有路径均以项目根目录为基准
project_root = "somedir/freqtrade"
i = 0
try:
    os.chdir(project_root)
    if not Path("LICENSE").is_file():
        i = 0
        while i < 4 and (not Path("LICENSE").is_file()):
            os.chdir(Path(Path.cwd(), "../"))
            i += 1
        project_root = Path.cwd()
except FileNotFoundError:
    print("请根据当前目录定义项目根目录")
print(Path.cwd())
```

### 配置 Freqtrade 环境

```python
from freqtrade.configuration import Configuration

# 根据你的需求自定义以下内容。

# 初始化空配置对象
config = Configuration.from_files([])
# 可选（推荐），使用已有配置文件
# config = Configuration.from_files(["user_data/config.json"])

# 定义一些常量
config["timeframe"] = "5m"
# 策略类名
config["strategy"] = "SampleStrategy"
# 数据位置
data_location = config["datadir"]
# 要分析的交易对 - 这里只用一个交易对
pair = "BTC/USDT"
```

```python
# 使用上面设置的值加载数据
from freqtrade.data.history import load_pair_history
from freqtrade.enums import CandleType

candles = load_pair_history(
    datadir=data_location,
    timeframe=config["timeframe"],
    pair=pair,
    data_format="json",  # 请确保与你的数据格式一致
    candle_type=CandleType.SPOT,
)

# 确认加载成功
print(f"已为 {pair} 从 {data_location} 加载 {len(candles)} 行数据")
candles.head()
```

## 加载并运行策略

* 每次更改策略文件后需重新运行

```python
# 使用上面设置的值加载策略
from freqtrade.data.dataprovider import DataProvider
from freqtrade.resolvers import StrategyResolver

strategy = StrategyResolver.load_strategy(config)
strategy.dp = DataProvider(config, None, None)
strategy.ft_bot_start()

# 用策略生成买/卖信号
df = strategy.analyze_ticker(candles, {"pair": pair})
df.tail()
```

### 显示交易详情

* 注意，使用 `data.head()` 也可以，但大多数指标在 dataframe 顶部有一些"启动"数据。
* 可能遇到的问题：
    * dataframe 末尾有 NaN 值的列
    * 用于 `crossed*()` 函数的列单位完全不同
* 与完整回测的对比：
    * `analyze_ticker()` 输出某个交易对有 200 个买入信号，并不意味着回测时会有 200 笔交易。
    * 假如买入条件仅为 `df['rsi'] < 30`，则会连续多次为同一交易对生成"买入"信号（直到 rsi 回到 >29）。机器人只会在这些信号中的第一个（且有可用交易槽"max_open_trades"时）或中间某个信号买入。

```python
# 报告结果
print(f"共生成 {df['enter_long'].sum()} 个入场信号")
data = df.set_index("date", drop=False)
data.tail()
```

## 在 Jupyter notebook 中加载已有对象

以下单元格假设你已通过命令行生成了数据。  
它们可以帮助你更深入地分析结果，避免信息过载。

### 加载回测结果到 pandas dataframe

分析交易 dataframe（下方绘图也会用到）

```python
from freqtrade.data.btanalysis import load_backtest_data, load_backtest_stats

# 如果 backtest_dir 指向目录，会自动加载最新回测文件。
backtest_dir = config["user_data_dir"] / "backtest_results"
# backtest_dir 也可以指向具体文件
# backtest_dir = (
#   config["user_data_dir"] / "backtest_results/backtest-result-2020-07-01_20-04-22.json"
# )
```

```python
# 用以下命令可获取完整回测统计信息。
# 包含生成回测结果所用的全部信息。
stats = load_backtest_stats(backtest_dir)

strategy = "SampleStrategy"
# 所有统计信息都按策略分组，如果回测时用过 `--strategy-list`，这里也会体现。
# 示例用法：
print(stats["strategy"][strategy]["results_per_pair"])
# 获取本次回测用的交易对列表
print(stats["strategy"][strategy]["pairlist"])
# 获取市场变化（回测期间所有交易对的平均涨跌幅）
print(stats["strategy"][strategy]["market_change"])
# 最大回撤（绝对值）
print(stats["strategy"][strategy]["max_drawdown_abs"])
# 最大回撤起止时间
print(stats["strategy"][strategy]["drawdown_start"])
print(stats["strategy"][strategy]["drawdown_end"])

# 策略对比（仅多策略回测时有意义）
print(stats["strategy_comparison"])
```

```python
# 以 dataframe 形式加载回测交易
trades = load_backtest_data(backtest_dir)

# 按交易对统计各退出原因数量
trades.groupby("pair")["exit_reason"].value_counts()
```

## 绘制每日收益 / 资金曲线

```python
# 绘制资金曲线（第 1 天从 0 开始，每天加上当日回测收益）

import pandas as pd
import plotly.express as px

from freqtrade.configuration import Configuration
from freqtrade.data.btanalysis import load_backtest_stats

# strategy = 'SampleStrategy'
# config = Configuration.from_files(["user_data/config.json"])
# backtest_dir = config["user_data_dir"] / "backtest_results"

stats = load_backtest_stats(backtest_dir)
strategy_stats = stats["strategy"][strategy]

df = pd.DataFrame(columns=["dates", "equity"], data=strategy_stats["daily_profit"])
df["equity_daily"] = df["equity"].cumsum()

fig = px.line(df, x="dates", y="equity_daily")
fig.show()
```

### 加载实盘交易结果到 pandas dataframe

如果你已经有实盘交易数据，想分析自己的表现：

```python
from freqtrade.data.btanalysis import load_trades_from_db

# 从数据库获取交易
trades = load_trades_from_db("sqlite:///tradesv3.sqlite")

# 显示结果
trades.groupby("pair")["exit_reason"].value_counts()
```

## 分析已加载交易的并行度
这有助于结合高 `max_open_trades` 设置回测时，找到最佳的 `max_open_trades` 参数。

`analyze_trade_parallelism()` 返回一个带有 "open_trades" 列的时间序列 dataframe，表示每根 K 线时刻的持仓数量。

```python
from freqtrade.data.btanalysis import analyze_trade_parallelism

# 分析上述交易
parallel_trades = analyze_trade_parallelism(trades, "5m")

parallel_trades.plot()
```

## 绘图结果

Freqtrade 提供了基于 plotly 的交互式绘图能力。

```python
from freqtrade.plot.plotting import generate_candlestick_graph

# 限定绘图区间以保证 plotly 响应速度

# 只保留一个交易对的交易
trades_red = trades.loc[trades["pair"] == pair]

data_red = data["2019-06-01":"2019-06-10"]
# 生成蜡烛图
graph = generate_candlestick_graph(
    pair=pair,
    data=data_red,
    trades=trades_red,
    indicators1=["sma20", "ema50", "ema55"],
    indicators2=["rsi", "macd", "macdsignal", "macdhist"],
)
```

```python
# 内嵌显示图表
# graph.show()

# 在新窗口渲染图表
graph.show(renderer="browser")
```

## 绘制每笔交易平均收益的分布图

```python
import plotly.figure_factory as ff

hist_data = [trades.profit_ratio]
group_labels = ["profit_ratio"]  # 数据集名称

fig = ff.create_distplot(hist_data, group_labels, bin_size=0.01)
fig.show()
```

如果你有更好的数据分析思路，欢迎提交 issue 或 Pull Request 丰富本文档！
