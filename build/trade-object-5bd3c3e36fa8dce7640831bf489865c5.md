---
title: 交易对象文档
subject: Freqtrade 交易对象指南
subtitle: 交易对象属性和方法概述
short_title: 交易对象指南
description: 本文档详细介绍了 Freqtrade 中的交易对象(Trade)，包括其属性、方法以及在策略中的使用方式。
---

# 交易对象

## Trade

freqtrade 进入的仓位存储在 `Trade` 对象中 - 该对象会被持久化到数据库。
这是 freqtrade 的核心概念 - 你会在文档的许多部分遇到它，这些部分很可能会指向这个位置。

它将在许多[策略回调](strategy-callbacks.md)中传递给策略。传递给策略的对象不能直接修改。间接修改可能基于回调结果发生。

## Trade - 可用属性

以下属性可用于每个单独的交易 - 并且可以使用 `trade.<property>` 访问（例如 `trade.pair`）。

|  属性 | 数据类型 | 描述 |
|------------|-------------|-------------|
| `pair` | string | 交易对。 |
| `is_open` | boolean | 交易当前是否开放，或已结束。 |
| `open_rate` | float | 进入交易时的价格（在交易调整的情况下为平均入场价格）。 |
| `close_rate` | float | 平仓价格 - 仅在 is_open = False 时设置。 |
| `stake_amount` | float | 以 Stake（或 Quote）货币计量的金额。 |
| `amount` | float | 当前拥有的资产/基础货币数量。在初始订单成交前将为 0.0。 |
| `open_date` | datetime | 交易开始的时间戳 **请使用 `open_date_utc` 代替** |
| `open_date_utc` | datetime | 交易开始的时间戳 - UTC 时间。 |
| `close_date` | datetime | 交易结束的时间戳 **请使用 `close_date_utc` 代替** |
| `close_date_utc` | datetime | 交易结束的时间戳 - UTC 时间。 |
| `close_profit` | float | 交易结束时的相对利润。`0.01` == 1% |
| `close_profit_abs` | float | 交易结束时的绝对利润（以 stake 货币计）。 |
| `realized_profit` | float | 交易仍然开放时已实现的绝对利润（以 stake 货币计）。 |
| `leverage` | float | 此交易使用的杠杆 - 在现货市场中默认为 1.0。 |
| `enter_tag` | string | 通过 dataframe 中的 `enter_tag` 列提供的入场标签。 |
| `is_short` | boolean | 做空交易为 True，否则为 False。 |
| `orders` | Order[] | 附加到此交易的订单对象列表（包括已成交和已取消的订单）。 |
| `date_last_filled_utc` | datetime | 最后一个成交订单的时间。 |
| `entry_side` | "buy" / "sell" | 交易进入的订单方向。 |
| `exit_side` | "buy" / "sell" | 将导致交易退出/仓位减少的订单方向。 |
| `trade_direction` | "long" / "short" | 文本形式的交易方向 - 做多或做空。 |
| `nr_of_successful_entries` | int | 成功（已成交）的入场订单数量。 |
| `nr_of_successful_exits` | int | 成功（已成交）的出场订单数量。 |
| `has_open_orders` | boolean | 交易是否有未完成订单（不包括止损订单）。 |

## 类方法

以下是类方法 - 它们返回通用信息，通常会导致对数据库的显式查询。
它们可以作为 `Trade.<method>` 使用 - 例如 `open_trades = Trade.get_open_trade_count()`

:::{warning} 回测/超优化
大多数方法在回测/超优化和实盘/模拟模式下都能工作。

在回测期间，它仅限于在[策略回调](strategy-callbacks.md)中使用。不支持在 `populate_*()` 方法中使用，否则会导致错误结果。
:::

### get_trades_proxy

当你的策略需要一些关于现有（开放或关闭）交易的信息时 - 最好使用 `Trade.get_trades_proxy()`。

用法：

