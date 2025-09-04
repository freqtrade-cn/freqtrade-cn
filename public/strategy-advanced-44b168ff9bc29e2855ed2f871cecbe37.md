---
title: Freqtrade 高级策略指南
subject: Freqtrade 策略开发
subtitle: 高级策略开发概念和方法
short_title: 高级策略
description: 本文档详细介绍了 Freqtrade 交易机器人的高级策略开发概念,包括持久化存储、回调方法等进阶功能。
---

# 高级策略

本页解释了一些可用于策略的高级概念。
如果你是初学者，请先熟悉 [Freqtrade 基础](bot-basics.md) 和 [策略定制](strategy-customization.md) 中描述的方法。

这里描述的方法的调用顺序在 [机器人执行逻辑](bot-basics.md#bot-execution-logic) 中有详细说明。这些文档也有助于决定哪种方法最适合你的定制需求。

:::{note}
回调方法应该*只*在策略使用它们时实现。
:::

:::{tip}
通过运行 `freqtrade new-strategy --strategy MyAwesomeStrategy --template advanced` 来获取包含所有可用回调方法的策略模板
:::

## 存储信息（持久化）

Freqtrade 允许在数据库中存储/检索与特定交易相关的用户自定义信息。

使用交易对象，可以使用 `trade.set_custom_data(key='my_key', value=my_value)` 存储信息，使用 `trade.get_custom_data(key='my_key')` 检索信息。每个数据条目都与一个交易和一个用户提供的键（类型为 `string`）相关联。这意味着这只能在也提供交易对象的回调中使用。

为了使数据能够存储在数据库中，freqtrade 必须序列化数据。这是通过将数据转换为 JSON 格式的字符串来完成的。
Freqtrade 将在检索时尝试反转此操作，因此从策略的角度来看，这应该无关紧要。

```python
from freqtrade.persistence import Trade
from datetime import timedelta

class AwesomeStrategy(IStrategy):

    def bot_loop_start(self, **kwargs) -> None:
        for trade in Trade.get_open_order_trades():
            fills = trade.select_filled_orders(trade.entry_side)
            if trade.pair == 'ETH/USDT':
                trade_entry_type = trade.get_custom_data(key='entry_type')
                if trade_entry_type is None:
                    trade_entry_type = 'breakout' if 'entry_1' in trade.enter_tag else 'dip'
                elif fills > 1:
                    trade_entry_type = 'buy_up'
                trade.set_custom_data(key='entry_type', value=trade_entry_type)
        return super().bot_loop_start(**kwargs)

    def adjust_entry_price(self, trade: Trade, order: Order | None, pair: str,
                           current_time: datetime, proposed_rate: float, current_order_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:
        # 对于 BTC/USDT 交易对，在入场触发后的前 10 分钟内使用限价单并跟随 SMA200 作为价格目标。
        if (
            pair == 'BTC/USDT' 
            and entry_tag == 'long_sma200' 
            and side == 'long' 
            and (current_time - timedelta(minutes=10)) > trade.open_date_utc 
            and order.filled == 0.0
        ):
            dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
            current_candle = dataframe.iloc[-1].squeeze()
            # 存储入场调整信息
            existing_count = trade.get_custom_data('num_entry_adjustments', default=0)
            if not existing_count:
                existing_count = 1
            else:
                existing_count += 1
            trade.set_custom_data(key='num_entry_adjustments', value=existing_count)

            # 调整订单价格
            return current_candle['sma_200']

        # 默认：维持现有订单
        return current_order_rate

    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float, current_profit: float, **kwargs):

        entry_adjustment_count = trade.get_custom_data(key='num_entry_adjustments')
        trade_entry_type = trade.get_custom_data(key='entry_type')
        if entry_adjustment_count is None:
            if current_profit > 0.01 and (current_time - timedelta(minutes=100) > trade.open_date_utc):
                return True, 'exit_1'
        else
            if entry_adjustment_count > 0 and if current_profit > 0.05:
                return True, 'exit_2'
            if trade_entry_type == 'breakout' and current_profit > 0.1:
                return True, 'exit_3

        return False, None
```

上面是一个简单的例子 - 有更简单的方法来检索交易数据，如入场调整。

:::{note}
建议使用简单的数据类型 `[bool, int, float, str]` 以确保序列化需要存储的数据时不会出现问题。

存储大量数据可能会导致意外的副作用，如数据库变大（因此也会变慢）。
:::

:::{warning} 不可序列化的数据
如果提供的数据无法序列化，将记录警告，并且指定 `key` 的条目将包含 `None` 作为数据。
:::

:::{note} 所有属性
通过 Trade 对象（下面假设为 `trade`）可以访问以下自定义数据访问器：

* `trade.get_custom_data(key='something', default=0)` - 返回以提供的类型给出的实际值。
* `trade.get_custom_data_entry(key='something')` - 返回条目 - 包括元数据。可以通过 `.value` 属性访问值。
* `trade.set_custom_data(key='something', value={'some': 'value'})` - 设置或更新此交易的相应键。值必须是可序列化的 - 我们建议保持存储的数据相对较小。

"value" 可以是任何类型（在设置和接收时都是）- 但必须是 json 可序列化的。
:::

## 存储信息（非持久化）

:::{warning} 已弃用
这种存储信息的方法已弃用，我们建议不要使用非持久化存储。
请改用[持久化存储](#storing-information-persistent)。

因此其内容已被折叠。
:::

:::{attention} 存储信息
可以通过在策略类中创建新字典来存储信息。

变量的名称可以随意选择，但应该以 `custom_` 为前缀，以避免与预定义的策略变量发生命名冲突。

```python
class AwesomeStrategy(IStrategy):
    # 创建自定义字典
    custom_info = {}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # 检查条目是否已存在
        if not metadata["pair"] in self.custom_info:
            # 为此交易对创建空条目
            self.custom_info[metadata["pair"]] = {}

        if "crosstime" in self.custom_info[metadata["pair"]]:
            self.custom_info[metadata["pair"]]["crosstime"] += 1
        else:
            self.custom_info[metadata["pair"]]["crosstime"] = 1
```
:::

:::{warning}
数据在机器人重启（或配置重载）后不会持久化。此外，数据量应该保持较小（不要使用 DataFrame 等），否则机器人将开始消耗大量内存，最终耗尽内存并崩溃。
:::

:::{note}
如果数据是特定于交易对的，请确保在字典中使用交易对作为键之一。
:::


## 数据框访问

你可以通过从数据提供者查询来在各种策略函数中访问数据框。

```python
from freqtrade.exchange import timeframe_to_prev_date

class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: 'Trade', order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: 'datetime', **kwargs) -> bool:
        # 获取交易对数据框。
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)

        # 获取最后一个可用的蜡烛图。不要使用 current_time 查找最新的蜡烛图，因为
        # current_time 指向当前未完成的蜡烛图，其数据不可用。
        last_candle = dataframe.iloc[-1].squeeze()
        # <...>

        # 在模拟/实盘运行中，交易开始日期不会匹配蜡烛图开始日期，因此必须
        # 进行四舍五入。
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        # 查找交易蜡烛图。
        trade_candle = dataframe.loc[dataframe['date'] == trade_date]
        # 对于刚刚开始的交易，trade_candle 可能为空，因为它仍然未完成。
        if not trade_candle.empty:
            trade_candle = trade_candle.squeeze()
            # <...>
```

:::{warning} 使用 .iloc[-1]
你可以在这里使用 `.iloc[-1]`，因为 `get_analyzed_dataframe()` 只返回回测允许看到的蜡烛图。

这在 `populate_*` 方法中不起作用，所以确保不要在该区域使用 `.iloc[]`。

此外，这只能从 2021.5 版本开始工作。
:::

## 入场标签

当你的策略有多个买入信号时，你可以命名触发的信号。
然后你可以在 `custom_exit` 中访问你的买入信号

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] < 35) &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe

