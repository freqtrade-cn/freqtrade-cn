---
title: 高级安装后配置
subject: FreqTrade 高级安装指南
subtitle: 安装后的高级配置选项
short_title: 高级安装
description: 本文档介绍了 FreqTrade 安装后可以执行的高级任务和配置选项,包括多实例运行、日志配置等高级特性。
---

# 高级安装后任务

本页介绍了一些在机器人安装后可以执行的高级任务和配置选项，在某些环境下可能会用到。

如果你不了解这里提到的内容，通常你并不需要这些操作。

## 运行多个 Freqtrade 实例

本节将介绍如何在同一台机器上同时运行多个机器人。

### 需要注意的事项

* 使用不同的数据库文件。
* 使用不同的 Telegram 机器人（需要多个不同的配置文件，仅在启用 Telegram 时适用）。
* 使用不同的端口（仅在启用 Freqtrade REST API Web 服务器时适用）。

### 不同的数据库文件

为了跟踪你的交易、利润等，freqtrade 使用 SQLite 数据库存储各种信息，如你过去执行的交易和当前持有的仓位。这不仅可以让你跟踪利润，更重要的是，即使机器人进程重启或意外终止，也能保持活动状态的追踪。

默认情况下，freqtrade 会为 dry-run 和实盘机器人分别使用不同的数据库文件（假设配置和命令行参数中都未指定 database-url）。实盘模式下默认数据库为 `tradesv3.sqlite`，dry-run 模式下为 `tradesv3.dryrun.sqlite`。

用于指定这些文件路径的可选参数是 `--db-url`，它需要一个有效的 SQLAlchemy url。
因此，当你仅用 config 和 strategy 参数以 dry-run 模式启动机器人时，以下两条命令效果相同：

```bash
freqtrade trade -c MyConfig.json -s MyStrategy
# 等价于
freqtrade trade -c MyConfig.json -s MyStrategy --db-url sqlite:///tradesv3.dryrun.sqlite
```

这意味着，如果你在两个不同的终端运行 trade 命令（例如分别测试 USDT 和 BTC 交易），你需要为它们指定不同的数据库。

如果你指定的数据库文件不存在，freqtrade 会自动创建一个同名数据库。因此，你可以用如下命令（在两个终端分别执行）测试自定义策略的 BTC 和 USDT 本位：

```bash
# 终端 1：
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.dryrun.sqlite
# 终端 2：
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.dryrun.sqlite
```

同理，如果你想在实盘模式下做同样的事，也需要为每个实例创建至少一个新数据库，并指定“实盘”数据库路径，例如：

```bash
# 终端 1：
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.live.sqlite
# 终端 2：
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.live.sqlite
```

关于 sqlite 数据库的更多用法（如手动录入或删除交易），请参见 [SQL 速查表](sql_cheatsheet.md)。

### 使用 docker 运行多个实例

要用 docker 运行多个 freqtrade 实例，你需要编辑 docker-compose.yml 文件，将所有实例作为独立服务添加。你可以将配置拆分为多个文件，便于模块化管理，这样如果需要修改所有机器人的通用配置，只需改一个文件即可。

``` yml
---
version: '3'
services:
  freqtrade1:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # 使用带绘图功能的镜像
    # image: freqtradeorg/freqtrade:develop_plot
    # 仅在需要额外依赖时才需 build
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade1
    volumes:
      - "./user_data:/freqtrade/user_data"
    # 在本地 8080 端口暴露 api
    # 启用前请阅读 https://www.freqtrade.io/zh_CN/latest/rest-api/ 文档
     ports:
     - "127.0.0.1:8080:8080"
    # 运行 `docker compose up` 时的默认命令
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade1.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade1.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade1.json
      --strategy SampleStrategy
  
  freqtrade2:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # 使用带绘图功能的镜像
    # image: freqtradeorg/freqtrade:develop_plot
    # 仅在需要额外依赖时才需 build
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade2
    volumes:
      - "./user_data:/freqtrade/user_data"
    # 在本地 8081 端口暴露 api
    # 启用前请阅读 https://www.freqtrade.io/zh_CN/latest/rest-api/ 文档
    ports:
      - "127.0.0.1:8081:8080"
    # 运行 `docker compose up` 时的默认命令
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade2.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade2.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade2.json
      --strategy SampleStrategy

```

你可以使用任意命名方式，freqtrade1 和 freqtrade2 只是示例。注意，每个实例都需要不同的数据库文件、端口映射和 telegram 配置。

## 使用不同的数据库系统

