```
用法: freqtrade recursive-analysis [-h] [-v] [--no-color] [--logfile FILE]
                                    [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                    [-s NAME] [--strategy-path PATH]
                                    [--recursive-strategy-search]
                                    [--freqaimodel NAME]
                                    [--freqaimodel-path PATH] [-i TIMEFRAME]
                                    [--timerange TIMERANGE]
                                    [--data-format-ohlcv {json,jsongz,feather,parquet}]
                                    [-p PAIRS [PAIRS ...]]
                                    [--startup-candle STARTUP_CANDLE [STARTUP_CANDLE ...]]

选项:
  -h, --help            显示帮助信息并退出。
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间周期（`1m`, `5m`, `30m`, `1h`, `1d`）。
  --timerange TIMERANGE
                        指定要使用的数据时间范围。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的蜡烛图（OHLCV）数据的存储格式。
                        （默认：`feather`）。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限制命令仅处理这些交易对。交易对以空格分隔。
  --startup-candle STARTUP_CANDLE [STARTUP_CANDLE ...]
                        指定要检查的启动蜡烛图（`199`, `499`, `999`, `1999`）。

通用参数:
  -v, --verbose         详细模式（-vv 获取更多信息，-vvv 获取所有消息）。
  --no-color            禁用超参数优化结果的着色。在将输出重定向到文件时可能有用。
  --logfile FILE, --log-file FILE
                        记录到指定的文件。特殊值包括：
                        'syslog', 'journald'。有关更多详细信息，请参阅文档。
  -V, --version         显示程序版本号并退出。
  -c PATH, --config PATH
                        指定配置文件（默认：`userdir/config.json` 或 `config.json`，以存在的为准）。
                        可以使用多个 --config 选项。可以设置为 `-` 以从标准输入读取配置。
  -d PATH, --datadir PATH, --data-dir PATH
                        交易所历史回测数据的基本目录路径。要查看期货数据，需要额外使用 trading-mode。
  --userdir PATH, --user-data-dir PATH
                        用户数据目录的路径。

策略参数:
  -s NAME, --strategy NAME
                        指定机器人使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的 freqaimodels。
  --freqaimodel-path PATH
                        指定 freqaimodels 的额外查找路径。
```
