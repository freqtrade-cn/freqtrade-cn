---
title: 策略自定义指南
subject: Freqtrade 策略开发
subtitle: 如何开发自己的交易策略
short_title: 策略自定义
description: 本文档介绍如何在 Freqtrade 中自定义交易策略,包括添加新指标、设置交易规则等内容。
tags: [策略开发, 自定义, 指标, 交易规则, 编程]
categories: [策略开发]
prerequisites: [Python基础, 策略101, 机器人基础]
difficulty: advanced
estimated_time: 90分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 9
toc: true
---

# 策略自定义

本页将介绍如何自定义你的策略、添加新指标以及设置交易规则。

如果你还没有了解过，建议先阅读：

- [Freqtrade 策略 101](strategy-101.md)，快速入门策略开发
- [Freqtrade 机器人基础](bot-basics.md)，了解机器人整体运行机制

## 开发你自己的策略

机器人自带了一个默认策略文件。

此外，[策略仓库](https://github.com/freqtrade/freqtrade-strategies)中还提供了其他策略。

不过你很可能有自己的策略想法。本文档旨在帮助你将想法转化为可运行的策略。

### 生成策略模板

你可以通过以下命令快速开始：

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

这会基于模板创建一个名为 `AwesomeStrategy` 的新策略，文件路径为 `user_data/strategies/AwesomeStrategy.py`。

:::{warning} 名称区别
策略的*名称*和文件名是有区别的。大多数命令中，Freqtrade 使用的是*策略类名*，而不是文件名。

`new-strategy` 命令生成的示例策略并不会直接盈利。
:::

:::{tip} 不同模板级别
`freqtrade new-strategy` 有一个额外参数 `--template`，可控制生成策略的复杂程度。用 `--template minimal` 可获得一个空白策略，无任何指标示例；用 `--template advanced` 可获得包含更多复杂特性的模板。
:::

### 策略结构剖析

一个策略文件包含构建策略逻辑所需的全部信息：

- K线数据（OHLCV 格式）
- 各类指标
- 入场逻辑
  - 信号
- 出场逻辑
  - 信号
  - 最小收益率（ROI）
  - 回调函数（自定义函数）
- 止损
  - 固定/绝对止损
  - 跟踪止损
  - 回调函数（自定义函数）
- 定价（可选）
- 持仓调整（可选）

机器人自带一个名为 `SampleStrategy` 的示例策略，可作为参考：`user_data/strategies/sample_strategy.py`。

你可以用参数 `--strategy SampleStrategy` 进行测试。注意这里用的是策略类名，而不是文件名。

此外，还有一个名为 `INTERFACE_VERSION` 的属性，用于定义策略接口的版本。当前版本为 `3`，如果未在策略中显式设置，则默认为 `3`。

你可能会看到旧策略设置为接口版本 `2`，未来版本会要求升级到 `v3`。

用 `trade` 命令即可启动机器人进入 `dry` 或 `live` 模式：

```bash
freqtrade trade --strategy AwesomeStrategy
```

### 机器人运行模式

Freqtrade 策略可在 5 种主要模式下被机器人处理：

- 回测（backtesting）
- 超参优化（hyperopting）
- 模拟盘（dry/forward testing）
- 实盘（live）
- FreqAI（本页不涉及）

关于如何设置 dry 或 live 模式，请查阅[配置文档](configuration.md)。

**测试时请始终使用 dry 模式，这样可以在不冒资金风险的情况下了解策略实际表现。**

## 深入剖析

**以下内容将以 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py) 为参考。**

:::{caution} 策略与回测
为避免回测与 dry/live 模式下出现问题和意外差异，请注意回测时 `populate_*()` 方法会一次性传入完整时间区间。

因此建议使用向量化操作（针对整个 dataframe，而非循环），避免用 `df.iloc[-1]`，而应用 `df.shift()` 获取前一根K线。
:::

:::{warning} 警惕未来数据
回测时 `populate_*()` 方法会传入完整时间区间，策略作者需确保不使用未来数据。

