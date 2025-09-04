---
title: 数据下载
description: 了解如何使用 Freqtrade 下载历史交易数据，包括 K 线数据、回测数据和超参优化所需数据的下载方法和命令参数说明
---

# 数据下载

## 获取回测和超参优化所需数据

要下载回测和超参优化所需的 K 线（OHLCV）数据，请使用 `freqtrade download-data` 命令。

如果未指定额外参数，freqtrade 会默认下载最近 30 天的 "1m" 和 "5m" 周期数据。
交易所和交易对将从 `config.json` 读取（如用 `-c/--config` 指定）。
未提供配置时，`--exchange` 参数为必填。

你可以用相对时间范围（如 `--days 20`）或绝对起始点（如 `--timerange 20200101-`）。增量下载建议用相对方式。

:::{tip} 提示：更新已有数据
如果你已在数据目录有回测数据，想要补齐到今天，freqtrade 会自动计算缺失区间，仅下载缺失部分，无需指定 `--days` 或 `--timerange`。原有数据会保留，仅补全缺失部分。

如果你新增了没有数据的新交易对，建议用 `--new-pairs-days xx`，新对会下载指定天数，旧对只补缺失部分。
:::

### 用法

```{include} commands/download-data.md
```

:::{tip} 下载某个计价货币下所有交易对
通常你会想下载某个计价货币下的所有交易对。可用如下简写：

`freqtrade download-data --exchange binance --pairs ".*/USDT" <...>`。pairs 字符串会自动扩展为交易所所有活跃对。

如需下载已下架对的数据，加 `--include-inactive-pairs`。
:::

:::{warning} 启动期
`download-data` 与策略无关。建议一次性下载大量数据，后续逐步补充。

因此，`download-data` 不关心策略定义的 "startup-period"。如需从特定时间点开始回测，请自行多下载一些天数（注意 startup period）。
:::

### 开始下载

最简单的命令（假设有 `config.json`）：

```bash
freqtrade download-data --exchange binance
```

这会为配置文件中定义的所有交易对下载历史K线（OHLCV）数据。

也可直接指定交易对：

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT
```

或用正则（如下载所有 USDT 对）：

```bash
freqtrade download-data --exchange binance --pairs ".*/USDT"
```

### 其他说明

* 如需用非默认目录，添加 `--datadir user_data/data/some_directory`。
* 如需更换下载数据的交易所，用 `--exchange <exchange>` 或指定不同配置文件。
* 如需用其他目录下的 pairs.json，用 `--pairs-file some_other_dir/pairs.json`。
* 只下载 10 天历史K线，用 `--days 10`（默认 30 天）。
* 从固定起点下载历史K线，用 `--timerange 20200101-`，即从 2020-01-01 起下载全部数据。
* 如已有部分数据，起点会被忽略，仅补齐缺失部分。
* 用 `--timeframes` 指定下载哪些周期的K线，默认 `--timeframes 1m 5m`。
* 用 `-c/--config` 可用配置文件中定义的交易所、周期和交易对。此时用 config 中的白名单作为下载对列表，无需 pairs.json。可与大多数其他参数组合。

:::{caution} 权限拒绝错误
如果你的 `user_data` 目录是 docker 创建的，可能会遇到如下错误：

```
cp: cannot create regular file 'user_data/data/binance/pairs.json': Permission denied
```

可用如下命令修复权限：

```
sudo chown -R $UID:$GID user_data
```
:::

### 下载当前区间前的更多数据

假设你已用 `--timerange 20220101-` 下载了 2022 年全部数据，现在想补充更早的数据。
可用 `--prepend` 配合 `--timerange`（指定结束日期）实现：

```bash
freqtrade download-data --exchange binance --pairs ETH/USDT XRP/USDT BTC/USDT --prepend --timerange 20210101-20220101
```

:::{caution} 
此模式下，如已有数据，freqtrade 会自动用现有数据的起点作为结束日期，忽略你指定的 end-date。
:::

### 数据格式

Freqtrade 目前支持以下数据格式：

* `feather` - 基于 Apache Arrow 的数据格式
* `json` - 纯文本 json 文件
* `jsongz` - gzip 压缩的 json 文件
* `parquet` - 列式存储（仅 OHLCV）

默认情况下，OHLCV 和成交数据都用 `feather` 格式存储。

可通过命令行参数 `--data-format-ohlcv` 和 `--data-format-trades` 更改。
如需永久生效，建议在配置文件中添加：

```jsonc
    // ...
    "dataformat_ohlcv": "feather",
    "dataformat_trades": "feather",
    // ...
