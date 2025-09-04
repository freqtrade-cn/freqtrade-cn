```
用法: freqtrade backtesting-analysis [-h] [-v] [--no-color] [--logfile FILE]
                                      [-V] [-c PATH] [-d PATH]
                                      [--userdir PATH]
                                      [--backtest-filename PATH]
                                      [--backtest-directory PATH]
                                      [--analysis-groups {0,1,2,3,4,5} [{0,1,2,3,4,5} ...]]
                                      [--enter-reason-list ENTER_REASON_LIST [ENTER_REASON_LIST ...]]
                                      [--exit-reason-list EXIT_REASON_LIST [EXIT_REASON_LIST ...]]
                                      [--indicator-list INDICATOR_LIST [INDICATOR_LIST ...]]
                                      [--entry-only] [--exit-only]
                                      [--timerange TIMERANGE]
                                      [--rejected-signals] [--analysis-to-csv]
                                      [--analysis-csv-path ANALYSIS_CSV_PATH]

选项:
  -h, --help            显示帮助信息并退出
  --backtest-filename PATH, --export-filename PATH
                        使用此文件名作为回测结果。示例：
                        `--backtest-filename=backtest_results_2020-09-27_16-20-48.json`。
                        假设以 `user_data/backtest_results/` 或 `--export-directory` 作为基础目录。
  --analysis-groups {0,1,2,3,4,5} [{0,1,2,3,4,5} ...]
                        分组输出 - 0: 按入场标签的简单盈亏，1: 按入场标签，
                        2: 按入场标签和出场标签，3: 按交易对和入场标签，
                        4: 按交易对、入场标签和出场标签（这可能会变得相当大），
                        5: 按出场标签
  --enter-reason-list ENTER_REASON_LIST [ENTER_REASON_LIST ...]
                        要分析的入场信号列表（用空格分隔）。
                        默认：全部。例如：'entry_tag_a entry_tag_b'
  --exit-reason-list EXIT_REASON_LIST [EXIT_REASON_LIST ...]
                        要分析的出场信号列表（用空格分隔）。
                        默认：全部。例如：'exit_tag_a roi stop_loss trailing_stop_loss'
  --indicator-list INDICATOR_LIST [INDICATOR_LIST ...]
                        要分析的指标列表（用空格分隔）。例如：
                        'close rsi bb_lowerband profit_abs'
  --entry-only          仅分析入场信号。
  --exit-only           仅分析出场信号。
  --timerange TIMERANGE
                        指定要使用的数据时间范围。
  --rejected-signals    分析被拒绝的信号。
  --analysis-to-csv     将选定的分析表保存为单独的CSV文件。
  --analysis-csv-path ANALYSIS_CSV_PATH
                        如果启用了 --analysis-to-csv，指定保存分析CSV文件的路径。
                        默认：user_data/backtesting_results/

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