Freqtrade 使用 SQLAlchemy，支持多种数据库系统。因此，理论上支持多种数据库。
Freqtrade 不会自动安装任何额外的数据库驱动。请参考 [SQLAlchemy 文档](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls) 获取各数据库系统的安装说明。

已知可用的数据库系统有：

* sqlite（默认）
* PostgreSQL
* MariaDB

:::{warning}
使用下列数据库系统时，需自行负责管理。Freqtrade 团队不提供安装、维护（或备份）等支持。
:::

### PostgreSQL

安装：
`pip install psycopg2-binary`

用法：
`... --db-url postgresql+psycopg2://<用户名>:<密码>@localhost:5432/<数据库>`

Freqtrade 启动时会自动创建所需表。

如运行多个 Freqtrade 实例，需为每个实例单独建库，或为每个连接使用不同用户/Schema。

### MariaDB / MySQL

Freqtrade 通过 SQLAlchemy 支持 MariaDB。

安装：
`pip install pymysql`

用法：
`... --db-url mysql+pymysql://<用户名>:<密码>@localhost:3306/<数据库>`

(configure-the-bot-running-as-a-systemd-service)=

## 配置机器人为 systemd 服务

将 `freqtrade.service` 文件复制到你的 systemd 用户目录（通常为 `~/.config/systemd/user`），并根据实际情况修改 `WorkingDirectory` 和 `ExecStart`。

:::{note}
某些系统（如 Raspbian）不会从用户目录加载服务单元文件。这种情况下，请将 `freqtrade.service` 复制到 `/etc/systemd/user/`（需超级用户权限）。
:::

之后可用如下命令启动守护进程：

```bash
systemctl --user start freqtrade
```

如需在用户注销后仍能运行，需为 freqtrade 用户启用 linger：

```bash
sudo loginctl enable-linger "$USER"
```

如以服务方式运行机器人，可用 systemd 服务管理器作为软件 watchdog 监控 freqtrade 状态，并在故障时自动重启。如果配置文件中设置了 `internals.sd_notify` 为 true，或命令行使用了 `--sd-notify`，机器人会通过 sd_notify 协议向 systemd 发送心跳，并在状态变化时告知 systemd 当前状态（运行、暂停或停止）。

`freqtrade.service.watchdog` 文件提供了使用 systemd 作为 watchdog 的服务单元配置示例。

:::{note}
如果机器人运行在 Docker 容器中，sd_notify 与 systemd 服务管理器之间的通信将无法工作。
:::

(advanced-logging)=

## 高级日志配置

Freqtrade 使用 Python 的默认 logging 模块。
Python 支持非常丰富的[日志配置](https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig)，远超本页所能涵盖。

如果 freqtrade 配置文件未提供 `log_config`，则默认使用彩色终端输出格式。使用 `--logfile logfile.log` 会启用 RotatingFileHandler。

如不满意日志格式或 RotatingFileHandler 的默认设置，可通过在配置文件中添加 `log_config` 自定义日志。

默认配置大致如下，文件处理器已提供但未启用（`filename` 行被注释）。取消注释并填写有效路径即可启用。

```json hl_lines="5-7 13-16 27"
{
  "log_config": {
      "version": 1,
      "formatters": {
          "basic": {
              "format": "%(message)s"
          },
          "standard": {
              "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
          }
      },
      "handlers": {
          "console": {
              "class": "freqtrade.loggers.ft_rich_handler.FtRichHandler",
              "formatter": "basic"
          },
          "file": {
              "class": "logging.handlers.RotatingFileHandler",
              "formatter": "standard",
              // "filename": "someRandomLogFile.log",
              "maxBytes": 10485760,
              "backupCount": 10
          }
      },
      "root": {
          "handlers": [
              "console",
              // "file"
          ],
          "level": "INFO",
      }
  }
}
```

:::{note} 高亮行
上述代码块中高亮的行定义了 Rich handler，属于同一组。

格式化器 "standard" 和 "file" 属于 FileHandler。
:::

每个 handler 必须使用已定义的 formatter（按名称），其 class 必须可用且为有效日志类。
要实际启用 handler，需在 "root" 段的 "handlers" 列表中添加。
如省略该段，freqtrade 不会输出日志（至少不会用未配置的 handler 输出）。

