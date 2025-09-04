---
title: REST API 指南
subject: Freqtrade REST API 文档
subtitle: 如何配置和使用 REST API
short_title: REST API
keywords: [REST API, Freqtrade, 接口, 配置, 安全]
description: 本文档介绍了 Freqtrade 的 REST API 功能,包括如何配置 API 服务器、安全设置、身份验证以及各种 API 端点的使用方法。
tags: [REST API, 接口, 配置, 安全, 自动化]
categories: [开发者文档]
prerequisites: [Freqtrade已安装, 配置文件]
difficulty: intermediate
estimated_time: 30分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 13
toc: true
---

# REST API

## FreqUI

FreqUI 现在有了自己的[专用文档部分](freq-ui.md) - 请参考该部分获取所有关于 FreqUI 的信息。

## 配置

通过在配置中添加 api_server 部分并将 `api_server.enabled` 设置为 `true` 来启用 REST API。

示例配置：

```json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "127.0.0.1",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "somethingrandom",
        "CORS_origins": [],
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        "ws_token": "sercet_Ws_t0ken"
    },
```

:::{danger} 安全警告
默认情况下，配置仅监听 localhost（因此无法从其他系统访问）。我们强烈建议不要将此 API 暴露到互联网，并选择一个强且唯一的密码，因为其他人可能会控制您的机器人。
:::

:::{note} 远程服务器上的 API/UI 访问
如果您在 VPS 上运行，您应该考虑使用 ssh 隧道或设置 VPN（openVPN、wireguard）来连接到您的机器人。

这将确保 freqUI 不会直接暴露在互联网上，出于安全原因不建议这样做（freqUI 默认不支持 https）。

这些工具的设置不在本教程范围内，但可以在互联网上找到许多好的教程。
:::

然后，您可以通过在浏览器中访问 `http://127.0.0.1:8080/api/v1/ping` 来检查 API 是否正常运行。
这应该返回响应：

```output
{"status":"pong"}
```

所有其他端点都返回敏感信息并需要身份验证，因此无法通过 Web 浏览器访问。

### 安全

要生成安全密码，最好使用密码管理器，或使用以下代码。

```python
import secrets
secrets.token_hex()
```

:::{hint} JWT token
使用相同的方法也可以生成 JWT 密钥（`jwt_secret_key`）。
:::

:::{danger} 密码选择
请确保选择一个非常强且唯一的密码，以保护您的机器人免受未授权访问。

同时将 `jwt_secret_key` 更改为随机值（不需要记住它，但它将用于加密您的会话，所以最好是一个唯一的值！）。
:::

(configuration-with-docker)=
### Docker 配置

如果您使用 docker 运行机器人，您需要让机器人监听传入连接。安全由 docker 处理。

```json
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    },
```

确保您的 docker-compose 文件中有以下 2 行：

```yml
    ports:
      - "127.0.0.1:8080:8080"
```

:::{danger} 安全警告
通过在 docker 端口映射中使用 `"8080:8080"`（或 `"0.0.0.0:8080:8080"`），API 将对连接到服务器正确端口的每个人可用，因此其他人可能能够控制您的机器人。

如果您在安全环境（如家庭网络）中运行机器人，这**可能**是安全的，但不建议将 API 暴露到互联网。
:::

## REST API

### 使用 API

我们建议使用支持的 `freqtrade-client` 包（也可作为 `scripts/rest_client.py` 使用）来使用 API。

可以通过使用 `pip install freqtrade-client` 独立于任何正在运行的 freqtrade 机器人安装此命令。

该模块设计为轻量级，仅依赖于 `requests` 和 `python-rapidjson` 模块，跳过了 freqtrade 所需的所有重依赖。

```bash
freqtrade-client <command> [optional parameters]
```

默认情况下，脚本假设使用 `127.0.0.1`（localhost）和端口 `8080`，但您可以指定配置文件来覆盖此行为。

#### 最小化客户端配置

```json
{
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "username": "Freqtrader",
        "password": "SuperSecret1!",
        //...
    }
}
```

```bash
freqtrade-client --config rest_config.json <command> [optional parameters]
```

具有多个参数的命令可能需要关键字参数（为了清晰）- 可以按如下方式提供：

```bash
freqtrade-client --config rest_config.json forceenter BTC/USDT long enter_tag=GutFeeling
```

此方法适用于所有参数 - 使用 "show" 命令查看可用参数列表。

