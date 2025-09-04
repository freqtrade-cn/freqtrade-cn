---
title: 配置 Freqtrade
description: 了解如何配置 Freqtrade 交易机器人,包括配置文件、环境变量等多种配置方式
subject: 配置指南
subtitle: 完整的配置参数说明
short_title: 配置
tags: [配置, 参数, 环境变量, 设置]
categories: [基础教程]
prerequisites: [安装完成]
difficulty: intermediate
estimated_time: 60分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 4
toc: true
---

# 配置机器人

Freqtrade 拥有众多可配置的功能和选项。

默认情况下，这些设置通过配置文件进行管理（见下文）。

## Freqtrade 配置文件

机器人在运行过程中会使用一组配置参数，这些参数共同构成了机器人的配置。通常，机器人会从一个文件（Freqtrade 配置文件）中读取其配置。

默认情况下，机器人会从当前工作目录下的 `config.json` 文件加载配置。

你可以通过 `-c/--config` 命令行选项指定机器人使用的其他配置文件。

如果你使用了[快速入门](docker_quickstart.md#docker-quick-start)方法安装机器人，安装脚本应该已经为你创建了默认的配置文件（`config.json`）。

如果没有创建默认配置文件，建议使用 `freqtrade new-config --config user_data/config.json` 生成一个基础配置文件。

Freqtrade 配置文件需采用 `JSON` 格式编写。

除了标准的 JSON 语法外，你还可以在配置文件中使用单行 `// ...` 和多行 `/* ... */` 注释，以及参数列表中的尾随逗号。

如果你不熟悉 JSON 格式也不用担心——只需用你喜欢的编辑器打开配置文件，修改你需要的参数，保存更改，最后重启机器人或在停止后重新运行即可。机器人在启动时会验证配置文件的语法，如果你编辑时出现错误，会在启动时警告并指出有问题的行。

(environment-variables)=

### 环境变量

可以通过环境变量设置 Freqtrade 配置中的选项。
这会优先于配置文件或策略中的对应值。

:::{note} __ 和 ___ 的说明

下文中的 `__` 是两个 `_`

下文中的 `___` 是三个 `_`

:::

环境变量必须以 `FREQTRADE__` 为前缀，才能被 freqtrade 加载到配置中。

`__` 作为层级分隔符，因此格式应为 `FREQTRADE__{section}__{key}`。
例如，定义 `export FREQTRADE__STAKE_AMOUNT=200` 会在配置中生成 `{stake_amount: 200}`。

更复杂的例子如 `export FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>`，用于保护你的交易所密钥安全。这样会将值放入配置的 `exchange.key` 部分。
使用这种方式，所有配置项都可以通过环境变量设置。

请注意，环境变量会覆盖配置文件中的对应设置，但命令行参数始终具有最高优先级。

常见示例：

```bash
FREQTRADE__TELEGRAM__CHAT_ID=<telegramchatid>
FREQTRADE__TELEGRAM__TOKEN=<telegramToken>
FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
FREQTRADE__EXCHANGE__SECRET=<yourExchangeSecret>
```

`JSON` 列表会被解析为 `json`，因此你可以如下设置交易对列表：

```bash
export FREQTRADE__EXCHANGE__PAIR_WHITELIST='["BTC/USDT", "ETH/USDT"]'
```

:::{caution} 3 种变量
检测到的环境变量会在启动时记录日志——如果你发现某个值不是你期望的，请确认它是否被环境变量覆盖。

上面提到了 3 种变量,它们的优先级从高到低如下:

1. 命令行参数优先级最高
1. 环境变量次之
1. 配置文件中的配置优先级最低

:::

:::{tip} 验证合并结果
你可以使用 [show-config 子命令](utils.md#show-config) 查看最终合并后的配置。
:::

:::{warning} 加载顺序
环境变量在初始配置加载后才会被加载。因此，无法通过环境变量提供配置文件路径。请使用命令行参数指定 `--config path/to/config.json`。

这同样适用于 `user_dir`。虽然可以通过环境变量设置用户目录，但配置不会从该位置加载。
:::

(multiple-configuration-files)=

### 多配置文件

可以为机器人指定多个配置文件，或让机器人从标准输入流读取配置参数。

你可以在 `add_config_files` 中指定额外的配置文件。

该参数指定的文件会在初始配置文件加载后合并。

文件路径相对于初始配置文件解析。

这类似于使用多个 `--config` 参数，但更简单，无需每次都为所有命令指定所有文件。

:::{tip} 验证合并结果
你可以使用 [show-config 子命令](utils.md#show-config) 查看最终合并后的配置。
:::

:::{tip} 用多配置文件保护密钥
你可以使用第二个配置文件存放密钥，这样可以分享"主"配置文件，同时保护 API 密钥。

第二个文件只需覆盖你想要隐藏的内容。

如果某个键在多个配置文件中出现，则"最后指定的配置"优先（如 `config-private.json`）。

对于一次性命令，也可以如下指定多个 "--config" 参数：

```bash
freqtrade trade --config user_data/config1.json --config user_data/config-private.json <...>
```

下面的方式等价于上例，但通过配置文件中的 `add_config_files`，便于复用，某个配置文件 "user_data/config.json" 的配置片段如下：

```json
"add_config_files": [
    "config1.json",
    "config-private.json"
]
```

```bash
freqtrade trade --config user_data/config.json <...>
```

:::

:::{caution} 配置冲突处理
如果 `config.json` 和 `config-import.json` 都有同一配置项，则父配置优先。

如下例，合并后 `max_open_trades` 为 3，因为主配置覆盖了可复用的"import"配置。

"user_data/config.json" 部分代码如下：

```json
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "add_config_files": [
        "config-import.json"
    ]
}
```

"user_data/config-import.json" 部分代码如下：

```json
{
    "max_open_trades": 10,
    "stake_amount": "unlimited",
}
```

合并后的配置：

```json title="Result"
{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "stake_amount": "unlimited"
}
```

如果 `add_config_files` 中有多个文件，则它们被视为同一级，后出现的覆盖前面的（除非父级已定义该键）。
:::

## 编辑器自动补全与校验

如果你使用支持 JSON schema 的编辑器，可以通过在配置文件顶部添加以下内容，获得自动补全和校验功能：

```json
{
    "$schema": "https://schema.freqtrade.io/schema.json",
}
```

:::{caution} 开发版 schema
开发版 schema 可通过 `https://schema.freqtrade.io/schema_dev.json` 获取，但建议使用稳定版以获得最佳体验。
:::

(configuration-parameters)=

## 配置参数

下表列出了所有可用的配置参数。

Freqtrade 也可以通过命令行（CLI）参数加载许多选项（可通过 `--help` 查看详情）。

(configuration-option-prevalence)=

### 配置项优先级

所有选项的优先级如下：

* 命令行参数优先于其他所有选项
* [环境变量](#environment-variables)
* 配置文件按顺序使用（最后的优先），并覆盖策略配置
* 策略配置仅在未通过配置文件或命令行参数设置时使用。下表中带有 [策略可覆盖](#configuration-parameters) 标记的选项即为如此。

(parameters-list)=

### 参数表

必填参数标记为 **必需**，表示必须通过某种方式设置。

```{note} 参数表格
:class: col-page-right
|  参数 | 描述 | 是否必需 | 数据类型 |
|------------|-------------|-------------|-------------|
| `max_open_trades` | **必需**  | 允许机器人同时持有的最大交易数。每个交易对只能有一个持仓，因此交易对列表长度也是一个限制。如果为 -1，则不限制（即可能无限开仓，受交易对列表限制）。[详见下文](#configuring-amount-per-trade)。[策略可覆盖](#parameters-in-the-strategy)。  | 正整数或 -1。 |
| `stake_currency` | **必需**  | 用于交易的加密货币。 | 字符串 |
| `stake_amount` | **必需**  | 每笔交易使用的加密货币数量。设为 "unlimited" 可让机器人使用全部可用余额。[详见下文](#configuring-amount-per-trade)。 | 正浮点数或 "unlimited"。 |
| `tradable_balance_ratio` |  | 机器人可用于交易的总账户余额比例。[详见下文](#configuring-amount-per-trade)。<br>*默认值为 `0.99`（99%）。* | 0.1 到 1.0 之间的正浮点数。 |
| `available_capital` |  | 机器人可用的起始资金。适用于在同一交易所账户运行多个机器人。[详见下文](#configuring-amount-per-trade)。 | 正浮点数。 |
| `amend_last_stake_amount` |  | 如有需要，使用减少的最后一笔交易金额。[详见下文](#configuring-amount-per-trade)。<br>*默认值为 `false`。* | 布尔值 |
| `last_stake_amount_min_ratio` |  | 定义最后一笔交易金额的最小比例，仅在 `amend_last_stake_amount` 为 true 时生效。[详见下文](#configuring-amount-per-trade)。<br>*默认值为 `0.5`。* | 浮点数（比例） |
| `amount_reserve_percent` |  | 在最小交易金额中预留部分金额。机器人在计算最小交易金额时会预留 `amount_reserve_percent` + 止损值，以避免下单被拒。[详见下文](#configuring-amount-per-trade)。<br>*默认值为 `0.05`（5%）。* | 正浮点数（比例）。 |
| `timeframe` |  | 使用的时间周期（如 `1m`、`5m`、`15m`、`30m`、`1h` ...）。通常在策略中指定。[策略可覆盖](#parameters-in-the-strategy)。 | 字符串 |
| `fiat_display_currency` |  | 用于显示利润的法币。[详见下文](#what-values-can-be-used-for-fiat_display_currency)。 | 字符串 |
| `dry_run` | **必需**  | 定义机器人是否处于模拟（Dry Run）或实盘模式。<br>*默认值为 `true`。* | 布尔值 |
| `dry_run_wallet` |  | 定义机器人在 Dry Run 模式下模拟钱包的起始金额。[详见下文](#dry-run-wallet)<br>*默认值为 `1000`。* | 浮点数或字典 |
| `cancel_open_orders_on_exit` |  | 当通过 `/stop` RPC 命令、按下 `Ctrl+C` 或机器人异常退出时，取消未成交订单。设为 true 可在市场崩盘时通过 `/stop` 取消未成交订单，不影响已开仓位。<br>*默认值为 `false`。* | 布尔值 |
| `process_only_new_candles` |  | 仅在新 K 线到来时处理指标。若为 false，每次循环都会处理指标，可能增加系统负载，但对依赖 tick 数据的策略有用。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `true`。* | 布尔值 |
| `minimal_roi` | **必需**  | 设置机器人退出交易的最小收益率阈值。[详见下文](#understand-minimal_roi)。[策略可覆盖](#parameters-in-the-strategy)。 | 字典 |
| `stoploss` | **必需**  | 机器人使用的止损值（比例）。详见[止损文档](stoploss.md)。[策略可覆盖](#parameters-in-the-strategy)。 | 浮点数（比例） |
| `trailing_stop` |  | 启用跟踪止损（基于配置或策略文件中的 `stoploss`）。详见[止损文档](stoploss.md#trailing-stop-loss)。[策略可覆盖](#parameters-in-the-strategy)。 | 布尔值 |
| `trailing_stop_positive` |  | 达到盈利后调整止损。详见[止损文档](stoploss.md#trailing-stop-loss-custom-positive-loss)。[策略可覆盖](#parameters-in-the-strategy)。 | 浮点数 |
| `trailing_stop_positive_offset` |  | 何时应用 `trailing_stop_positive` 的偏移量。百分比值，需为正数。详见[止损文档](stoploss.md#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset)。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0.0`（无偏移）。* | 浮点数 |
| `trailing_only_offset_is_reached` |  | 仅在达到偏移量时应用跟踪止损。[止损文档](stoploss.md)。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* | 布尔值 |
| `fee` |  | 回测/模拟时使用的手续费。通常不需配置，freqtrade 会使用交易所默认手续费。按比例设置（如 0.001 = 0.1%）。每笔交易买卖各收一次手续费。 | 浮点数（比例） |
| `futures_funding_rate` |  | 当无法从交易所获取历史资金费率时使用的用户自定义资金费率。不会覆盖真实历史费率。建议除非测试特定币种且了解资金费率影响，否则设为 0。[详见此处](leverage.md#unavailable-funding-rates)<br>*默认值为 `None`。* | 浮点数 |
| `trading_mode` |  | 指定常规交易(regularly)、杠杆交易(leverage)或合约交易(contracts)。[杠杆文档](leverage.md)。<br>*默认值为 `"spot"`。* | 字符串 |
| `margin_mode` |  | 杠杆交易时，决定保证金是全仓还是逐仓。[杠杆文档](leverage.md)。 | 字符串 |
| `liquidation_buffer` |  | 设置止损价与强平价之间的安全缓冲区比例，防止触发强平。[杠杆文档](leverage.md)。<br>*默认值为 `0.05`。* | 浮点数 |
|  **未成交超时** | | |
| `unfilledtimeout.entry` | **必需**  | 机器人等待未成交买单的时间（分钟或秒），超时后取消订单。[策略可覆盖](#parameters-in-the-strategy)。 | 整数 |
| `unfilledtimeout.exit` | **必需**  | 机器人等待未成交卖单的时间（分钟或秒），超时后取消订单并以当前价格重新下单（只要有信号）。[策略可覆盖](#parameters-in-the-strategy)。 | 整数 |
| `unfilledtimeout.unit` |  | 未成交超时的单位。注意：若设为 "seconds"，则 "internals.process_throttle_secs" 必须小于等于超时时间。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `"minutes"`。* | 字符串 |
| `unfilledtimeout.exit_timeout_count` |  | 卖单超时次数上限。达到该次数后触发紧急退出。0 表示不限制。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0`。* | 整数 |
| **定价** | | |
| `entry_pricing.price_side` |  | 选择机器人获取买入价格时参考的盘口方向。[详见下文](#entry-price)。<br>*默认值为 `"same"`。* | 字符串（`ask`、`bid`、`same` 或 `other`） |
| `entry_pricing.price_last_balance` | **必需**  | 插值买入价格。[详见下文](#entry-price-without-orderbook-enabled)。 |  |
| `entry_pricing.use_order_book` |  | 启用基于盘口的买入。[详见下文](#entry-price-with-orderbook-enabled)。<br>*默认值为 `true`。* | 布尔值 |
| `entry_pricing.order_book_top` |  | 机器人在盘口中选取第 N 个价格作为买入价。如为 2，则选取盘口中的第二个价格。[详见下文](#entry-price-with-orderbook-enabled)。<br>*默认值为 `1`。* | 正整数 |
| `entry_pricing.check_depth_of_market.enabled` |  | 若盘口买卖单差额达到设定值则不买入。[详见下文](#check-depth-of-market)。<br>*默认值为 `false`。* | 布尔值 |
| `entry_pricing.check_depth_of_market.bids_to_ask_delta` |  | 盘口买卖单差额比例。小于 1 表示卖单大于买单，大于 1 表示买单大于卖单。[详见下文](#check-depth-of-market)。<br>*默认值为 `0`。* | 浮点数（比例） |
| `exit_pricing.price_side` |  | 选择机器人获取卖出价格时参考的盘口方向。[详见下文](#exit-price-side)。<br>*默认值为 `"same"`。* | 字符串（`ask`、`bid`、`same` 或 `other`） |
| `exit_pricing.price_last_balance` |  | 插值卖出价格。[详见下文](#exit-price-without-orderbook-enabled)。  | |
| `exit_pricing.use_order_book` |  | 启用基于盘口的卖出。[详见下文](#exit-price-without-orderbook-enabled)。<br>*默认值为 `true`。* | 布尔值 |
| `exit_pricing.order_book_top` |  | 机器人在盘口中选取第 N 个价格作为卖出价。如为 2，则选取盘口中的第二个价格。[详见下文](#exit-price-without-orderbook-enabled)。<br>*默认值为 `1`。* | 正整数 |
| `custom_price_max_distance_ratio` |  | 配置当前价与自定义买入/卖出价的最大距离比例。<br>*默认值为 `0.02`（2%）。* | 正浮点数 |
|  **订单/信号处理** | | |
| `use_exit_signal` |  | 是否使用策略生成的退出信号，配合 `minimal_roi`。设为 false 时禁用 "exit_long" 和 "exit_short" 列。对其他退出方式（止损、ROI、回调）无影响。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `true`。* | 布尔值 |
| `exit_profit_only` |  | 仅在达到 `exit_profit_offset` 后才考虑退出。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* | 布尔值 |
| `exit_profit_offset` |  | 仅在高于该值时退出信号才生效，仅与 `exit_profit_only=True` 联用。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0.0`。* | 浮点数（比例） |
| `ignore_roi_if_entry_signal` |  | 若买入信号仍然有效，则不退出。优先于 `minimal_roi` 和 `use_exit_signal`。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* | 布尔值 |
| `ignore_buying_expired_candle_after` |  | 指定买入信号过期的秒数。 | 整数 |
| `order_types` |  | 根据操作（"entry"、"exit"、"stoploss"、"emergency_exit"、"force_exit"、"force_entry"）配置订单类型。[详见下文](#understand-order_types)。[策略可覆盖](#parameters-in-the-strategy)。 | 字典 |
| `order_time_in_force` |  | 配置买入和卖出订单的有效期策略。[详见下文](#understand-order_time_in_force)。[策略可覆盖](#parameters-in-the-strategy)。 | 字典 |
| `position_adjustment_enable` |  | 启用策略使用持仓调整（加仓/减仓）。[详见此处](strategy-callbacks.md#adjust-trade-position)。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* | 布尔值 |
| `max_entry_position_adjustment` |  | 每笔持仓最多可加仓次数，-1 表示不限制。[详见此处](strategy-callbacks.md#adjust-trade-position)。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `-1`。* | 正整数或 -1 |
| **交易所** | | |
| `exchange.name` | **必需**  | 交易所名称。 | 字符串 |
| `exchange.key` |  | 交易所 API key，仅生产模式下必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `exchange.secret` |  | 交易所 API secret，仅生产模式下必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `exchange.password` |  | 交易所 API password，仅部分交易所和生产模式下必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `exchange.uid` |  | 交易所 API uid，仅部分交易所和生产模式下必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `exchange.pair_whitelist` |  | 机器人用于交易和回测的交易对列表。支持正则表达式（如 .*/BTC）。不适用于 VolumePairList。[详见](plugins.md#pairlists-and-pairlist-handlers)。 | 列表 |
| `exchange.pair_blacklist` |  | 机器人必须避免交易和回测的交易对列表。[详见](plugins.md#pairlists-and-pairlist-handlers)。 | 列表 |
| `exchange.ccxt_config` |  | 传递给 ccxt 实例（同步和异步）的额外参数。参数因交易所而异，详见 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。请避免在此处添加密钥（请用专用字段），以免泄露到日志。 | 字典 |
| `exchange.ccxt_sync_config` |  | 传递给同步 ccxt 实例的额外参数。参数因交易所而异，详见 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。 | 字典 |
| `exchange.ccxt_async_config` |  | 传递给异步 ccxt 实例的额外参数。参数因交易所而异，详见 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation)。 | 字典 |
| `exchange.enable_ws` |  | 启用交易所 Websocket。[详见](#consuming-exchange-websockets)。<br>*默认值为 `true`。* | 布尔值 |
| `exchange.markets_refresh_interval` |  | 市场信息刷新间隔（分钟）。<br>*默认值为 `60` 分钟。* | 正整数 |
| `exchange.skip_open_order_update` |  | 启动时跳过未成交订单更新，仅在实盘下相关。<br>*默认值为 `false`* | 布尔值 |
| `exchange.unknown_fee_rate` |  | 计算手续费时的备用值，适用于手续费为非可交易币种的交易所。此值会与"手续费成本"相乘。<br>*默认值为 `None`* | 浮点数 |
| `exchange.log_responses` |  | 记录交易所响应，仅调试模式下使用，谨慎开启。<br>*默认值为 `false`* | 布尔值 |
| `exchange.only_from_ccxt` |  | 禁止从 data.binance.vision 下载数据。设为 false 可加快下载速度，但若该站点不可用可能出错。<br>*默认值为 `false`* | 布尔值 |
| `experimental.block_bad_exchanges` |  | 屏蔽已知与 freqtrade 不兼容的交易所。除非测试新交易所，否则建议保持默认。<br>*默认值为 `true`。* | 布尔值 |
| **插件** | | |
| `edge.*` |  | 详见 [edge 配置文档](edge.md)。  | |
| `pairlists` |  | 定义一个或多个交易对列表。[详见](plugins.md#pairlists-and-pairlist-handlers)。<br>*默认值为 `StaticPairList`。* | 字典列表 |
| **Telegram** | | |
| `telegram.enabled` |  | 启用 Telegram。| 布尔值  |
| `telegram.token` |  | Telegram 机器人 token，仅在 `telegram.enabled` 为 true 时必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `telegram.chat_id` |  | 你的 Telegram 账号 id，仅在 `telegram.enabled` 为 true 时必需。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `telegram.balance_dust_level` |  | 尘埃级别（以 stake 货币计），余额低于此值的币种不会在 `/balance` 中显示。 | 浮点数 |
| `telegram.reload` |  | 允许 Telegram 消息中显示"重载"按钮。<br>*默认值为 `true`。* | 布尔值 |
| `telegram.notification_settings.*` |  | 详细通知设置。详见 [telegram 文档](telegram-usage.md)。 | 字典 |
| `telegram.allow_custom_messages` |  | 允许策略通过 dataprovider.send_msg() 发送 Telegram 消息。 | 布尔值 |
| **Webhook** | | |
| `webhook.enabled` |  | 启用 Webhook 通知。 | 布尔值 |
| `webhook.url` |  | Webhook 地址，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.entry` |  | 买入时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.entry_cancel` |  | 买单取消时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.entry_fill` |  | 买单成交时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.exit` |  | 卖出时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.exit_cancel` |  | 卖单取消时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.exit_fill` |  | 卖单成交时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.status` |  | 状态调用时发送的 payload，仅在 `webhook.enabled` 为 true 时必需。<br>详见 [webhook 文档](webhook-config.md)。 | 字符串 |
| `webhook.allow_custom_messages` |  | 允许策略通过 dataprovider.send_msg() 发送 Webhook 消息。 | 布尔值 |
| **Rest API / FreqUI / 生产者-消费者** | | |
| `api_server.enabled` |  | 启用 API Server。<br>详见 [API Server 文档](rest-api.md)。 | 布尔值 |
| `api_server.listen_ip_address` |  | 绑定 IP 地址。<br>详见 [API Server 文档](rest-api.md)。 | IPv4 |
| `api_server.listen_port` |  | 绑定端口。<br>详见 [API Server 文档](rest-api.md)。| 1024-65535 之间的整数 |
| `api_server.verbosity` |  | 日志详细程度。<br>`info` 显示所有 RPC 调用，`error` 仅显示错误。| 枚举，`info` 或 `error`。默认 `info`。 |
| `api_server.username` |  | API server 用户名。<br>详见 [API Server 文档](rest-api.md)。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `api_server.password` |  | API server 密码。<br>详见 [API Server 文档](rest-api.md)。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `api_server.ws_token` |  | 消息 WebSocket 的 API token。<br>详见 [API Server 文档](rest-api.md)。<br>**请妥善保管，切勿泄露。** | 字符串 |
| `bot_name` |  | 机器人名称。通过 API 传递给客户端，可用于区分/命名机器人。<br>*默认值为 `freqtrade`* | 字符串 |
| `external_message_consumer` |  | 启用[生产者/消费者模式](producer-consumer.md)。 | 字典 |
| **其他** | | |
| `initial_state` |  | 定义应用初始状态。设为 stopped 时，需通过 `/start` RPC 命令手动启动。<br>*默认值为 `stopped`。* | 枚举，`running`、`paused` 或 `stopped` |
| `force_entry_enable` |  | 启用强制建仓的 RPC 命令。详见下文。 | 布尔值 |
| `disable_dataframe_checks` |  | 禁用对策略方法返回的 OHLCV dataframe 的正确性检查。仅在你明确知道自己在做什么时使用。[策略可覆盖](#parameters-in-the-strategy)。<br>*默认值为 `False`*。 | 布尔值 |
| `internals.process_throttle_secs` |  | 设置每次机器人主循环的最小持续时间（秒）。<br>*默认值为 `5` 秒。* | 正整数 |
| `internals.heartbeat_interval` |  | 每 N 秒打印一次心跳消息。设为 0 可禁用心跳。<br>*默认值为 `60` 秒。* | 正整数或 0 |
| `internals.sd_notify` |  | 启用 sd_notify 协议，向 systemd 服务管理器报告机器人状态并发送保活信号。详见 [此处](advanced-setup.md#configure-the-bot-running-as-a-systemd-service)。 | 布尔值 |
| `strategy` | **必需**  | 定义要使用的策略类。建议通过 `--strategy NAME` 设置。 | 类名 |
| `strategy_path` |  | 添加额外的策略查找路径（必须为目录）。 | 字符串 |
| `recursive_strategy_search` |  | 设为 true 时递归搜索 `user_data/strategies` 子目录下的策略。 | 布尔值 |
| `user_data_dir` |  | 用户数据目录。<br>*默认值为 `./user_data/`*。 | 字符串 |
| `db_url` |  | 声明要使用的数据库 URL。注意：若 `dry_run` 为 true，<br>默认为 `sqlite:///tradesv3.dryrun.sqlite`，<br>生产模式为 `sqlite:///tradesv3.sqlite`。 | 字符串，SQLAlchemy 连接字符串 |
| `logfile` |  | 指定日志文件名。采用滚动策略，最多保留 10 个 1 MB 日志文件。 | 字符串 |
| `add_config_files` |  | 额外配置文件。这些文件会与当前配置文件合并。路径相对于初始文件。<br>*默认值为 `[]`*。 | 字符串列表 |
| `dataformat_ohlcv` |  | 存储历史 K 线（OHLCV）数据的数据格式。<br>*默认值为 `feather`*。 | 字符串 |
| `dataformat_trades` |  | 存储历史成交数据的数据格式。<br>*默认值为 `feather`*。 | 字符串 |
| `reduce_df_footprint` |  | 将所有数值列转换为 float32/int32，以减少内存/磁盘占用（并加快回测/超参优化/FreqAI 训练推理）。 | 布尔值。默认值：`False`。 |
| `log_config` |  | python logging 的日志配置字典。[详见](advanced-setup.md#advanced-logging) | 字典。默认值：`FtRichHandler` |
```

(parameters-in-the-strategy)=

### 策略中的参数

以下参数可在配置文件或策略中设置。

配置文件中的值会覆盖策略中的值。

* `minimal_roi`
* `timeframe`
* `stoploss`
* `max_open_trades`
* `trailing_stop`
* `trailing_stop_positive`
* `trailing_stop_positive_offset`
* `trailing_only_offset_is_reached`
* `use_custom_stoploss`
* `process_only_new_candles`
* `order_types`
* `order_time_in_force`
* `unfilledtimeout`
* `disable_dataframe_checks`
* `use_exit_signal`
* `exit_profit_only`
* `exit_profit_offset`
* `ignore_roi_if_entry_signal`
* `ignore_buying_expired_candle_after`
* `position_adjustment_enable`
* `max_entry_position_adjustment`

(configuring-amount-per-trade)=

### 每笔交易的金额配置

有多种方式配置机器人每笔交易使用的 stake 货币数量。所有方法都遵循[可交易余额配置](#tradable-balance)的说明。

#### 最小交易金额

最小交易金额取决于交易所和交易对，通常可在交易所支持页面查到。

假设 XRP/USD 的最小可交易数量为 20 XRP（由交易所规定），价格为 0.6 美元，则买入该交易对的最小金额为 `20 * 0.6 ~= 12`。

该交易所还规定所有订单必须大于 10 美元，但本例不受影响。

为保证安全执行，freqtrade 不允许用 10.1 美元买入，而是确保有足够空间在交易对下方设置止损（加上 `amount_reserve_percent`，默认 5%）。

预留 5% 后，最小交易金额约为 12.6 美元（`12 * (1 + 0.05)`）。若再考虑 10% 止损，则为约 14 美元（`12.6 / (1 - 0.1)`）。

为防止止损过大导致计算值过高，计算出的最小交易金额不会超过实际限制的 1.5 倍。

:::{warning}
由于交易所的限制通常较为稳定且不常更新，部分交易对可能因币价大幅上涨而显示较高的最小限额。freqtrade 会将 stake-amount 调整为该值，除非超过期望值的 30%，此时会拒绝交易。
:::

(dry-run-wallet)=

#### Dry-run 钱包

在 dry-run 模式下，机器人会用模拟钱包执行交易。该钱包的起始余额由 `dry_run_wallet` 定义（默认 1000）。

对于更复杂的场景，也可以为 `dry_run_wallet` 分配一个字典，分别定义每种货币的起始余额。

```json
"dry_run_wallet": {
    "BTC": 0.01,
    "ETH": 2,
    "USDT": 1000
}
```

命令行参数（`--dry-run-wallet`）可覆盖配置值，但仅适用于浮点数，字典需在配置文件中设置。

:::{caution}
非 stake 货币的余额不会用于交易，但会在钱包余额中显示。

在全仓杠杆交易所，钱包余额可用于计算可用保证金。
:::

(tradable-balance)=

#### 可交易余额

默认情况下，机器人假定"总余额 - 1%"可用于交易，使用[动态交易金额](#dynamic-stake-amount)时，会将总余额按 `max_open_trades` 平均分配。

**freqtrade 会为手续费预留 1%，默认不会动用。**

可通过 `tradable_balance_ratio` 设置"保留"金额。

例如，钱包有 10 ETH，`tradable_balance_ratio=0.5`（50%），则机器人最多用 5 ETH 交易，其余不参与。

:::{danger}
运行多个机器人时**不应**使用此设置。请参考[分配可用资金](#assign-available-capital)。
:::

:::{warning}
`tradable_balance_ratio` 作用于当前余额（可用余额 + 持仓）。

假设起始余额 1000，`tradable_balance_ratio=0.99` 并不能保证始终有 10 单位货币可用。

例如，若总余额降至 500（因亏损或提现），可用余额也可能降至 5。
:::

(assign-available-capital)=

#### 分配可用资金

若在同一交易所账户运行多个机器人，为充分利用复利，可为每个机器人限定起始余额。

可通过设置 `available_capital` 实现。

假设账户有 10000 USDT，想运行 2 个策略，则设 `available_capital=5000`，每个机器人初始资金 5000 USDT。

机器人会将该余额平均分配到 `max_open_trades`。

盈利会增加该机器人的持仓，不影响其他机器人。

调整 `available_capital` 需重载配置。减少资金时不会平仓，差额在交易结束后返还钱包。实际效果取决于调整与平仓间的价格变动。

:::{warning} 与 `tradable_balance_ratio` 不兼容
设置此项会替换 `tradable_balance_ratio` 的配置。
:::

#### 调整最后一笔交易金额

假设可交易余额 1000 USDT，`stake_amount=400`，`max_open_trades=3`。

机器人会开 2 单，最后一单因余额不足 400 USDT 无法开仓。

可通过 `amend_last_stake_amount` 设为 `True`，使机器人自动将最后一单金额降为可用余额。

如上例：

* 交易1：400 USDT
* 交易2：400 USDT
* 交易3：200 USDT

:::{caution}
仅在[静态交易金额](#static-stake-amount)下生效，[动态交易金额](#dynamic-stake-amount)会自动平均分配余额。
:::

:::{caution}
最小最后一单金额可通过 `last_stake_amount_min_ratio` 配置，默认 0.5（50%），即最小为 `stake_amount * 0.5`，避免过小金额被交易所拒绝。
:::

(static-stake-amount)=

#### 静态交易金额

`stake_amount` 静态配置每笔交易的金额。

最小配置值为 0.0001，但请查阅交易所最低限额。

该设置与 `max_open_trades` 联用，最大持仓为 `stake_amount * max_open_trades`。

如配置 `max_open_trades=3`、`stake_amount=0.05`，则最多用 (0.05 BTC x 3) = 0.15 BTC。

:::{caution}
此设置遵循[可交易余额配置](#tradable-balance)。
:::

(dynamic-stake-amount)=

#### 动态交易金额

也可用动态交易金额，自动将可用余额按允许交易数平均分配。

配置方法：`stake_amount="unlimited"`，建议同时设 `tradable_balance_ratio=0.99`（99%），以预留手续费。

此时每笔交易金额为：

```python
currency_balance / (max_open_trades - current_open_trades)
```

如需让机器人用尽所有 `stake_currency`（减去 `tradable_balance_ratio`），配置如下：

```json
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
```

:::{tip} 复利
此配置可根据机器人表现动态调整持仓（亏损时减少，盈利时增加），实现复利。
:::

:::{warning} Dry-Run 模式下
Dry-Run、回测或超参优化时，`stake_amount : "unlimited",` 会用 `dry_run_wallet` 作为初始余额模拟。请合理设置 `dry_run_wallet`，否则可能模拟极大或极小金额，不符合实际。
:::

#### 动态交易金额与持仓调整

若需在无限持仓下用持仓调整，需在策略中实现 `custom_stake_amount`，返回合适的金额。

典型值为建议金额的 25%-50%，具体视策略和预留资金而定。

如持仓调整假设可加仓 2 次，则应预留 66.6667% 作为缓冲。

如假设可加仓 1 次且金额为原始的 3 倍，则 `custom_stake_amount` 应返回建议金额的 25%，其余 75% 预留。

```{include} includes/pricing.md
```

## 更多配置细节

(understand-minimal_roi)=

### 理解 minimal_roi

`minimal_roi` 配置参数为 JSON 对象，键为分钟数，值为最小收益率（比例）。
示例：

```json
"minimal_roi": {
    "40": 0.0,    # 40 分钟后只要不亏损就退出
    "30": 0.01,   # 30 分钟后至少盈利 1% 就退出
    "20": 0.02,   # 20 分钟后至少盈利 2% 就退出
    "0":  0.04    # 立即盈利 4% 就退出
},
```

大多数策略文件已包含最优的 `minimal_roi`。
该参数可在策略或配置文件中设置，配置文件优先。
若两者都未设置，则默认 1000%（`{"0": 10}`），即除非盈利 1000% 否则不退出。

:::{warning} 强制定时退出的特殊用法
ROI 设为 `"<N>": -1` 时，机器人会在 N 分钟后强制退出，无论盈亏。
:::

### 理解 force_entry_enable

`force_entry_enable` 配置参数允许通过 Telegram 和 REST API 使用强制建仓命令（`/forcelong`、`/forceshort`）。
出于安全考虑，默认禁用，启用时 freqtrade 会在启动时警告。
例如，发送 `/forceenter ETH/BTC`，机器人会买入该交易对并持有，直到出现常规退出信号（ROI、止损、/forceexit）。

部分策略下此功能风险较大，请谨慎使用。

详见 [telegram 文档](telegram-usage.md)。

### 忽略过期 K 线

使用较大周期（如 1h）且 `max_open_trades` 较低时，最后一根 K 线可能在交易位空出时被处理。此时买入信号可能已过最佳时机。

可通过设置 `ignore_buying_expired_candle_after`，指定买入信号过期秒数。

如策略用 1h 周期，只想在新 K 线前 5 分钟买入，可如下配置：

```json
  {
    //...
    "ignore_buying_expired_candle_after": 300,
    // ...
  }
```

:::{caution}

该设置会在每根新 K 线时重置，无法防止信号在第 2、3 根 K 线继续执行。建议用"触发"型买入信号，仅在一根 K 线内有效。

(understand-order_types)=

### 理解 order_types

`order_types` 配置参数将操作（`entry`、`exit`、`stoploss`、`emergency_exit`、`force_exit`、`force_entry`）映射为订单类型（`market`、`limit` 等），并配置止损是否在交易所上、止损更新间隔（秒）。

可用限价单买入、限价单卖出、用市价单止损。
也可设置止损"在交易所"，即买单成交后立即挂止损单。

配置文件中的 `order_types` 会整体覆盖策略中的设置，因此需在同一处配置完整字典。

若配置了该项，以下 4 个值（`entry`、`exit`、`stoploss`、`stoploss_on_exchange`）必须全部存在，否则机器人无法启动。

更多关于（`emergency_exit`、`force_exit`、`force_entry`、`stoploss_on_exchange`、`stoploss_on_exchange_interval`、`stoploss_on_exchange_limit_ratio`）请见[止损文档](stoploss.md)。

策略示例：

```python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": False,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99,
}
```

配置文件示例：

```json
"order_types": {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

:::{warning} 市价单支持
并非所有交易所都支持市价单。
若不支持，会提示：`"Exchange <yourexchange> does not support market orders."`，机器人将拒绝启动。
:::

:::{warning} 使用市价单
使用市价单时请仔细阅读[市价单定价](#market-order-pricing)章节。
:::

:::{warning} 止损在交易所
`order_types.stoploss_on_exchange_interval` 非必填，若不确定请勿更改。更多止损机制详见[止损文档](stoploss.md)。

若启用 `order_types.stoploss_on_exchange`，且手动取消了交易所止损单，机器人会重新挂单。
:::

:::{warning} order_types.stoploss_on_exchange 失败
若止损在交易所挂单失败，将触发"紧急退出"，默认用市价单平仓。可通过 `order_types` 字典中的 `emergency_exit` 更改，但不建议。
:::

(understand-order_time_in_force)=

### 理解 order_time_in_force

`order_time_in_force` 配置参数定义订单在交易所的执行策略。

常见有：

**GTC（Good Till Canceled）：**

默认策略，订单会一直保留，直到被用户取消。可全部或部分成交，未成交部分会一直挂单。

**FOK（Fill Or Kill）：**

若订单未能立即且全部成交，则被交易所取消。

**IOC（Immediate Or Canceled）：**

与 FOK 类似，但可部分成交，未成交部分自动取消。

**PO（Post only）：**

仅作为挂单，若不能作为挂单则取消。即订单必须至少挂在盘口一段时间。

请查看[交易所文档](exchanges.md)了解您的交易所支持的有效时间值。

#### time_in_force 配置

`order_time_in_force` 参数为字典，包含买入和卖出的策略。
可在配置文件或策略中设置。

配置文件中设置的值会覆盖策略中的值，遵循常规的[优先级规则](#configuration-option-prevalence)。

可选值：`GTC`（默认）、`FOK`、`IOC`。

```python
"order_time_in_force": {
    "entry": "GTC",
    "exit": "GTC"
},
```

:::{warning}
    请勿随意更改，除非你了解不同策略对交易所的影响。
:::

### 法币转换

Freqtrade 使用 Coingecko API 将币值转换为法币，用于 Telegram 报告。
法币可通过配置文件的 `fiat_display_currency` 设置。

移除 `fiat_display_currency` 可跳过 coingecko 初始化，不会显示法币转换，对机器人运行无影响。

(what-values-can-be-used-for-fiat_display_currency)=

#### 法币可用值

`fiat_display_currency` 配置参数设置 Telegram 报告中币转法币的基准货币。

可用值：

```json
"AUD", "BRL", "CAD", "CHF", "CLP", "CNY", 
"CZK", "DKK", "EUR", "GBP", "HKD", "HUF", 
"IDR", "ILS", "INR", "JPY", "KRW", "MXN", 
"MYR", "NOK", "NZD", "PHP", "PKR", "PLN", 
"RUB", "SEK", "SGD", "THB", "TRY", "TWD", 
"ZAR", "USD"
```

除法币外，还支持部分加密货币：

```json
"BTC", "ETH", "XRP", "LTC", "BCH", "BNB"
```

#### Coingecko 限流问题

部分 IP 段下，coingecko 限流严重。此时可在配置中添加 coingecko API key。

```json
{
    "fiat_display_currency": "USD",
    "coingecko": {
        "api_key": "your-api",
        "is_demo": true
    }
}
```

Freqtrade 支持 Coingecko Demo 和 Pro API key。

Coingecko API key 非机器人运行必需，仅用于 Telegram 报告中的币转法币，通常无需 key 也可用。

(consuming-exchange-websockets)=

## 使用交易所 Websocket

Freqtrade 可通过 ccxt.pro 消费交易所 websocket。

Freqtrade 旨在确保数据始终可用。

若 websocket 连接失败（或被禁用），机器人会回退到 REST API。

如遇到疑似 websocket 问题，可通过 `exchange.enable_ws` 关闭（默认 true）。

```jsonc
"exchange": {
    // ...
    "enable_ws": false,
    // ...
}
```

如需使用代理，详见[代理配置](#using-a-proxy-with-freqtrade)。

:::{hint} "逐步上线"
正在逐步上线，确保机器人稳定。目前仅限 ohlcv 数据流，且仅支持部分交易所，后续会持续增加。
:::

(dry-run-live-mode)=

## 使用 Dry-run 模式

建议先用 Dry-run 模式运行机器人，观察策略表现。

**Dry-run 模式下不会动用真实资金，仅做实时模拟。**

1. 编辑 `config.json` 配置文件。
2. 将 `dry_run` 设为 true，并指定 `db_url`。

```json
"dry_run": true,
"db_url": "sqlite:///tradesv3.dryrun.sqlite",
```

3. 移除交易所 `API key` 和 `secret`（可填空或假值）：

```json
"exchange": {
    "name": "binance",
    "key": "key",
    "secret": "secret",
    //"password": "", // 可选，并非所有交易所都需要
    // ...
}
```

`Dry-run` 模式下满意后，可切换到生产模式。

:::{caution}
`Dry-run` 模式下有模拟钱包，起始资金为 `dry_run_wallet`（默认 1000）。
:::

(using-dry-run-mode)=

### Dry-run 注意事项

* API key 可选。仅执行只读操作（不会更改账户状态）。
* 钱包（`/balance`）基于 `dry_run_wallet` 模拟。
* 订单为模拟，不会提交到交易所。
* 市价单按下单时盘口成交量成交，最大滑点 5%。
* 限价单在价格达到目标时成交，或按 `unfilledtimeout` 超时设置取消。
* 限价单若与市价差超 1%，会转为市价单立即成交。
* 配合 `stoploss_on_exchange` 时，假定止损价成交。
* 未成交订单（非已成交交易，后者存数据库）在机器人重启后仍保留，假定未成交。

## 切换到生产模式

生产模式下，机器人会动用真实资金。请谨慎操作，错误策略可能导致全部亏损。

切换到生产模式时，建议使用不同/全新数据库，避免 `dry-run` 交易影响真实资金和统计。

### 配置交易所账户

需在交易所网站创建 API Key（通常为 `key` 和 `secret`，部分还需 `password`），并填入配置文件或通过 `freqtrade new-config` 命令输入。

API Key 仅在实盘（生产模式）下必需，`dry-run` 模式可填空。

### 切换到生产模式

**编辑 `config.json` 文件。**

**将 dry_run 设为 false，并根据需要调整数据库 URL：**

```json
"dry_run": false,
```

**填入交易所 API key（可用假 key）：**

```json
{
    "exchange": {
        "name": "binance",
        "key": "af8ddd35195e9dc500b9a6f799f6f5c93d89193b",
        "secret": "08a9dc6db3d7b53e1acebd9275677f4b0a04f1a5",
        //"password": "", // 可选，并非所有交易所都需要
        // ...
    }
    //...
}
```

请务必阅读[交易所](exchanges.md)文档，了解各交易所的特殊配置。

:::{tip} 保护密钥
建议用第二个配置文件存放 API key。

将上述片段放入新文件（如 `config-private.json`），主配置文件仅存通用设置。

启动时用 `freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>` 加载密钥。

**切勿**与他人分享私密配置文件或交易所密钥！
:::

(using-a-proxy-with-freqtrade)=

## Freqtrade 使用代理

如需为 freqtrade 配置代理，导出 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量：

```bash
export HTTP_PROXY="http://addr:port"
export HTTPS_PROXY="http://addr:port"
freqtrade
```

### 代理交易所请求

如需为交易所连接配置代理，需在 ccxt 配置中指定：

```json
{ 
  "exchange": {
    "ccxt_config": {
      "httpsProxy": "http://addr:port",
      "wsProxy": "http://addr:port",
    }
  }
}
```

更多代理类型请查阅 [ccxt 代理文档](https://docs.ccxt.com/#/README?id=proxy)。

## 下一步

现在你已配置好 config.json，下一步请[启动机器人](bot-usage.md)。