:::{tip} 显式日志配置
建议将日志配置从主配置文件中提取出来，通过[多配置文件](configuration.md#multiple-configuration-files)功能传递给机器人，避免重复代码。
:::

---

在许多 Linux 系统上，机器人可配置为将日志发送到 `syslog` 或 `journald` 系统服务。Windows 下也可远程记录到 syslog 服务器。`--logfile` 命令行参数可用于此目的。

### 日志输出到 syslog

要将 Freqtrade 日志发送到本地或远程 `syslog`，请用 "log_config" 配置日志。

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "syslog_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // 其他 handler
      "syslog": {
         "class": "logging.handlers.SysLogHandler",
          "formatter": "syslog_fmt",
          // 可用上面任一 address
          "address": "/dev/log"
      }
    },
    "root": {
      "handlers": [
        // 其他 handler
        "syslog",
        
      ]
    }

  }
}
```

[更多日志 handler 配置](#advanced-logging) 可用于同时输出到控制台等。

#### Syslog 用法

日志以 `user` facility 发送到 syslog。可用如下命令查看：

* `tail -f /var/log/user`
* 或安装图形日志查看器（如 Ubuntu 的 'Log File Viewer'）。

许多系统上，syslog（rsyslog）和 journald 互通，因此日志可用 journalctl 或 syslog 工具查看。
你可根据习惯自由组合。

如需将机器人日志重定向到专用日志文件，在 `/etc/rsyslog.d/50-default.conf` 末尾添加：

```
if $programname startswith "freqtrade" then -/var/log/freqtrade.log
```

如需开启 rsyslog 的重复消息抑制（如多条心跳合并为一条），在 `/etc/rsyslog.conf` 添加：

```
# 过滤重复消息
$RepeatedMsgReduction on
```

#### Syslog 地址

syslog 地址可为 Unix 域套接字（文件名）或 UDP 套接字（IP:端口）。

常见示例：

* `"address": "/dev/log"` —— 绝大多数系统的本地 syslog。
* `"address": "/var/run/syslog"` —— MacOS 使用。
* `"address": "localhost:514"` —— 本地 syslog 的 UDP。
* `"address": "<ip>:514"` —— 远程 syslog。

:::{tip} 已废弃：命令行配置 syslog
`--logfile syslog:<syslog_address>` —— 通过 `<syslog_address>` 发送日志到 syslog。

地址可为 Unix 域套接字或 UDP（IP:端口）。

常见用法：

* `--logfile syslog:/dev/log` —— 本地 syslog。
* `--logfile syslog` —— 等同于上。
* `--logfile syslog:/var/run/syslog` —— MacOS。
* `--logfile syslog:localhost:514` —— 本地 UDP。
* `--logfile syslog:<ip>:514` —— 远程 syslog。
:::

### 日志输出到 journald

需安装 `cysystemd` python 包（`pip install cysystemd`），Windows 不支持 journald。

要将 Freqtrade 日志发送到 journald，请在配置中添加如下片段：

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "journald_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // 其他 handler
      "journald": {
         "class": "cysystemd.journal.JournaldLogHandler",
          "formatter": "journald_fmt",
      }
    },
    "root": {
      "handlers": [
        // .. 
        "journald",
        
      ]
    }

  }
}
```

[更多日志 handler 配置](#advanced-logging) 可用于同时输出到控制台等。

日志以 `user` facility 发送到 journald。可用如下命令查看：

* `journalctl -f` —— 实时查看所有 journald 日志。
* `journalctl -f -u freqtrade.service` —— 机器人以 systemd 服务运行时查看。

journalctl 还有许多过滤选项，详见其手册。

许多系统上，syslog（rsyslog）和 journald 互通，因此日志可用 `--logfile syslog` 或 `--logfile journald`，并可用 journalctl 或 syslog 工具查看。
你可根据习惯自由组合。

:::{tip} 已废弃：命令行配置 journald
用 `--logfile journald` 命令行参数将日志发送到 journald。
:::

### 日志格式为 JSON

你也可以将默认输出流配置为 JSON 格式。
"fmt_dict" 属性定义了 json 输出的键及 [python logging LogRecord 属性](https://docs.python.org/3/library/logging.html#logrecord-attributes)。

如下配置将默认输出改为 JSON。该 formatter 也可与 RotatingFileHandler 配合使用。建议至少保留一种人类可读格式。

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
       "json": {
          "()": "freqtrade.loggers.json_formatter.JsonFormatter",
          "fmt_dict": {
              "timestamp": "asctime",
              "level": "levelname",
              "logger": "name",
              "message": "message"
          }
      }
    },
    "handlers": {
      // 其他 handler
      "jsonStream": {
          "class": "logging.StreamHandler",
          "formatter": "json"
      }
    },
    "root": {
      "handlers": [
        // .. 
        "jsonStream",
        
      ]
    }

  }
}
```