:::{note} 编程使用"
`freqtrade-client` 包（可独立于 freqtrade 安装）可用于您自己的脚本中以与 freqtrade API 交互。

为此，请使用以下内容：

```python
from freqtrade_client import FtRestClient


client = FtRestClient(server_url, username, password)

# 获取机器人状态
ping = client.ping()
print(ping)
# ... 
```

有关可用命令的完整列表，请参阅下面的列表。
:::

可以使用 `help` 命令从 rest-client 脚本中列出可能的命令。

```bash
freqtrade-client help
```

```output
可能的命令：

available_pairs
	返回基于时间框架/基础货币选择的可用交易对（回测数据）

        :param timeframe: 仅具有此时间框架的可用交易对。
        :param stake_currency: 仅包含此时间框架的交易对

balance
	获取账户余额。

blacklist
	显示当前黑名单。

        :param add: 要添加的币种列表（例如："BNB/BTC"）

cancel_open_order
	取消交易的开放订单。

        :param trade_id: 取消此交易的开放订单。

count
	返回开放交易的数量。

daily
	返回每天的利润和交易数量。

delete_lock
	从数据库中删除（禁用）锁定。

        :param lock_id: 要删除的锁定的 ID

delete_trade
	从数据库中删除交易。
        尝试关闭开放订单。需要在交易所手动处理此资产。

        :param trade_id: 从数据库中删除具有此 ID 的交易。

edge
	返回有关 Edge 的信息。

forcebuy
	购买资产。

        :param pair: 要购买的交易对（ETH/BTC）
        :param price: 可选 - 购买价格

forceenter
	强制进入交易

        :param pair: 要购买的交易对（ETH/BTC）
        :param side: 'long' 或 'short'
        :param price: 可选 - 购买价格

forceexit
	强制退出交易。

        :param tradeid: 交易的 ID（可以通过状态命令获取）
        :param ordertype: 要使用的订单类型（必须是 market 或 limit）
        :param amount: 要卖出的数量。如果未给出则全部卖出

health
	提供运行机器人的快速健康检查。

lock_add
    手动锁定特定交易对

        :param pair: 要锁定的交易对
        :param until: 锁定到此日期（格式 "2024-03-30 16:00:00Z"）
        :param side: 要锁定的方向（long、short、*）
        :param reason: 锁定的原因        

locks
	返回当前锁定

logs
	显示最新日志。

        :param limit: 将日志消息限制为最后 <limit> 条日志。无限制获取整个日志。

pair_candles
	返回 <pair><timeframe> 的实时数据框。

        :param pair: 获取数据的交易对
        :param timeframe: 仅具有此时间框架的可用交易对。
        :param limit: 将结果限制为最后 n 个蜡烛图。

pair_history
	返回历史分析数据框

        :param pair: 获取数据的交易对
        :param timeframe: 仅具有此时间框架的可用交易对。
        :param strategy: 用于分析和获取值的策略
        :param timerange: 获取数据的时间范围（与 --timerange 端点相同的格式）

performance
	返回不同币种的性能。

ping
	简单 ping

plot_config
	如果策略定义了一个，则返回绘图配置。

profit
	返回利润摘要。

reload_config
	重新加载配置。

show_config
        返回与交易操作相关的部分配置。

start
	如果机器人处于停止状态，则启动机器人。

pause
	如果机器人处于运行状态，则暂停机器人。如果在停止状态触发，将处理开放仓位。

stats
	返回统计报告（持续时间、卖出原因）。

status
	获取开放交易的状态。

stop
	停止机器人。使用 `start` 重新启动。

stopbuy
	停止买入（但优雅地处理卖出）。使用 `reload_config` 重置。

strategies
	列出可用策略

strategy
	获取策略详情

        :param strategy: 策略类名

sysinfo
	提供系统信息（CPU、RAM 使用情况）

trade
	返回特定交易

        :param trade_id: 指定要获取的交易。

trades
	按 id 排序返回交易历史

        :param limit: 将交易限制为最后 X 笔交易。最多 500 笔交易。
        :param offset: 按此数量的交易偏移。

list_open_trades_custom_data
    返回包含开放交易自定义数据的字典

        :param key: str, 可选 - 自定义数据的键
        :param limit: 将交易限制为 X 笔交易。
        :param offset: 按此数量的交易偏移。

list_custom_data
    返回包含指定交易自定义数据的字典

        :param trade_id: int - 交易的 ID
        :param key: str, 可选 - 自定义数据的键

version
	返回机器人的版本。

whitelist
	显示当前白名单。

```

