---
title: Freqtrade 策略开发指南
subject: Freqtrade 策略开发
subtitle: 从零开始学习 Freqtrade 策略开发
short_title: 策略开发指南
description: 本文档介绍了 Freqtrade 策略开发的基础知识,包括必备概念、策略结构以及如何开发自己的交易策略。
---

# Freqtrade 策略 101：策略开发快速入门

本快速入门假设你已了解交易基础，并已阅读 [Freqtrade 基础](bot-basics.md) 页面。

## 必备知识

在 `Freqtrade` 中，策略是一个 `Python` 类，用于定义买入和卖出加密货币“资产”的逻辑。

资产通过 `pairs`（交易对）定义，代表“币种”和“计价币”。币种是你用另一种货币作为计价币进行交易的资产。

交易所提供的数据以 `K线`（candles）形式给出，每根 K 线包含六个值：`date`、`open`、`high`、`low`、`close` 和 `volume`。

`技术分析`（Technical analysis）函数通过各种计算和统计公式分析 `K` 线数据，生成称为“指标”（indicators）的二级值。

指标在交易对的 K 线上被分析，用于生成“信号”（signals）。

信号会在加密货币“交易所”上转化为“订单”（orders），即“交易”（trades）。

我们使用“进场”（entry）和“出场”（exit）来代替“买入”和“卖出”，因为 Freqtrade 同时支持“多头”（long）和“空头”（short）交易。

- **多头（long）**：你用计价币买入币种，例如用 `USDT` 买入 `BTC`，之后以高于买入价的价格卖出获利。多头交易的盈利来自币种相对计价币的升值。
- **空头（short）**：你从交易所借入币种，之后以更低的价格买回并归还，赚取差价。空头交易的盈利来自币种相对计价币的贬值（你以更低的价格还清借款）。

虽然 `Freqtrade` 支持部分交易所的现货和合约市场，但为简化起见，本教程仅关注现货（多头）交易。

## 基础策略结构

### 主数据帧(dataframe)

`Freqtrade` 策略使用一种带有行和列的表格数据结构，称为 `数据帧(dataframe)`，用于生成进出场信号。

你配置的每个交易对都有自己的数据帧。

数据帧以 `date` 列为索引，例如 `2024-06-31 12:00`。

接下来的 5 列分别代表 `open`、`high`、`low`、`close` 和 `volume`（OHLCV）数据。

### 填充指标值(populate_indicators)

`populate_indicators` 函数会向数据帧添加代表技术分析指标值的列。

常见指标包括`相对强弱指数（RSI）`、`布林带（Bollinger Bands）`、`资金流指数（MFI）`、`均线（MA）`、`平均真实波幅（ATR）`等。

通过调用技术分析函数（如 ta-lib 的 RSI 函数 `ta.RSI()`）并赋值给某列名（如 `rsi`），即可向数据帧添加指标列：

```python
dataframe['rsi'] = ta.RSI(dataframe)
```

