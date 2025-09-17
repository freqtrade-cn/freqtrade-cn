"""text
Available commands and their arguments:

Command: trade    help="Trade module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  --db-url
  --sd-notify
  --dry-run
  --dry-run-wallet, --starting-balance
  --fee

Command: create-userdir    help="Create user-data directory."
  -h, --help
  --userdir, --user-data-dir
  --reset

Command: new-config    help="Create new config"
  -h, --help
  -c, --config

Command: show-config    help="Show resolved config"
  -h, --help
  --userdir, --user-data-dir
  -c, --config
  --show-sensitive

Command: new-strategy    help="Create new strategy"
  -h, --help
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --template

Command: download-data    help="Download backtesting data."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -p, --pairs
  --pairs-file
  --days
  --new-pairs-days
  --include-inactive-pairs
  --timerange
  --dl-trades
  --convert
  --exchange
  -t, --timeframes
  --erase
  --data-format-ohlcv
  --data-format-trades
  --trading-mode, --tradingmode
  --prepend

Command: convert-data    help="Convert candle (OHLCV) data from one format to another."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -p, --pairs
  --format-from
  --format-to
  --erase
  --exchange
  -t, --timeframes
  --trading-mode, --tradingmode
  --candle-types

Command: convert-trade-data    help="Convert trade data from one format to another."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -p, --pairs
  --format-from
  --format-to
  --erase
  --exchange

Command: trades-to-ohlcv    help="Convert trade data to OHLCV data."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -p, --pairs
  -t, --timeframes
  --exchange
  --data-format-ohlcv
  --data-format-trades
  --trading-mode, --tradingmode

Command: list-data    help="List downloaded data."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --exchange
  --data-format-ohlcv
  --data-format-trades
  --trades
  -p, --pairs
  --trading-mode, --tradingmode
  --show-timerange

Command: backtesting    help="Backtesting module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -i, --timeframe
  --timerange
  --data-format-ohlcv
  --max-open-trades
  --stake-amount
  --fee
  -p, --pairs
  --eps, --enable-position-stacking
  --enable-protections, --enableprotections
  --dry-run-wallet, --starting-balance
  --timeframe-detail
  --strategy-list
  --export
  --export-filename, --backtest-filename
  --breakdown
  --cache
  --freqai-backtest-live-models

Command: backtesting-show    help="Show past Backtest results"
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --export-filename, --backtest-filename
  --show-pair-list
  --breakdown

Command: backtesting-analysis    help="Backtest Analysis module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --export-filename, --backtest-filename
  --analysis-groups
  --enter-reason-list
  --exit-reason-list
  --indicator-list
  --entry-only
  --exit-only
  --timerange
  --rejected-signals
  --analysis-to-csv
  --analysis-csv-path

Command: edge    help="Edge module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -i, --timeframe
  --timerange
  --data-format-ohlcv
  --max-open-trades
  --stake-amount
  --fee
  -p, --pairs
  --stoplosses

Command: hyperopt    help="Hyperopt module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -i, --timeframe
  --timerange
  --data-format-ohlcv
  --max-open-trades
  --stake-amount
  --fee
  -p, --pairs
  --hyperopt
  --hyperopt-path
  --eps, --enable-position-stacking
  --enable-protections, --enableprotections
  --dry-run-wallet, --starting-balance
  --timeframe-detail
  -e, --epochs
  --spaces
  --print-all
  --print-json
  -j, --job-workers
  --random-state
  --min-trades
  --hyperopt-loss, --hyperoptloss
  --disable-param-export
  --ignore-missing-spaces, --ignore-unparameterized-spaces
  --analyze-per-epoch
  --early-stop

Command: hyperopt-list    help="List Hyperopt results"
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --best
  --profitable
  --min-trades
  --max-trades
  --min-avg-time
  --max-avg-time
  --min-avg-profit
  --max-avg-profit
  --min-total-profit
  --max-total-profit
  --min-objective
  --max-objective
  --print-json
  --no-details
  --hyperopt-filename
  --export-csv

Command: hyperopt-show    help="Show details of Hyperopt results"
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --best
  --profitable
  -n, --index
  --print-json
  --hyperopt-filename
  --no-header
  --disable-param-export
  --breakdown

Command: list-exchanges    help="Print available exchanges."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -1, --one-column
  -a, --all

Command: list-markets    help="Print markets on exchange."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --exchange
  --print-list
  --print-json
  -1, --one-column
  --print-csv
  --base
  --quote
  -a, --all
  --trading-mode, --tradingmode

Command: list-pairs    help="Print pairs on exchange."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --exchange
  --print-list
  --print-json
  -1, --one-column
  --print-csv
  --base
  --quote
  -a, --all
  --trading-mode, --tradingmode

Command: list-strategies    help="Print available strategies."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --strategy-path
  -1, --one-column
  --recursive-strategy-search

Command: list-hyperoptloss    help="Print available hyperopt loss functions."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --hyperopt-path
  -1, --one-column

Command: list-freqaimodels    help="Print available freqAI models."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --freqaimodel-path
  -1, --one-column

Command: list-timeframes    help="Print available timeframes for the exchange."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --exchange
  -1, --one-column

Command: show-trades    help="Show trades."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --db-url
  --trade-ids
  --print-json

Command: test-pairlist    help="Test your pairlist configuration."
  -h, --help
  --userdir, --user-data-dir
  -v, --verbose
  -c, --config
  --quote
  -1, --one-column
  --print-json
  --exchange

Command: convert-db    help="Migrate database to different system"
  -h, --help
  --db-url
  --db-url-from

Command: install-ui    help="Install FreqUI"
  -h, --help
  --erase
  --prerelease
  --ui-version

Command: plot-dataframe    help="Plot candles with indicators."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -p, --pairs
  --indicators1
  --indicators2
  --plot-limit
  --db-url
  --trade-source
  --export
  --export-filename, --backtest-filename
  --timerange
  -i, --timeframe
  --no-trades

Command: plot-profit    help="Generate plot showing profits."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -p, --pairs
  --timerange
  --export
  --export-filename, --backtest-filename
  --db-url
  --trade-source
  -i, --timeframe
  --auto-open

Command: webserver    help="Webserver module."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir

Command: strategy-updater    help="updates outdated strategy files to the current version"
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  --strategy-list
  --strategy-path
  --recursive-strategy-search

Command: lookahead-analysis    help="Check for potential look ahead bias."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -i, --timeframe
  --timerange
  --data-format-ohlcv
  --max-open-trades
  --stake-amount
  --fee
  -p, --pairs
  --enable-protections, --enableprotections
  --dry-run-wallet, --starting-balance
  --timeframe-detail
  --strategy-list
  --export
  --export-filename, --backtest-filename
  --freqai-backtest-live-models
  --minimum-trade-amount
  --targeted-trade-amount
  --lookahead-analysis-exportfilename

Command: recursive-analysis    help="Check for potential recursive formula issue."
  -h, --help
  -v, --verbose
  --no-color
  --logfile, --log-file
  -V, --version
  -c, --config
  -d, --datadir, --data-dir
  --userdir, --user-data-dir
  -s, --strategy
  --strategy-path
  --recursive-strategy-search
  --freqaimodel
  --freqaimodel-path
  -i, --timeframe
  --timerange
  --data-format-ohlcv
  -p, --pairs
  --startup-candle
"""