### 可用端点

如果您希望通过其他方式手动调用 REST API，例如直接通过 `curl`，下表显示了相关的 URL 端点和参数。
下表中的所有端点都需要以 API 的基本 URL 为前缀，例如 `http://127.0.0.1:8080/api/v1/` - 因此命令变为 `http://127.0.0.1:8080/api/v1/<command>`。

|  端点 | 方法 | 描述 / 参数 |
|-----------|--------|--------------------------|
| `/ping` | GET | 测试 API 就绪性的简单命令 - 不需要身份验证。 |
| `/start` | POST | 启动交易者。 |
| `/pause` | POST | 暂停交易者。根据其规则优雅地处理开放交易。不进入新仓位。 |
| `/stop` | POST | 停止交易者。 |
| `/stopbuy` | POST | 停止交易者开仓。根据其规则优雅地关闭开放交易。 |
| `/reload_config` | POST | 重新加载配置文件。 |
| `/trades` | GET | 列出最后交易。每次调用限制为 500 笔交易。 |
| `/trade/<tradeid>` | GET | 获取特定交易。<br/>*参数：*<br/>- `tradeid` (`int`) |
| `/trades/<tradeid>` | DELETE | 从数据库中删除交易。尝试关闭开放订单。需要在交易所手动处理此交易。<br/>*参数：*<br/>- `tradeid` (`int`)  |
| `/trades/<tradeid>/open-order` | DELETE | 取消此交易的开放订单。<br/>*参数：*<br/>- `tradeid` (`int`)  |
| `/trades/<tradeid>/reload` | POST | 从交易所重新加载交易。仅在实盘模式下工作，可能有助于恢复在交易所手动卖出的交易。<br/>*参数：*<br/>- `tradeid` (`int`)  |
| `/show_config` | GET | 显示当前配置中与操作相关的部分设置。 |
| `/logs` | GET | 显示最后日志消息。 |
| `/status` | GET | 列出所有开放交易。 |
| `/count` | GET | 显示已使用和可用的交易数量。 |
| `/entries` | GET | 显示给定交易对（如果未给出交易对则为所有交易对）的每个入场标签的利润统计。交易对是可选的。<br/>*参数：*<br/>- `pair` (`str`)  |
| `/exits` | GET | 显示给定交易对（如果未给出交易对则为所有交易对）的每个出场原因的利润统计。交易对是可选的。<br/>*参数：*<br/>- `pair` (`str`)  |
| `/mix_tags` | GET | 显示给定交易对（如果未给出交易对则为所有交易对）的每个入场标签+出场原因组合的利润统计。交易对是可选的。<br/>*参数：*<br/>- `pair` (`str`)  |
| `/locks` | GET | 显示当前锁定的交易对。 |
| `/locks` | POST | 将交易对锁定到 "until"。（Until 将向上取整到最近的时间框架）。Side 是可选的，是 `long` 或 `short`（默认为 `long`）。Reason 是可选的。<br/>*参数：*<br/>- `<pair>` (`str`)<br/>- `<until>` (`datetime`)<br/>- `[side]` (`str`)<br/>- `[reason]` (`str`)  |
| `/locks/<lockid>` | DELETE | 通过 id 删除（禁用）锁定。<br/>*参数：*<br/>- `lockid` (`int`)  |
| `/profit` | GET | 显示已关闭交易的盈亏摘要和一些关于您表现的统计信息。 |
| `/forceexit` | POST | 立即退出给定交易（忽略 `minimum_roi`），使用给定订单类型（"market" 或 "limit"，如果未指定则使用您的配置设置），以及选择的数量（如果未指定则全部卖出）。如果 `all` 作为 `tradeid` 提供，则所有当前开放交易将被强制退出。<br/>*参数：*<br/>- `<tradeid>` (`int` 或 `str`)<br/>- `<ordertype>` (`str`)<br/>- `[amount]` (`float`) |
| `/forceenter` | POST | 立即进入给定交易对。Side 是可选的，是 `long` 或 `short`（默认为 `long`）。Rate 是可选的。（`force_entry_enable` 必须设置为 True）<br/>*参数：*<br/>- `<pair>` (`str`)<br/>- `<side>` (`str`)<br/>- `[rate]` (`float`) |
| `/performance` | GET | 显示按交易对分组的每个已完成交易的性能。 |
| `/balance` | GET | 显示每个货币的账户余额。 |
| `/daily` | GET | 显示过去 n 天（n 默认为 7）每天的盈亏。<br/>*参数：*<br/>- `<n>` (`int`) |
| `/weekly` | GET | 显示过去 n 天（n 默认为 4）每周的盈亏。<br/>*参数：*<br/>- `<n>` (`int`) |
| `/monthly` | GET | 显示过去 n 天（n 默认为 3）每月的盈亏。<br/>*参数：*<br/>- `<n>` (`int`) |
| `/stats` | GET | 显示盈亏原因摘要以及平均持有时间。 |
| `/whitelist` | GET | 显示当前白名单。 |
| `/blacklist` | GET | 显示当前黑名单。 |
| `/blacklist` | POST | 将指定交易对添加到黑名单。<br/>*参数：*<br/>- `pair` (`str`) |
| `/blacklist` | DELETE | 从黑名单中删除指定的交易对列表。<br/>*参数：*<br/>- `[pair,pair]` (`list[str]`)  |
| `/edge` | GET | 如果启用，显示 Edge 验证的交易对。 |
| `/pair_candles` | GET | 在机器人运行时返回交易对/时间框架组合的数据框。**Alpha** |
| `/pair_candles` | POST | 在机器人运行时返回交易对/时间框架组合的数据框，通过提供的列列表过滤返回。**Alpha**<br/>*参数：*<br/>- `<column_list>` (`list[str]`) |
| `/pair_history` | GET | 返回给定时间范围的分析数据框，由给定策略分析。**Alpha** |
| `/pair_history` | POST | 返回给定时间范围的分析数据框，由给定策略分析，通过提供的列列表过滤返回。**Alpha**<br/>*参数：*<br/>- `<column_list>` (`list[str]`) |
| `/plot_config` | GET | 从策略获取绘图配置（如果未配置则返回空）。**Alpha** |
| `/strategies` | GET | 列出策略目录中的策略。**Alpha** |
| `/strategy/<strategy>` | GET | 通过策略类名获取特定策略内容。**Alpha**<br/>*参数：*<br/>- `<strategy>` (`str`) |
| `/available_pairs` | GET | 列出可用的回测数据。**Alpha** |
| `/version` | GET | 显示版本。 |
| `/sysinfo` | GET | 显示有关系统负载的信息。 |
| `/health` | GET | 显示机器人健康状态（最后机器人循环）。 |

