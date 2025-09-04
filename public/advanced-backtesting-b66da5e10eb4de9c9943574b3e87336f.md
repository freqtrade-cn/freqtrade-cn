---
title: 高级回测分析
subject: FreqTrade 高级回测指南
subtitle: 深入了解回测分析功能
short_title: 高级回测
description: 本文档介绍了 FreqTrade 的高级回测分析功能,包括买入/卖出标签分析、信号导出等高级特性。
---

# 高级回测分析

## 分析买入/入场和卖出/出场标签

了解策略根据用于标记不同买入条件的买入/入场标签的行为可能很有帮助。你可能希望看到比默认回测输出更复杂的每个买入和卖出条件的统计信息。你还可能想要确定导致开仓信号的蜡烛上的指标值。

:::{note}
以下买入原因分析仅适用于回测，*不适用于超参数优化（hyperopt）*。
:::

我们需要在回测时使用 `--export` 选项设置为 `signals`，以启用信号**和**交易的导出：

```bash
freqtrade backtesting -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange=<timerange> --export=signals
```

这会让 freqtrade 输出一个包含策略、交易对及其对应蜡烛 DataFrame 的 pickle 字典，这些蜡烛导致了入场和出场信号。
根据你的策略产生的入场次数，这个文件可能会很大，因此请定期检查你的 `user_data/backtest_results` 文件夹，删除旧的导出文件。

在运行下次回测前，请确保删除旧的回测结果，或使用 `--cache none` 选项运行回测，以确保不使用缓存结果。

如果一切顺利，你现在应该能在 `user_data/backtest_results` 文件夹中看到 `backtest-result-{timestamp}_signals.pkl` 和 `backtest-result-{timestamp}_exited.pkl` 文件。

要分析入场/出场标签，现在需要使用 `freqtrade backtesting-analysis` 命令，并提供 `--analysis-groups` 选项和空格分隔的参数：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 1 2 3 4 5
```

该命令将读取最近的回测结果。`--analysis-groups` 选项用于指定显示每组或每笔交易利润的各种表格输出，范围从最简单（0）到最详细的每交易对、每买入和每卖出标签（4）：

* 0：按 enter_tag 汇总的整体胜率和利润
* 1：按 enter_tag 分组的利润汇总
* 2：按 enter_tag 和 exit_tag 分组的利润汇总
* 3：按交易对和 enter_tag 分组的利润汇总
* 4：按交易对、enter_ 和 exit_tag 分组的利润汇总（此表可能非常大）
* 5：按 exit_tag 分组的利润汇总

更多选项可通过 `-h` 参数查看。

以下是翻译后的中文内容，保持了Markdown格式：

### 使用 backtest-filename

默认情况下，`backtesting-analysis` 处理 `user_data/backtest_results` 目录中最新的回测结果。
如果您想分析之前回测的结果，请使用 `--backtest-filename` 选项来指定所需的文件。这让您可以随时重新访问和分析历史回测输出，只需提供相关回测结果的文件名：

``` bash
freqtrade backtesting-analysis -c <config.json> --timeframe <tf> --strategy <strategy_name> --timerange <timerange> --export signals --backtest-filename backtest-result-2025-03-05_20-38-34.zip
```

您应该会在日志中看到类似以下的输出，其中包含导出的时间戳文件名：

```
2022-06-14 16:28:32,698 - freqtrade.misc - INFO - dumping json to "mystrat_backtest-2022-06-14_16-28-32.json"
```

然后您可以在 `backtesting-analysis` 中使用该文件名：

```
freqtrade backtesting-analysis -c <config.json> --backtest-filename=mystrat_backtest-2022-06-14_16-28-32.json
```

要使用来自不同结果目录的结果，您可以使用 `--backtest-directory` 来指定目录：

``` bash
freqtrade backtesting-analysis -c <config.json> --backtest-directory custom_results/ --backtest-filename mystrat_backtest-2022-06-14_16-28-32.json
```

### 调整要显示的买入和卖出标签

要在输出中只显示特定的买入和卖出标签，请使用以下两个选项：

```
--enter-reason-list : 要分析的入场信号的空格分隔列表。默认："all"
--exit-reason-list : 要分析的出场信号的空格分隔列表。默认："all"
```

例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss
```

### 输出信号蜡烛的指标

