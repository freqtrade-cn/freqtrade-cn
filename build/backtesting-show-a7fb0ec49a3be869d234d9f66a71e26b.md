```
用法: freqtrade backtesting-show [-h] [-v] [--no-color] [--logfile FILE] [-V]
                                  [-c PATH] [-d PATH] [--userdir PATH]
                                  [--backtest-filename PATH]
                                  [--backtest-directory PATH]
                                  [--show-pair-list]
                                  [--breakdown {day,week,month,year} [{day,week,month,year} ...]]

选项:
  -h, --help            显示帮助信息并退出
  --backtest-filename PATH, --export-filename PATH
                        使用此文件名作为回测结果。示例：
                        `--backtest-filename=backtest_results_2020-09-27_16-20-48.json`。
                        假设以 `user_data/backtest_results/` 或 `--export-directory` 作为基础目录。
  --backtest-directory PATH, --export-directory PATH
                        用于回测结果的目录。示例：
                        `--export-directory=user_data/backtest_results/`。
  --show-pair-list      按利润排序显示回测交易对列表。
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        显示按[日、周、月、年]的回测明细。

Common arguments:
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