:::{warning} Alpha 状态
上面标记为 *Alpha 状态* 的端点可能随时更改，恕不另行通知。
:::

### 消息 WebSocket

API 服务器包含一个 websocket 端点，用于订阅来自 freqtrade 机器人的 RPC 消息。
这可用于消费来自机器人的实时数据，例如入场/出场成交消息、白名单更改、交易对的填充指标等。

这也用于在 Freqtrade 中设置[生产者/消费者模式](producer-consumer.md)。

假设您的 rest API 设置为 `127.0.0.1` 端口 `8080`，端点可在 `http://localhost:8080/api/v1/message/ws` 访问。

要访问 websocket 端点，需要在端点 URL 中将 `ws_token` 作为查询参数。

要生成安全的 `ws_token`，您可以运行以下代码：

```python
>>> import secrets
>>> secrets.token_urlsafe(25)
'hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q'
```

然后，您将在 `api_server` 配置中的 `ws_token` 下添加该令牌。如下所示：

```json
"api_server": {
    "enabled": true,
    "listen_ip_address": "127.0.0.1",
    "listen_port": 8080,
    "verbosity": "error",
    "enable_openapi": false,
    "jwt_secret_key": "somethingrandom",
    "CORS_origins": [],
    "username": "Freqtrader",
    "password": "SuperSecret1!",
    "ws_token": "hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q" // <-----
},
```

您现在可以连接到端点 `http://localhost:8080/api/v1/message/ws?token=hZ-y58LXyX_HZ8O1cJzVyN6ePWrLpNQv4Q`。

:::{danger} 重用示例令牌
请不要使用上面的示例令牌。为确保安全，请生成一个全新的令牌。
:::

#### 使用 WebSocket

连接到 WebSocket 后，机器人将向订阅它们的任何人广播 RPC 消息。要订阅消息列表，您必须通过 WebSocket 发送 JSON 请求，如下所示。`data` 键必须是消息类型字符串列表。

