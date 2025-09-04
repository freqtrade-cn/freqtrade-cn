---
title: FreqAI 参数表
subject: FreqAI 配置文档
subtitle: FreqAI 可用参数的完整列表
short_title: 参数表
description: 本文档详细列出了 FreqAI 的所有配置参数,包括通用配置参数和特征参数,并对每个参数的功能、数据类型和默认值进行了说明。
---

# 参数表

下表列出了 FreqAI 可用的所有配置参数。一些参数在 `config_examples/config_freqai.example.json` 中有示例。

必填参数标记为 **必需**，必须按建议方式之一进行设置。

## 通用配置参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`config.freqai` 树下的通用配置参数** |
| `freqai` | **必需。** <br> 包含所有 FreqAI 控制参数的父字典。<br> **数据类型：** 字典。 |
| `train_period_days` | **必需。** <br> 用于训练数据的天数（滑动窗口宽度）。<br> **数据类型：** 正整数。 |
| `backtest_period_days` | **必需。** <br> 在回测期间，从训练好的模型推理的天数，然后滑动上面定义的 `train_period_days` 窗口并重新训练模型（详见[此处](freqai-running.md#backtesting)）。可以为小数天，但请注意，提供的 `timerange` 会被此数值除以，以得出完成回测所需的训练次数。<br> **数据类型：** 浮点数。 |
| `identifier` | **必需。** <br> 当前模型的唯一 ID。如果模型保存到磁盘，`identifier` 允许重新加载特定的预训练模型/数据。<br> **数据类型：** 字符串。 |
| `live_retrain_hours` | dry/live 运行期间的再训练频率。<br> **数据类型：** 大于 0 的浮点数。<br> 默认值：`0`（模型尽可能频繁地再训练）。 |
| `expiration_hours` | 如果模型超过 `expiration_hours` 小时，则避免做出预测。<br> **数据类型：** 正整数。<br> 默认值：`0`（模型永不过期）。 |
| `purge_old_models` | 磁盘上保留的模型数量（与回测无关）。默认值为 2，表示 dry/live 运行会在磁盘上保留最新的 2 个模型。设置为 0 保留所有模型。此参数也接受布尔值以保持向后兼容。<br> **数据类型：** 整数。<br> 默认值：`2`。 |
| `save_backtest_models` | 回测时是否将模型保存到磁盘。回测通过保存预测数据并在后续运行中直接重用这些数据（如需调整入场/出场参数）来实现最高效。将回测模型保存到磁盘还允许用相同的模型 `identifier` 启动 dry/live 实例时复用这些模型文件。<br> **数据类型：** 布尔值。<br> 默认值：`False`（不保存模型）。 |
| `fit_live_predictions_candles` | 用于从预测数据而非训练数据集计算目标（标签）统计的历史蜡烛数量（详见[此处](freqai-configuration.md#creating-a-dynamic-target-threshold)）。<br> **数据类型：** 正整数。 |
| `continual_learning` | 使用最近训练模型的最终状态作为新模型的起点，实现增量学习（详见[此处](freqai-running.md#continual-learning)）。请注意，这目前是增量学习的朴素实现，在市场偏离模型时很容易过拟合/陷入局部最优。我们主要为实验目的保留此功能，以便为加密市场等混沌系统的更成熟增量学习方法做好准备。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `write_metrics_to_disk` | 将训练时长、推理时长和 CPU 使用情况收集到 json 文件中。<br> **数据类型：** 布尔值。<br> 默认值：`False` |
| `data_kitchen_thread_count` | <br> 指定用于数据处理（异常值方法、归一化等）的线程数。对训练所用线程数无影响。如果用户未设置（默认），FreqAI 会使用最大线程数减 2（为 Freqtrade 机器人和 FreqUI 留出 1 个物理核心）。<br> **数据类型：** 正整数。 |
| `activate_tensorboard` | <br> 是否为支持 tensorboard 的模块（目前为强化学习、XGBoost、Catboost 和 PyTorch）激活 tensorboard。tensorboard 需要安装 Torch，因此你需要使用 torch/RL docker 镜像，或在安装时选择安装 Torch。<br> **数据类型：** 布尔值。<br> 默认值：`True`。 |
| `wait_for_training_iteration_on_reload` | <br> 使用 /reload 或 ctrl-c 时，是否等待当前训练迭代完成后再优雅关闭。如果设为 `False`，FreqAI 会中断当前训练迭代，更快地优雅关闭，但会丢失当前训练迭代。<br> **数据类型：** 布尔值。<br> 默认值：`True`。 |

## 特征参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.feature_parameters` 子字典下的特征参数** |
| `feature_parameters` | 包含用于特征集工程的参数的字典。详情和示例见[此处](freqai-feature-engineering.md)。<br> **数据类型：** 字典。 |
| `include_timeframes` | 所有在 `feature_engineering_expand_*()` 中创建的指标所用的时间框架列表。该列表作为特征添加到基础指标数据集中。<br> **数据类型：** 时间框架（字符串）列表。 |
| `include_corr_pairlist` | FreqAI 会将该相关币种列表作为额外特征添加到所有 `pair_whitelist` 币种。特征工程期间在 `feature_engineering_expand_*()` 中设置的所有指标都会为每个相关币种创建。相关币种特征会添加到基础指标数据集中。<br> **数据类型：** 资产（字符串）列表。 |
| `label_period_candles` | 创建标签时面向未来的蜡烛数量。可在 `set_freqai_targets()` 中使用（详细用法见 `templates/FreqaiExampleStrategy.py`）。此参数非必需，你可以自定义标签并选择是否使用该参数。请参见 `templates/FreqaiExampleStrategy.py` 示例。<br> **数据类型：** 正整数。 |
| `include_shifted_candles` | 将前几根蜡烛的特征添加到后续蜡烛，以引入历史信息。如果使用，FreqAI 会复制并移动所有特征，使前 `include_shifted_candles` 根蜡烛的信息可用于后续蜡烛。<br> **数据类型：** 正整数。 |
| `weight_factor` | 按时间新旧对训练数据点加权（详见[此处](freqai-feature-engineering.md#weighting-features-for-temporal-importance)）。<br> **数据类型：** 正浮点数（通常 < 1）。 |
| `indicator_max_period_candles` | **已废弃（#7325）**。已被在[策略](freqai-configuration.md#building-a-freqai-strategy)中设置的 `startup_candle_count` 替代。`startup_candle_count` 与时间框架无关，定义了在 `feature_engineering_*()` 中创建指标时用到的最大*周期*。FreqAI 会结合 `include_time_frames` 中的最大时间框架计算需要下载的数据点数，以确保首个数据点不含 NaN。<br> **数据类型：** 正整数。 |
| `indicator_periods_candles` | 用于计算指标的周期。指标会添加到基础指标数据集中。<br> **数据类型：** 正整数列表。 |
| `principal_component_analysis` | 是否自动用主成分分析（PCA）降维。详见[此处](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `plot_feature_importances` | 为每个模型创建特征重要性图，显示前/后 `plot_feature_importances` 个特征。图表保存在 `user_data/models/<identifier>/sub-train-<COIN>_<timestamp>.html`。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `DI_threshold` | 设置大于 0 时启用 Dissimilarity Index 进行异常值检测。详见[此处](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)。<br> **数据类型：** 正浮点数（通常 < 1）。 |
| `use_SVM_to_remove_outliers` | 训练支持向量机（SVM）检测并移除训练集和输入数据中的异常值。详见[此处](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 布尔值。 |
| `svm_params` | Sklearn `SGDOneClassSVM()` 的所有可用参数。部分参数详见[此处](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 字典。 |
| `use_DBSCAN_to_remove_outliers` | 用 DBSCAN 算法聚类数据，识别并移除训练和预测数据中的异常值。详见[此处](freqai-feature-engineering.md#identifying-outliers-with-dbscan)。<br> **数据类型：** 布尔值。 |
| `noise_standard_deviation` | 若设置，FreqAI 会为训练特征添加噪声以防止过拟合。FreqAI 会从高斯分布生成标准差为 `noise_standard_deviation` 的随机偏差并加到所有数据点。`noise_standard_deviation` 应相对于归一化空间设置，即在 -1 到 1 之间。因为 FreqAI 中的数据总是归一化到 -1 到 1，`noise_standard_deviation: 0.05` 意味着 32% 的数据会被随机增减超过 2.5%（即落在第一个标准差内的数据百分比）。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `outlier_protection_percentage` | 启用后可防止异常值检测方法丢弃过多数据。如果 SVM 或 DBSCAN 检测为异常值的点超过 `outlier_protection_percentage`%，FreqAI 会记录警告并忽略异常值检测，即保留原始数据集。如果触发了异常值保护，则不会基于该训练集做出预测。<br> **数据类型：** 浮点数。<br> 默认值：`30`。 |
| `reverse_train_test_order` | 拆分特征数据集（见下文），用最新的数据拆分做训练，历史数据做测试。这样模型可以训练到最新数据点，同时避免过拟合。但在使用前应理解该参数的非常规性质。<br> **数据类型：** 布尔值。<br> 默认值：`False`（不反转）。
| `shuffle_after_split` | 拆分数据为训练集和测试集后，分别对两者进行洗牌。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `buffer_train_data_candles` | 在指标填充后，从训练数据的开头和结尾各裁剪 `buffer_train_data_candles` 个数据点。主要用于预测极大/极小值时，argrelextrema 函数无法在时间范围边缘判断极值。为提高模型准确性，最好在完整时间范围上计算 argrelextrema，然后用此参数按核函数裁剪边缘数据。若目标为移动价格，则不需要此 buffer，因为时间范围末尾的移动蜡烛会是 NaN，FreqAI 会自动从训练集中剔除这些数据。<br> **数据类型：** 整数。<br> 默认值：`0`。

## 数据拆分参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.data_split_parameters` 子字典下的数据拆分参数**
| `data_split_parameters` | 包含所有 scikit-learn `test_train_split()` 可用参数的字典，详见[此处](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)（外部网站）。<br> **数据类型：** 字典。
| `test_size` | 用于测试而非训练的数据比例。<br> **数据类型：** 小于 1 的正浮点数。
| `shuffle` | 训练期间是否对训练数据点洗牌。通常在时间序列预测中为保持数据的时间顺序，设为 `False`。<br> **数据类型：** 布尔值。<br> 默认值：`False`。

## 模型训练参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.model_training_parameters` 子字典下的模型训练参数**
| `model_training_parameters` | 灵活的字典，包含所选模型库的所有可用参数。例如，若用 `LightGBMRegressor`，该字典可包含 [LightGBMRegressor](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.LGBMRegressor.html) 的任意参数（外部网站）。如选用其他模型，则可包含该模型的任意参数。当前可用模型列表见[此处](freqai-configuration.md#using-different-prediction-models)。<br> **数据类型：** 字典。
| `n_estimators` | 训练模型时要拟合的提升树数量。<br> **数据类型：** 整数。
| `learning_rate` | 训练模型时的提升学习率。<br> **数据类型：** 浮点数。
| `n_jobs`, `thread_count`, `task_type` | 设置并行处理的线程数和 `task_type`（`gpu` 或 `cpu`）。不同模型库参数名不同。<br> **数据类型：** 浮点数。

## 强化学习参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.rl_config` 子字典下的强化学习参数**
| `rl_config` | 包含强化学习模型控制参数的字典。<br> **数据类型：** 字典。
| `train_cycles` | 训练步数将设为 `train_cycles * 训练数据点数`。<br> **数据类型：** 整数。
| `max_trade_duration_candles`| 指导智能体训练时保持交易低于期望长度。用法见 `prediction_models/ReinforcementLearner.py` 中可自定义的 `calculate_reward()` 函数。<br> **数据类型：** 整数。
| `model_type` | stable_baselines3 或 SBcontrib 的模型字符串。可用字符串包括：`'TRPO', 'ARS', 'RecurrentPPO', 'MaskablePPO', 'PPO', 'A2C', 'DQN'`。用户应确保 `model_training_parameters` 与所选 stable_baselines3 模型的参数一致，详见其文档。[PPO 文档](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html)（外部网站）<br> **数据类型：** 字符串。
| `policy_type` | stable_baselines3 可用的策略类型之一。<br> **数据类型：** 字符串。
| `max_training_drawdown_pct` | 智能体在训练期间允许的最大回撤。<br> **数据类型：** 浮点数。<br> 默认值：0.8
| `cpu_count` | 分配给强化学习训练进程的线程/CPU 数（取决于是否选择了 `ReinforcementLearning_multiproc`）。建议保持默认，默认值为物理核心总数减 1。<br> **数据类型：** 整数。
| `model_reward_parameters` | 用于 `ReinforcementLearner.py` 中可自定义 `calculate_reward()` 函数的参数。<br> **数据类型：** 整数。
| `add_state_info` | 告诉 FreqAI 在训练和推理时将状态信息包含在特征集中。当前状态变量包括交易时长、当前利润、持仓方向。仅在 dry/live 运行时可用，回测时自动关闭。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `net_arch` | 网络结构，详见 [`stable_baselines3` 文档](https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html#examples)。简言之：`[<共享层>, dict(vf=[<非共享价值网络层>], pi=[<非共享策略网络层>])]`。默认值为 `[128, 128]`，即 2 层共享隐藏层，每层 128 单元。
| `randomize_starting_position` | 随机化每个 episode 的起始点以避免过拟合。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `drop_ohlc_from_features` | 训练期间不将归一化 ohlc 数据包含在传递给智能体的特征集中（ohlc 仍用于驱动环境）。<br> **数据类型：** 布尔值。<br> **默认值：** `False`
| `progress_bar` | 显示进度条，包括当前进度、已用时间和预计剩余时间。<br> **数据类型：** 布尔值。<br> 默认值：`False`。

## PyTorch 参数

### 通用

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.model_training_parameters` 子字典下的模型训练参数**
| `learning_rate` | 传递给优化器的学习率。<br> **数据类型：** 浮点数。<br> 默认值：`3e-4`。
| `model_kwargs` | 传递给模型类的参数。<br> **数据类型：** 字典。<br> 默认值：`{}`。
| `trainer_kwargs` | 传递给训练器类的参数。<br> **数据类型：** 字典。<br> 默认值：`{}`。

### trainer_kwargs

| 参数    | 描述 |
|--------------|-------------|
|              |  **`freqai.model_training_parameters.model_kwargs` 子字典下的模型训练参数**
| `n_epochs`   | `n_epochs` 参数是 PyTorch 训练循环中的关键设置，决定整个训练集将用于更新模型参数的次数。一个 epoch 表示完整遍历一次训练集。会覆盖 `n_steps`。`n_epochs` 或 `n_steps` 必须设置一个。<br><br> **数据类型：** int，可选。<br> 默认值：`10`。
| `n_steps`    | 设置 `n_epochs` 的另一种方式——训练迭代次数。这里的迭代指调用 `optimizer.step()` 的次数。如果设置了 `n_epochs`，则忽略。简化公式：<br><br> n_epochs = n_steps / (n_obs / batch_size) <br><br> 这样做的动机是 `n_steps` 更易于优化，并能在不同 n_obs（数据点数）下保持稳定。<br><br> **数据类型：** int，可选。<br> 默认值：`None`。
| `batch_size` | 训练时使用的批次大小。<br><br> **数据类型：** int。<br> 默认值：`64`。


## 其他参数

|  参数 | 描述 |
|------------|-------------|
|  |  **其他参数**
| `freqai.keras` | 如果所选模型使用 Keras（典型于基于 TensorFlow 的预测模型），需激活此标志，以便模型保存/加载遵循 Keras 标准。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `freqai.conv_width` | 神经网络输入张量的宽度。可替代移动蜡烛（`include_shifted_candles`），通过将历史数据点作为张量的第二维输入。技术上也可用于回归器，但只会增加计算开销，不影响模型训练/预测。<br> **数据类型：** 整数。<br> 默认值：`2`。
| `freqai.reduce_df_footprint` | 将所有数值列转换为 float32/int32，旨在减少内存/磁盘占用并加快训练/推理。此参数在 Freqtrade 配置文件主级别设置（不在 FreqAI 内部）。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
