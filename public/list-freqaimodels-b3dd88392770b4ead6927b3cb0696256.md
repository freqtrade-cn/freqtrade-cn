```
用法: freqtrade list-freqaimodels [-h] [-v] [--no-color] [--logfile FILE]
                                   [-V] [-c PATH] [-d PATH] [--userdir PATH]
                                   [--freqaimodel-path PATH] [-1]

选项:
  -h, --help            显示帮助信息并退出
  --freqaimodel-path PATH
                        为 freqaimodels 指定额外的查找路径。
  -1, --one-column      以单列格式输出。

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
