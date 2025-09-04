```
用法: freqtrade hyperopt-show [-h] [-v] [--no-color] [--logfile FILE] [-V]
                               [-c PATH] [-d PATH] [--userdir PATH] [--best]
                               [--profitable] [-n INT] [--print-json]
                               [--hyperopt-filename FILENAME] [--no-header]
                               [--disable-param-export]
                               [--breakdown {day,week,month,year} [{day,week,month,year} ...]]

选项:
  -h, --help            显示帮助信息并退出
  --best                仅选择最佳 epoch。
  --profitable          仅选择有利润的 epoch。
  -n INT, --index INT   指定要打印详细信息的 epoch 索引。
  --print-json          以 JSON 格式输出结果。
  --hyperopt-filename FILENAME
                        超参数优化结果文件名。例如：`--hyperopt-filename=hyperopt_results_2020-09-27_16-20-48.pickle`
  --no-header           不打印 epoch 详细信息标题。
  --disable-param-export
                        禁用自动导出超参数。
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        按 [日、周、月、年] 显示回测明细。

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
