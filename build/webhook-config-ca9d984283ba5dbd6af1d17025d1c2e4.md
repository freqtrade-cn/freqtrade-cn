---
title: Webhook 配置指南
subject: Freqtrade Webhook 文档
subtitle: Webhook 配置和使用说明
short_title: Webhook 指南
description: Freqtrade 的 Webhook 功能允许您在交易事件发生时发送通知到外部服务。本文档详细介绍了如何配置和使用 Webhook。
tags: [Webhook, 通知, 外部服务, 事件, 配置]
categories: [通知与监控]
prerequisites: [Freqtrade已安装, 外部服务]
difficulty: intermediate
estimated_time: 25分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 18
toc: true
---

# Webhook 使用

## 配置

通过在配置文件中添加 webhook 部分并将 `webhook.enabled` 设置为 `true` 来启用 webhook。

示例配置（使用 IFTTT 测试）。

```json
  "webhook": {
        "enabled": true,
        "url": "https://maker.ifttt.com/trigger/<YOUREVENT>/with/key/<YOURKEY>/",
        "entry": {
            "value1": "买入 {pair}",
            "value2": "限价 {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
        "entry_cancel": {
            "value1": "取消 {pair} 的开放买单",
            "value2": "限价 {limit:8f}",
            "value3": "{stake_amount:8f} {stake_currency}"
        },
         "entry_fill": {
            "value1": "{pair} 买单已成交",
            "value2": "成交价 {open_rate:8f}",
            "value3": ""
        },
        "exit": {
            "value1": "退出 {pair}",
            "value2": "限价 {limit:8f}",
            "value3": "利润: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_cancel": {
            "value1": "取消 {pair} 的开放卖单",
            "value2": "限价 {limit:8f}",
            "value3": "利润: {profit_amount:8f} {stake_currency} ({profit_ratio})"
        },
        "exit_fill": {
            "value1": "{pair} 卖单已成交",
            "value2": "成交价 {close_rate:8f}",
            "value3": ""
        },
        "status": {
            "value1": "状态: {status}",
            "value2": "",
            "value3": ""
        }
    },
```

`webhook.url` 中的 url 应该指向您的 webhook 的正确 url。如果您使用 [IFTTT](https://ifttt.com)（如上面的示例所示），请在 url 中插入您的事件和密钥。

您可以将 POST 正文格式设置为表单编码（默认）、JSON 编码或原始数据。分别使用 `"format": "form"`、`"format": "json"` 或 `"format": "raw"`。Mattermost Cloud 集成的示例配置：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURSUBDOMAIN>.cloud.mattermost.com/hooks/<YOURHOOK>",
        "format": "json",
        "status": {
            "text": "状态: {status}"
        }
    },
```

结果将是一个 POST 请求，例如 `{"text":"状态: running"}` 正文和 `Content-Type: application/json` 头，这将在 Mattermost 频道中显示 `状态: running` 消息。

使用表单编码或 JSON 编码配置时，您可以配置任意数量的有效负载值，键和值都将在 POST 请求中输出。但是，使用原始数据格式时，您只能配置一个值，并且它**必须**命名为 `"data"`。在这种情况下，数据键不会在 POST 请求中输出，只会输出值。例如：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "format": "raw",
        "webhookstatus": {
            "data": "状态: {status}"
        }
    },
```

结果将是一个 POST 请求，例如 `状态: running` 正文和 `Content-Type: text/plain` 头。

### 嵌套 Webhook 配置

一些 webhook 目标需要嵌套结构。
这可以通过将内容设置为字典或列表而不是直接设置为文本来实现。

这仅支持 JSON 格式。

```json
"webhook": {
    "enabled": true,
    "url": "https://<yourhookurl>",
    "format": "json",
    "status": {
        "msgtype": "text",
        "text": {
            "content": "Status update: {status}"
        }
    }
}
```

结果将是一个带有例如 `{"msgtype":"text","text":{"content":"Status update: running"}}` 主体和 `Content-Type: application/json` 头的 POST 请求。

## 其他配置

`webhook.retries` 参数可以设置为 webhook 请求在失败时（即 HTTP 响应状态不是 200）应尝试的最大重试次数。默认情况下，这设置为 `0`，表示禁用。还可以设置额外的 `webhook.retry_delay` 参数来指定重试尝试之间的时间（以秒为单位）。默认情况下，这设置为 `0.1`（即 100ms）。请注意，如果 webhook 存在连接问题，增加重试次数或重试延迟可能会减慢交易者的速度。
您还可以指定 `webhook.timeout` - 它定义了机器人在认为其他主机无响应之前将等待多长时间（默认为 10 秒）。