`freqtrade backtesting-analysis` 的真正强大之处在于能够打印信号蜡烛上的指标值，以便对买入信号指标进行细致调查和调优。要打印给定指标集的列，请使用 `--indicator-list` 选项：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-groups 0 2 --enter-reason-list enter_tag_a enter_tag_b --exit-reason-list roi custom_exit_tag_a stop_loss --indicator-list rsi rsi_1h bb_lowerband ema_9 macd macdsignal
```

这些指标必须存在于你的策略主 DataFrame（主时间框架或信息性时间框架）中，否则它们将在脚本输出中被忽略。

:::{note} 指标列表
指标值将在入场和出场点都显示。如果指定了 `--indicator-list all`，则只会显示入场点的指标，以避免因策略不同而导致的列表过大。
:::

分析中自动包含一系列蜡烛和交易相关字段，只需在 indicator-list 中包含即可自动访问，包括：

- **open_date     ：** 交易开仓时间
- **close_date    ：** 交易平仓时间
- **min_rate      ：** 持仓期间最低价
- **max_rate      ：** 持仓期间最高价
- **open          ：** 信号蜡烛开盘价
- **close         ：** 信号蜡烛收盘价
- **high          ：** 信号蜡烛最高价
- **low           ：** 信号蜡烛最低价
- **volume        ：** 信号蜡烛成交量
- **profit_ratio  ：** 交易利润率
- **profit_abs    ：** 交易的绝对利润

#### 指标值示例输出

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen 
```

在此示例中，
我们希望显示交易入场和出场点的 `chikou_span` 和 `tenkan_sen` 指标值。

指标输出示例：

| 交易对      | 开仓时间                 | 入场原因 | 出场原因 | chikou_span (入场) | tenkan_sen (入场) | chikou_span (出场) | tenkan_sen (出场) |
|-----------|---------------------------|--------------|-------------|---------------------|--------------------|--------------------|-------------------|
| DOGE/USDT | 2024-07-06 00:35:00+00:00 |              | exit_signal | 0.105               | 0.106              | 0.105              | 0.107             |
| BTC/USDT  | 2024-08-05 14:20:00+00:00 |              | roi         | 54643.440           | 51696.400          | 54386.000          | 52072.010         |

如表所示，`chikou_span (入场)` 表示交易开仓时的指标值，
而 `chikou_span (出场)` 则表示平仓时的指标值。
这种详细的指标值视图增强了分析能力。

指标后缀 `(入场)` 和 `(出场)` 用于区分交易开仓和平仓时的指标值。

:::{note} 交易级指标
某些交易级指标没有 `(入场)` 或 `(出场)` 后缀。这些指标包括：`pair`, `stake_amount`, 
    `max_stake_amount`, `amount`, `open_date`, `close_date`, `open_rate`, `close_rate`, `fee_open`, `fee_close`, `trade_duration`, 
    `profit_ratio`, `profit_abs`, `exit_reason`,`initial_stop_loss_abs`, `initial_stop_loss_ratio`, `stop_loss_abs`, `stop_loss_ratio`, 
    `min_rate`, `max_rate`, `is_open`, `enter_tag`, `leverage`, `is_short`, `open_timestamp`, `close_timestamp` 和 `orders`
:::

#### 基于入场或出场信号过滤指标

`--indicator-list` 选项默认会显示入场和出场信号的指标值。要仅显示入场信号的指标值，可使用 `--entry-only` 参数。类似地，仅显示出场信号指标值可用 `--exit-only` 参数。

示例：仅显示入场信号指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --entry-only
```

示例：仅显示出场信号指标值：

```bash
freqtrade backtesting-analysis -c user_data/config.json --analysis-groups 0 --indicator-list chikou_span tenkan_sen --exit-only
```

:::{note} 
使用这些过滤器时，指标名称不会带有 `(入场)` 或 `(出场)` 后缀。
:::


### 按日期过滤交易输出

要仅显示回测时间范围内某些日期之间的交易，请在 `YYYYMMDD-[YYYYMMDD]` 格式下提供常用的 `timerange` 选项：

```
--timerange : 用于过滤输出交易的时间范围，起始日期包含，结束日期不包含。例如：20220101-20221231
```

例如，如果你的回测时间范围是 `20220101-20221231`，但只想输出 1 月份的交易：

```bash
freqtrade backtesting-analysis -c <config.json> --timerange 20220101-20220201
```

### 打印被拒绝的信号

使用 `--rejected-signals` 选项打印被拒绝的信号。

```bash
freqtrade backtesting-analysis -c <config.json> --rejected-signals
```

### 将表格写入 CSV

部分表格输出可能很大，直接打印到终端不太方便。
使用 `--analysis-to-csv` 选项可禁用表格标准输出，并将其写入 CSV 文件。

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv
```

默认情况下，这会为你在 `backtesting-analysis` 命令中指定的每个输出表写一个文件，例如：

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --rejected-signals --analysis-groups 0 1
```

这会写入 `user_data/backtest_results`：

* rejected_signals.csv
* group_0.csv
* group_1.csv

要自定义文件写入路径，还可指定 `--analysis-csv-path` 选项。

```bash
freqtrade backtesting-analysis -c <config.json> --analysis-to-csv --analysis-csv-path another/data/path/
```
