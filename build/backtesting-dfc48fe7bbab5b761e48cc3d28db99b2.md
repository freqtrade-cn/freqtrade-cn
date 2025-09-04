---
title: 回测
description: 了解如何使用 Freqtrade 的回测功能验证和优化你的交易策略,包括回测命令、参数配置和结果分析
subject: 回测指南
subtitle: 策略回测和性能验证
short_title: 回测
tags: [回测, 策略验证, 性能分析, 历史数据]
categories: [策略开发]
prerequisites: [策略开发, 历史数据]
difficulty: intermediate
estimated_time: 45分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 8
toc: true
---

# 回测（Backtesting）

本页将介绍如何通过回测验证你的策略表现。

回测需要有历史数据。

如何获取你关注的交易对和交易所的数据，请参阅文档的[数据下载](data-download.md)部分。

## 回测命令参考

```{include} commands/backtesting.md
```

## 用回测测试你的策略

现在你已经有了完善的入场和出场策略，并准备了一些历史数据，你就可以用真实数据来测试它了。这就是我们所说的[回测](https://zh.wikipedia.org/wiki/%E5%9B%9E%E6%B5%8B)。

回测会使用你配置文件中的加密货币（交易对），并默认从 `user_data/data/<exchange>` 加载历史K线（OHLCV）数据。
如果没有该交易所/交易对/周期的数据，回测会提示你先用 `freqtrade download-data` 下载。详情请参阅[数据下载](data-download.md)部分。

回测结果可以帮助你判断机器人获利的概率是否大于亏损。

所有收益计算都包含手续费，freqtrade 会用交易所的默认手续费进行计算。

:::{warning} 回测中使用动态交易对列表
回测支持动态交易对列表（但并非所有处理器都可用于回测），但这依赖于当前市场状态，无法反映历史时期的真实交易对列表。
并且，使用非 StaticPairlist 时，回测结果无法保证可复现。
详情请阅读[交易对列表文档](plugins.md#pairlists)。

如需获得可复现结果，建议用 [`test-pairlist`](utils.md#test-pairlist) 命令生成静态交易对列表。
:::

:::{caution}
Freqtrade 默认会将回测结果导出到 `user_data/backtest_results`。
导出的交易可用于[进一步分析](#further-backtest-result-analysis)或用 [plotting 子命令](plotting.md#plot-price-and-indicators)（`freqtrade plot-dataframe`）进行可视化。
:::

### 起始余额

回测需要一个起始余额，可通过命令行参数 `--dry-run-wallet <余额>` 或 `--starting-balance <余额>`，也可通过配置项 `dry_run_wallet` 设置。
该金额必须大于 `stake_amount`, 否则机器人无法模拟任何交易。

### 动态交易金额

回测支持[动态交易金额](configuration.md#dynamic-stake-amount)，即将 `stake_amount` 配置为 "unlimited", 会将起始余额按 `max_open_trades` 平均分配。
前期盈利会导致后续交易金额增加，实现复利。

### 回测命令示例

### 用 5 分钟K线（OHLCV）数据（默认）

```bash
freqtrade backtesting --strategy AwesomeStrategy
```

其中 `--strategy AwesomeStrategy` / `-s AwesomeStrategy` 指的是策略类名，该类在 `user_data/strategies` 目录下的 python 文件中。

---

### 用 1 分钟K线（OHLCV）数据

```bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1m
```

---

### 自定义起始余额为 1000（以 stake 货币计）

```bash
freqtrade backtesting --strategy AwesomeStrategy --dry-run-wallet 1000
```

---

### 使用不同的本地历史K线（OHLCV）数据源

假设你从 Binance 下载了历史数据，保存在 `user_data/data/binance-20180101` 目录。
可用如下命令进行回测：

```bash
freqtrade backtesting --strategy AwesomeStrategy 
    --datadir user_data/data/binance-20180101 
```

---

### 对比多个策略

```bash
freqtrade backtesting --strategy-list SampleStrategy1 AwesomeStrategy --timeframe 5m
```

其中 `SampleStrategy1` 和 `AwesomeStrategy` 均为策略类名。

---

### 不导出交易到文件

```bash
freqtrade backtesting --strategy backtesting --export none --config config.json 
```

仅当你确定不需要后续可视化或分析结果时使用。

---

### 导出交易到自定义文件夹

```bash
freqtrade backtesting --strategy backtesting 
    --export trades 
    --backtest-directory=user_data/custom-backtest-results
```

更多关于[策略启动期](strategy-customization.md#strategy-startup-period)的内容请阅读相关文档。

---

### 自定义手续费

有时你的账户有手续费返还（如大额账户或月交易量达标），ccxt 无法感知。
此时可用 `--fee` 命令行参数指定手续费。
手续费为比例，回测时会收取两次（开仓和平仓各一次）。

例如，单笔手续费为 0.1%（即 0.001），可用如下命令：

```bash
freqtrade backtesting --fee 0.001
```

:::{caution} 参数使用场景
仅在你想测试不同手续费时使用该参数。默认情况下，回测会自动获取交易所默认手续费。
:::

---

### 用 timerange 缩小测试集

用 `--timerange` 参数可指定测试集时间范围。

如用 `--timerange=20190501-`，则从 2019 年 5 月 1 日起用所有可用数据：

```bash
freqtrade backtesting --timerange=20190501-
```

也可指定具体日期区间。

完整 timerange 语法：

- 用到 2018/01/31 的数据：`--timerange=-20180131`
- 用 2018/01/31 之后的数据：`--timerange=20180131-`
- 用 2018/01/31 到 2018/03/01 的数据：`--timerange=20180131-20180301`
- 用 POSIX/epoch 时间戳区间：`--timerange=1527595200-1527618600`

## 理解回测结果

回测最重要的是理解结果。

回测结果大致如下：

```
                                                 BACKTESTING REPORT                                                  
┏━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃          Pair ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃    Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ LTC/USDT:USDT │     16 │          1.0 │          56.176 │         5.62 │        16:16:00 │   16     0     0   100 │
│ ETC/USDT:USDT │     12 │         0.72 │          30.936 │         3.09 │         9:55:00 │   11     0     1  91.7 │
│ ETH/USDT:USDT │      8 │         0.66 │          17.864 │         1.79 │ 1 day, 13:55:00 │    7     0     1  87.5 │
│ XLM/USDT:USDT │     10 │         0.31 │          11.054 │         1.11 │        12:08:00 │    9     0     1  90.0 │
│ BTC/USDT:USDT │      8 │         0.21 │           7.289 │         0.73 │ 3 days, 1:24:00 │    6     0     2  75.0 │
│ XRP/USDT:USDT │      9 │        -0.14 │          -7.261 │        -0.73 │        21:18:00 │    8     0     1  88.9 │
│ DOT/USDT:USDT │      6 │         -0.4 │          -9.187 │        -0.92 │         5:35:00 │    4     0     2  66.7 │
│ ADA/USDT:USDT │      8 │        -1.76 │         -52.098 │        -5.21 │        11:38:00 │    6     0     2  75.0 │
│         TOTAL │     77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
└───────────────┴────────┴──────────────┴─────────────────┴──────────────┴─────────────────┴────────────────────────┘
                                               LEFT OPEN TRADES REPORT                                                
┏━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃          Pair ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃     Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ BTC/USDT:USDT │      1 │        -4.14 │          -9.930 │        -0.99 │ 17 days, 8:00:00 │    0     0     1     0 │
│ ETC/USDT:USDT │      1 │        -4.24 │         -15.365 │        -1.54 │         10:40:00 │    0     0     1     0 │
│ DOT/USDT:USDT │      1 │        -5.29 │         -19.125 │        -1.91 │         11:30:00 │    0     0     1     0 │
│         TOTAL │      3 │        -4.56 │         -44.420 │        -4.44 │  6 days, 2:03:00 │    0     0     3     0 │
└───────────────┴────────┴──────────────┴─────────────────┴──────────────┴──────────────────┴────────────────────────┘
                                                ENTER TAG STATS                                                
┏━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Enter Tag ┃ Entries ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃ Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│     OTHER │      77 │         0.22 │          54.774 │         5.48 │     22:12:00 │   67     0    10  87.0 │
│     TOTAL │      77 │         0.22 │          54.774 │         5.48 │     22:12:00 │   67     0    10  87.0 │
└───────────┴─────────┴──────────────┴─────────────────┴──────────────┴──────────────┴────────────────────────┘
                                                EXIT REASON STATS                                                 
┏━━━━━━━━━━━━━┳━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Exit Reason ┃ Exits ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃    Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━━╇━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│         roi │    67 │         1.05 │         242.179 │        24.22 │        15:49:00 │   67     0     0   100 │
│ exit_signal │     4 │        -2.23 │         -31.217 │        -3.12 │  1 day, 8:38:00 │    0     0     4     0 │
│  force_exit │     3 │        -4.56 │         -44.420 │        -4.44 │ 6 days, 2:03:00 │    0     0     3     0 │
│   stop_loss │     3 │       -10.14 │        -111.768 │       -11.18 │  1 day, 3:05:00 │    0     0     3     0 │
│       TOTAL │    77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
└─────────────┴───────┴──────────────┴─────────────────┴──────────────┴─────────────────┴────────────────────────┘
                                                        MIXED TAG STATS                                                        
┏━━━━━━━━━━━┳━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Enter Tag ┃ Exit Reason ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃    Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━╇━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│           │         roi │     67 │         1.05 │         242.179 │        24.22 │        15:49:00 │   67     0     0   100 │
│           │ exit_signal │      4 │        -2.23 │         -31.217 │        -3.12 │  1 day, 8:38:00 │    0     0     4     0 │
│           │  force_exit │      3 │        -4.56 │         -44.420 │        -4.44 │ 6 days, 2:03:00 │    0     0     3     0 │
│           │   stop_loss │      3 │       -10.14 │        -111.768 │       -11.18 │  1 day, 3:05:00 │    0     0     3     0 │
│     TOTAL │             │     77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
└───────────┴─────────────┴────────┴──────────────┴─────────────────┴──────────────┴─────────────────┴────────────────────────┘
                          SUMMARY METRICS                          
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Metric                        ┃ Value                           ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Backtesting from              │ 2025-07-01 00:00:00             │
│ Backtesting to                │ 2025-08-01 00:00:00             │
│ Trading Mode                  │ Isolated Futures                │
│ Max open trades               │ 3                               │
│                               │                                 │
│ Total/Daily Avg Trades        │ 77 / 2.48                       │
│ Starting balance              │ 1000 USDT                       │
│ Final balance                 │ 1054.774 USDT                   │
│ Absolute profit               │ 54.774 USDT                     │
│ Total profit %                │ 5.48%                           │
│ CAGR %                        │ 87.36%                          │
│ Sortino                       │ 2.48                            │
│ Sharpe                        │ 3.75                            │
│ Calmar                        │ 40.99                           │
│ SQN                           │ 0.69                            │
│ Profit factor                 │ 1.29                            │
│ Expectancy (Ratio)            │ 0.71 (0.04)                     │
│ Avg. daily profit             │ 1.767 USDT                      │
│ Avg. stake amount             │ 345.016 USDT                    │
│ Total trade volume            │ 53316.954 USDT                  │
│                               │                                 │
│ Long / Short trades           │ 67 / 10                         │
│ Long / Short profit %         │ 8.94% / -3.47%                  │
│ Long / Short profit USDT      │ 89.425 / -34.651                │
│                               │                                 │
│ Best Pair                     │ LTC/USDT:USDT 5.62%             │
│ Worst Pair                    │ ADA/USDT:USDT -5.21%            │
│ Best trade                    │ ETC/USDT:USDT 2.00%             │
│ Worst trade                   │ ADA/USDT:USDT -10.17%           │
│ Best day                      │ 26.91 USDT                      │
│ Worst day                     │ -47.741 USDT                    │
│ Days win/draw/lose            │ 20 / 6 / 5                      │
│ Min/Max/Avg. Duration Winners │ 0d 00:35 / 5d 18:15 / 0d 15:49  │
│ Min/Max/Avg. Duration Losers  │ 0d 10:40 / 17d 08:00 / 2d 17:00 │
│ Max Consecutive Wins / Loss   │ 36 / 3                          │
│ Rejected Entry signals        │ 258                             │
│ Entry/Exit Timeouts           │ 0 / 0                           │
│                               │                                 │
│ Min balance                   │ 1003.168 USDT                   │
│ Max balance                   │ 1149.421 USDT                   │
│ Max % of account underwater   │ 8.23%                           │
│ Absolute drawdown             │ 94.647 USDT (8.23%)             │
│ Drawdown duration             │ 9 days 08:50:00                 │
│ Profit at drawdown start      │ 149.421 USDT                    │
│ Profit at drawdown end        │ 54.774 USDT                     │
│ Drawdown start                │ 2025-07-22 15:10:00             │
│ Drawdown end                  │ 2025-08-01 00:00:00             │
│ Market change                 │ 30.51%                          │
└───────────────────────────────┴─────────────────────────────────┘

Backtested 2025-07-01 00:00:00 -> 2025-08-01 00:00:00 | Max open trades : 3
                                                            STRATEGY SUMMARY                                                            
┏━━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┓
┃       Strategy ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃ Avg Duration ┃  Win  Draw  Loss  Win% ┃           Drawdown ┃
┡━━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━┩
│ SampleStrategy │     77 │         0.22 │          54.774 │         5.48 │     22:12:00 │   67     0    10  87.0 │ 94.647 USDT  8.23% │
└────────────────┴────────┴──────────────┴─────────────────┴──────────────┴──────────────┴────────────────────────┴────────────────────┘
```

### 回测报告表格

第一张表包含机器人所有交易，包括"未平仓交易"。

最后一行给出策略总体表现，如：

```
│         TOTAL │     77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
```

机器人共进行了 `77` 笔交易，平均持仓时长 `22:12:00`, 总收益率 `5.48%`, 即用 1000 USDT 起始资金赚取了 `54.774 USDT`。

`Avg Profit %` 列为所有交易的平均收益率。

`Tot Profit %` 列为相对于起始余额的总收益率。

如上例，起始余额 1000 USDT，绝对收益 54.774 USDT，则 `Tot Profit %` = `(54.774 / 1000) * 100 ~= 5.48%`。

策略表现受入场、出场、`minimal_roi` 和 `stop_loss` 等多因素影响。

例如，若 `minimal_roi` 仅为：

```json
"minimal_roi": {
    "0":  0.01
},
```

则每次盈利 1% 就会平仓，无法获得更高收益。

反之，若 `minimal_roi` 过高，如 `"0":  0.55`（55%），几乎不可能达到。
因此，策略表现是多种因素的综合结果，包括配置和交易对选择。

### 未平仓交易表

第二张表为回测期结束时被"强制平仓"的所有交易，便于完整展示。
这些交易也包含在第一张表中，但单独列出便于分析。

以下是翻译后的中文内容，保持了Markdown格式：

### 进入标签统计表

第三个表格提供按进入标签（例如 `enter_long`、`enter_short`）分类的交易明细，显示每个标签的进入次数、平均利润百分比、基础货币总利润、总利润百分比、平均持续时间以及盈利、平局和亏损的次数。

### 退出原因统计表

第四个表格包含退出原因的汇总（例如 `exit_signal`、`roi`、`stop_loss`、`force_exit`）。此表格可以告诉您哪个领域需要额外改进（例如，如果许多 `exit_signal` 交易都是亏损的，您应该致力于改善退出信号或考虑禁用它）。

### 混合标签统计表

第五个表格结合了进入标签和退出原因，提供了不同进入标签与特定退出原因组合表现的详细视图。这有助于识别哪些进入和退出策略的组合最为有效。

### 汇总指标

回测报告最后是汇总指标表，它包含了您的策略在回测数据上表现的关键指标。。

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Metric                        ┃ Value                           ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Backtesting from              │ 2025-07-01 00:00:00             │
│ Backtesting to                │ 2025-08-01 00:00:00             │
│ Trading Mode                  │ Isolated Futures                │
│ Max open trades               │ 3                               │
│                               │                                 │
│ Total/Daily Avg Trades        │ 72 / 2.32                       │
│ Starting balance              │ 1000 USDT                       │
│ Final balance                 │ 1106.734 USDT                   │
│ Absolute profit               │ 106.734 USDT                    │
│ Total profit %                │ 10.67%                          │
│ CAGR %                        │ 230.04%                         │
│ Sortino                       │ 4.99                            │
│ Sharpe                        │ 8.00                            │
│ Calmar                        │ 77.76                           │
│ SQN                           │ 1.52                            │
│ Profit factor                 │ 1.79                            │
│ Expectancy (Ratio)            │ 1.48 (0.07)                     │
│ Avg. daily profit             │ 3.443 USDT                      │
│ Avg. stake amount             │ 363.133 USDT                    │
│ Total trade volume            │ 52466.174 USDT                  │
│                               │                                 │
│ Best Pair                     │ LTC/USDT:USDT 4.48%             │
│ Worst Pair                    │ ADA/USDT:USDT -1.78%            │
│ Best trade                    │ ETC/USDT:USDT 2.00%             │
│ Worst trade                   │ ADA/USDT:USDT -10.17%           │
│ Best day                      │ 23.535 USDT                     │
│ Worst day                     │ -49.813 USDT                    │
│ Days win/draw/lose            │ 21 / 6 / 4                      │
│ Min/Max/Avg. Duration Winners │ 0d 00:35 / 5d 18:15 / 0d 15:30  │
│ Min/Max/Avg. Duration Losers  │ 0d 12:00 / 17d 08:00 / 3d 23:28 │
│ Max Consecutive Wins / Loss   │ 58 / 4                          │
│ Rejected Entry signals        │ 254                             │
│ Entry/Exit Timeouts           │ 0 / 0                           │
│                               │                                 │
│ Min balance                   │ 1003.168 USDT                   │
│ Max balance                   │ 1209 USDT                       │
│ Max % of account underwater   │ 8.46%                           │
│ Absolute drawdown             │ 102.266 USDT (8.46%)            │
│ Drawdown duration             │ 9 days 08:50:00                 │
│ Profit at drawdown start      │ 209 USDT                        │
│ Profit at drawdown end        │ 106.734 USDT                    │
│ Drawdown start                │ 2025-07-22 15:10:00             │
│ Drawdown end                  │ 2025-08-01 00:00:00             │
│ Market change                 │ 30.51%                          │
└───────────────────────────────┴─────────────────────────────────┘
```

- `Backtesting from` / `Backtesting to`：回测区间（通常由 `--timerange` 指定）
- `Trading Mode`：现货或合约
- `Max open trades`：`max_open_trades`（或 `--max-open-trades`）设置，或交易对列表长度（取较小值）
- `Total/Daily Avg Trades`：总交易数 / 日均交易数（总交易数除以天数）
- `Starting balance`：起始余额（由 dry-run-wallet 指定）
- `Final balance`：最终余额 = 起始余额 + 绝对收益
- `Absolute profit`：以 stake 货币计的收益
- `Total profit %`：总收益率，对应第一张表 TOTAL 行的 Tot Profit %，计算方式为 `(最终余额-起始余额)/起始余额`
- `CAGR %`：年化复合增长率
- `Sortino`：年化 Sortino 比率
- `Sharpe`：年化 Sharpe 比率
- `Calmar`：年化 Calmar 比率
- `SQN`：系统质量指数（Van Tharp 提出）
- `Profit factor`：所有盈利交易的利润总和除以所有亏损交易的损失总和。
- `Expectancy (Ratio)`：期望比率，即每笔交易的平均利润或损失。负的期望比率意味着您的策略不盈利。
- `Avg. daily profit`：每日平均利润，计算公式为 `(总利润 / 回测天数)`。
- `Avg. stake amount`：平均持仓金额（动态交易金额时为平均值）
- `Total trade volume`：总交易量
- `Long / Short trades`：多头/空头交易次数分类（仅在有空头交易时显示）。
- `Long / Short profit %`：多头和空头交易的利润百分比（仅在有空头交易时显示）。
- `Long / Short profit USDT`：多头和空头交易在基础货币中的利润（仅在有空头交易时显示）。
- `Best Pair` / `Worst Pair`：表现最好/最差的交易对及其 Tot Profit %
- `Best Trade` / `Worst Trade`：单笔最大盈利/亏损
- `Best day` / `Worst day`：单日最大盈利/亏损
- `Days win/draw/lose`：盈利/亏损天数（平局通常是没有平仓交易的天数）。
- `Min/Max/Avg. Duration Winners`：盈利交易的最小、最大和平均持续时间。
- `Min/Max/Avg. Duration Losers`：亏损交易的最小、最大和平均持续时间。
- `Max Consecutive Wins / Loss`：最大连胜/连败次数
- `Rejected Entry signals`：因已达最大持仓数而被拒绝的入场信号
- `Entry/Exit Timeouts`：未成交的入场/出场订单数（仅自定义定价时适用）
- `Min balance` / `Max balance`：回测期间最低/最高钱包余额
- `Max % of account underwater`：自模拟开始以来账户从峰值下降的最大百分比。计算公式为 `(最大余额 - 当前余额) / (最大余额)` 的最大值。
- `Absolute drawdown`：经历的最大绝对回撤，包括相对于账户的百分比，计算公式为 `(绝对回撤) / (回撤高点 + 起始余额)`。
- `Drawdown duration`：最大回撤期间的持续时间。
- `Profit at drawdown start` / `Profit at drawdown end`：最大回撤期间开始和结束时的利润。
- `Drawdown start` / `Drawdown end`：最大回撤的开始和结束日期时间（也可以通过 `plot-dataframe` 子命令进行可视化）。
- `Market change`：回测期间市场的变化。计算为所有交易对从第一根到最后一根K线使用"收盘价"列的变化平均值。

### 日/周/月/年分解

可用 `--breakdown <>` 参数查看日/周/月/年分解结果。

如需可视化每月和每年分解，可用：

```bash
freqtrade backtesting --strategy MyAwesomeStrategy --breakdown month year
```

```output
                                MONTH BREAKDOWN
┏━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃      Month ┃ Trades ┃ Tot Profit USDT ┃ Profit Factor ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 31/01/2020 │     12 │          44.451 │          7.28 │   10     0     2  83.3 │
│ 29/02/2020 │     30 │           45.41 │          2.36 │   17     0    13  56.7 │
│ 31/03/2020 │     35 │         142.024 │          2.42 │   14     0    21  40.0 │
│ 30/04/2020 │     67 │         -23.692 │          0.81 │   24     0    43  35.8 │
...
...
│ 30/04/2025 │    203 │          -63.43 │          0.81 │   73     0   130  36.0 │
│ 31/05/2025 │    142 │         104.675 │          1.28 │   59     0    83  41.5 │
│ 30/06/2025 │    177 │          -1.014 │           1.0 │   85     0    92  48.0 │
│ 31/07/2025 │    155 │         232.762 │           1.6 │   63     0    92  40.6 │
└────────────┴────────┴─────────────────┴───────────────┴────────────────────────┘
                                  YEAR BREAKDOWN
┏━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃       Year ┃ Trades ┃ Tot Profit USDT ┃ Profit Factor ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 31/12/2020 │    896 │         868.889 │          1.46 │  351     0   545  39.2 │
│ 31/12/2021 │   1778 │        4487.163 │          1.93 │  745     0  1033  41.9 │
│ 31/12/2022 │   1736 │          938.27 │          1.27 │  698     0  1038  40.2 │
│ 31/12/2023 │   1712 │        1677.126 │          1.68 │  670     0  1042  39.1 │
│ 31/12/2024 │   1609 │        3198.424 │          2.22 │  773     0   836  48.0 │
│ 31/12/2025 │   1042 │         716.174 │          1.33 │  420     0   622  40.3 │
└────────────┴────────┴─────────────────┴───────────────┴────────────────────────┘
```

输出将显示包含所选期间已实现绝对利润（以基础货币计）的表格，以及其他统计信息，如交易次数、利润因子以及在此期间实现（平仓）的盈利、平局和亏损分布。

### 回测结果缓存

为节省时间，回测默认会复用一天内相同策略和配置的缓存结果。若需强制重新回测，可加 `--cache none` 参数。

:::{warning} 强制新回测
对于开放式 timerange（如 `--timerange 20210101-`），缓存会自动禁用，因为无法保证底层数据未变。若原回测因数据缺失导致结果不完整，后续补数据后也可能误用缓存。此时请用 `--cache none` 强制新回测。
:::

(further-backtest-result-analysis)=

### 进一步分析回测结果

freqtrade 默认会导出交易到文件，便于后续分析。
你可以用 pandas 加载这些交易，具体见[数据分析](strategy_analysis_example.md#load-backtest-results-to-pandas-dataframe)部分。

### 回测输出文件

freqtrade 生成的输出文件为 zip 包，包含：

- 回测报告（json 格式）
- 市场变化数据（feather 格式）
- 策略文件副本
- 策略参数副本（如用到参数文件）
- 配置文件副本（已脱敏）

这样可保证结果可复现（前提是数据一致）。

仅包含策略和配置文件，不包含依赖。

(assumptions-made-by-backtesting)=

## 回测的假设

由于回测无法获知K线内的详细价格变化，因此需做如下假设：

- 遵守交易所[交易限制](#trading-limits-in-backtesting)
- 入场价为开盘价，除非有自定义定价逻辑
- 只要价格在K线高低区间内，所有订单都能成交（无滑点）
- 出场信号在下一根K线开盘价成交
- 平仓后释放持仓位，可用于新交易对
- 出场信号优先于止损（假定信号在K线开盘触发）
- ROI
  - 出场价与高点比较，但用 ROI 值（如 ROI=2%，高点=5%，则以2%出场）
  - 出场价不会低于K线最低点，2% ROI 可能实际以2.4%成交（若低点为2.4%）
  - ROI 在触发K线生效时（如 `120: 0.02`，1hK线，从 `60: 0.05`），用K线开盘价作为出场价
  - ROI 为 `<N>=-1` 时，强制平仓用低点，除非 N 恰好在K线开盘
- 止损严格按止损价成交，即使低点更低，但实际亏损会比止损价多2倍手续费
- 止损在同一K线内优先于 ROI，因此回测中止损次数可能多于 dry-run/实盘
- 低点先于高点（止损优先保护资金）
- 跟踪止损
  - 仅当止损价低于K线低点时才调整（否则会被触发）
  - 入场K线触发跟踪止损时，用"最小偏移量"(`stop_positive_offset`)为基准（非高点），再计算止损。自定义止损不适用此规则。
  - 先高点再低点，止损用调整后的止损价
  - ROI 优先于跟踪止损，若两者都触发则以 ROI 为准
- 出场原因仅说明触发条件，不代表盈亏（如负 ROI 也可能显示为 exit_signal）
- 多信号同K线时的处理顺序：
  - 出场信号
  - 止损
  - ROI
  - 跟踪止损
- 反向开仓（仅合约）时，若平仓K线有反向信号，则直接反向建仓

基于上述假设，回测尽量贴近真实交易。但回测**永远无法**替代 dry-run。
请牢记，历史表现不代表未来。

此外，策略作者应仔细阅读[开发策略时的常见错误](strategy-customization.md#common-mistakes-when-developing-strategies)部分，避免回测用到实际不可用的数据。

(trading-limits-in-backtesting)=

### 回测中的交易限制

交易所有最小/最大下单量、最小/最大持仓金额等限制，通常在交易所文档"交易规则"部分列出，不同交易对差异很大。

回测（以及实盘和 dry-run）会遵守这些限制，并确保止损价低于最小下单金额，因此实际下单金额会略高于交易所要求。
但 freqtrade 无法获知历史限制。

这可能导致用历史价格计算的最小下单金额被高估（如历史高价时），出现最小金额 > 50 美元的情况。

例如：

BTC 最小可交易量为 0.001。
BTC 当前价格 22,000 美元（0.001 BTC = 22 美元），但回测区间内最高价为 50,000 美元。
此时最小金额可能为 `0.001 * 50_000 = 50` 美元。

#### 交易精度限制

大多数交易所对价格和数量有精度限制，不能买 1.0020401 个，或以 1.24567123123 的价格成交。
实际会按交易所定义四舍五入或截断，如数量 1.002，价格 1.24567。

这些精度值基于当前交易所限制（如上文所述），历史精度不可用。

## 提高回测精度

回测最大局限是无法获知K线内价格变化（高点先于收盘还是反之？）。
假设用 1 小时K线回测，则每根K线有 4 个价格（开、高、低、收）。

虽然回测会做一些假设（见上文），但永远无法完美，始终有偏差。
为缓解此问题，freqtrade 支持用更小周期模拟K线内波动。

只需在回测命令后加 `--timeframe-detail 5m`：

```bash
freqtrade backtesting --strategy AwesomeStrategy 
    --timeframe 1h --timeframe-detail 5m
```

这会加载 1 小时主周期和 5 分钟细节周期的数据。
策略分析仍用 1 小时周期。
但有信号或持仓时，会用 5 分钟周期分析细节。
这样能更准确模拟K线内波动，尤其在高周期下结果差异明显。

入场仍以主K线开盘价为准，但释放持仓位可能提前（如 5 分钟K线触发出场信号），可用于新交易。

所有回调函数（如 `custom_exit()`、`custom_stoploss()` 等）在持仓后会对每根 5 分钟K线运行一次（如 1 小时主周期、5 分钟细节周期，则每单 12 次）。

`--timeframe-detail` 必须小于主周期，否则回测无法启动。

显然，这会占用更多内存（5 分钟数据量大于 1 小时），也会影响运行时间（取决于交易数量和持仓时长）。数据需提前下载好。

:::{tip} 确保策略未利用回测假设
建议在策略开发最后阶段用此功能，确保策略未利用[回测假设](#assumptions-made-by-backtesting)漏洞。若细节模式下表现与常规模式相近，实盘表现也有较大概率接近（但只有 dry-run 才能真正验证）。
:::

:::{hint} 极端差异示例
在极端情况下（所有对在 10:00 K线有入场信号，max_open_trades=1），用 `--timeframe-detail` 回测可能出现如下交易序列：

| 交易对 | 入场时间 | 出场时间 | 持仓时长 |
|------|------------|-----------| -------- |
| BTC/USDT | 2024-01-01 10:00:00 | 2021-01-01 10:05:00 | 5m |
| ETH/USDT | 2024-01-01 10:05:00 | 2021-01-01 10:15:00 | 10m |
| XRP/USDT | 2024-01-01 10:15:00 | 2021-01-01 10:30:00 | 15m |
| SOL/USDT | 2024-01-01 10:15:00 | 2021-01-01 11:05:00 | 50m |
| BTC/USDT | 2024-01-01 11:05:00 | 2021-01-01 12:00:00 | 55m |

若不用细节数据，则为：

| 交易对 | 入场时间 | 出场时间 | 持仓时长 |
|------|------------|-----------| -------- |
| BTC/USDT | 2024-01-01 10:00:00 | 2021-01-01 11:00:00 | 1h |
| BTC/USDT | 2024-01-01 11:00:00 | 2021-01-01 12:00:00 | 1h |

差异显著，因为常规模式下每根K线只处理前 `max_open_trades` 个信号，持仓位只在K线结束时释放，导致新交易只能在下一根K线开仓。
:::

## 回测多策略

如需对比多个策略，可用策略列表进行回测。

每次仅支持一个周期，但数据只加载一次，适合对比多策略时节省时间。

所有策略需在同一目录，或用 `--recursive-strategy-search` 支持子目录。

```bash
freqtrade backtesting --timerange 20180401-20180410 
    --timeframe 5m 
    --strategy-list Strategy001 Strategy002 --export trades
```

结果会保存到 `user_data/backtest_results/backtest-result-<datetime>.json`，包含所有策略结果。
会有一张表对比各策略胜率/亏损（同第一张表 TOTAL 行）。
详细输出会依次展示每个策略，注意向上滚动查看。

```text
================================================== STRATEGY SUMMARY ===================================================================
| Strategy    |  Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration   |  Wins |  Draws | Losses | Drawdown % |
|-------------+---------+----------------+------------------+----------------+----------------+-------+--------+--------+------------|
| Strategy1   |     429 |           0.36 |       0.00762792 |          76.20 | 4:12:00        |   186 |      0 |    243 |       45.2 |
| Strategy2   |    1487 |          -0.13 |      -0.00988917 |         -98.79 | 4:43:00        |   662 |      0 |    825 |     241.68 |
```

## 下一步,寻找最优参数

恭喜，策略已盈利。如果机器人还能帮你找到最优参数呢？

下一步请学习[如何用 Hyperopt 寻找最优参数](hyperopt.md)