```json
{
  "type": "subscribe",
  "data": ["whitelist", "analyzed_df"] // 消息类型字符串列表
}
```

有关消息类型的列表，请参阅 `freqtrade/enums/rpcmessagetype.py` 中的 RPCMessageType 枚举

现在，只要连接处于活动状态，每当在机器人中发送这些类型的 RPC 消息时，您都会通过 WebSocket 收到它们。它们通常采用与请求相同的形式：

```json
{
  "type": "analyzed_df",
  "data": {
      "key": ["NEO/BTC", "5m", "spot"],
      "df": {}, // 数据框
      "la": "2022-09-08 22:14:41.457786+00:00"
  }
}
```

#### 反向代理设置

使用 [Nginx](https://nginx.org/en/docs/) 时，WebSocket 需要以下配置（注意此配置不完整，缺少一些信息，不能直接使用）：

请确保将 `<freqtrade_listen_ip>`（以及后续端口）替换为与您的配置/设置匹配的 IP 和端口。

```text
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    #...

    server {
        #...

        location / {
            proxy_http_version 1.1;
            proxy_pass http://<freqtrade_listen_ip>:8080;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
        }
    }
}
```

要正确配置您的反向代理（安全地），请查阅其文档以了解如何代理 websocket。

- **Traefik**：Traefik 默认支持 websocket，请参阅[文档](https://doc.traefik.io/traefik/)
- **Caddy**：Caddy v2 默认支持 websocket，请参阅[文档](https://caddyserver.com/docs/v2-upgrade#proxy)

:::{tip} SSL 证书
您可以使用 certbot 等工具设置 ssl 证书，通过使用上述任何反向代理通过加密连接访问机器人的 UI。

虽然这将保护传输中的数据，但我们不建议在您的专用网络（VPN、SSH 隧道）之外运行 freqtrade API。
:::

### OpenAPI 界面

要启用内置的 openAPI 界面（Swagger UI），请在 api_server 配置中指定 `"enable_openapi": true`。
这将在 `/docs` 端点启用 Swagger UI。默认情况下，它在 http://localhost:8080/docs 运行 - 但这取决于您的设置。

### 使用 JWT 令牌的高级 API 使用

:::{note}
以下应该在应用程序（Freqtrade REST API 客户端，通过 API 获取信息）中完成，不打算定期使用。
:::

Freqtrade 的 REST API 还提供 JWT（JSON Web Tokens）。
您可以使用以下命令登录，然后使用生成的 access_token。

```bash
> curl -X POST --user Freqtrader http://localhost:8080/api/v1/token/login
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiZWQ1ZWI3YjAtYjMwMy00YzAyLTg2N2MtNWViMjIxNWQ2YTMxIiwiZXhwIjoxNTkxNzExNjgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJ0eXBlIjoicmVmcmVzaCJ9.d1AT_jYICyTAjD0fiQAr52rkRqtxCjUGEMwlNuuzgNQ"}

> access_token="eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk2ODEsIm5iZiI6MTU4OTExOTY4MSwianRpIjoiMmEwYmY0NWUtMjhmOS00YTUzLTlmNzItMmM5ZWVlYThkNzc2IiwiZXhwIjoxNTg5MTIwNTgxLCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.qt6MAXYIa-l556OM7arBvYJ0SDI9J8bIk3_glDujF5g"
# 使用 access_token 进行身份验证
> curl -X GET --header "Authorization: Bearer ${access_token}" http://localhost:8080/api/v1/count

```

由于访问令牌有短超时（15 分钟）- 应该定期使用 `token/refresh` 请求获取新的访问令牌：

```bash
> curl -X POST --header "Authorization: Bearer ${refresh_token}"http://localhost:8080/api/v1/token/refresh
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1ODkxMTk5NzQsIm5iZiI6MTU4OTExOTk3NCwianRpIjoiMDBjNTlhMWUtMjBmYS00ZTk0LTliZjAtNWQwNTg2MTdiZDIyIiwiZXhwIjoxNTg5MTIwODc0LCJpZGVudGl0eSI6eyJ1IjoiRnJlcXRyYWRlciJ9LCJmcmVzaCI6ZmFsc2UsInR5cGUiOiJhY2Nlc3MifQ.1seHlII3WprjjclY6DpRhen0rqdF4j6jbvxIhUFaSbs"}
```

```{include} includes/cors.md
```
