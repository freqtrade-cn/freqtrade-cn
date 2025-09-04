---
title: FreqAI 运行指南
subject: FreqAI 使用文档
subtitle: 如何训练和部署自适应机器学习模型
short_title: FreqAI 运行
description: FreqAI 是一个自适应机器学习框架，本文档介绍了如何在实时和回测模式下运行 FreqAI，包括模型训练、部署和管理等内容。
---

# 运行 FreqAI

有两种方式可以训练和部署自适应机器学习模型——实时部署和历史回测。在这两种情况下，FreqAI 都会运行/模拟模型的周期性再训练，如下图所示：

![freqai-window](assets/freqai_moving-window.jpg)

## 实时部署

可以使用以下命令以 dry/live 模式运行 FreqAI：

```bash
freqtrade trade --strategy FreqaiExampleStrategy --config config_freqai.example.json --freqaimodel LightGBMRegressor
```

启动后，FreqAI 会根据配置设置开始训练一个新的模型，并生成一个新的 `identifier`。训练完成后，该模型将用于对新到的蜡烛数据进行预测，直到有新模型可用。通常，FreqAI 会尽可能频繁地生成新模型，并管理一个内部队列以尽量保持所有币对的模型都是最新的。FreqAI 总是使用最近训练的模型对实时数据进行预测。如果你不希望 FreqAI 频繁再训练新模型，可以设置 `live_retrain_hours`，让 FreqAI 至少等待指定小时数后再训练新模型。此外，你可以设置 `expired_hours`，让 FreqAI 避免使用超过该小时数的旧模型进行预测。