:::{tip} 技术分析库
不同的库生成指标值的方式不同。请查阅各自文档了解如何集成到你的策略中。你也可以参考 [Freqtrade 示例策略](https://github.com/freqtrade/freqtrade-strategies) 获取灵感。
:::

### 填充进场信号()

`populate_entry_trend` 函数定义进场信号的条件。

数据帧会新增 `enter_long` 列，当该列值为 `1` 时，Freqtrade 视为进场信号。

:::{tip} 做空
若要做空交易，请使用 `enter_short` 列。
:::

### 填充出场信号

`populate_exit_trend` 函数定义出场信号的条件。

数据帧会新增 `exit_long` 列，当该列值为 `1` 时，Freqtrade 视为出场信号。

:::{tip}  做空
若要做空平仓，请使用 `exit_short` 列。
:::

## 一个简单的策略

下面是一个最小化的 Freqtrade 策略示例：

```python
from freqtrade.strategy import IStrategy
from pandas import DataFrame
import talib.abstract as ta

class MyStrategy(IStrategy):

    timeframe = '15m'

    # 初始止损设为 -10%
    stoploss = -0.10

    # 盈利大于 1% 时随时出场
    minimal_roi = {"0": 0.01}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 生成技术分析指标值
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 根据指标值生成进场信号
        dataframe.loc[
            (dataframe['rsi'] < 30),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 根据指标值生成出场信号
        dataframe.loc[
            (dataframe['rsi'] > 70),
            'exit_long'] = 1

        return dataframe
```

## 交易执行

当检测到信号（进场或出场列为 `1`）时，Freqtrade 会尝试下单，即创建一个 `trade` 或 `position`。

每个新交易会占用一个“槽位”（slot）。槽位代表可同时开启的新交易最大数量。

槽位数量由 `max_open_trades` [配置项](configuration.md) 决定。

但在以下情况下，生成信号未必会创建交易订单，包括：

- 剩余计价币不足以买入资产，或钱包中无足够资产卖出（包括手续费）
- 剩余可用槽位不足（已开仓数量等于 `max_open_trades`）
- 某交易对已有持仓（Freqtrade 不支持同一对叠加持仓，但可[调整已有持仓](strategy-callbacks.md#adjust-trade-position)）
- 同一根 K 线上同时出现进场和出场信号，视为[信号冲突](strategy-customization.md#colliding-signals)，不会下单
- 策略通过相关 [entry](strategy-callbacks.md#trade-entry-buy-order-confirmation) 或 [exit](strategy-callbacks.md#trade-exit-sell-order-confirmation) 回调主动拒绝下单

更多细节请阅读 [策略自定义](strategy-customization.md) 文档。

## 回测与前向测试

策略开发可能漫长且令人沮丧，因为将人的“直觉”转化为可运行的计算机策略（“量化”）并不总是直观的。

因此，策略应经过测试以验证其是否按预期工作。

Freqtrade 提供两种测试模式：

- **回测（backtesting）**：使用你[从交易所下载的历史数据](data-download.md)，回测可快速评估策略表现。但结果很容易被扭曲，使策略看起来比实际更赚钱。详见[回测文档](backtesting.md)。
- **模拟盘（dry run）**：也称“前向测试”，用实时数据，但不会在交易所实际下单，只在 Freqtrade 内部跟踪信号和交易。前向测试实时运行，结果更可靠但耗时更长。

通过在 [配置文件](configuration.md#using-dry-run-mode) 中将 `dry_run` 设为 true 启用模拟盘。

:::{warning} 回测结果可能极不准确
回测结果与实际可能有很大差异。请查阅[回测假设](backtesting.md#assumptions-made-by-backtesting)和[常见策略错误](strategy-customization.md#common-mistakes-when-developing-strategies)。

一些网站展示的 Freqtrade 策略回测结果很亮眼，但请勿认为这些结果可实现或真实。
:::

:::{tip} 有用的命令
Freqtrade 提供两个有用命令检查策略基本缺陷：[lookahead-analysis](lookahead-analysis.md) 和 [recursive-analysis](recursive-analysis.md)。
:::

### 评估回测与模拟盘结果

回测后务必用模拟盘运行策略，比较两者结果是否足够接近。

如有明显差异，请检查进出场信号在两种模式下是否一致且出现在同一根 K 线上。但 dry run 和回测之间总会有差异：

- 回测假定所有订单都成交。dry run 下若用限价单或交易所无成交量，可能无法成交。
- 回测在信号出现的 K 线收盘后，假定以下一根 K 线开盘价进场（除非策略有自定义定价回调）。dry run 下信号到实际下单有延迟。这是因为新 K 线到来时，Freqtrade 需分析所有交易对数据，实际下单会比 K 线开盘稍有延迟。
- dry run 下的进场价格可能与回测不同，导致收益计算也不同。因此 ROI、止损、追踪止损和回调出场等结果不完全一致很正常。
- 计算机处理新 K 线和信号、下单的延迟越大，价格波动越不可控。请确保你的电脑能在合理时间内处理所有交易对数据。若数据处理延迟严重，Freqtrade 会在日志中警告。

## 控制和监控运行中的机器人

机器人在 dry run 或实盘模式下运行时，Freqtrade 提供六种控制和监控方式：

- **[FreqUI](freq-ui.md)**：最易上手的 Web 界面，可查看和控制机器人当前活动。
- **[Telegram](telegram-usage.md)**：在移动设备上集成 Telegram，可接收机器人活动提醒并控制部分功能。
- **[FTUI](https://github.com/freqtrade/ftui)**：命令行界面，仅用于监控运行中的机器人。
- **[freqtrade-client](rest-api.md#consuming-the-api)**：Python 实现的 REST API 客户端，便于在 Python 应用或命令行中请求和消费机器人响应。
- **[REST API 接口](rest-api.md#available-endpoints)**：REST API 允许开发者自定义工具与 Freqtrade 机器人交互。
- **[Webhooks](webhook-config.md)**：Freqtrade 可通过 webhook 向其他服务（如 discord）发送信息。

### 日志

Freqtrade 会生成详细的调试日志，帮助你了解机器人运行状况。请熟悉日志中的信息和错误消息。

日志默认输出到标准输出（命令行）。如需写入文件，许多 freqtrade 命令（包括 `trade`）支持 `--logfile` 选项。

示例请查阅 [FAQ](faq.md#how-do-i-search-the-bot-logs-for-something)。

## 最后的思考

量化交易很难，大多数公开策略表现并不好，因为要让策略在多种场景下持续盈利需要大量时间和精力。

因此，直接用公开策略并通过回测评估表现往往问题多多。但 Freqtrade 提供了多种工具，帮助你做出决策并尽职调查。

盈利的方式有很多，没有任何单一技巧或配置能让表现不佳的策略变好。

Freqtrade 是一个开源平台，拥有庞大且乐于助人的社区——欢迎加入我们的 [discord 频道](https://discord.gg/p7nuUNVfP7) 与他人交流你的策略！

始终只投资你愿意失去的资金。

## 总结

在 Freqtrade 中开发策略，就是基于技术指标定义进出场信号。按照上述结构和方法，你可以创建并测试自己的交易策略。

常见问题可在我们的 [FAQ](faq.md) 查阅。

欲深入了解，请参考更详细的 [Freqtrade 策略自定义文档](strategy-customization.md)。