```

如下载时更改了默认格式，配置文件中的 `dataformat_ohlcv` 和 `dataformat_trades` 也需同步修改。

:::{caution} 
可用 [convert-data](#sub-command-convert-data) 和 [convert-trade-data](#sub-command-convert-trade-data) 命令在不同格式间转换。
:::

#### 数据格式对比

以下对比基于如下数据，使用 linux `time` 命令：

```
Found 6 pair / timeframe combinations.
+----------+-------------+--------+---------------------+---------------------+
|     Pair |   Timeframe |   Type |                From |                  To |
|----------+-------------+--------+---------------------+---------------------|
| BTC/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:25:00 |
| ETH/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:26:00 |
| BTC/USDT |          1m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:30:00 |
| XRP/USDT |          5m |   spot | 2018-05-04 08:10:00 | 2022-09-13 19:15:00 |
| XRP/USDT |          1m |   spot | 2018-05-04 08:11:00 | 2022-09-13 19:22:00 |
| ETH/USDT |          5m |   spot | 2017-08-17 04:00:00 | 2022-09-13 19:20:00 |
+----------+-------------+--------+---------------------+---------------------+
```

用如下命令测试读取速度：

```bash
time freqtrade list-data --show-timerange --data-format-ohlcv <dataformat>
```

|  格式 | 大小 | 读取时间 |
|------------|-------------|-------------|
| `feather` | 72Mb | 3.5s |
| `json` | 149Mb | 25.6s |
| `jsongz` | 39Mb | 27s |
| `parquet` | 83Mb | 3.8s |

大小为上述 BTC/USDT 1m spot 区间。

推荐用默认 feather 或 parquet 格式，兼顾性能和体积。

### 交易对文件

除了 `config.json` 的白名单，也可用 `pairs.json` 文件。
如用 Binance：

* 创建目录 `user_data/data/binance`，将 `pairs.json` 放入该目录
* 编辑 `pairs.json`，写入你关注的交易对

```bash
mkdir -p user_data/data/binance
touch user_data/data/binance/pairs.json
```

`pairs.json` 格式为简单 `json` 列表。
可混用不同计价货币，仅用于下载。

```json
[
    "ETH/BTC",
    "ETH/USDT",
    "BTC/USDT",
    "XRP/ETH"
]
```

:::{caution} 
仅在未加载配置文件时（自动或用 `--config`）才用 `pairs.json`。

可用 `--pairs-file pairs.json` 强制使用，但推荐用配置文件中的白名单（`exchange.pair_whitelist` 或 `pairs`）。
:::

## 子命令 convert data

```{include} commands/convert-data.md
```
### 数据格式转换示例

如下命令会将 `~/.freqtrade/data/binance` 下所有K线（OHLCV）数据从 `json` 转为 `jsongz`，并删除原 `json` 文件（`--erase` 参数）：

```bash
freqtrade convert-data --format-from json --format-to jsongz --datadir ~/.freqtrade/data/binance -t 5m 15m --erase
```

## 子命令 convert trade data

```{include} commands/convert-trade-data.md
```
### 成交数据转换示例

如下命令会将 `~/.freqtrade/data/kraken` 下所有成交数据从 `jsongz` 转为 `json`，并删除原 `jsongz` 文件（`--erase` 参数）：

```bash
freqtrade convert-trade-data --format-from jsongz --format-to json --datadir ~/.freqtrade/data/kraken --erase
```

## 子命令 trades to ohlcv

如需用 `--dl-trades`（仅 kraken）下载数据，最后一步需将成交数据转为K线数据。
本命令可让你为更多周期重复此步骤，无需重新下载。

```{include} commands/trades-to-ohlcv.md
```

### trade-to-ohlcv 转换示例

```bash
freqtrade trades-to-ohlcv --exchange kraken -t 5m 1h 1d --pairs BTC/EUR ETH/EUR
```

## 子命令 list-data

可用 `list-data` 子命令查看已下载数据。

```{include} commands/list-data.md
```

### list-data 示例

```bash
> freqtrade list-data --userdir ~/.freqtrade/user_data/

              Found 33 pair / timeframe combinations.
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┓
┃          Pair ┃                                 Timeframe ┃ Type ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━┩
│       ADA/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ADA/ETH │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│       ETH/BTC │     5m, 15m, 30m, 1h, 2h, 4h, 6h, 12h, 1d │ spot │
│      ETH/USDT │                  5m, 15m, 30m, 1h, 2h, 4h │ spot │
└───────────────┴───────────────────────────────────────────┴──────┘

```

显示所有成交数据及时间区间：

```bash
> freqtrade list-data --show --trades
                     Found trades data for 1 pair.                     
┏━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━┓
┃    Pair ┃ Type ┃                From ┃                  To ┃ Trades ┃
┡━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━┩
│ XRP/ETH │ spot │ 2019-10-11 00:00:11 │ 2019-10-13 11:19:28 │  12477 │
└─────────┴──────┴─────────────────────┴─────────────────────┴────────┘

```

## 成交（tick）数据

默认情况下，`download-data` 子命令下载 K 线（OHLCV）数据。大多数交易所也支持通过 API 下载历史成交数据。
如需多个周期，成交数据只需下载一次，后续可本地重采样。

因数据量大，默认用 feather 格式存储，文件名为 `<pair>-trades.feather`（如 ETH_BTC-trades.feather）。支持增量模式，如每周用 `--days 8` 下载一次即可。

如需此模式，只需加 `--dl-trades`。此时会切换为下载成交数据。
如加 `--convert`，则会自动重采样并覆盖已有 OHLCV 数据。

:::{warning} 请勿随意使用
除非你用 kraken，否则不建议用此方式（kraken 不提供历史 OHLCV 数据）。
其他交易所直接下载 OHLCV 更快。
:::

:::{warning} kraken 用户
kraken 用户请先阅读[此处](exchanges.md#historic-kraken-data)。
:::

示例：

```bash
freqtrade download-data --exchange kraken --pairs XRP/EUR ETH/EUR --days 20 --dl-trades
```

:::{caution} 
虽然此方法用异步调用，但因需依次请求，速度较慢。
:::

## 下一步

恭喜，你已下载好数据，可以[开始回测](backtesting.md)你的策略了。