重试的示例配置：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "timeout": 10,
        "retries": 3,
        "retry_delay": 0.2,
        "status": {
            "status": "状态: {status}"
        }
    },
```

可以通过策略中的 `self.dp.send_msg()` 函数将自定义消息发送到 Webhook 端点。要启用此功能，请将 `allow_custom_messages` 选项设置为 `true`：

```json
  "webhook": {
        "enabled": true,
        "url": "https://<YOURHOOKURL>",
        "allow_custom_messages": true,
        "strategy_msg": {
            "status": "策略消息: {msg}"
        }
    },
```

可以为不同事件配置不同的有效负载。并非所有字段都是必需的，但您应该至少配置其中一个字典，否则 webhook 将永远不会被调用。

## Webhook 消息类型

### 入场 / 入场成交

当机器人下达多头/空头订单以增加仓位时，或者当该订单成交时，`webhook.entry` 和 `webhook.entry_fill` 中的字段会被相应地填充。参数使用 string.format 填充。
可能的参数有：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* ~~`limit` # 已弃用 - 不应再使用。~~
* `open_rate`
* `amount`
* `open_date`
* `stake_amount`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `order_type`
* `current_rate`
* `enter_tag`

以下是翻译后的中文内容，保持了Markdown格式：

### 出场 / 出场成交

当机器人下达出场订单或该出场订单成交时，`webhook.exit` 和 `webhook.exit_fill` 中的字段会被相应地填充。参数使用 string.format 填充。
可能的参数有：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `amount`
* `open_rate`
* `close_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `enter_tag`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`
* `sub_trade`
* `is_final_exit`

### 出场取消

当机器人取消出场订单时，`webhook.exit_cancel` 中的字段会被填充。参数使用 string.format 填充。
可能的参数有：

* `trade_id`
* `exchange`
* `pair`
* `direction`
* `leverage`
* `gain`
* `order_rate`
* `amount`
* `open_rate`
* `current_rate`
* `profit_amount`
* `profit_ratio`
* `stake_currency`
* `base_currency`
* `quote_currency`
* `fiat_currency`
* `exit_reason`
* `order_type`
* `open_date`
* `close_date`

### 状态

`webhook.status` 中的字段用于常规状态消息（已启动 / 已停止 / ...）。参数使用 string.format 填充。

这里唯一可能的值是 `{status}`。

## Discord

Discord 提供了一种特殊形式的 webhook。
您可以按如下方式配置：

```json
"discord": {
    "enabled": true,
    "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
    "exit_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Close rate": "{close_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Close date": "{close_date:%Y-%m-%d %H:%M:%S}"},
        {"Profit": "{profit_amount} {stake_currency}"},
        {"Profitability": "{profit_ratio:.2%}"},
        {"Enter tag": "{enter_tag}"},
        {"Exit Reason": "{exit_reason}"},
        {"Strategy": "{strategy}"},
        {"Timeframe": "{timeframe}"},
    ],
    "entry_fill": [
        {"Trade ID": "{trade_id}"},
        {"Exchange": "{exchange}"},
        {"Pair": "{pair}"},
        {"Direction": "{direction}"},
        {"Open rate": "{open_rate}"},
        {"Amount": "{amount}"},
        {"Open date": "{open_date:%Y-%m-%d %H:%M:%S}"},
        {"Enter tag": "{enter_tag}"},
        {"Strategy": "{strategy} {timeframe}"},
    ]
}
```

上述配置代表默认设置（`exit_fill` 和 `entry_fill` 是可选的，将默认使用上述配置） - 显然可以进行修改。
要禁用这两个默认值中的任何一个（`entry_fill` / `exit_fill`），您可以为它们分配一个空数组（`exit_fill: []`）。

可用字段对应于 webhook 的字段，并在相应的 webhook 部分中有文档说明。

默认情况下，通知将如下所示。

![discord-notification](assets/discord_notification.png)

可以通过 dataprovider.send_msg() 函数从策略向 Discord 端点发送自定义消息。要启用此功能，请将 `allow_custom_messages` 选项设置为 `true`：

```json
  "discord": {
        "enabled": true,
        "webhook_url": "https://discord.com/api/webhooks/<Your webhook URL ...>",
        "allow_custom_messages": true,
    },
```
