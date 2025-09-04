---
title: 策略回调指南
subject: Freqtrade 策略回调文档
subtitle: 策略回调函数的完整概述
short_title: 策略回调
description: 本文档详细介绍了 Freqtrade 交易机器人中可用的各种策略回调函数,包括它们的用途、调用时机以及使用方法。
---

# 策略回调

虽然主要的策略函数（`populate_indicators()`, `populate_entry_trend()`, `populate_exit_trend()`) ）应以向量化方式使用，并且只会在[回测时调用一次](bot-basics.md#backtesting-hyperopt-execution-logic)，但回调函数会在"需要时"被调用。

因此，你应避免在回调中进行繁重的计算，以免在操作过程中造成延迟。

根据所用回调的不同，它们可能在进入/退出交易时调用，或在交易期间多次调用。
当前可用的回调有：

* [机器人启动 `bot_start()`](#bot-start)
* [机器人循环开始 `bot_loop_start()`](#bot-loop-start)
* [仓位大小管理 `custom_stake_amount()`](#stake-size-management)
* [自定义退出信号 `custom_exit()`](#custom-exit-signal)
* [自定义止损 `custom_stoploss()`](#custom-stoploss)
* [`custom_entry_price()` 和 `custom_exit_price()`](#custom-order-price-rules)
* [`check_entry_timeout()` 和 `check_exit_timeout()`](#custom-order-timeout-rules)
* [`confirm_trade_entry()`](#trade-entry-buy-order-confirmation)
* [`confirm_trade_exit()`](#trade-exit-sell-order-confirmation)
* [`adjust_trade_position()`](#adjust-trade-position)
* [`adjust_entry_price()`](#adjust-entry-price)
* [`leverage()`](#leverage-callback)
* [`order_filled()`](#order-filled-callback)

:::{tip} 回调调用顺序
你可以在 [bot-basics](bot-basics.md#bot-execution-logic) 中找到回调的调用顺序
:::

```{include} includes/strategy-imports.md
```

```{include} includes/strategy-exit-comparisons.md
```

(bot-start)=

## 机器人启动

一个简单的回调，在策略加载时只调用一次。
这可用于执行只需执行一次的操作，并在数据供应商 (dataprovider) 和钱包设置后运行。

```python
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def bot_start(self, **kwargs) -> None:
        """
        Called only once after bot instantiation.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # Assign this to the class by using self.*
            # can then be used by populate_* methods
            self.custom_remote_data = requests.get("https://some_remote_source.example.com")

```

在 hyperopt 时，这个回调只会在启动时运行一次。

(bot-loop-start)=

## 机器人循环开始

一个简单的回调，在每次机器人节流循环开始时调用一次（在 dry/live 模式下大约每 5 秒，除非另有配置），或在回测/hyperopt 模式下每根 K 线调用一次。
这可用于执行与交易对无关的计算（适用于所有交易对）、加载外部数据等。

```python
# Default imports
import requests

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def bot_loop_start(self, current_time: datetime, **kwargs) -> None:
        """
        Called at the start of the bot iteration (one loop).
        Might be used to perform pair-independent tasks
        (e.g. gather some remote resource for comparison)
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        if self.config["runmode"].value in ("live", "dry_run"):
            # Assign this to the class by using self.*
            # can then be used by populate_* methods
            self.remote_data = requests.get("https://some_remote_source.example.com")

```

(stake-size-management)=

## 仓位大小管理

在进入交易前调用，使你可以在下新单时管理仓位大小。

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
        current_candle = dataframe.iloc[-1].squeeze()

        if current_candle["fastk_rsi_1h"] > current_candle["fastd_rsi_1h"]:
            if self.config["stake_amount"] == "unlimited":
                # Use entire available wallet during favorable conditions when in compounding mode.
                return max_stake
            else:
                # Compound profits during favorable conditions instead of using a static stake.
                return self.wallets.get_total_stake_amount() / self.config["max_open_trades"]

        # Use default stake amount.
        return proposed_stake
```

如果你的代码抛出异常，Freqtrade 会回退到 `proposed_stake` 的值。异常本身会被记录。

:::{tip}
你不必确保 `min_stake <= returned_value <= max_stake`。交易会成功，因为返回值会被限制在支持的范围内，并且此操作会被记录。
:::

:::{tip}
返回 `0` 或 `None` 会阻止下单。
:::

(custom-exit-signal)=

## 自定义退出信号

在每次节流迭代（大约每 5 秒）时对开放交易调用，直到交易关闭。

允许定义自定义退出信号，指示应关闭指定仓位（完全退出）。这在需要为每个单独的交易自定义退出条件，或者需要交易数据来做出退出决定时非常有用。

例如，你可以使用 `custom_exit()` 实现 1:2 的风险回报比。

不过，使用 `custom_exit()` 信号代替止损 *不推荐*。在这方面，它不如使用 `custom_stoploss()` 有效，后者还允许你在交易所保持止损。

:::{note}
从此方法返回一个（非空）`string` 或 `True` 等同于在指定时间设置退出信号。如果已经设置了退出信号，或者退出信号被禁用（`use_exit_signal=False`），则不会调用此方法。`string` 的最大长度为 64 个字符。超过此限制将导致消息被截断为 64 个字符。

`custom_exit()` 将忽略 `exit_profit_only`，并且除非 `use_exit_signal=False`，否则始终会被调用，即使有新的进入信号。
:::

一个示例，说明如何根据当前利润使用不同的指标，并退出已开放超过一天的交易：

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                    current_profit: float, **kwargs):
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # Above 20% profit, sell when rsi < 80
        if current_profit > 0.2:
            if last_candle["rsi"] < 80:
                return "rsi_below_80"

        # Between 2% and 10%, sell if EMA-long above EMA-short
        if 0.02 < current_profit < 0.1:
            if last_candle["emalong"] > last_candle["emashort"]:
                return "ema_long_below_80"

        # Sell any positions at a loss if they are held for more than one day.
        if current_profit < 0.0 and (current_time - trade.open_date_utc).days >= 1:
            return "unclog"
```

请参阅 [Dataframe 访问](strategy-advanced.md#dataframe-access) 以获取有关在策略回调中使用 dataframe 的更多信息。

(custom-stoploss)=

## 自定义止损

在每次迭代（大约每 5 秒）时对开放交易调用，直到交易关闭。

必须通过在策略对象上设置 `use_custom_stoploss=True` 来启用自定义止损方法。

止损价格只能向上移动 - 如果从 `custom_stoploss` 返回的止损值会导致止损价格低于之前设置的价格，则会被忽略。传统的 `stoploss` 值作为绝对下限，并将作为初始止损（在此方法首次为交易调用之前），并且仍然是强制性的。  

由于自定义止损作为常规、变化的止损，其行为类似于 `trailing_stop` - 因此，由于此原因退出的交易将具有 `"trailing_stop_loss"` 的退出原因。

该方法必须返回一个止损值（浮点数/数字），作为当前价格的百分比。

例如，如果 `current_rate` 是 200 美元，则返回 `0.02` 将设置止损价格比当前价格低 2%，即 196 美元。

在回测期间，`current_rate`（和 `current_profit`）是针对 K 线的高点（或空头交易的低点）提供的，而最终的止损是针对 K 线的低点（或空头交易的高点）进行评估的。

返回值的绝对值被使用（符号被忽略），因此返回 `0.05` 或 `-0.05` 具有相同的结果，即止损比当前价格低 5%。
返回 `None` 将被解释为"不希望更改"，并且是当你不想修改止损时的唯一安全返回方式。
`NaN` 和 `inf` 值被视为无效，并将被忽略（与 `None` 相同）。

交易所上的止损与 `trailing_stop` 类似，并且交易所上的止损会根据 `stoploss_on_exchange_interval` 进行更新（[有关交易所止损的更多详细信息](stoploss.md#stop-loss-on-exchangefreqtrade)）。

如果你在期货市场上，请注意 [止损和杠杆](stoploss.md#stoploss-and-leverage) 部分，因为从 `custom_stoploss` 返回的止损值是该交易的风险，而不是相对价格变动。

:::{important} 使用日期
所有基于时间的计算都应基于 `current_time` - 不鼓励使用 `datetime.now()` 或 `datetime.utcnow()`，因为这会影响回测支持。
:::

:::{tip} 追踪止损
在使用自定义止损值时，建议禁用 `trailing_stop`。两者可以协同工作，但你可能会遇到追踪止损将价格推高，而你的自定义函数不希望这种情况，导致冲突行为。
:::

### 调整止损后调整仓位

根据你的策略，你可能需要在 [仓位调整](#adjust-trade-position) 后调整止损的方向。
为此，freqtrade 将在订单成交后额外调用 `after_fill=True`，这将允许策略在任意方向移动止损（也可以扩大止损与当前价格之间的差距，这在其他情况下是禁止的）。

:::{important} 向后兼容性
只有在你的 `custom_stoploss` 函数定义中包含 `after_fill` 参数时，才会进行此调用。

因此，这不会影响（也不会让现有运行的策略感到意外）。
:::

### 自定义止损示例

下一节将展示一些使用自定义止损函数的示例。
当然，还有更多可能性，所有示例都可以随意组合。

#### 通过自定义止损实现追踪止损

要模拟 4% 的常规追踪止损（在达到的最高价格后追踪 4%），你可以使用以下非常简单的方法：

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:
        """
        自定义止损逻辑，返回相对于 current_rate 的新距离（作为比率）。
        例如，返回 -0.05 将创建一个比 current_rate 低 5% 的止损。
        自定义止损绝不能低于 self.stoploss，后者作为硬性最大损失。

        完整文档请访问 https://www.freqtrade.io/en/latest/strategy-advanced/

        如果策略未实现，则返回初始止损值。
        仅在 use_custom_stoploss 设置为 True 时调用。

        :param pair: 当前分析的交易对
        :param trade: 交易对象。
        :param current_time: 当前时间的 datetime 对象
        :param current_rate: 基于 exit_pricing 中的定价设置计算的比率。
        :param current_profit: 当前利润（作为比率），基于 current_rate 计算。
        :param after_fill: 如果止损在订单成交后调用，则为 True。
        :param **kwargs: 保持此参数，以免未来更新破坏你的策略。
        :return float: 相对于 current_rate 的新止损值
        """
        return -0.04 * trade.leverage
```

#### 基于时间的追踪止损

在前 60 分钟内使用初始止损，之后更改为 10% 的追踪止损，并在 2 小时（120 分钟）后使用 5% 的追踪止损。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:

        # 确保最长的间隔在前 - 这些条件从上到下评估。
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05 * trade.leverage
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10 * trade.leverage
        return None
```

#### 基于时间的追踪止损，带有成交后调整

在前 60 分钟内使用初始止损，之后更改为 10% 的追踪止损，并在 2 小时（120 分钟）后使用 5% 的追踪止损。
如果额外订单成交，将止损设置为新 `open_rate` 下方 10%（[所有入场的平均值](#position-adjust-calculations)）。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool, 
                        **kwargs) -> float | None:

        if after_fill: 
            # 在额外订单后，从新开仓率下方 10% 的止损开始
            return stoploss_from_open(0.10, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        # 确保最长的间隔在前 - 这些条件从上到下评估。
        if current_time - timedelta(minutes=120) > trade.open_date_utc:
            return -0.05 * trade.leverage
        elif current_time - timedelta(minutes=60) > trade.open_date_utc:
            return -0.10 * trade.leverage
        return None
```

#### 每个交易对不同的止损

根据交易对使用不同的止损。

在此示例中，我们将为 `ETH/BTC` 和 `XRP/BTC` 使用 10% 的追踪止损，为 `LTC/BTC` 使用 5% 的追踪止损，并为所有其他交易对使用 15% 的追踪止损。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if pair in ("ETH/BTC", "XRP/BTC"):
            return -0.10 * trade.leverage
        elif pair in ("LTC/BTC"):
            return -0.05 * trade.leverage
        return -0.15 * trade.leverage
```

#### 带正偏移的追踪止损

在利润超过 4% 之前使用初始止损，然后使用当前利润的 50% 作为追踪止损，最小为 2.5%，最大为 5%。

请注意，止损只能增加，低于当前止损的值将被忽略。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* 方法

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        if current_profit < 0.04:
            return None # 返回 None 以继续使用初始止损

        # 达到所需偏移后，允许止损追踪一半的利润
        desired_stoploss = current_profit / 2

        # 使用最小 2.5% 和最大 5%
        return max(min(desired_stoploss, 0.05), 0.025) * trade.leverage
```

#### 阶梯式止损

此示例不是连续追踪当前价格，而是根据当前利润设置固定的止损价格水平。

* 在达到 20% 利润之前使用常规止损
* 一旦利润 > 20% - 将止损设置为开仓价格上方 7%。
* 一旦利润 > 25% - 将止损设置为开仓价格上方 15%。
* 一旦利润 > 40% - 将止损设置为开仓价格上方 25%。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # evaluate highest to lowest, so that highest possible stop is used
        if current_profit > 0.40:
            return stoploss_from_open(0.25, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.25:
            return stoploss_from_open(0.15, current_profit, is_short=trade.is_short, leverage=trade.leverage)
        elif current_profit > 0.20:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)

        # return maximum stoploss value, keeping current stoploss price unchanged
        return None
```

#### 使用数据框架中的指标实现自定义止损示例

绝对止损值可以从存储在数据框架中的指标派生。此示例使用价格下方的抛物线 SAR 作为止损。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # <...>
        dataframe["sar"] = ta.SAR(dataframe)

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()

        # Use parabolic sar as absolute stoploss price
        stoploss_price = last_candle["sar"]

        # Convert absolute price to percentage relative to current_rate
        if stoploss_price < current_rate:
            return stoploss_from_absolute(stoploss_price, current_rate, is_short=trade.is_short)

        # return maximum stoploss value, keeping current stoploss price unchanged
        return None
```

有关在策略回调中使用数据框架的更多信息，请参见[数据框架访问](strategy-advanced.md#dataframe-access)。

### 止损计算的常用辅助函数

#### 相对于开仓价格的止损

从 `custom_stoploss()` 返回的止损值必须指定相对于 `current_rate` 的百分比，但有时你可能想要指定相对于_开仓_价格的止损。
`stoploss_from_open()` 是一个辅助函数，用于计算可以从 `custom_stoploss` 返回的止损值，该值将等同于开仓点上方所需的交易利润。

:::{hint} 从自定义止损函数返回相对于开仓价格的止损

假设开仓价格为 \$100，`current_price` 为 \$121（`current_profit` 将为 `0.21`）。

如果我们想要在开仓价格上方 7% 设置止损价格，我们可以调用 `stoploss_from_open(0.07, current_profit, False)`，它将返回 `0.1157024793`。\$121 下方 11.57% 是 \$107，这与 \$100 上方 7% 相同。

此函数会考虑杠杆 - 所以在 10 倍杠杆下，实际止损将是 \$100 上方 0.7%（0.7% * 10x = 7%）。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    use_custom_stoploss = True

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:

        # once the profit has risen above 10%, keep the stoploss at 7% above the open price
        if current_profit > 0.10:
            return stoploss_from_open(0.07, current_profit, is_short=trade.is_short, leverage=trade.leverage)

        return 1

```

完整示例可以在文档的[自定义止损](strategy-callbacks.md#custom-stoploss)部分找到。
:::

:::{note}
向 `stoploss_from_open()` 提供无效输入可能会产生"CustomStoploss 函数未返回有效止损"警告。
如果 `current_profit` 参数低于指定的 `open_relative_stop`，这种情况可能会发生。当交易关闭被 `confirm_trade_exit()` 方法阻止时，可能会出现这种情况。可以通过在 `confirm_trade_exit()` 中检查 `exit_reason` 来永不阻止止损卖出，或者使用 `return stoploss_from_open(...) or 1` 惯用法来解决警告，这将在 `current_profit < open_relative_stop` 时请求不更改止损。
:::

#### 从绝对价格计算止损百分比

从 `custom_stoploss()` 返回的止损值始终指定相对于 `current_rate` 的百分比。为了在指定的绝对价格水平设置止损，我们需要使用 `stop_rate` 来计算相对于 `current_rate` 的百分比，这将给出与从开仓价格指定百分比相同的结果。

辅助函数 `stoploss_from_absolute()` 可用于将绝对价格转换为可以从 `custom_stoploss()` 返回的当前价格相对止损。

:::{hint} 从自定义止损函数返回使用绝对价格的止损

如果我们想要在当前价格下方 2xATR 追踪止损价格，我们可以调用 `stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), current_rate=current_rate, is_short=trade.is_short, leverage=trade.leverage)`。
对于期货，我们需要调整方向（向上或向下）以及调整杠杆，因为 [`custom_stoploss`](strategy-callbacks.md#custom-stoploss) 回调返回的是["此交易的风险"](stoploss.md#stoploss-and-leverage) - 而不是相对价格变动。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    use_custom_stoploss = True

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe["atr"] = ta.ATR(dataframe, timeperiod=14)
        return dataframe

    def custom_stoploss(self, pair: str, trade: Trade, current_time: datetime,
                        current_rate: float, current_profit: float, after_fill: bool,
                        **kwargs) -> float | None:
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        candle = dataframe.iloc[-1].squeeze()
        side = 1 if trade.is_short else -1
        return stoploss_from_absolute(current_rate + (side * candle["atr"] * 2), 
                                        current_rate=current_rate, 
                                        is_short=trade.is_short,
                                        leverage=trade.leverage)

```

:::

---

## 自定义 ROI

对开放交易每次迭代（大约每 5 秒）调用，直到交易关闭。

必须通过在策略对象上设置 `use_custom_roi=True` 来启用自定义 ROI 方法的使用。

此方法允许你定义退出交易的自定义最小 ROI 阈值，以比率表示（例如，`0.05` 表示 5% 利润）。如果同时定义了 `minimal_roi` 和 `custom_roi`，则较低的阈值将触发退出。例如，如果 `minimal_roi` 设置为 `{"0": 0.10}`（0 分钟时为 10%）且 `custom_roi` 返回 `0.05`，则当利润达到 5% 时交易将退出。同样，如果 `custom_roi` 返回 `0.10` 且 `minimal_roi` 设置为 `{"0": 0.05}`（0 分钟时为 5%），则当利润达到 5% 时交易将关闭。

该方法必须返回一个表示新 ROI 阈值的浮点数比率，或返回 `None` 以回退到 `minimal_roi` 逻辑。返回 `NaN` 或 `inf` 值被视为无效，将被视为 `None`，导致机器人使用 `minimal_roi` 配置。

### 自定义 ROI 示例

以下示例说明如何使用 `custom_roi` 函数实现不同的 ROI 逻辑。

#### 按交易方向自定义 ROI

根据 `side` 使用不同的 ROI 阈值。在此示例中，多头入场为 5%，空头入场为 2%。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    use_custom_roi = True

    # ... populate_* methods

    def custom_roi(self, pair: str, trade: Trade, current_time: datetime, trade_duration: int,
                   entry_tag: str | None, side: str, **kwargs) -> float | None:
        """
        自定义 ROI 逻辑，返回新的最小 ROI 阈值（以比率表示，例如 0.05 表示 +5%）。
        仅在 use_custom_roi 设置为 True 时调用。

        如果与 minimal_roi 同时使用，当达到较低的阈值时将触发退出。
        示例：如果 minimal_roi = {"0": 0.01} 且 custom_roi 返回 0.05，
        当利润达到 5% 时将触发退出。

        :param pair: 当前分析的交易对。
        :param trade: 交易对象。
        :param current_time: datetime 对象，包含当前日期时间。
        :param trade_duration: 当前交易持续时间（分钟）。
        :param entry_tag: 如果买入信号提供了可选的 entry_tag（buy_tag）。
        :param side: 'long' 或 'short' - 表示当前交易的方向。
        :param **kwargs: 确保保留此参数，以便更新不会破坏你的策略。
        :return float: 新的 ROI 值（比率），或 None 以回退到 minimal_roi 逻辑。
        """
        return 0.05 if side == "long" else 0.02
```

#### 按交易对自定义 ROI

根据 `pair` 使用不同的 ROI 阈值。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    use_custom_roi = True

    # ... populate_* methods

    def custom_roi(self, pair: str, trade: Trade, current_time: datetime, trade_duration: int,
                   entry_tag: str | None, side: str, **kwargs) -> float | None:

        stake = trade.stake_currency
        roi_map = {
            f"BTC/{stake}": 0.02, # BTC 为 2%
            f"ETH/{stake}": 0.03, # ETH 为 3%
            f"XRP/{stake}": 0.04, # XRP 为 4%
        }

        return roi_map.get(pair, 0.01) # 其他交易对为 1%
```

#### 按入场标签自定义 ROI

根据买入信号提供的 `entry_tag` 使用不同的 ROI 阈值。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    use_custom_roi = True

    # ... populate_* methods

    def custom_roi(self, pair: str, trade: Trade, current_time: datetime, trade_duration: int,
                   entry_tag: str | None, side: str, **kwargs) -> float | None:

        roi_by_tag = {
            "breakout": 0.08,       # 如果标签是 "breakout" 则为 8%
            "rsi_overbought": 0.05, # 如果标签是 "rsi_overbought" 则为 5%
            "mean_reversion": 0.03, # 如果标签是 "mean_reversion" 则为 3%
        }

        return roi_by_tag.get(entry_tag, 0.01)  # 如果标签未知则为 1%
```

#### 基于 ATR 的自定义 ROI

ROI 值可以从存储在数据框架中的指标派生。此示例使用 ATR 比率作为 ROI。

```python
# Default imports
# <...>
import talib.abstract as ta

class AwesomeStrategy(IStrategy):

    use_custom_roi = True

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # <...>
        dataframe["atr"] = ta.ATR(dataframe, timeperiod=10)

    def custom_roi(self, pair: str, trade: Trade, current_time: datetime, trade_duration: int,
                   entry_tag: str | None, side: str, **kwargs) -> float | None:

        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        atr_ratio = last_candle["atr"] / last_candle["close"]

        return atr_ratio # 返回 ATR 值作为比率
```

---

## 自定义订单价格规则

默认情况下，freqtrade 使用订单簿自动设置订单价格（[相关文档](configuration.md#prices-used-for-orders)），你也可以选择基于你的策略创建自定义订单价格。

你可以通过在策略文件中创建 `custom_entry_price()` 函数自定义入场价格，以及 `custom_exit_price()` 函数自定义出场价格。

每个方法都会在向交易所下单前被调用。

:::{tip}
如果你的自定义定价函数返回 None 或无效值，价格将回退到 `proposed_rate`，即基于常规定价配置的价格。

使用 custom_entry_price 时，Trade 对象会在与该交易相关的第一个入场订单创建时可用，对于第一次入场，`trade` 参数值为 `None`。
:::

### 自定义订单入场和出场价格示例

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def custom_entry_price(self, pair: str, trade: Trade | None, current_time: datetime, proposed_rate: float,
                           entry_tag: str | None, side: str, **kwargs) -> float:

        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair,
                                                                timeframe=self.timeframe)
        new_entryprice = dataframe["bollinger_10_lowerband"].iat[-1]

        return new_entryprice

    def custom_exit_price(self, pair: str, trade: Trade,
                          current_time: datetime, proposed_rate: float,
                          current_profit: float, exit_tag: str | None, **kwargs) -> float:

        dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=pair,
                                                                timeframe=self.timeframe)
        new_exitprice = dataframe["bollinger_10_upperband"].iat[-1]

        return new_exitprice

```

:::{warning}
修改入场和出场价格仅适用于限价单。根据选择的价格，这可能导致大量未成交订单。默认情况下，当前价格与自定义价格之间的最大允许距离为 2%，该值可通过配置中的 `custom_price_max_distance_ratio` 参数更改。

**示例**：

如果 new_entryprice 是 97，proposed_rate 是 100，`custom_price_max_distance_ratio` 设置为 2%，则保留的有效自定义入场价格将是 98，比当前（建议）价格低 2%。
:::

:::{warning} 回测
自定义价格在回测中受支持（自 2021.12 起），如果价格落在 K 线的最高/最低范围内，订单将会成交。

未能立即成交的订单将受到常规超时处理，这会在每根（详细）K 线发生一次。

`custom_exit_price()` 仅针对类型为 exit_signal、自定义退出和部分退出的卖出调用。所有其他退出类型将使用常规回测价格。

:::

---

## 自定义订单超时规则

简单的、基于时间的订单超时可以通过策略或在配置的 `unfilledtimeout` 部分进行配置。

然而，freqtrade 还为两种订单类型提供了自定义回调，允许你基于自定义标准决定订单是否超时。

:::{note}
回测会在订单价格落在 K 线的最高/最低范围内时成交订单。

以下回调将针对不会立即成交的订单（使用自定义定价）在每个（详细）K 线调用一次。
:::

### 自定义订单超时示例

对每个未成交订单调用，直到订单成交或取消。

`check_entry_timeout()` 用于交易入场，而 `check_exit_timeout()` 用于交易出场订单。

下面是一个简单的示例，它根据资产价格应用不同的未成交超时。

它对高价资产应用较短的超时，而对低价币种允许更长的成交时间。

函数必须返回 `True`（取消订单）或 `False`（保持订单活跃）。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    # 设置 unfilledtimeout 为 25 小时，因为下面的最大超时是 24 小时。
    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False


    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        if trade.open_rate > 100 and trade.open_date_utc < current_time - timedelta(minutes=5):
            return True
        elif trade.open_rate > 10 and trade.open_date_utc < current_time - timedelta(minutes=3):
            return True
        elif trade.open_rate < 1 and trade.open_date_utc < current_time - timedelta(hours=24):
           return True
        return False
```

:::{warning}
对于上面的示例，`unfilledtimeout` 必须设置为大于 24 小时的值，否则该类型的超时将首先应用。
:::

### 自定义订单超时示例（使用额外数据）

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    # 设置 unfilledtimeout 为 25 小时，因为下面的最大超时是 24 小时。
    unfilledtimeout = {
        "entry": 60 * 25,
        "exit": 60 * 25
    }

    def check_entry_timeout(self, pair: str, trade: Trade, order: Order,
                            current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["bids"][0][0]
        # 如果价格比订单价格高出 2% 以上，取消买单。
        if current_price > order.price * 1.02:
            return True
        return False


    def check_exit_timeout(self, pair: str, trade: Trade, order: Order,
                           current_time: datetime, **kwargs) -> bool:
        ob = self.dp.orderbook(pair, 1)
        current_price = ob["asks"][0][0]
        # 如果价格比订单价格低 2% 以上，取消卖单。
        if current_price < order.price * 0.98:
            return True
        return False
```

---

## 机器人订单确认

确认交易入场/出场。
这些是在订单下单之前最后调用的方法。

(trade-entry-buy-order-confirmation)=

### 交易入场（买单）确认

`confirm_trade_entry()` 可用于在最后一刻中止交易入场（可能是因为价格不符合预期）。

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def confirm_trade_entry(self, pair: str, order_type: str, amount: float, rate: float,
                            time_in_force: str, current_time: datetime, entry_tag: str | None,
                            side: str, **kwargs) -> bool:
        """
        Called right before placing a entry order.
        Timing for this function is critical, so avoid doing heavy computations or
        network requests in this method.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns True (always confirming).

        :param pair: Pair that's about to be bought/shorted.
        :param order_type: Order type (as configured in order_types). usually limit or market.
        :param amount: Amount in target (base) currency that's going to be traded.
        :param rate: Rate that's going to be used when using limit orders 
                     or current rate for market orders.
        :param time_in_force: Time in force. Defaults to GTC (Good-til-cancelled).
        :param current_time: datetime object, containing the current datetime
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: "long" or "short" - indicating the direction of the proposed trade
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return bool: When True is returned, then the buy-order is placed on the exchange.
            False aborts the process
        """
        return True

```

### 交易出场（卖单）确认

`confirm_trade_exit()` 可用于在最后一刻中止交易出场（可能是因为价格不符合预期）。

`confirm_trade_exit()` 可能会在同一迭代中多次调用同一交易，如果适用不同的退出原因。
退出原因（如果适用）将按以下顺序排列：

* `exit_signal` / `custom_exit`
* `stop_loss`
* `roi`
* `trailing_stop_loss`

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def confirm_trade_exit(self, pair: str, trade: Trade, order_type: str, amount: float,
                           rate: float, time_in_force: str, exit_reason: str,
                           current_time: datetime, **kwargs) -> bool:
        """
        Called right before placing a regular exit order.
        Timing for this function is critical, so avoid doing heavy computations or
        network requests in this method.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns True (always confirming).

        :param pair: Pair for trade that's about to be exited.
        :param trade: trade object.
        :param order_type: Order type (as configured in order_types). usually limit or market.
        :param amount: Amount in base currency.
        :param rate: Rate that's going to be used when using limit orders
                     or current rate for market orders.
        :param time_in_force: Time in force. Defaults to GTC (Good-til-cancelled).
        :param exit_reason: Exit reason.
            Can be any of ["roi", "stop_loss", "stoploss_on_exchange", "trailing_stop_loss",
                           "exit_signal", "force_exit", "emergency_exit"]
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return bool: When True, then the exit-order is placed on the exchange.
            False aborts the process
        """
        if exit_reason == "force_exit" and trade.calc_profit_ratio(rate) < 0:
            # Reject force-sells with negative profit
            # This is just a sample, please adjust to your needs
            # (this does not necessarily make sense, assuming you know when you're force-selling)
            return False
        return True

```

:::{warning}
`confirm_trade_exit()` 可以阻止止损退出，导致重大损失，因为这将忽略止损退出。

`confirm_trade_exit()` 不会为清算调用 - 因为清算是由交易所强制执行的，因此无法拒绝。
:::

(adjust-trade-position)=

## 调整交易仓位

`position_adjustment_enable` 策略属性启用了策略中 `adjust_trade_position()` 回调的使用。
出于性能原因，默认情况下它是禁用的，如果启用，freqtrade 将在启动时显示警告消息。
`adjust_trade_position()` 可用于执行额外的订单，例如使用 DCA（美元成本平均）管理风险或增加或减少仓位。

额外订单也会导致额外费用，并且这些订单不计入 `max_open_trades`。

当有未成交订单（买单或卖单）等待执行时，也会调用此回调 - 如果数量、价格或方向不同，将取消现有的未成交订单以放置新订单。部分成交的订单也将被取消，并将被回调返回的新数量替换。

`adjust_trade_position()` 在交易期间非常频繁地被调用，因此你必须尽可能保持实现的性能。

仓位调整将始终应用于交易的方向，因此正值将始终增加你的仓位（负值将减少你的仓位），无论它是多头还是空头交易。
调整订单可以通过返回一个 2 元素元组来分配标签，第一个元素是调整数量，第二个元素是标签（例如 `return 250, "increase_favorable_conditions"`）。

无法修改杠杆，返回的仓位金额假定为应用杠杆之前。

当前分配给仓位的组合仓位金额保存在 `trade.stake_amount` 中。因此，`trade.stake_amount` 将在每次通过 `adjust_trade_position()` 进行额外入场和部分出场时更新。

:::{danger} 松散逻辑
在模拟和实盘运行中，此函数将每 `throttle_process_secs`（默认为 5 秒）调用一次。如果你有松散的逻辑（例如，如果最后一根 K 线的 RSI 低于 30，则增加仓位），你的机器人将每 5 秒进行一次额外重新入场，直到你用完资金，达到 `max_position_adjustment` 限制，或者出现 RSI 大于 30 的新 K 线。

部分出场也可能发生同样的情况。  

因此，请确保有严格的逻辑和/或检查最后成交的订单以及是否已有未成交订单。
:::

:::{warning} 多次仓位调整的性能
仓位调整可以是增加策略输出的好方法 - 但如果广泛使用此功能，也可能有缺点。  

每个订单将在交易期间附加到交易对象 - 因此增加内存使用。

因此，不建议使用长时间持续和 10 次甚至 100 次仓位调整的交易，应该定期关闭以避免影响性能。
:::

:::{warning} 回测
在回测期间，此回调在 `timeframe` 或 `timeframe_detail` 中的每根 K 线调用一次，因此运行时性能将受到影响。

这也可能导致实盘和回测之间的结果不同，因为回测只能在每根 K 线调整一次交易，而实盘可以在每根 K 线多次调整交易。
:::

### 增加仓位

当需要进行额外入场订单时（即增加仓位——多头为买单，空头为卖单），策略应返回一个介于 `min_stake` 和 `max_stake` 之间的正数 **stake_amount**（以仓位货币计价）。

如果钱包中没有足够的资金（返回值高于 `max_stake`），则该信号会被忽略。
`max_entry_position_adjustment` 属性用于限制机器人每笔交易（在首次入场订单之外）可执行的额外入场次数。默认值为 -1，表示机器人对调整入场次数没有限制。

一旦达到你在 `max_entry_position_adjustment` 上设置的最大额外入场次数，额外入场将被忽略，但回调仍会被调用以寻找部分出场机会。

:::{warning} 关于仓位大小
使用固定仓位大小意味着首次订单将使用该金额，就像没有仓位调整一样。

如果你希望通过 DCA 进行额外买入，请确保钱包中留有足够的资金。

对 DCA 订单使用 "unlimited" 仓位金额时，需要你实现 `custom_stake_amount()` 回调，以避免初始订单占用全部资金。
:::

### 减少仓位

策略应返回一个负的 stake_amount（以仓位货币计价）用于部分出场。

返回当前持有的全部仓位（`-trade.stake_amount`）将导致完全出场。  

返回超过上述值（即剩余 stake_amount 变为负数）将导致机器人忽略该信号。

对于部分出场，重要的是要知道用于计算部分出场订单币种数量的公式是 `部分出场数量 = negative_stake_amount * trade.amount / trade.stake_amount`，其中 `negative_stake_amount` 是 `adjust_trade_position` 函数返回的值。正如公式所示，该公式与当前仓位盈亏无关，只与 `trade.amount` 和 `trade.stake_amount` 有关，这两个值不会受到价格波动影响。

例如，假设你以 50 的开盘价买入 2 个 SHITCOIN/USDT，意味着该交易的仓位金额为 100 USDT。现在价格涨到 200，你想卖出一半。这时你需要返回 `trade.stake_amount` 的 -50%（0.5 * 100 USDT），即 -50。机器人会计算需要卖出的数量，即 `50 * 2 / 100`，等于 1 个 SHITCOIN/USDT。如果你返回 -200（2 * 200 的 50%），机器人会忽略，因为 `trade.stake_amount` 只有 100 USDT，而你要求卖出 200 USDT，相当于卖出 4 个 SHITCOIN/USDT。

回到上面的例子，当前价格为 200，你的交易当前 USDT 价值为 400 USDT。假设你想部分卖出 100 USDT，以取回初始投资并将利润留在交易中，希望价格继续上涨。这时你需要先计算要卖出的确切数量。由于你想按当前价格卖出价值 100 USDT 的币，部分卖出的确切数量为 `100 * 2 / 400`，等于 0.5 个 SHITCOIN/USDT。既然知道了要卖出的数量（0.5），你在 `adjust_trade_position` 函数中需要返回的值是 `-部分出场数量 * trade.stake_amount / trade.amount`，即 -25。机器人会卖出 0.5 个 SHITCOIN/USDT，交易中保留 1.5 个。你将从部分出场中获得 100 USDT。

:::{warning} 止损计算

止损仍然以初始开仓价为基准计算，而不是均价。

常规止损规则仍然适用（不能下移止损）。

虽然 `/stopentry` 命令会阻止机器人进入新交易，但仓位调整功能会继续在已有交易上买入新订单。
:::

```python
# Default imports

class DigDeeperStrategy(IStrategy):

    position_adjustment_enable = True

    # Attempts to handle large drops with DCA. High stoploss is required.
    stoploss = -0.30

    # ... populate_* methods

    # Example specific variables
    max_entry_position_adjustment = 3
    # This number is explained a bit further down
    max_dca_multiplier = 5.5

    # This is called when placing the initial order (opening trade)
    def custom_stake_amount(self, pair: str, current_time: datetime, current_rate: float,
                            proposed_stake: float, min_stake: float | None, max_stake: float,
                            leverage: float, entry_tag: str | None, side: str,
                            **kwargs) -> float:

        # We need to leave most of the funds for possible further DCA orders
        # This also applies to fixed stakes
        return proposed_stake / self.max_dca_multiplier

    def adjust_trade_position(self, trade: Trade, current_time: datetime,
                              current_rate: float, current_profit: float,
                              min_stake: float | None, max_stake: float,
                              current_entry_rate: float, current_exit_rate: float,
                              current_entry_profit: float, current_exit_profit: float,
                              **kwargs
                              ) -> float | None | tuple[float | None, str | None]:
        """
        Custom trade adjustment logic, returning the stake amount that a trade should be
        increased or decreased.
        This means extra entry or exit orders with additional fees.
        Only called when `position_adjustment_enable` is set to True.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-advanced/

        When not implemented by a strategy, returns None

        :param trade: trade object.
        :param current_time: datetime object, containing the current datetime
        :param current_rate: Current entry rate (same as current_entry_profit)
        :param current_profit: Current profit (as ratio), calculated based on current_rate 
                               (same as current_entry_profit).
        :param min_stake: Minimal stake size allowed by exchange (for both entries and exits)
        :param max_stake: Maximum stake allowed (either through balance, or by exchange limits).
        :param current_entry_rate: Current rate using entry pricing.
        :param current_exit_rate: Current rate using exit pricing.
        :param current_entry_profit: Current profit using entry pricing.
        :param current_exit_profit: Current profit using exit pricing.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return float: Stake amount to adjust your trade,
                       Positive values to increase position, Negative values to decrease position.
                       Return None for no action.
                       Optionally, return a tuple with a 2nd element with an order reason
        """
        if trade.has_open_orders:
            # Only act if no orders are open
            return

        if current_profit > 0.05 and trade.nr_of_successful_exits == 0:
            # Take half of the profit at +5%
            return -(trade.stake_amount / 2), "half_profit_5%"

        if current_profit > -0.05:
            return None

        # Obtain pair dataframe (just to show how to access it)
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        # Only buy when not actively falling price.
        last_candle = dataframe.iloc[-1].squeeze()
        previous_candle = dataframe.iloc[-2].squeeze()
        if last_candle["close"] < previous_candle["close"]:
            return None

        filled_entries = trade.select_filled_orders(trade.entry_side)
        count_of_entries = trade.nr_of_successful_entries
        # Allow up to 3 additional increasingly larger buys (4 in total)
        # Initial buy is 1x
        # If that falls to -5% profit, we buy 1.25x more, average profit should increase to roughly -2.2%
        # If that falls down to -5% again, we buy 1.5x more
        # If that falls once again down to -5%, we buy 1.75x more
        # Total stake for this trade would be 1 + 1.25 + 1.5 + 1.75 = 5.5x of the initial allowed stake.
        # That is why max_dca_multiplier is 5.5
        # Hope you have a deep wallet!
        try:
            # This returns first order stake size
            stake_amount = filled_entries[0].stake_amount_filled
            # This then calculates current safety order size
            stake_amount = stake_amount * (1 + (count_of_entries * 0.25))
            return stake_amount, "1/3rd_increase"
        except Exception as exception:
            return None

        return None

```

### 仓位调整计算

* 入场价格使用加权平均计算。
* 出场不会影响平均入场价格。
* 部分出场的相对利润是相对于此时的平均入场价格。
* 最终出场的相对利润是基于总投资资本计算的。（见下面的例子）

:::{hint} 计算示例
*此示例假设零手续费，且为一个虚拟币的多头仓位。*  

* 买入 100@8\$
* 买入 100@9\$ -> 平均价格：8.5\$
* 卖出 100@10\$ -> 平均价格：8.5\$，已实现利润 150\$，17.65%
* 买入 150@11\$ -> 平均价格：10\$，已实现利润 150\$，17.65%
* 卖出 100@12\$ -> 平均价格：10\$，总已实现利润 350\$，20%
* 卖出 150@14\$ -> 平均价格：10\$，总已实现利润 950\$，40%  <- *这将是最后的"出场"消息*

这笔交易的总利润是 950\$，总投资为 3350\$（`100@8$ + 100@9$ + 150@11$`）。因此，最终相对利润为 28.35%（`950 / 3350`）。
:::

## 调整订单价格

策略开发者可以使用 `adjust_order_price()` 回调在新K线到达时刷新/替换限价订单。  

除非订单在当前K线内已被（重新）放置，否则此回调在每次迭代时都会被调用一次 - 限制每个订单的最大（重新）放置次数为每根K线一次。

这也意味着第一次调用将在初始订单放置后的下一根K线开始时进行。

请注意，`custom_entry_price()`/`custom_exit_price()` 仍然是在信号发出时决定初始限价订单价格目标的方法。

可以通过返回 `None` 来取消此回调中的订单。

返回 `current_order_rate` 将保持订单在交易所"原样"。

返回任何其他价格将取消现有订单，并替换为新订单。

如果原始订单的取消失败，则不会替换订单 - 尽管订单很可能已在交易所被取消。如果这种情况发生在初始入场时，将导致订单被删除，而在仓位调整订单中，将导致交易规模保持不变。  
如果订单已部分成交，则不会替换订单。但是，如果需要/希望，你可以使用 [`adjust_trade_position()`](#adjust-trade-position) 来调整交易规模到预期的仓位大小。

:::{warning} 常规超时
入场 `unfilledtimeout` 机制（以及 `check_entry_timeout()`/`check_exit_timeout()`）优先于此回调。

通过上述方法取消的订单不会调用此回调。请确保更新超时值以符合你的预期。
:::

```python
# Default imports

class AwesomeStrategy(IStrategy):

    # ... populate_* methods

    def adjust_order_price(
        self,
        trade: Trade,
        order: Order | None,
        pair: str,
        current_time: datetime,
        proposed_rate: float,
        current_order_rate: float,
        entry_tag: str | None,
        side: str,
        is_entry: bool,
        **kwargs,
    ) -> float | None:
        """
        Exit and entry order price re-adjustment logic, returning the user desired limit price.
        This only executes when a order was already placed, still open (unfilled fully or partially)
        and not timed out on subsequent candles after entry trigger.

        For full documentation please go to https://www.freqtrade.io/en/latest/strategy-callbacks/

        When not implemented by a strategy, returns current_order_rate as default.
        If current_order_rate is returned then the existing order is maintained.
        If None is returned then order gets canceled but not replaced by a new one.

        :param pair: Pair that's currently analyzed
        :param trade: Trade object.
        :param order: Order object
        :param current_time: datetime object, containing the current datetime
        :param proposed_rate: Rate, calculated based on pricing settings in entry_pricing.
        :param current_order_rate: Rate of the existing order in place.
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: 'long' or 'short' - indicating the direction of the proposed trade
        :param is_entry: True if the order is an entry order, False if it's an exit order.
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return float or None: New entry price value if provided
        """

        # Limit entry orders to use and follow SMA200 as price target for the first 10 minutes since entry trigger for BTC/USDT pair.
        if (
            is_entry
            and pair == "BTC/USDT" 
            and entry_tag == "long_sma200" 
            and side == "long" 
            and (current_time - timedelta(minutes=10)) <= trade.open_date_utc
        ):
            # just cancel the order if it has been filled more than half of the amount
            if order.filled > order.remaining:
                return None
            else:
                dataframe, _ = self.dp.get_analyzed_dataframe(pair=pair, timeframe=self.timeframe)
                current_candle = dataframe.iloc[-1].squeeze()
                # desired price
                return current_candle["sma_200"]
        # default: maintain existing order
        return current_order_rate
```

:::{danger} 与 `adjust_*_price()` 的不兼容性
如果你同时实现了 `adjust_order_price()` 和 `adjust_entry_price()`/`adjust_exit_price()`，则只会使用 `adjust_order_price()`。

如果你需要调整入场/出场价格，你可以选择在 `adjust_order_price()` 中实现逻辑，或者使用分开的 `adjust_entry_price()` / `adjust_exit_price()` 回调，但不能同时使用两者。

混合使用这些功能是不支持的，会在机器人启动时引发错误。
:::

### 调整入场价格

策略开发者可以使用 `adjust_entry_price()` 回调在新K线到达时刷新/替换入场限价订单。

这是 `adjust_order_price()` 的一个子集，仅用于入场订单。

其余所有行为与 `adjust_order_price()` 相同。

交易开始日期（`trade.open_date_utc`）将保持在第一个订单放置的时间。

请确保注意这一点 - 并相应地调整其他回调中的逻辑，使用第一个成交订单的日期。

### 调整出场价格

策略开发者可以使用 `adjust_exit_price()` 回调在新K线到达时刷新/替换出场限价订单。

这是 `adjust_order_price()` 的一个子集，仅用于出场订单。

其余所有行为与 `adjust_order_price()` 相同。

## 杠杆回调

在允许杠杆交易的市场中，此方法必须返回所需的杠杆倍数（默认为 1 -> 无杠杆）。

假设资金为 500USDT，杠杆倍数为 3 的交易将产生 500 x 3 = 1500 USDT 的仓位。

超过 `max_leverage` 的值将被调整为 `max_leverage`。
对于不支持杠杆的市场/交易所，此方法将被忽略。

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def leverage(self, pair: str, current_time: datetime, current_rate: float,
                 proposed_leverage: float, max_leverage: float, entry_tag: str | None, side: str,
                 **kwargs) -> float:
        """
        Customize leverage for each new trade. This method is only called in futures mode.

        :param pair: Pair that's currently analyzed
        :param current_time: datetime object, containing the current datetime
        :param current_rate: Rate, calculated based on pricing settings in exit_pricing.
        :param proposed_leverage: A leverage proposed by the bot.
        :param max_leverage: Max leverage allowed on this pair
        :param entry_tag: Optional entry_tag (buy_tag) if provided with the buy signal.
        :param side: "long" or "short" - indicating the direction of the proposed trade
        :return: A leverage amount, which is between 1.0 and max_leverage.
        """
        return 1.0
```

所有利润计算都包含杠杆。止损和 ROI 的计算也包含杠杆。

在 10 倍杠杆下设置 10% 的止损，将在价格下跌 1% 时触发止损。

## 订单成交回调

`order_filled()` 回调可用于在订单成交后根据当前交易状态执行特定操作。

它将独立于订单类型（入场、出场、止损或仓位调整）被调用。

假设你的策略需要在开仓时存储该 K 线的最高价，可以通过如下示例所示的回调实现这一点。

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def order_filled(self, pair: str, trade: Trade, order: Order, current_time: datetime, **kwargs) -> None:
        """
        Called right after an order fills. 
        Will be called for all order types (entry, exit, stoploss, position adjustment).
        :param pair: Pair for trade
        :param trade: trade object.
        :param order: Order object.
        :param current_time: datetime object, containing the current datetime
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        """
        # Obtain pair dataframe (just to show how to access it)
        dataframe, _ = self.dp.get_analyzed_dataframe(trade.pair, self.timeframe)
        last_candle = dataframe.iloc[-1].squeeze()
        
        if (trade.nr_of_successful_entries == 1) and (order.ft_order_side == trade.entry_side):
            trade.set_custom_data(key="entry_candle_high", value=last_candle["high"])

        return None

```

## 图表注释回调

每当 freqUI 请求显示图表数据时，都会调用图表注释回调。
此回调在交易周期中没有实际意义，仅用于图表展示。

策略可以返回一个 `AnnotationType` 对象列表，这些对象会显示在图表上。
根据返回内容，图表可以显示水平区域、垂直区域或框选区域。

完整对象如下所示：

```json
{
    "type": "area", // Type of the annotation, currently only "area" is supported
    "start": "2024-01-01 15:00:00", // Start date of the area
    "end": "2024-01-01 16:00:00",  // End date of the area
    "y_start": 94000.2,  // Price / y axis value
    "y_end": 98000, // Price / y axis value
    "color": "",
    "label": "some label"
}
```

下面的示例将在图表上标记第 8 小时和第 15 小时的区域，使用灰色突出显示市场开盘和收盘时间。
这显然是一个非常基础的示例。

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def plot_annotations(
        self, pair: str, start_date: datetime, end_date: datetime, dataframe: DataFrame, **kwargs
    ) -> list[AnnotationType]:
        """
        Retrieve area annotations for a chart.
        Must be returned as array, with type, label, color, start, end, y_start, y_end.
        All settings except for type are optional - though it usually makes sense to include either
        "start and end" or "y_start and y_end" for either horizontal or vertical plots
        (or all 4 for boxes).
        :param pair: Pair that's currently analyzed
        :param start_date: Start date of the chart data being requested
        :param end_date: End date of the chart data being requested
        :param dataframe: DataFrame with the analyzed data for the chart
        :param **kwargs: Ensure to keep this here so updates to this won't break your strategy.
        :return: List of AnnotationType objects
        """
        annotations = []
        while start_dt < end_date:
            start_dt += timedelta(hours=1)
            if start_dt.hour in (8, 15):
                annotations.append(
                    {
                        "type": "area",
                        "label": "Trade open and close hours",
                        "start": start_dt,
                        "end": start_dt + timedelta(hours=1),
                        # Omitting y_start and y_end will result in a vertical area spanning the whole height of the main Chart
                        "color": "rgba(133, 133, 133, 0.4)",
                    }
                )

        return annotations

```

所有条目都将被验证，如果不符合预期的模式，将不会被传递给 UI，并且会记录错误。

:::{warning} 过多图片注释
使用过多注释可能导致 UI 卡顿，特别是在绘制大量历史数据时。

请谨慎使用注释功能。
:::

### 图表注释示例

![FreqUI - 图表注释](assets/freqUI-chart-annotations-dark.png)
![FreqUI - 图表注释](assets/freqUI-chart-annotations-light.png)

:::{important} 上图使用的代码
这是一个示例代码，仅供参考。

```python
# Default imports

class AwesomeStrategy(IStrategy):
    def plot_annotations(
        self, pair: str, start_date: datetime, end_date: datetime, dataframe: DataFrame, **kwargs
    ) -> list[AnnotationType]:
        annotations = []
        while start_dt < end_date:
            start_dt += timedelta(hours=1)
            if (start_dt.hour % 4) == 0:
                mark_areas.append(
                    {
                        "type": "area",
                        "label": "4h",
                        "start": start_dt,
                        "end": start_dt + timedelta(hours=1),
                        "color": "rgba(133, 133, 133, 0.4)",
                    }
                )
            elif (start_dt.hour % 2) == 0:
            price = dataframe.loc[dataframe["date"] == start_dt, ["close"]].mean()
                mark_areas.append(
                    {
                        "type": "area",
                        "label": "2h",
                        "start": start_dt,
                        "end": start_dt + timedelta(hours=1),
                        "y_end": price * 1.01,
                        "y_start": price * 0.99,
                        "color": "rgba(0, 255, 0, 0.4)",
                    }
                )

        return annotations

```

:::
