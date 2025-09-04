---
title: 生产者/消费者模式
subject: Freqtrade 生产者/消费者指南
subtitle: 如何配置和使用生产者/消费者模式
short_title: 生产者/消费者
description: 本文档介绍了 Freqtrade 的生产者/消费者模式,包括如何配置多个机器人实例共享计算指标和信号,以及相关的 websocket 设置。
tags: [生产者, 消费者, 分布式, 多实例, WebSocket]
categories: [高级功能]
prerequisites: [Freqtrade已安装, 网络配置]
difficulty: advanced
estimated_time: 45分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 17
toc: true
---

# 生产者/消费者模式

freqtrade 提供了一种机制，使一个实例（也称为 `consumer`）可以通过消息 websocket 监听上游 freqtrade 实例（也称为 `producer`）的消息。主要是 `analyzed_df` 和 `whitelist` 消息。这允许在多个机器人中重用计算出的指标（和信号），而无需多次计算它们。

有关设置消息 websocket 的 `api_server` 配置（这将是你的生产者），请参阅 Rest API 文档中的[消息 Websocket](rest-api.md#message-websocket)。

:::{note}
我们强烈建议将 `ws_token` 设置为只有你自己知道的随机值，以避免未经授权访问你的机器人。
:::

## 配置

通过在消费者的配置文件中添加 `external_message_consumer` 部分来启用订阅实例。

```json
{
    //...
   "external_message_consumer": {
        "enabled": true,
        "producers": [
            {
                "name": "default", // 这可以是任何你喜欢的名称，默认为 "default"
                "host": "127.0.0.1", // 来自你的生产者 api_server 配置的主机
                "port": 8080, // 来自你的生产者 api_server 配置的端口
                "secure": false, // 使用安全 websockets 连接，默认为 false
                "ws_token": "sercet_Ws_t0ken" // 来自你的生产者 api_server 配置的 ws_token
            }
        ],
        // 以下配置是可选的，通常不需要
        // "wait_timeout": 300,
        // "ping_timeout": 10,
        // "sleep_time": 10,
        // "remove_entry_exit_signals": false,
        // "message_size_limit": 8
    }
    //...
}
```

|  参数 | 描述 |
|------------|-------------|
| `enabled` | **必需。** 启用消费者模式。如果设置为 false，则忽略此部分中的所有其他设置。<br>*默认为 `false`。*<br> **数据类型：** 布尔值。 |
| `producers` | **必需。** 生产者列表 <br> **数据类型：** 数组。 |
| `producers.name` | **必需。** 此生产者的名称。如果使用多个生产者，此名称必须用于调用 `get_producer_pairs()` 和 `get_producer_df()`。<br> **数据类型：** 字符串 |
| `producers.host` | **必需。** 来自你的生产者的主机名或 IP 地址。<br> **数据类型：** 字符串 |
| `producers.port` | **必需。** 与上述主机匹配的端口。<br>*默认为 `8080`。*<br> **数据类型：** 整数 |
| `producers.secure` | **可选。** 在 websockets 连接中使用 ssl。默认为 False。<br> **数据类型：** 字符串 |
| `producers.ws_token` | **必需。** 在生产者上配置的 `ws_token`。<br> **数据类型：** 字符串 |
| | **可选设置** |
| `wait_timeout` | 如果未收到消息，直到我们再次 ping 的超时时间。<br>*默认为 `300`。*<br> **数据类型：** 整数 - 以秒为单位。 |
| `ping_timeout` | Ping 超时 <br>*默认为 `10`。*<br> **数据类型：** 整数 - 以秒为单位。 |
| `sleep_time` | 重试连接前的睡眠时间。<br>*默认为 `10`。*<br> **数据类型：** 整数 - 以秒为单位。 |
| `remove_entry_exit_signals` | 在接收数据框时从数据框中删除信号列（将它们设置为 0）。<br>*默认为 `false`。*<br> **数据类型：** 布尔值。 |
| `initial_candle_limit` | 从生产者预期的初始蜡烛图数量。<br>*默认为 `1500`。*<br> **数据类型：** 整数 - 蜡烛图数量。 |
| `message_size_limit` | 每条消息的大小限制<br>*默认为 `8`。*<br> **数据类型：** 整数 - 兆字节。 |

消费者实例不是（或除了）在 `populate_indicators()` 中计算指标，而是监听与生产者实例消息的连接（或在高级配置中监听多个生产者实例），并请求生产者最近为活跃白名单中的每个交易对分析的数据框。

然后，消费者实例将拥有分析数据框的完整副本，而无需自己计算它们。

## 示例

### 示例 - 生产者策略

一个具有多个指标的简单策略。策略本身不需要特殊考虑。

```python
class ProducerStrategy(IStrategy):
    #...
    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        以标准的 freqtrade 方式计算指标，然后可以广播到其他实例
        """
        dataframe['rsi'] = ta.RSI(dataframe)
        bollinger = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=2)
        dataframe['bb_lowerband'] = bollinger['lower']
        dataframe['bb_middleband'] = bollinger['mid']
        dataframe['bb_upperband'] = bollinger['upper']
        dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        为给定的数据框填充入场信号
        """
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], self.buy_rsi.value)) &
                (dataframe['tema'] <= dataframe['bb_middleband']) &
                (dataframe['tema'] > dataframe['tema'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

:::{tip} FreqAI
你可以使用这个在功能强大的机器上设置 [FreqAI](freqai.md)，同时在树莓派等简单机器上运行消费者，这些消费者可以以不同的方式解释生产者生成的信号。
:::

### 示例 - 消费者策略

一个逻辑上等效的策略，它自己不计算任何指标，但将具有相同的分析数据框可用于基于生产者在生产者中计算的指标做出交易决策。在此示例中，消费者具有相同的入场标准，但这不是必需的。消费者可以使用不同的逻辑来入场/出场交易，并且只使用指定的指标。

```python
class ConsumerStrategy(IStrategy):
    #...
    process_only_new_candles = False # 消费者必需

    _columns_to_expect = ['rsi_default', 'tema_default', 'bb_middleband_default']

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        使用 websocket api 从另一个 freqtrade 实例获取预填充的指标。
        使用 `self.dp.get_producer_df(pair)` 获取数据框
        """
        pair = metadata['pair']
        timeframe = self.timeframe

        producer_pairs = self.dp.get_producer_pairs()
        # 你可以通过以下方式指定从哪个生产者获取交易对：
        # self.dp.get_producer_pairs("my_other_producer")

        # 此函数返回分析的数据框，以及它被分析的时间
        producer_dataframe, _ = self.dp.get_producer_df(pair)
        # 如果生产者提供，你可以获取其他数据：
        # self.dp.get_producer_df(
        #   pair,
        #   timeframe="1h",
        #   candle_type=CandleType.SPOT,
        #   producer_name="my_other_producer"
        # )

        if not producer_dataframe.empty:
            # 如果你计划直接传递生产者的入场/出场信号，
            # 指定 ffill=False，否则会有意外的结果
            merged_dataframe = merge_informative_pair(dataframe, producer_dataframe,
                                                      timeframe, timeframe,
                                                      append_timeframe=False,
                                                      suffix="default")
            return merged_dataframe
        else:
            dataframe[self._columns_to_expect] = 0

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        为给定的数据框填充入场信号
        """
        # 使用数据框列，就像我们自己计算它们一样
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi_default'], self.buy_rsi.value)) &
                (dataframe['tema_default'] <= dataframe['bb_middleband_default']) &
                (dataframe['tema_default'] > dataframe['tema_default'].shift(1)) &
                (dataframe['volume'] > 0)
            ),
            'enter_long'] = 1

        return dataframe
```

:::{tip} 使用上游信号
通过设置 `remove_entry_exit_signals=false`，你也可以直接使用生产者的信号。它们应该可以作为 `enter_long_default` 使用（假设使用了 `suffix="default"`）- 并且可以直接用作信号，或作为附加指标。
:::
