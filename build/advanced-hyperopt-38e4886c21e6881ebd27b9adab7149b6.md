---
title: 高级超参数优化
subject: FreqTrade 高级超参数优化指南
subtitle: 深入了解超参数优化功能
short_title: 高级超参数优化
description: 本文档介绍了 FreqTrade 的高级超参数优化功能,包括自定义损失函数等高级特性。
---

# 高级超参数优化（Hyperopt）

本页介绍一些高级 Hyperopt 主题，可能比创建普通超参数优化类需要更高的编码技能和 Python 知识。

## 创建和使用自定义损失函数

要使用自定义损失函数类，请确保你的自定义 hyperopt 损失类中定义了 `hyperopt_loss_function` 函数。
对于下方的示例，你需要在 hyperopt 命令中添加参数 `--hyperopt-loss SuperDuperHyperOptLoss`，以便使用该函数。

下面是一个示例，与默认 Hyperopt 损失实现完全相同。完整示例可见于 [userdata/hyperopts](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_hyperopt_loss.py)。

```python
from datetime import datetime
from typing import Any, Dict

from pandas import DataFrame

from freqtrade.constants import Config
from freqtrade.optimize.hyperopt import IHyperOptLoss

TARGET_TRADES = 600
EXPECTED_MAX_PROFIT = 3.0
MAX_ACCEPTED_TRADE_DURATION = 300

class SuperDuperHyperOptLoss(IHyperOptLoss):
    """
    定义 hyperopt 的默认损失函数
    """

    @staticmethod
    def hyperopt_loss_function(
        *,
        results: DataFrame,
        trade_count: int,
        min_date: datetime,
        max_date: datetime,
        config: Config,
        processed: dict[str, DataFrame],
        backtest_stats: dict[str, Any],
        starting_balance: float,
        **kwargs,
    ) -> float:
        """
        目标函数，结果越小越好
        这是传统算法（freqtrade 一直使用的）。权重分配如下：
        * 0.4 给交易时长
        * 0.25：避免交易亏损
        * 1.0 给总利润，相对于上面定义的期望值（`EXPECTED_MAX_PROFIT`）
        """
        total_profit = results['profit_ratio'].sum()
        trade_duration = results['trade_duration'].mean()

        trade_loss = 1 - 0.25 * exp(-(trade_count - TARGET_TRADES) ** 2 / 10 ** 5.8)
        profit_loss = max(0, 1 - total_profit / EXPECTED_MAX_PROFIT)
        duration_loss = 0.4 * min(trade_duration / MAX_ACCEPTED_TRADE_DURATION, 1)
        result = trade_loss + profit_loss + duration_loss
        return result
```

目前，参数说明如下：

* `results`：包含结果交易的 DataFrame。
    结果中可用的列如下（与回测时 `--export trades` 输出文件一致）：  
    `pair, profit_ratio, profit_abs, open_date, open_rate, fee_open, close_date, close_rate, fee_close, amount, trade_duration, is_open, exit_reason, stake_amount, min_rate, max_rate, stop_loss_ratio, stop_loss_abs`
* `trade_count`：交易数量（等同于 `len(results)`）
* `min_date`：使用的时间范围起始日期
* `max_date`：使用的时间范围结束日期
* `config`：使用的配置对象（注意：如果某些策略相关参数属于 hyperopt 空间，这里未必会更新全部参数）。
* `processed`：以交易对为键的 DataFrame 字典，包含回测用到的数据。
* `backtest_stats`：回测统计信息，格式与回测文件“strategy”子结构一致。可用字段见 `optimize_reports.py` 中的 `generate_strategy_stats()`。
* `starting_balance`：回测使用的初始资金。

该函数需返回一个浮点数（`float`）。数值越小，结果越好。参数和权重分配可自行调整。

:::{note}
该函数每个 epoch 调用一次——请尽量优化代码，避免拖慢 hyperopt。
:::

:::{note} `*args` 和 `**kwargs`
请保留接口中的 `*args` 和 `**kwargs`，以便未来扩展。
:::

(overriding-pre-defined-spaces)=

## 覆盖预定义空间

要覆盖预定义空间（如 `roi_space`、`generate_roi_table`、`stoploss_space`、`trailing_space`、`max_open_trades_space`），请在策略中定义一个嵌套的 Hyperopt 类，并如下定义所需空间：

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal

class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        # 自定义止损空间
        def stoploss_space():
            return [SKDecimal(-0.05, -0.01, decimals=3, name='stoploss')]

        # 自定义 ROI 空间
        def roi_space() -> List[Dimension]:
            return [
                Integer(10, 120, name='roi_t1'),
                Integer(10, 60, name='roi_t2'),
                Integer(10, 40, name='roi_t3'),
                SKDecimal(0.01, 0.04, decimals=3, name='roi_p1'),
                SKDecimal(0.01, 0.07, decimals=3, name='roi_p2'),
                SKDecimal(0.01, 0.20, decimals=3, name='roi_p3'),
            ]

        def generate_roi_table(params: Dict) -> dict[int, float]:

            roi_table = {}
            roi_table[0] = params['roi_p1'] + params['roi_p2'] + params['roi_p3']
            roi_table[params['roi_t3']] = params['roi_p1'] + params['roi_p2']
            roi_table[params['roi_t3'] + params['roi_t2']] = params['roi_p1']
            roi_table[params['roi_t3'] + params['roi_t2'] + params['roi_t1']] = 0

            return roi_table

        def trailing_space() -> List[Dimension]:
            # 这里所有参数都是必需的，只能修改类型或范围。
            return [
                # 固定为 true，如果优化 trailing_stop，则假定始终使用追踪止损。
                Categorical([True], name='trailing_stop'),

                SKDecimal(0.01, 0.35, decimals=3, name='trailing_stop_positive'),
                # 'trailing_stop_positive_offset' 应大于 'trailing_stop_positive'，
                # 所以这里用中间参数表示两者的差值。'trailing_stop_positive_offset' 的值在
                # generate_trailing_params() 方法中构造。
                # 这类似于用于构造 ROI 表的 hyperspace 维度。
                SKDecimal(0.001, 0.1, decimals=3, name='trailing_stop_positive_offset_p1'),

                Categorical([True, False], name='trailing_only_offset_is_reached'),
        ]

        # 自定义最大持仓数空间
        def max_open_trades_space(self) -> List[Dimension]:
            return [
                Integer(-1, 10, name='max_open_trades'),
            ]
```

:::{note}
所有覆盖都是可选的，可根据需要混合使用。
:::

### 动态参数

参数也可以动态定义，但在 [`bot_start()` 回调](strategy-callbacks.md#bot-start) 被调用后，实例必须能访问到这些参数。

```python

class MyAwesomeStrategy(IStrategy):

    def bot_start(self, **kwargs) -> None:
        self.buy_adx = IntParameter(20, 30, default=30, optimize=True)

    # ...
```

:::{warning}
以这种方式创建的参数不会出现在 `list-strategies` 的参数计数中。
:::

### 覆盖 Base estimator

你可以通过在 Hyperopt 子类中实现 `generate_estimator()`，为 Hyperopt 定义自己的 optuna 采样器。

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            return "NSGAIIISampler"

```

可用值包括 "NSGAIISampler"、"TPESampler"、"GPSampler"、"CmaEsSampler"、"NSGAIIISampler"、"QMCSampler"（详情见 [optuna-samplers 文档](https://optuna.readthedocs.io/en/stable/reference/samplers/index.html)），也可以是继承自 `optuna.samplers.BaseSampler` 的类实例。

有时需要自行研究以发现更多采样器（如 optunahub 提供的）。

:::{note}
虽然可以自定义 estimator，但需要你自己研究参数并分析/理解应使用哪些参数。
如果不确定，建议直接使用默认值（`"NSGAIIISampler"` 被证明最为通用）。
:::

:::{hint} 使用 Optunahub 的 `AutoSampler`

[AutoSampler 文档](https://hub.optuna.org/samplers/auto_sampler/)

安装必要依赖
```bash
pip install optunahub cmaes torch scipy
```
在策略中实现 `generate_estimator()`

```python
# ...
from freqtrade.strategy.interface import IStrategy
from typing import List
import optunahub
# ... 

class my_strategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List["Dimension"], **kwargs):
            if "random_state" in kwargs.keys():
                return optunahub.load_module("samplers/auto_sampler").AutoSampler(seed=kwargs["random_state"])
            else:
                return optunahub.load_module("samplers/auto_sampler").AutoSampler()

```

显然，optuna 支持的其他采样器也可用同样方式集成。
:::

## 空间类型选项

对于附加空间，scikit-optimize（结合 Freqtrade）提供以下空间类型：

* `Categorical` - 从类别列表中选择（如 `Categorical(['a', 'b', 'c'], name="cat")`）
* `Integer` - 从整数范围中选择（如 `Integer(1, 10, name='rsi')`）
* `SKDecimal` - 从有限精度的小数范围中选择（如 `SKDecimal(0.1, 0.5, decimals=3, name='adx')`）。*仅 Freqtrade 提供。*
* `Real` - 从全精度小数范围中选择（如 `Real(0.1, 0.5, name='adx')`）

你可以从 `freqtrade.optimize.space` 导入所有这些类型，尽管 `Categorical`、`Integer` 和 `Real` 只是 scikit-optimize 空间的别名。`SKDecimal` 由 freqtrade 提供，用于更快的优化。

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal, Real  # noqa
```

:::{hint} SKDecimal vs. Real
我们建议几乎所有场景都使用 `SKDecimal`，而不是 `Real` 空间。虽然 Real 空间提供全精度（高达约 16 位小数），但这种精度很少需要，且会导致 hyperopt 时间过长。

假设定义了一个较小的空间（`SKDecimal(0.10, 0.15, decimals=2, name='xxx')`）——SKDecimal 只有 5 种可能（`[0.10, 0.11, 0.12, 0.13, 0.14, 0.15]`）。

相应的 real 空间 `Real(0.10, 0.15 name='xxx')` 则有几乎无限多的可能（`[0.10, 0.010000000001, 0.010000000002, ... 0.014999999999, 0.01500000000]`）。
:::