def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                current_profit: float, **kwargs):
    dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
    last_candle = dataframe.iloc[-1].squeeze()
    if trade.enter_tag == 'buy_signal_rsi' and last_candle['rsi'] > 80:
        return 'sell_signal_rsi'
    return None

```

:::{note}
`enter_tag` 限制为 100 个字符，剩余数据将被截断。
:::

:::{warning}
只有一个 `enter_tag` 列，用于做多和做空交易。

因此，必须将此列视为"最后写入获胜"（毕竟它只是一个数据框列）。

在复杂的情况下，当多个信号冲突（或如果信号基于不同条件再次停用）时，这可能导致奇怪的结果，错误的标签应用于入场信号。

这些结果是策略覆盖先前标签的结果 - 最后一个标签将"粘住"，并将是 freqtrade 使用的标签。
:::

## 出场标签

类似于[入场标签](#enter-tag)，你也可以指定出场标签。

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] > 70) &
            (dataframe['volume'] > 0)
        ),
        ['exit_long', 'exit_tag']] = (1, 'exit_rsi')

    return dataframe
```

提供的出场标签然后用作卖出原因 - 并在回测结果中显示为如此。

:::{note}
`exit_reason` 限制为 100 个字符，剩余数据将被截断。
:::

## 策略版本

你可以通过使用 "version" 方法来实现自定义策略版本控制，并返回你希望此策略具有的版本。

