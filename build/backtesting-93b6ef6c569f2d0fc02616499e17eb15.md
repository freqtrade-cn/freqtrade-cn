```
用法: freqtrade backtesting [-h] [-v] [--no-color] [--logfile FILE] [-V]
                             [-c PATH] [-d PATH] [--userdir PATH] [-s NAME]
                             [--strategy-path PATH]
                             [--recursive-strategy-search]
                             [--freqaimodel NAME] [--freqaimodel-path PATH]
                             [-i TIMEFRAME] [--timerange TIMERANGE]
                             [--data-format-ohlcv {json,jsongz,feather,parquet}]
                             [--max-open-trades INT]
                             [--stake-amount STAKE_AMOUNT] [--fee FLOAT]
                             [-p PAIRS [PAIRS ...]] [--eps]
                             [--enable-protections]
                             [--dry-run-wallet DRY_RUN_WALLET]
                             [--timeframe-detail TIMEFRAME_DETAIL]
                             [--strategy-list STRATEGY_LIST [STRATEGY_LIST ...]]
                             [--export {none,trades,signals}]
                             [--backtest-filename PATH]
                             [--backtest-directory PATH]
                             [--breakdown {day,week,month,year} [{day,week,month,year} ...]]
                             [--cache {none,day,week,month}]
                             [--freqai-backtest-live-models]

选项:
  -h, --help            显示帮助信息并退出
  -i TIMEFRAME, --timeframe TIMEFRAME
                        指定时间框架 (`1m`, `5m`, `30m`, `1h`, `1d`)。
  --timerange TIMERANGE
                        指定要使用的数据时间范围。
  --data-format-ohlcv {json,jsongz,feather,parquet}
                        下载的K线（OHLCV）数据的存储格式。
                        （默认：`feather`）。
  --max-open-trades INT
                        覆盖配置设置中的 `max_open_trades` 值。
  --stake-amount STAKE_AMOUNT
                        覆盖配置设置中的 `stake_amount` 值。
  --fee FLOAT           指定手续费比率。将应用两次（在交易进入和退出时）。
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        限制命令仅用于这些交易对。交易对之间用空格分隔。
  --eps, --enable-position-stacking
                        允许多次购买同一交易对（仓位叠加）。
  --enable-protections, --enableprotections
                        为回测启用保护机制。这将显著降低回测速度，但会包含已配置的保护机制。
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        起始余额，用于回测/超参数优化和模拟运行。
  --timeframe-detail TIMEFRAME_DETAIL
                        为回测指定详细时间框架 (`1m`, `5m`, `30m`, `1h`, `1d`)。
  --strategy-list STRATEGY_LIST [STRATEGY_LIST ...]
                        提供要回测的策略列表（用空格分隔）。请注意，时间框架需要在配置中或通过命令行设置。
                        当与 `--export trades` 一起使用时，策略名称会被注入到文件名中
                        （因此 `backtest-data.json` 会变成 `backtest-data-SampleStrategy.json`）。
  --export {none,trades,signals}
                        导出回测结果（默认：trades）。
  --backtest-filename PATH, --export-filename PATH
                        使用此文件名作为回测结果。示例：
                        `--backtest-
                        filename=backtest_results_2020-09-27_16-20-48.json`。
                        假设以 `user_data/backtest_results/` 或
                        `--export-directory` 作为基础目录。
  --backtest-directory PATH, --export-directory PATH
                        用于回测结果的目录。示例：
                        `--export-directory=user_data/backtest_results/`。
  --breakdown {day,week,month,year} [{day,week,month,year} ...]
                        显示按[日、周、月、年]的回测明细。
  --cache {none,day,week,month}
                        加载不超过指定时间的缓存回测结果（默认：day）。
  --freqai-backtest-live-models
                        使用已准备好的模型运行回测。

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

策略参数:
  -s NAME, --strategy NAME
                        指定机器人要使用的策略类名。
  --strategy-path PATH  指定额外的策略查找路径。
  --recursive-strategy-search
                        在策略文件夹中递归搜索策略。
  --freqaimodel NAME    指定自定义的 freqaimodels。
  --freqaimodel-path PATH
                        为 freqaimodels 指定额外的查找路径。

```
