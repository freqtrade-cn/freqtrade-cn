```
用法: freqtrade download-data [-h] [-v] [--no-color] [--logfile FILE] [-V]
                               [-c PATH] [-d PATH] [--userdir PATH]
                               [-p PAIRS [PAIRS ...]] [--pairs-file FILE]
                               [--days INT] [--new-pairs-days INT]
                               [--include-inactive-pairs]
                               [--timerange TIMERANGE] [--dl-trades]
                               [--convert] [--exchange EXCHANGE]
                               [-t TIMEFRAMES [TIMEFRAMES ...]] [--erase]
                               [--data-format-ohlcv {json,jsongz,feather,parquet}]
                               [--data-format-trades {json,jsongz,feather,parquet}]
                               [--trading-mode {spot,margin,futures}]
                               [--prepend]

选项:
  -h, --help            显示帮助信息并退出
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限制命令仅用于这些交易对。交易对之间用空格分隔。
  --pairs-file FILE     包含交易对列表的文件。优先于 --pairs 或配置文件中的 pairs。
  --days INT            下载指定天数的数据。
  --new-pairs-days INT  为新交易对下载指定天数的数据。
                        默认：`None`。
  --include-inactive-pairs
                        也下载非活跃交易对的数据。
  --timerange TIMERANGE
                        指定要使用的数据时间范围。
  --dl-trades           下载成交单数据而不是 OHLCV 数据。
  --convert             将下载的成交单数据转换为 OHLCV 数据。仅在与 `--dl-trades` 结合使用时适用。
                        对于没有历史 OHLCV 的交易所（如 Kraken）会自动转换。
                        如果未提供此参数，请使用 `trades-to-ohlcv` 命令手动转换。
  --exchange EXCHANGE   交易所名称。仅在未提供配置时有效。
  -t TIMEFRAMES [TIMEFRAMES ...], --timeframes TIMEFRAMES [TIMEFRAMES ...]
                        指定要下载的行情数据。空格分隔的列表。
                        默认：`1m 5m`。
  --erase               清除所选交易所/交易对/时间框架的所有现有数据。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的K线（OHLCV）数据的存储格式。
                        （默认：`feather`）。
  --data-format-trades {json,jsongz,feather,parquet}
                        下载的成交单数据的存储格式。（默认：`feather`）。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式。
  --prepend             允许数据前置。（数据追加被禁用）

通用参数:
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取所有消息）。
  --no-color            禁用超参数优化结果的着色。在将输出重定向到文件时可能有用。
  --logfile FILE, --log-file FILE
                        记录到指定的文件。特殊值包括：
                        'syslog', 'journald'。有关更多详细信息，请参阅文档。
  -V, --version         显示程序版本号并退出
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或 `config.json`，以存在的为准）。
                        可以使用多个 --config 选项。可以设置为 `-` 以从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        交易所历史回测数据的基本目录路径。要查看期货数据，需要额外使用 trading-mode。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录的路径。

```