```python
def version(self) -> str:
    """
    返回策略的版本。
    """
    return "1.1"
```

:::{note}
你应该确保同时实现适当的版本控制（如 git 仓库），因为 freqtrade 不会保留你的策略的历史版本，所以由用户负责能够最终回滚到策略的先前版本。
:::

## 派生策略

策略可以从其他策略派生。这避免了自定义策略代码的重复。你可以使用这种技术来覆盖主策略的小部分，保持其余部分不变：

```python title="user_data/strategies/myawesomestrategy.py"
class MyAwesomeStrategy(IStrategy):
    ...
    stoploss = 0.13
    trailing_stop = False
    # 所有其他属性和方法都在这里，就像
    # 在任何自定义策略中一样...
    ...

```

```python title="user_data/strategies/MyAwesomeStrategy2.py"
from myawesomestrategy import MyAwesomeStrategy
class MyAwesomeStrategy2(MyAwesomeStrategy):
    # 覆盖某些内容
    stoploss = 0.08
    trailing_stop = True
```

属性和方法都可以被覆盖，以你需要的方式改变原始策略的行为。

虽然在技术上可以在同一文件中保持子类，但这可能会导致超优化参数文件的一些问题，因此我们建议使用单独的策略文件，并如上所示导入父策略。

## 嵌入策略

Freqtrade 为你提供了一种简单的方法来将策略嵌入到配置文件中。
这是通过利用 BASE64 编码并在你选择的配置文件的策略配置字段中提供此字符串来完成的。

### 将字符串编码为 BASE64

这是一个快速示例，如何在 python 中生成 BASE64 字符串

```python
from base64 import urlsafe_b64encode

with open(file, 'r') as f:
    content = f.read()
content = urlsafe_b64encode(content.encode('utf-8'))
```

变量 'content' 将包含 BASE64 编码形式的策略文件。现在可以在配置文件中设置如下

```json
"strategy": "NameOfStrategy:BASE64String"
```

请确保 'NameOfStrategy' 与策略名称完全相同！

## 性能警告

在执行策略时，有时会在日志中看到以下内容

> PerformanceWarning: DataFrame is highly fragmented.

这是来自 [`pandas`](https://github.com/pandas-dev/pandas) 的警告，正如警告继续说的：
使用 `pd.concat(axis=1)`。
这可能会有轻微的性能影响，通常只在超优化期间可见（在优化指标时）。

例如：

```python
for val in self.buy_ema_short.range:
    dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
```

应该重写为

```python
frames = [dataframe]
for val in self.buy_ema_short.range:
    frames.append(DataFrame({
        f'ema_short_{val}': ta.EMA(dataframe, timeperiod=val)
    }))

# 组合所有数据框，并重新分配原始数据框列
dataframe = pd.concat(frames, axis=1)
```

然而，Freqtrade 通过在 `populate_indicators()` 方法之后立即运行 `dataframe.copy()` 来抵消这一点 - 因此这的性能影响应该很低或不存在。