常见错误模式见文档后面的[常见错误](#开发策略时的常见错误)章节。
:::

:::{tip} 前视与递归分析
Freqtrade 提供了两条命令帮助检测常见的前视（未来数据）和递归偏差（指标值方差）问题。在 dry/live 前，建议先用 [lookahead](lookahead-analysis.md) 和 [recursive](recursive-analysis.md) 工具分析。
:::

### DataFrame

Freqtrade 使用 [pandas](https://pandas.pydata.org/) 存储/提供 K 线（OHLCV）数据。
Pandas 是处理表格数据的强大库。

DataFrame 的每一行对应一根K线，最新的完整K线总是排在最后（按日期排序）。

用 pandas 的 `head()` 查看前几行：

```output
> dataframe.head()
                       date      open      high       low     close     volume
0 2021-11-09 23:25:00+00:00  67279.67  67321.84  67255.01  67300.97   44.62253
1 2021-11-09 23:30:00+00:00  67300.97  67301.34  67183.03  67187.01   61.38076
2 2021-11-09 23:35:00+00:00  67187.02  67187.02  67031.93  67123.81  113.42728
3 2021-11-09 23:40:00+00:00  67123.80  67222.40  67080.33  67160.48   78.96008
4 2021-11-09 23:45:00+00:00  67160.48  67160.48  66901.26  66943.37  111.39292
```

DataFrame 是一个表格，列不是单一值，而是一组数据。因此，像下面这样直接用 Python 比较会报错：

```python
    if dataframe['rsi'] > 30:
        dataframe['enter_long'] = 1
```

上述写法会报错：`The truth value of a Series is ambiguous [...]`。

应改为 pandas 向量化写法，对整个 dataframe 执行操作：

```python
    dataframe.loc[
        (dataframe['rsi'] > 30)
    , 'enter_long'] = 1
```

这样会在 RSI 大于 30 时，为新列 `enter_long` 赋值 1。

Freqtrade 会用这个新列作为入场信号，假定下一根K线开盘时开仓。

Pandas 支持高效的向量化计算，建议尽量避免循环，直接用向量化方法。

向量化操作会对整列数据进行计算，比逐行循环快得多。

:::{tip} 信号 vs 交易

- 信号由指标在K线收盘时生成，表示有意愿开仓。
- 交易是在实际下单（实盘时在交易所），会尽量在下一根K线开盘时成交。

:::

:::{warning} 交易下单假设

回测时，信号在K线收盘时生成，交易在下一根K线开盘时执行。

dry/live 模式下，因需先分析所有交易对 dataframe，再处理下单，可能有延迟。建议减少交易对数量，并用高主频 CPU。

:::

#### 为什么看不到"实时"K线数据？

Freqtrade 不会在 dataframe 中存储未完成/未收盘的K线。

用未完成数据做决策叫"重绘"，有些平台允许，但 Freqtrade 不支持。只有完整K线数据可用。

### 自定义指标

入场和出场信号需要用到指标。你可以在策略文件的 `populate_indicators()` 方法中添加更多指标。

只应添加在 `populate_entry_trend()`、`populate_exit_trend()` 或用于生成其他指标的指标，否则会影响性能。

务必返回 dataframe，且不要删除/修改 "open", "high", "low", "close", "volume" 这几列，否则会导致异常。

示例：

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    为给定 DataFrame 添加多种技术指标

    性能提示：为获得最佳性能，请只用你策略或超参优化用到的指标，否则会浪费内存和 CPU。
    :param dataframe: 交易所数据 DataFrame
    :param metadata: 额外信息，如当前交易对
    :return: 包含所有策略所需指标的 DataFrame
    """
    dataframe['sar'] = ta.SAR(dataframe)
    dataframe['adx'] = ta.ADX(dataframe)
    stoch = ta.STOCHF(dataframe)
    dataframe['fastd'] = stoch['fastd']
    dataframe['fastk'] = stoch['fastk']
    dataframe['bb_lower'] = ta.BBANDS(dataframe, nbdevup=2, nbdevdn=2)['lowerband']
    dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)
    dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)
    dataframe['mfi'] = ta.MFI(dataframe)
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
    dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
    dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
    dataframe['ao'] = awesome_oscillator(dataframe)
    macd = ta.MACD(dataframe)
    dataframe['macd'] = macd['macd']
    dataframe['macdsignal'] = macd['macdsignal']
    dataframe['macdhist'] = macd['macdhist']
    hilbert = ta.HT_SINE(dataframe)
    dataframe['htsine'] = hilbert['sine']
    dataframe['htleadsine'] = hilbert['leadsine']
    dataframe['plus_dm'] = ta.PLUS_DM(dataframe)
    dataframe['plus_di'] = ta.PLUS_DI(dataframe)
    dataframe['minus_dm'] = ta.MINUS_DM(dataframe)
    dataframe['minus_di'] = ta.MINUS_DI(dataframe)

    # 记得始终返回 dataframe
    return dataframe
```

:::{warning} 想要更多指标示例？
请参考 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py)，取消注释你需要的指标。
:::

#### 指标库

Freqtrade 默认安装了以下技术指标库：

- [ta-lib](https://ta-lib.github.io/ta-lib-python/)
- [pandas-ta](https://twopirllc.github.io/pandas-ta/)
- [technical](https://technical.freqtrade.io)

如有需要可安装其他技术指标库，或自行编写自定义指标。

### 策略启动期

部分指标在启动初期因数据不足会出现 NaN 或计算不准确。Freqtrade 无法自动判断不稳定期长度，会直接用 dataframe 中的指标值。

为此，策略可设置 `startup_candle_count` 属性。

该值应设为策略中所有指标所需的最大历史K线数。若用到高阶时间周期 informative pair，`startup_candle_count` 也无需改变，只需用最大周期。

可用 [recursive-analysis](recursive-analysis.md) 检查合适的 `startup_candle_count`。当递归分析显示方差为 0% 时，说明历史数据已足够。

如本例策略用到 ema100，则应设为 400（`startup_candle_count = 400`），以保证 ema100 计算准确。

```python
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
```

告知机器人所需历史长度后，回测和超参优化可从指定时间点开始。

:::{warning} 多次获取 OHLCV 的警告
若收到 `WARNING - Using 3 calls to get OHLCV...`，请考虑是否真的需要这么多历史数据。过多会导致多次网络请求，拖慢机器人刷新速度。

最多允许 5 次请求，避免过载。
:::

:::{warning}
`startup_candle_count` 应小于 `ohlcv_candle_limit * 5`（大多数交易所为 500 * 5），Dry-Run/Live 模式下只会有这么多K线。
:::

#### 示例

假设用 EMA100 策略回测 2019 年 1 月的 5m K 线：

```bash
freqtrade backtesting --timerange 20190101-20190201 --timeframe 5m
```

若 `startup_candle_count` 设为 400，回测会自动加载 400 根K线的历史数据，即从 `20190101 - (400 * 5m)`，约等于 2018-12-30 11:40:00。

若有该数据，指标会用扩展时间段计算，启动期（到 2019-01-01 00:00:00）会被剔除。

:::{warning} 启动期数据不可用
若历史数据不足，回测会自动调整起始时间。例如本例会从 2019-01-02 09:20:00 开始。
:::

### 入场信号规则

编辑策略文件中的 `populate_entry_trend()` 方法，更新入场逻辑。

务必返回 dataframe，且不要删除/修改 "open", "high", "low", "close", "volume" 这几列，否则会导致异常。

该方法还需定义新列 `enter_long`（做空为 `enter_short`），入场时为 1，无操作为 0。即使只做空也必须设置 `enter_long`。

可用 `enter_tag` 列为信号命名，便于后续调试和分析。

示例（来自 sample_strategy.py）：

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    基于技术指标，为给定 dataframe 生成买入信号
    :param dataframe: 已填充指标的 DataFrame
    :param metadata: 额外信息，如当前交易对
    :return: 包含买入信号的 DataFrame
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI 上穿 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # 条件
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # 条件
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

:::{warning} 做空信号
做空可通过设置 `enter_short`（对应多头的 `enter_long`）实现。
`enter_tag` 列用法相同。
做空需交易所和市场支持！如需做空，务必设置 [`can_short`](#can-short) 为 True。

```python
# 支持多空
can_short = True

def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &
            (dataframe['tema'] <= dataframe['bb_middleband']) &
            (dataframe['tema'] > dataframe['tema'].shift(1)) &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    dataframe.loc[
        (
            (qtpylib.crossed_below(dataframe['rsi'], 70)) &
            (dataframe['tema'] > dataframe['bb_middleband']) &
            (dataframe['tema'] < dataframe['tema'].shift(1)) &
            (dataframe['volume'] > 0)
        ),
        ['enter_short', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

:::

:::{tip}
买入需要有卖方，因此需 `dataframe['volume'] > 0`，避免在无成交量时买入/卖出。
:::

### 出场信号规则

编辑策略文件中的 `populate_exit_trend()` 方法，更新出场逻辑。

可通过在配置或策略中设置 `use_exit_signal` 为 false 禁用出场信号。

`use_exit_signal` 不影响[信号冲突规则](#信号冲突)，后者仍然生效，可能阻止入场。

务必返回 dataframe，且不要删除/修改 "open", "high", "low", "close", "volume" 这几列，否则会导致异常。

该方法还需定义新列 `exit_long`（做空为 `exit_short`），出场时为 1，无操作为 0。

可用 `exit_tag` 列为信号命名，便于后续调试和分析。

示例（来自 sample_strategy.py）：

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    基于技术指标，为给定 dataframe 生成卖出信号
    :param dataframe: 已填充指标的 DataFrame
    :param metadata: 额外信息，如当前交易对
    :return: 包含卖出信号的 DataFrame
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # RSI 上穿 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # 条件
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # 条件
            (dataframe['volume'] > 0)  # 确保有成交量
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    return dataframe
```

:::{warning} 做空平仓
做空平仓可通过设置 `exit_short`（对应多头的 `exit_long`）实现。

`exit_tag` 列用法相同。

做空需交易所和市场支持！如需做空，务必设置 [`can_short`](#can-short) 为 True。

```python
    # 支持多空
    can_short = True

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 70)) &
                (dataframe['tema'] > dataframe['bb_middleband']) &
                (dataframe['tema'] < dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
        dataframe.loc[
            (
                (qtpylib.crossed_below(dataframe['rsi'], 30)) &
                (dataframe['tema'] < dataframe['bb_middleband']) &
                (dataframe['tema'] > dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            ['exit_short', 'exit_tag']] = (1, 'rsi_too_low')
        return dataframe
```

:::

### 最小收益率（ROI）

`minimal_roi` 策略变量定义了交易在退出前应达到的最小收益率，与出场信号无关。

格式如下，是一个 python 字典，键为开仓后经过的分钟数，值为百分比：

```python
minimal_roi = {
    "40": 0.0,
    "30": 0.01,
    "20": 0.02,
    "0": 0.04
}
```

上述配置含义：

- 达到 4% 盈利时退出
- 20 分钟后达到 2% 盈利时退出
- 30 分钟后达到 1% 盈利时退出
- 40 分钟后只要不亏损就退出

计算时包含手续费。

#### 禁用最小 ROI

如需完全禁用 ROI，将其设为空字典：

```python
minimal_roi = {}
```

#### ROI 中用计算表达式

如需按K线周期（timeframe）设置时间，可用如下写法：

```python
from freqtrade.exchange import timeframe_to_minutes

class AwesomeStrategy(IStrategy):

    timeframe = "1d"
    timeframe_mins = timeframe_to_minutes(timeframe)
    minimal_roi = {
        "0": 0.05,                      # 前 3 根 K 线 5%
        str(timeframe_mins * 3): 0.02,  # 3 根 K 线后 2%
        str(timeframe_mins * 6): 0.01,  # 6 根 K 线后 1%
    }
```

:::{note} 未立即成交的订单
`minimal_roi` 以 `trade.open_date` 为基准，即首次下单时间。

对于未立即成交的限价单（如用 `custom_entry_price()`），或用 `adjust_entry_price()` 调整价格的情况，时间仍以首次下单为准。
:::

### 止损

强烈建议设置止损，以保护资金免受极端行情影响。

如设置 10% 止损：

```python
stoploss = -0.10
```

更多止损功能详见[止损专页](stoploss.md)。

### 时间周期（Timeframe）

即策略用的K线周期。

常见值有 "1m"、"5m"、"15m"、"1h"，也可用交易所支持的其他周期。

同一入场/出场信号在不同周期下效果可能完全不同。

在策略方法中可通过 `self.timeframe` 访问。

### 是否支持做空

如需在合约市场做空，需设置 `can_short = True`。

启用后，策略在现货市场会加载失败。

如 `enter_short` 列有 1，但 `can_short = False`（默认），则即使配置了合约市场也不会做空。

### Metadata 字典

`metadata` 字典（`populate_entry_trend`、`populate_exit_trend`、`populate_indicators` 可用）包含额外信息。
目前有 `pair`，可通过 `metadata['pair']` 获取，如 `XRP/BTC`（合约市场为 `XRP/BTC:BTC`）。

metadata 不应被修改，也不会在策略函数间持久化。

如需持久化信息，请查阅[信息存储](strategy-advanced.md#storing-information-persistent)。

```{include} includes/strategy-imports.md
```

## 策略文件加载

默认情况下，freqtrade 会尝试从 `userdir`（默认 `user_data/strategies`）下所有 `.py` 文件加载策略。

假设你的策略叫 `AwesomeStrategy`，文件为 `user_data/strategies/AwesomeStrategy.py`，可用如下命令 dry（或 live，视配置而定）运行：

```bash
freqtrade trade --strategy AwesomeStrategy
```

注意这里用的是类名，不是文件名。

用 `freqtrade list-strategies` 可查看所有可加载的策略（正确目录下的所有策略）。会有"状态"字段，提示潜在问题。

:::{tip} 自定义策略目录
可用 `--strategy-path user_data/otherPath` 指定其他目录。所有需用策略的命令均支持此参数。
:::

## 辅助交易对（Informative Pairs）

(get-data-for-non-tradeable-pairs)=

### 获取非交易对的数据

有些策略需要参考更大周期的其他交易对数据。

这些辅助对的 OHLCV 数据会在常规白名单刷新时一并下载，可通过 `DataProvider` 获取。

这些对**不会**被交易，除非也在白名单或被动态白名单（如 `VolumePairlist`）选中。

需用元组指定，格式为 `("pair", "timeframe")`，第一个为交易对，第二个为周期。

示例：

```python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/TUSD", "15m"),
            ]
```

完整示例见[DataProvider 部分](#完整数据提供者示例)。

:::{warning}
这些对会随白名单刷新，建议列表不要太长。

只要交易所支持，所有周期和对都可用。

但建议尽量用重采样方式获取大周期，避免请求过多被封。
:::

:::{attention} 其他K线类型
informative_pairs 还可加第三个元素，显式指定K线类型。

可用性取决于交易模式和交易所。一般现货对不能用于合约市场，反之亦然。

详情见交易所文档。
:::

```python
    def informative_pairs(self):
        return [
            ("ETH/USDT", "5m", ""),   # 默认 K 线类型，随 trading_mode 变化（推荐）
            ("ETH/USDT", "5m", "spot"),   # 强制用现货 K 线（仅现货机器人）
            ("BTC/TUSD", "15m", "futures"),  # 用合约 K 线（仅合约机器人）
            ("BTC/TUSD", "15m", "mark"),  # 用标记 K 线（仅合约机器人）
        ]
```

### Informative pairs 装饰器（`@informative()`）

可用 `@informative` 装饰器快速定义 informative pair。所有被装饰的 `populate_indicators_*` 方法独立运行，不能访问其他 informative pair 的数据。但所有 informative dataframe 会合并传递给主 `populate_indicators()`。

:::{caution}
若需用一个 informative pair 的数据生成另一个 informative pair，请勿用 `@informative` 装饰器，而应手动定义 informative pair，见[DataProvider 部分](#完整数据提供者示例)。
:::

超参优化时，hyperoptable 参数不支持 `.value`，请用 `.range`。见[优化指标参数](hyperopt.md#optimizing-an-indicator-parameter)。

:::{note} 完整文档

```python
    def informative(
        timeframe: str,
        asset: str = "",
        fmt: str | Callable[[Any], str] | None = None,
        *,
        candle_type: CandleType | str | None = None,
        ffill: bool = True,
    ) -> Callable[[PopulateIndicators], PopulateIndicators]:
        """
        用于 populate_indicators_Nn(self, dataframe, metadata) 的装饰器，定义 informative 指标。
        
        用法示例：

            @informative('1h')
            def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
                dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
                return dataframe

        :param timeframe: 辅助K线周期，必须大于等于主策略周期。
        :param asset: 辅助资产，如 BTC、BTC/USDT、ETH/BTC。不指定则用当前对。支持格式化字符串。
        :param fmt: 列名格式（str）或格式化函数。默认：
        * {base}_{quote}_{column}_{timeframe}（指定 asset 时）
        * {column}_{timeframe}（未指定 asset 时）
        支持变量：{base}、{BASE}、{quote}、{QUOTE}、{asset}、{column}、{timeframe}
        :param ffill: informative pair 合并后是否前向填充。
        :param candle_type: '', mark, index, premiumIndex, funding_rate
        """
```

:::

:::{hint} 快速定义 informative pair

大多数情况下无需用 `merge_informative_pair()`，可直接用装饰器：

```python

    from datetime import datetime
    from freqtrade.persistence import Trade
    from freqtrade.strategy import IStrategy, informative

    class AwesomeStrategy(IStrategy):
        
        # 可省略 informative_pairs 方法

        # 为每个对定义高阶周期 informative pair。可叠加装饰器。
        @informative('30m')
        @informative('1h')
        def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # 定义 BTC/STAKE informative pair。可用 {stake} 占位符。最终列名如 'btc_usdt_rsi_1h'。
        @informative('1h', 'BTC/{stake}')
        def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # 定义 BTC/ETH informative pair。需指定 quote。最终列名如 'eth_btc_rsi_1h'。
        @informative('1h', 'ETH/BTC')
        def populate_indicators_eth_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        # 可自定义列名格式
        @informative('1h', 'BTC/{stake}', '{column}_{timeframe}')
        def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi_upper'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
            return dataframe
```

:::

:::{attention}
访问其他对的 informative dataframe 时建议用字符串格式化，便于后续更换 stake 货币。

```python
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        stake = self.config['stake_currency']
        dataframe.loc[
            (
                (dataframe[f'btc_{stake}_rsi_1h'] < 35)
                &
                (dataframe['volume'] > 0)
            ),
            ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')
    
        return dataframe
```

或用列重命名去除 stake 货币：`@informative('1h', 'BTC/{stake}', fmt='{base}_{column}_{timeframe}')`。
:::

:::{warning} 方法名重复

被 `@informative()` 装饰的方法名必须唯一！重复会覆盖前面的定义且无报错，导致部分指标不可用。请仔细检查方法名。
:::

### *merge_informative_pair()*

该方法可安全、无前视偏差地将 informative pair 合并到主 dataframe。

功能：

- 列重命名，避免冲突
- 无前视偏差合并
- 可选前向填充

完整示例见[完整数据提供者示例](#完整数据提供者示例)。

所有 informative dataframe 列会以新名字出现在主 dataframe：

:::{important} 列重命名

假设 `inf_tf = '1d'`，合并后列名如下：

```python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # 原 dataframe
    'date_1d', 'open_1d', 'high_1d', 'low_1d', 'close_1d', 'rsi_1d'   # informative dataframe
```

:::

:::{important} 1h 列重命名

假设 `inf_tf = '1h'`，合并后列名如下：

```python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # 原 dataframe
    'date_1h', 'open_1h', 'high_1h', 'low_1h', 'close_1h', 'rsi_1h'   # informative dataframe
```

:::

:::{important} 自定义实现
可自定义实现如下：

```python
    # 日期右移 1 根K线
    # This is necessary since the data is always the "open date"
    # and a 15m candle starting at 12:15 should not know the close of the 1h candle from 12:00 to 13:00
    minutes = timeframe_to_minutes(inf_tf)
    # Only do this if the timeframes are different:
    informative['date_merge'] = informative["date"] + pd.to_timedelta(minutes, 'm')

    # Rename columns to be unique
    informative.columns = [f"{col}_{inf_tf}" for col in informative.columns]
    # Assuming inf_tf = '1d' - then the columns will now be:
    # date_1d, open_1d, high_1d, low_1d, close_1d, rsi_1d

    # Combine the 2 dataframes
    # all indicators on the informative sample MUST be calculated before this point
    dataframe = pd.merge(dataframe, informative, left_on='date', right_on=f'date_merge_{inf_tf}', how='left')
    # FFill to have the 1d value available in every row throughout the day.
    # Without this, comparisons would only work once per day.
    dataframe = dataframe.ffill()
```
:::

:::{warning} informative 周期 < 主周期
用本方法时，不建议 informative 周期小于主 dataframe 周期，否则无法利用更多细节。若需用更细粒度信息，需用更高级方法（超出本文档范围）。
:::

## 额外数据（DataProvider）

策略可通过 `DataProvider` 获取更多数据。

所有方法失败时返回 None，不抛异常。

请根据运行模式选择合适方法（见下文示例）。

:::{warning} 超参优化限制
DataProvider 在超参优化时可用，但仅限于策略内的 `populate_indicators()`，不可用于超参优化类文件。

也不可用于 `populate_entry_trend()` 和 `populate_exit_trend()`。
:::

### DataProvider 可用方法

- [`available_pairs`](#available_pairs) - 返回缓存的所有对及其周期（pair, timeframe）元组
- [`current_whitelist()`](#current_whitelist) - 返回当前白名单对列表，适合动态白名单（如 VolumePairlist）
- [`get_pair_dataframe(pair, timeframe)`](#get_pair_dataframepair-timeframe) - 通用方法，回测时返回历史数据，dry/live 时返回缓存数据
- [`get_analyzed_dataframe(pair, timeframe)`](#get_analyzed_dataframepair-timeframe) - 返回分析后的 dataframe 及最新分析时间
- `historic_ohlcv(pair, timeframe)` - 返回磁盘上的历史数据
- `market(pair)` - 返回交易对市场数据（手续费、限额、精度、活跃标志等）
- `ohlcv(pair, timeframe)` - 返回当前缓存的K线数据
- [`orderbook(pair, maximum)`](#orderbookpair-maximum) - 返回最新 orderbook 数据（bids/asks）
- [`ticker(pair)`](#tickerpair) - 返回当前 ticker 数据
- `runmode` - 当前运行模式

### 示例用法

### *available_pairs*

```python
for pair, timeframe in self.dp.available_pairs:
    print(f"available {pair}, {timeframe}")
```

### *current_whitelist()*

假设你开发了一个用 `5m` 周期、用 `1d` 周期信号的策略，且用 VolumePairList 动态选前 10 个交易对。

策略逻辑如下：

*每 5 分钟扫描前 10 个交易对，用 14 日 RSI 生成信号。*

因数据有限，无法用 5m K 线重采样成日线。大多数交易所只给 500-1000 根K线，约等于 1.74 天。我们需要至少 14 天！

因此需用 informative pair，且因白名单动态变化，不知用哪些对！

此时可用 `self.dp.current_whitelist()` 获取当前白名单对。

```python
    def informative_pairs(self):

        # 获取白名单所有对
        pairs = self.dp.current_whitelist()
        # 为每个对分配周期
        informative_pairs = [(pair, '1d') for pair in pairs]
        return informative_pairs
```

:::{caution} current_whitelist 用于绘图
current_whitelist 不支持 `plot-dataframe`，也不支持 FreqUI 可视化（webserver 模式），因 webserver 配置无需白名单。
:::

### *get_pair_dataframe(pair, timeframe)*

```python
# 获取第一个 informative pair 的K线数据
inf_pair, inf_timeframe = self.informative_pairs()[0]
informative = self.dp.get_pair_dataframe(pair=inf_pair,
                                         timeframe=inf_timeframe)
```

:::{warning} 回测注意
回测时，`dp.get_pair_dataframe()` 在 `populate_*()` 方法中返回完整时间区间。请勿"看未来"，否则 dry/live 时会有意外。

在 [回调函数](strategy-callbacks.md) 中，则只返回当前（模拟）K线前的数据。
:::

### *get_analyzed_dataframe(pair, timeframe)*

该方法供 freqtrade 内部用来判断最后信号，也可在特定回调中用来获取触发操作的信号（见[高级策略文档](strategy-advanced.md)）。

```python
# 获取当前 dataframe
dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=metadata['pair'],
                                                         timeframe=self.timeframe)
```

:::{caution} 无数据
若请求的对未缓存，返回空 dataframe。可用 `if dataframe.empty:` 检查。
:::

### *orderbook(pair, maximum)*

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ob = self.dp.orderbook(metadata['pair'], 1)
    dataframe['best_bid'] = ob['bids'][0][0]
    dataframe['best_ask'] = ob['asks'][0][0]
```

orderbook 结构与 [ccxt](https://github.com/ccxt/ccxt/wiki/Manual#order-book-structure) 一致：

``` js
{
    'bids': [
        [ price, amount ],
        ...
    ],
    'asks': [
        [ price, amount ],
        ...
    ],
}
```

用 `ob['bids'][0][0]` 可取最优买价，`ob['bids'][0][1]` 为该价位数量。

:::{warning} 回测注意
订单簿不是历史数据的一部分，这意味着如果使用此方法，回测和超参数优化将无法正常工作，因为该方法将返回最新值。
:::

### *ticker(pair)*

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ticker = self.dp.ticker(metadata['pair'])
    dataframe['last_price'] = ticker['last']
    dataframe['volume24h'] = ticker['quoteVolume']
    dataframe['vwap'] = ticker['vwap']
```

:::{warning}
虽然 ticker 数据结构是 ccxt 统一接口的一部分，但不同交易所返回的值可能会有所不同。例如，许多交易所不返回 `vwap` 值，一些交易所并不总是填充 `last` 字段（所以它可能为 None）等。因此，您需要仔细验证从交易所返回的 ticker 数据，并添加适当的错误处理/默认值。
:::

:::{warning} 回测警告
此方法将始终返回最新/实时值。因此，在回测/超参数优化期间使用而不检查运行模式将导致错误的结果，例如，您的整个 dataframe 将在所有行中包含相同的单个值。
:::

### 发送通知

dataprovider 的 `.send_msg()` 可在策略中发送自定义通知。
相同的通知在每个蜡烛图期间只会发送一次，除非第二个参数（`always_send`）设置为 True。

```python
    self.dp.send_msg(f"{metadata['pair']} just got hot!")

    # 强制发送此通知，避免缓存（请阅读下面的警告！）
    self.dp.send_msg(f"{metadata['pair']} just got hot!", always_send=True)
```

通知只会在交易模式（实盘/模拟盘）中发送 - 因此可以在回测中无条件调用此方法。

:::{warning} 消息轰炸
通过在此方法中设置 `always_send=True`，您可能会给自己发送大量消息。请谨慎使用此功能，并且只在您知道不会在整个蜡烛图期间频繁触发的情况下使用，以避免每5秒就收到一条消息。
:::

### 完整 DataProvider 示例

```python
from freqtrade.strategy import IStrategy, merge_informative_pair
from pandas import DataFrame

class SampleStrategy(IStrategy):
    # 策略初始化...

    timeframe = '5m'

    # ...

    def informative_pairs(self):

        # 获取白名单所有对
        pairs = self.dp.current_whitelist()
        # 为每个对分配周期
        informative_pairs = [(pair, '1d') for pair in pairs]
        # 可选：添加静态对
        informative_pairs += [("ETH/USDT", "5m"),
                              ("BTC/TUSD", "15m"),
                            ]
        return informative_pairs

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        if not self.dp:
            # DataProvider 不可用时不做任何操作
            return dataframe

        inf_tf = '1d'
        # 获取信息对
        informative = self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe=inf_tf)
        # 获取14天RSI
        informative['rsi'] = ta.RSI(informative, timeperiod=14)

        # 使用辅助函数 merge_informative_pair 安全地合并交易对
        # 自动重命名列并合并较短时间周期的 dataframe 和较长时间周期的信息对
        # 使用 ffill 使1天的值在一天中的每一行都可用
        # 如果没有这个，原始数据框和信息对的列之间的比较每天只能进行一次
        # 此方法的完整文档，见下文
        dataframe = merge_informative_pair(dataframe, informative, self.timeframe, inf_tf, ffill=True)

        # 计算原始数据框的RSI（5分钟时间周期）
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        # 其他操作
        # ...

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # RSI 上穿 30
                (dataframe['rsi_1d'] < 30) &                     # 日线 RSI < 30
                (dataframe['volume'] > 0)                        # 有成交量
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

```

***

## 额外数据（钱包 Wallets）

策略可通过 `wallets` 对象获取当前钱包/账户余额。

:::{note} 回测/超参优化
Wallets 在不同函数中表现不同。

在 `populate_*()` 方法中，返回完整钱包。

在 [回调函数](strategy-callbacks.md) 中，返回当前模拟钱包状态。
:::

调用前请检查 `wallets` 是否可用，避免回测时报错。

```python
if self.wallets:
    free_eth = self.wallets.get_free('ETH')
    used_eth = self.wallets.get_used('ETH')
    total_eth = self.wallets.get_total('ETH')
```

### Wallets 可用方法

- `get_free(asset)` - 当前可用余额
- `get_used(asset)` - 当前被占用余额（挂单）
- `get_total(asset)` - 总余额（前两者之和）

***

## 额外数据（交易记录 Trades）

可在策略中通过数据库查询历史交易。

在文件顶部，导入所需对象：

```python
from freqtrade.persistence import Trade
```

以下示例查询当天当前对的已平仓交易（可加其他过滤条件）：

```python
trades = Trade.get_trades_proxy(pair=metadata['pair'],
                                open_date=datetime.now(timezone.utc) - timedelta(days=1),
                                is_open=False,
            ]).order_by(Trade.close_date).all()
# 汇总该交易对的利润
curdayprofit = sum(trade.close_profit for trade in trades)
```

更多方法请查阅 [Trade 对象](trade-object.md) 文档。

:::{warning}
回测/超参优化时，`populate_*` 方法中无法获取交易历史，结果为空。
:::

## 阻止特定对交易

Freqtrade 会在某对平仓后自动锁定该对至当前K线结束，防止同一K线内频繁交易。

锁定对时会提示 `Pair <pair> is currently locked.`。

### 在策略中锁定对

有时希望在特定事件后锁定某对（如连续亏损）。

可用 `self.lock_pair(pair, until, [reason])` 实现，`until` 为未来时间点，`reason` 为可选说明。

可用 `self.unlock_pair(pair)` 或 `self.unlock_reason(<reason>)` 手动解锁，后者会解锁所有因该原因锁定的对。

用 `self.is_pair_locked(pair)` 检查对是否被锁定。

:::{note}
锁定会自动延长到下一个K线结束。如 5m 周期，锁到 10:18 会延长到 10:20。
:::

:::{warning}
手动锁定仅在实盘/模拟盘可用，回测时仅保护机制可锁定。
:::

#### 锁定对示例

```python
from freqtrade.persistence import Trade
from datetime import timedelta, datetime, timezone
# 放在策略文件顶部
# --------

# 在 populate_indicators 或 populate_entry_trend 中：
if self.config['runmode'].value in ('live', 'dry_run'):
    # 查询近两天已平仓交易
    trades = Trade.get_trades_proxy(
        pair=metadata['pair'], is_open=False, 
        open_date=datetime.now(timezone.utc) - timedelta(days=2))
    # 分析是否需要锁定
    sumprofit = sum(trade.close_profit for trade in trades)
    if sumprofit < 0:
        # 锁定 12 小时
        self.lock_pair(metadata['pair'], until=datetime.now(timezone.utc) + timedelta(hours=12))
```

## 打印主 dataframe

可在 `populate_entry_trend()` 或 `populate_exit_trend()` 中打印当前主 dataframe，便于调试。

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            #>> 你的条件 <<<
        ),
        ['enter_long', 'enter_tag']] = (1, 'somestring')

    # 打印分析的对
    print(f"result for {metadata['pair']}")

    # 打印最后 5 行
    print(dataframe.tail())

    return dataframe
```

如需打印更多行可用 `print(dataframe)`，但不建议，否则每对每 5 秒会输出约 500 行。

## 开发策略时的常见错误

### 回测时"看未来"

回测为提升性能会一次性分析整个 dataframe。策略作者需确保不"看未来"，即不使用 dry/live 时不可用的数据。

这是常见痛点，易导致回测与 dry/live 巨大差异。看未来的策略回测时表现极佳，实盘却很差。

常见错误包括：

- 不要用 `shift(-1)` 或负数，这会用到未来数据
- 不要在 `populate_` 函数中用 `.iloc[-1]` 或其他绝对位置，dry-run 和回测时表现不同。回调函数中可安全用 iloc
- 不要用 `dataframe['mean_volume'] = dataframe['volume'].mean()` 这类全列函数，回测时会用到未来数据。应用 rolling() 计算，如 `dataframe['volume'].rolling(<window>).mean()`
- 不要用 `.resample('1h')`，这会用左边界，应用 `.resample('1h', label='right')`
- 不要用 `.merge()` 合并大周期到小周期，应用 [informative pair](#辅助交易对) 工具。普通 merge 会有前视偏差

:::{tip} 如何发现问题
建议用 [lookahead-analysis](lookahead-analysis.md) 和 [recursive-analysis](recursive-analysis.md) 两个工具辅助排查。

但两者都不能保证完全无误。
:::

### 信号冲突

当信号冲突（如 `enter_long` 和 `exit_long` 同时为 1）时，freqtrade 会忽略入场信号，避免刚进场就立刻出场。

规则如下，若 3 个信号中有多个为 1，则忽略入场：

- `enter_long` -> `exit_long`, `enter_short`
- `enter_short` -> `exit_short`, `enter_long`

## 更多策略思路

如需更多策略思路，请访问 [策略仓库](https://github.com/freqtrade/freqtrade-strategies)。可作为学习参考，实际效果取决于市场、交易对等。请务必先回测，再 dry-run，谨慎实盘。

欢迎参考、改编，也欢迎向仓库提交新策略 PR。

## 下一步

现在你已经有了完美的策略，下一步请学习[如何回测](backtesting.md)。