训练好的模型默认会保存到磁盘，以便在回测或崩溃后复用。你可以在配置中设置 `"purge_old_models": true` 来[清理旧模型](#purging-old-model-data)，以节省磁盘空间。

要从已保存的回测模型（或先前崩溃的 dry/live 会话）启动 dry/live 运行，只需指定特定模型的 `identifier`：

```json
    "freqai": {
        "identifier": "example",
        "live_retrain_hours": 0.5
    }
```

在这种情况下，虽然 FreqAI 会用预训练模型启动，但仍会检查自模型训练以来经过了多长时间。如果自加载模型结束以来已过完整的 `live_retrain_hours`，FreqAI 会开始训练新模型。

### 自动数据下载

FreqAI 会自动下载所需的数据量，以确保通过定义的 `train_period_days` 和 `startup_candle_count` 训练模型（详细参数说明见[参数表](freqai-parameter-table.md)）。

### 保存预测数据

在特定 `identifier` 模型生命周期内的所有预测都会存储在 `historic_predictions.pkl` 文件中，以便在崩溃或配置更改后重新加载。

### 清理旧模型数据

FreqAI 每次成功训练后都会保存新模型文件。随着新模型的生成，这些文件会变得过时。如果你计划让 FreqAI 长时间运行并高频再训练，建议在配置中启用 `purge_old_models`：

```json
    "freqai": {
        "purge_old_models": 4,
    }
```

这会自动清理除最近四个模型外的所有旧模型，以节省磁盘空间。输入 "0" 则永远不会清理任何模型。

## 回测

可以用以下命令执行 FreqAI 回测模块：

```bash
freqtrade backtesting --strategy FreqaiExampleStrategy --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --freqaimodel LightGBMRegressor --timerange 20210501-20210701
```

如果使用现有配置文件从未执行过该命令，FreqAI 会为每个币对、每个回测窗口训练新模型。

回测模式需要在部署前[下载必要的数据](#downloading-data-to-cover-the-full-backtest-period)（与 dry/live 模式不同，后者 FreqAI 会自动下载数据）。你需要确保下载的数据时间范围大于回测时间范围，因为 FreqAI 需要在目标回测时间范围开始前的数据来训练模型，以便在回测时间范围的第一根蜡烛上就能做出预测。如何计算所需下载数据的更多细节见[此处](#deciding-the-size-of-the-sliding-training-window-and-backtesting-duration)。

:::{note} 模型复用
训练完成后，可以用相同配置文件再次执行回测，FreqAI 会找到已训练的模型并加载，而不会重新训练。这对于在策略内部调整（甚至超参数优化）买入和卖出条件非常有用。如果你*想*用相同配置文件重新训练新模型，只需更改 `identifier`。这样，你可以通过指定 `identifier` 随时切换到任何你想用的模型。
:::

:::{note}
回测会对每个回测窗口调用一次 `set_freqai_targets()`（窗口数量为完整回测时间范围除以 `backtest_period_days` 参数）。这样可以模拟 dry/live 行为而不会产生前视偏差。但 `feature_engineering_*()` 的特征定义会在整个训练时间范围上执行一次。因此要确保特征不会前视未来。关于前视偏差的更多细节见[常见错误](strategy-customization.md#common-mistakes-when-developing-strategies)。
:::

---

### 保存回测预测数据

为了便于调整你的策略（**不是**特征！），FreqAI 会自动保存回测期间的预测，以便在未来回测和实时运行时复用相同 `identifier` 模型的预测。这为**高层次超参数优化**进出场条件提供了性能提升。

在 `unique-id` 文件夹下会创建一个名为 `backtesting_predictions` 的目录，里面包含所有以 `feather` 格式存储的预测。

要更改**特征**，你**必须**在配置中设置新的 `identifier`，以通知 FreqAI 训练新模型。

如果你希望将某次回测生成的模型保存下来，以便后续直接用其启动实时部署而不是重新训练新模型，需在配置中设置 `save_backtest_models` 为 `True`。

:::{note}
为确保模型可复用，freqAI 会用长度为 1 的 dataframe 调用你的策略。如果你的策略生成相同特征所需的数据多于 1 条，则无法将回测预测用于实时部署，每次新回测都需更新 `identifier`。
:::

### 回测实时收集的预测

FreqAI 允许你通过回测参数 `--freqai-backtest-live-models` 复用实时历史预测。当你想复用 dry/run 生成的预测进行对比或其他研究时，这很有用。

此时不能指定 `--timerange` 参数，因为它会根据历史预测文件中的数据自动计算。

### 下载完整回测周期所需数据

对于 dry/live 部署，FreqAI 会自动下载所需数据。但要使用回测功能，你需要用 `download-data` 命令下载数据（详见[此处](data-download.md#data-downloading)）。你需要特别注意，下载的数据量要足够，确保在回测时间范围开始前有足够的训练数据。额外数据量可通过将时间范围起始日期向前移动 `train_period_days` 和 `startup_candle_count`（详细参数见[参数表](freqai-parameter-table.md)）来粗略估算。

例如，若要用[示例配置](freqai-configuration.md#setting-up-the-configuration-file)（`train_period_days` 设为 30，`startup_candle_count: 40`，最大 `include_timeframes` 为 1h）回测 `--timerange 20210501-20210701`，则下载数据的起始日期应为 `20210501` - 30 天 - 40 * 1h / 24 小时 = 20210330（比目标训练时间范围早 31.7 天）。

### 决定滑动训练窗口和回测周期的大小

回测时间范围通过配置文件中的常规 `--timerange` 参数定义。滑动训练窗口的持续时间由 `train_period_days` 设置，`backtest_period_days` 则是滑动回测窗口，二者均以天为单位（`backtest_period_days` 可为小数，表示在 live/dry 模式下的亚日再训练）。在[示例配置](freqai-configuration.md#setting-up-the-configuration-file)（位于 `config_examples/config_freqai.example.json`）中，用户要求 FreqAI 使用 30 天训练期，并在随后的 7 天进行回测。模型训练完成后，FreqAI 会回测接下来的 7 天。然后"滑动窗口"前移一周（模拟 FreqAI 在 live 模式下每周再训练一次），新模型使用前 30 天（包括前一个模型用于回测的 7 天）进行训练。如此反复，直到 `--timerange` 结束。这意味着如果你设置 `--timerange 20210501-20210701`，FreqAI 在该时间范围结束时会训练 8 个独立模型（因为整个范围包含 8 周）。

:::{note}
虽然允许使用小数的 `backtest_period_days`，但要注意 `--timerange` 会被该值整除，以确定 FreqAI 需要训练多少个模型来完成整个回测。例如，设置 10 天的 `--timerange` 和 0.1 的 `backtest_period_days`，FreqAI 需要为每个币对训练 100 个模型才能完成回测。正因如此，完整回测 FreqAI 的自适应训练会非常耗时。最好的测试方法是直接 dry run 并让其持续训练。在这种情况下，回测所需时间与 dry run 完全相同。
:::

## 定义模型过期时间

在 dry/live 模式下，FreqAI 会按顺序为每个币对训练模型（在主 Freqtrade 机器人之外的独立线程/GPU 上）。这意味着模型之间总会有年龄差异。如果你训练 50 个币对，每个币对训练需 5 分钟，最老的模型会超过 4 小时。这在策略的特征时间尺度（交易目标周期）小于 4 小时时可能不可接受。你可以在配置文件中设置 `expiration_hours`，只允许模型在小于指定小时数时参与交易：

```json
    "freqai": {
        "expiration_hours": 0.5,
    }
```

在示例配置中，用户只允许使用小于半小时的模型进行预测。

## 控制模型学习过程

模型训练参数取决于所选的机器学习库。FreqAI 允许你在配置的 `model_training_parameters` 字典中为任意库设置任意参数。示例配置（见 `config_examples/config_freqai.example.json`）展示了 `Catboost` 和 `LightGBM` 的部分参数示例，但你可以添加这些库或你实现的其他库的所有可用参数。

数据划分参数在 `data_split_parameters` 中定义，可为 scikit-learn 的 `train_test_split()` 函数的任意参数。`train_test_split()` 有一个 `shuffle` 参数，允许打乱数据或保持顺序。这对于避免因时间自相关数据而导致的训练偏差特别有用。更多参数细节见[scikit-learn 官网](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)（外部网站）。

FreqAI 特有参数 `label_period_candles` 定义用于 `labels` 的未来蜡烛数量偏移。在[示例配置](freqai-configuration.md#setting-up-the-configuration-file)中，用户要求 `labels` 预测未来 24 根蜡烛。

## 持续学习

你可以在配置中设置 `"continual_learning": true` 采用持续学习方案。启用后，初始模型会从头训练，后续训练会从前一次训练的最终模型状态开始。这让新模型拥有"记忆"。默认情况下为 `False`，即所有新模型都从头训练，不继承前一模型。

:::{danger} 持续学习强制参数空间恒定
由于 `continual_learning` 意味着模型参数空间*不能*在训练间变化，启用 `continual_learning` 时会自动禁用 `principal_component_analysis`。提示：PCA 会改变参数空间和特征数量，详细了解 PCA 见[此处](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)。
:::

:::{danger} 实验性功能
请注意，这目前是增量学习的朴素实现，在市场偏离模型时极易过拟合/陷入局部最优。FreqAI 主要为实验目的提供该机制，以便为加密市场等混沌系统的更成熟持续学习方法做好准备。
:::

## 超参数优化（Hyperopt）

你可以用与[常规 Freqtrade 超参数优化](hyperopt.md)相同的命令进行 hyperopt：

```bash
freqtrade hyperopt --hyperopt-loss SharpeHyperOptLoss --strategy FreqaiExampleStrategy --freqaimodel LightGBMRegressor --strategy-path freqtrade/templates --config config_examples/config_freqai.example.json --timerange 20220428-20220507
```

`hyperopt` 要求你像[回测](#backtesting)一样预先下载数据。此外，hyperopt FreqAI 策略时需注意以下限制：

- `--analyze-per-epoch` 超参数不兼容 FreqAI。
- 不能对 `feature_engineering_*()` 和 `set_freqai_targets()` 函数中的指标进行超参数优化。这意味着不能用 hyperopt 优化模型参数。除此之外，可以优化所有其他[空间](hyperopt.md#running-hyperopt-with-smaller-search-space)。
- 回测的所有说明同样适用于 hyperopt。

结合 hyperopt 和 FreqAI 的最佳方法是专注于优化进出场阈值/条件。你需要优化未用于特征的参数。例如，不应尝试优化特征创建中的滚动窗口长度，或任何会改变预测的 FreqAI 配置部分。为高效 hyperopt FreqAI 策略，FreqAI 会将预测保存为 dataframe 并复用。因此只需优化进出场阈值/条件。

FreqAI 中可超参数优化的一个好例子是 [Dissimilarity Index (DI)](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di) 的阈值 `DI_values`，超过该值的数据点视为异常值：

```python
di_max = IntParameter(low=1, high=20, default=10, space='buy', optimize=True, load=True)
dataframe['outlier'] = np.where(dataframe['DI_values'] > self.di_max.value/10, 1, 0)
```

这个超参数优化可以帮助你了解适合你参数空间的 `DI_values`。

## 使用 Tensorboard

:::{note} 可用性
FreqAI 支持多种模型的 tensorboard，包括 XGBoost、所有 PyTorch 模型、强化学习和 Catboost。如果你希望在其他模型类型中集成 Tensorboard，请在 [Freqtrade GitHub](https://github.com/freqtrade/freqtrade/issues) 提交 issue。
:::

:::{danger} 要求
Tensorboard 日志记录需要 FreqAI 的 torch 安装或 docker 镜像。
:::

最简单的使用 tensorboard 的方式是确保配置文件中 `freqai.activate_tensorboard` 设置为 `True`（默认），运行 FreqAI 后，在另一个 shell 中运行：

```bash
cd freqtrade
tensorboard --logdir user_data/models/unique-id
```

其中 `unique-id` 是 `freqai` 配置文件中设置的 `identifier`。如果你希望在浏览器 127.0.0.1:6060（6060 是 Tensorboard 默认端口）查看输出，必须在单独的 shell 中运行该命令。

![tensorboard](assets/tensorboard.jpg)

:::{note} 性能提示
Tensorboard 日志记录会降低训练速度，生产环境建议关闭。
:::