```python
from freqtrade.persistence import Trade
from datetime import timedelta

# ...
trade_hist = Trade.get_trades_proxy(pair='ETH/USDT', is_open=False, open_date=current_date - timedelta(days=2))

```

`get_trades_proxy()` 支持以下关键字参数。所有参数都是可选的 - 不带参数调用 `get_trades_proxy()` 将返回数据库中的所有交易列表。

* `pair` 例如 `pair='ETH/USDT'`
* `is_open` 例如 `is_open=False`
* `open_date` 例如 `open_date=current_date - timedelta(days=2)`
* `close_date` 例如 `close_date=current_date - timedelta(days=5)`

### get_open_trade_count

获取当前开放交易的数量

```python
from freqtrade.persistence import Trade
# ...
open_trades = Trade.get_open_trade_count()
```

### get_total_closed_profit

检索机器人到目前为止产生的总利润。
汇总所有已关闭交易的 `close_profit_abs`。

```python
from freqtrade.persistence import Trade

# ...
profit = Trade.get_total_closed_profit()
```

### total_open_trades_stakes

检索当前在交易中的总 stake_amount。

```python
from freqtrade.persistence import Trade

# ...
profit = Trade.total_open_trades_stakes()
```

### get_overall_performance

检索整体表现 - 类似于 `/performance` telegram 命令。

```python
from freqtrade.persistence import Trade

# ...
if self.config['runmode'].value in ('live', 'dry_run'):
    performance = Trade.get_overall_performance()
```

示例返回值：ETH/BTC 有 5 笔交易，总利润为 1.5%（比率为 0.015）。

```json
{"pair": "ETH/BTC", "profit": 0.015, "count": 5}
```

## Order 对象

`Order` 对象代表交易所上的订单（或模拟模式下的模拟订单）。
`Order` 对象将始终与其对应的 [`Trade`](#trade-object) 绑定，并且只在交易的上下文中才有意义。

### Order - 可用属性

Order 对象通常附加到交易上。
这里的大多数属性可能为 None，因为它们依赖于交易所的响应。

|  属性 | 数据类型 | 描述 |
|------------|-------------|-------------|
| `trade` | Trade | 此订单附加到的交易对象 |
| `ft_pair` | string | 此订单的交易对 |
| `ft_is_open` | boolean | 订单是否仍然开放？ |
| `order_type` | string | 交易所定义的订单类型 - 通常是市价单、限价单或止损单 |
| `status` | string | 由 [ccxt 的订单结构](https://docs.ccxt.com/#/README?id=order-structure) 定义的状态。通常是开放、关闭、过期、取消或拒绝 |
| `side` | string | 买入或卖出 |
| `price` | float | 订单放置的价格 |
| `average` | float | 订单成交的平均价格 |
| `amount` | float | 基础货币的数量 |
| `filled` | float | 已成交数量（以基础货币计）（请使用 `safe_filled` 代替） |
| `safe_filled` | float | 已成交数量（以基础货币计）- 保证不为 None |
| `remaining` | float | 剩余数量（请使用 `safe_remaining` 代替） |
| `safe_remaining` | float | 剩余数量 - 从交易所获取或计算得出。 |
| `cost` | float | 订单成本 - 通常是 average * filled（*在期货中取决于交易所，可能包含带或不带杠杆的成本，可能以合约计。*） |
| `stake_amount` | float | 用于此订单的 stake 金额。*2023.7 版本添加。* |
| `stake_amount_filled` | float | 用于此订单的已成交 stake 金额。*2024.11 版本添加。* |
| `order_date` | datetime | 订单创建日期 **请使用 `order_date_utc` 代替** |
| `order_date_utc` | datetime | 订单创建日期（UTC 时间） |
| `order_fill_date` | datetime | 订单成交日期 **请使用 `order_fill_utc` 代替** |
| `order_fill_date_utc` | datetime | 订单成交日期 |
