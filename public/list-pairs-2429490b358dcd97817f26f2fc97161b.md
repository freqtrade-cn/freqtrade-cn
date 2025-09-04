```
用法: freqtrade list-pairs [-h] [-v] [--no-color] [--logfile FILE] [-V]
                            [-c PATH] [-d PATH] [--userdir PATH]
                            [--exchange EXCHANGE] [--print-list]
                            [--print-json] [-1] [--print-csv]
                            [--base BASE_CURRENCY [BASE_CURRENCY ...]]
                            [--quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]]
                            [-a] [--trading-mode {spot,margin,futures}]

选项:
  -h, --help            显示帮助信息并退出
  --exchange EXCHANGE   交易所名称。仅在未提供配置时有效。
  --print-list          以列表形式打印交易对。默认以表格形式输出。
  --print-json          以 JSON 格式打印交易对列表。
  -1, --one-column      以单列格式输出。
  --print-csv           以 CSV 格式打印交易所交易对数据。
  --base BASE_CURRENCY [BASE_CURRENCY ...]
                        指定基础货币。空格分隔的列表。
  --quote QUOTE_CURRENCY [QUOTE_CURRENCY ...]
                        指定计价货币。空格分隔的列表。
  -a, --all             打印所有交易对。默认仅显示活跃的。
  --trading-mode {spot,margin,futures}, --tradingmode {spot,margin,futures}
                        选择交易模式。

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
