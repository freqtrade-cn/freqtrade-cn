---
title: FreqAI 配置指南
subject: FreqAI 配置文档
subtitle: 配置 FreqAI 的详细说明
short_title: FreqAI 配置
description: 本文档详细介绍了如何配置 FreqAI,包括配置文件设置和策略构建。FreqAI 是 Freqtrade 的机器学习框架,可用于构建和训练预测模型。
---

# 配置

FreqAI 通过典型的 [Freqtrade 配置文件](configuration.md) 和标准的 [Freqtrade 策略](strategy-customization.md) 进行配置。FreqAI 配置和策略文件的示例分别可在 `config_examples/config_freqai.example.json` 和 `freqtrade/templates/FreqaiExampleStrategy.py` 中找到。

## 配置文件设置

虽然有许多可选参数可供选择（详见[参数表](freqai-parameter-table.md#parameter-table)），但 FreqAI 配置至少必须包含以下参数（参数值仅为示例）：

```json
    "freqai": {
        "enabled": true,
        "purge_old_models": 2,
        "train_period_days": 30,
        "backtest_period_days": 7,
        "identifier" : "unique-id",
        "feature_parameters" : {
            "include_timeframes": ["5m","15m","4h"],
            "include_corr_pairlist": [
                "ETH/USD",
                "LINK/USD",
                "BNB/USD"
            ],
            "label_period_candles": 24,
            "include_shifted_candles": 2,
            "indicator_periods_candles": [10, 20]
        },
        "data_split_parameters" : {
            "test_size": 0.25
        }
    }
```

完整的配置示例可在 `config_examples/config_freqai.example.json` 中找到。

:::{note}
`identifier` 常被新手忽略，但它在你的配置中起着重要作用。这个值是你为某次运行选择的唯一 ID。保持它不变可以实现崩溃恢复和更快的回测。当你想尝试新的运行（新特征、新模型等）时，应更改此值（或删除 `user_data/models/unique-id` 文件夹）。更多细节见[参数表](freqai-parameter-table.md#feature-parameters)。
:::

## 构建 FreqAI 策略

FreqAI 策略需要在标准 [Freqtrade 策略](strategy-customization.md) 中包含以下代码：

```python
    # 用户应定义最大启动蜡烛数（传递给任一指标的最大蜡烛数）
    startup_candle_count: int = 20

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        # 模型将返回用户在 `set_freqai_targets()` 中创建的所有标签
        # （及附加目标）、是否应接受预测的指示、
        # 以及每个训练周期用户在 `set_freqai_targets()` 中创建的每个标签的目标均值/标准差。

        dataframe = self.freqai.start(dataframe, metadata, self)

        return dataframe

    def feature_engineering_expand_all(self, dataframe: DataFrame, period, **kwargs) -> DataFrame:
        """
        *仅在启用 FreqAI 的策略中有效*
        此函数会根据配置中定义的 `indicator_periods_candles`、`include_timeframes`、`include_shifted_candles` 和 `include_corr_pairs` 自动扩展定义的特征。换句话说，在此函数中定义的单个特征会自动扩展为
        `indicator_periods_candles` * `include_timeframes` * `include_shifted_candles` *
        `include_corr_pairs` 个特征添加到模型中。

        所有特征必须以 `%` 开头，FreqAI 内部才能识别。

        :param df: 将接收特征的策略数据框
        :param period: 指标的周期 - 用法示例：
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)
        """

        dataframe["%-rsi-period"] = ta.RSI(dataframe, timeperiod=period)
        dataframe["%-mfi-period"] = ta.MFI(dataframe, timeperiod=period)
        dataframe["%-adx-period"] = ta.ADX(dataframe, timeperiod=period)
        dataframe["%-sma-period"] = ta.SMA(dataframe, timeperiod=period)
        dataframe["%-ema-period"] = ta.EMA(dataframe, timeperiod=period)

        return dataframe

    def feature_engineering_expand_basic(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用 FreqAI 的策略中有效*
        此函数会根据配置中定义的 `include_timeframes`、`include_shifted_candles` 和 `include_corr_pairs` 自动扩展定义的特征。换句话说，在此函数中定义的单个特征会自动扩展为
        `include_timeframes` * `include_shifted_candles` * `include_corr_pairs`
        个特征添加到模型中。

        这里定义的特征*不会*在用户定义的 `indicator_periods_candles` 上自动复制。

        所有特征必须以 `%` 开头，FreqAI 内部才能识别。

        :param df: 将接收特征的策略数据框
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-ema-200"] = ta.EMA(dataframe, timeperiod=200)
        """
        dataframe["%-pct-change"] = dataframe["close"].pct_change()
        dataframe["%-raw_volume"] = dataframe["volume"]
        dataframe["%-raw_price"] = dataframe["close"]
        return dataframe

    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用 FreqAI 的策略中有效*
        此可选函数会在基础时间框架的数据框上调用一次。
        这是要调用的最终函数，这意味着进入此函数的数据框将包含所有其他
        freqai_feature_engineering_* 函数创建的所有特征和列。

        这是进行自定义特征提取（如 tsfresh）的好地方。
        也是定义不应自动扩展的特征（如星期几）的好地方。

        所有特征必须以 `%` 开头，FreqAI 内部才能识别。

        :param df: 将接收特征的策略数据框
        用法示例：dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        """
        dataframe["%-day_of_week"] = (dataframe["date"].dt.dayofweek + 1) / 7
        dataframe["%-hour_of_day"] = (dataframe["date"].dt.hour + 1) / 25
        return dataframe

    def set_freqai_targets(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        """
        *仅在启用 FreqAI 的策略中有效*
        设置模型目标的必需函数。
        所有目标必须以 `&` 开头，FreqAI 内部才能识别。

        :param df: 将接收目标的策略数据框
        用法示例：dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        dataframe["&-s_close"] = (
            dataframe["close"]
            .shift(-self.freqai_info["feature_parameters"]["label_period_candles"])
            .rolling(self.freqai_info["feature_parameters"]["label_period_candles"])
            .mean()
            / dataframe["close"]
            - 1
            )
        return dataframe
```

注意 `feature_engineering_*()` 是添加[特征](freqai-feature-engineering.md#feature-engineering)的地方，而 `set_freqai_targets()` 用于添加标签/目标。完整的策略示例见 `templates/FreqaiExampleStrategy.py`。

:::{note}
`self.freqai.start()` 函数不能在 `populate_indicators()` 之外调用。
:::

:::{note}
特征**必须**在 `feature_engineering_*()` 中定义。在 `populate_indicators()` 中定义 FreqAI 特征会导致算法在 live/dry 模式下失败。若要添加与特定交易对或时间框架无关的通用特征，应使用 `feature_engineering_standard()`（如 `freqtrade/templates/FreqaiExampleStrategy.py` 所示）。
:::

## 重要的数据框键模式

以下是你可以在典型策略数据框（`df[]`）中包含/使用的键：

|  DataFrame 键 | 描述 |
|------------|-------------|
| `df['&*']` | 任何在 `set_freqai_targets()` 中以 `&` 开头的数据框列都会被 FreqAI 视为训练目标（标签）（通常遵循 `&-s*` 命名约定）。例如，要预测未来 40 根蜡烛的收盘价，你可以设置 `df['&-s_close'] = df['close'].shift(-self.freqai_info["feature_parameters"]["label_period_candles"])`，并在配置中设置 `"label_period_candles": 40`。FreqAI 会做出预测并以相同的键（`df['&-s_close']`）返回结果，可在 `populate_entry/exit_trend()` 中使用。<br> **数据类型：** 取决于模型输出。
| `df['&*_std/mean']` | 训练期间（或使用 `fit_live_predictions_candles` 进行实时跟踪时）定义标签的标准差和均值。常用于了解预测的罕见程度（可用 z-score，如 `templates/FreqaiExampleStrategy.py` 所示，并在[此处](#creating-a-dynamic-target-threshold)解释，用于评估某一预测在训练期间或历史上出现的频率）。<br> **数据类型：** 浮点数。
| `df['do_predict']` | 异常点指示。返回值为 -2 到 2 的整数，表示预测是否可信。`do_predict==1` 表示预测可信。如果输入数据点的 Dissimilarity Index（DI，详见[此处](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)）高于配置阈值，FreqAI 会将 `do_predict` 减 1，结果为 `do_predict==0`。如果启用 `use_SVM_to_remove_outliers`，支持向量机（SVM，详见[此处](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)）也可能检测到训练和预测数据中的异常值，此时 SVM 也会将 `do_predict` 减 1。如果 SVM 和 DI 之一认为是异常点，结果为 `do_predict==0`；如果两者都认为是异常点，结果为 `do_predict==-1`。如果同时启用 SVM 和 DBSCAN（详见[此处](freqai-feature-engineering.md#identifying-outliers-with-dbscan)），且两者都检测到异常点，结果为 `do_predict==-2`。特殊情况是 `do_predict == 2`，表示模型因超出 `expired_hours` 而过期。<br> **数据类型：** -2 到 2 之间的整数。
| `df['DI_values']` | DI（Dissimilarity Index）值是 FreqAI 对预测置信度的代理。DI 越低，预测越接近训练数据，即置信度越高。详见[此处](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)。<br> **数据类型：** 浮点数。
| `df['%*']` | 任何在 `feature_engineering_*()` 中以 `%` 开头的数据框列都会被视为训练特征。例如，你可以通过设置 `df['%-rsi']` 将 RSI 纳入训练特征集（如 `templates/FreqaiExampleStrategy.py`）。更多细节见[此处](freqai-feature-engineering.md)。<br> **注意：** 由于以 `%` 开头的特征数量会迅速增加（如 `include_shifted_candles` 和 `include_timeframes` 的乘法功能，详见[参数表](freqai-parameter-table.md)），这些特征会在 FreqAI 返回给策略的数据框中被移除。若要保留某类特征用于绘图，应以 `%%` 开头（见下文）。<br> **数据类型：** 取决于用户创建的特征。
| `df['%%*']` | 任何在 `feature_engineering_*()` 中以 `%%` 开头的数据框列也会被视为训练特征，与上面的 `%` 类似。但此时这些特征会返回给策略，用于 FreqUI/plot-dataframe 绘图和 Dry/Live/Backtesting 监控。<br> **数据类型：** 取决于用户创建的特征。请注意，在 `feature_engineering_expand()` 中创建的特征会根据你配置的扩展自动采用 FreqAI 命名方案（如 `include_timeframes`、`include_corr_pairlist`、`indicators_periods_candles`、`include_shifted_candles`）。因此，如果你想绘制 `%%-rsi`，最终的绘图配置名称可能是：`%%-rsi-period_10_ETH/USDT:USDT_1h`，表示 `rsi` 特征，`period=10`，`timeframe=1h`，`pair=ETH/USDT:USDT`（如果用的是期货对会加 `:USDT`）。建议在 `populate_indicators()` 中 `self.freqai.start()` 之后加 `print(dataframe.columns)`，查看可用于绘图的所有特征。

## 设置 `startup_candle_count`

FreqAI 策略中的 `startup_candle_count` 需与标准 Freqtrade 策略设置方式一致（详见[此处](strategy-customization.md#strategy-startup-period)）。该值用于确保在调用 dataprovider 时有足够的数据，避免首次训练时出现 NaN。你可以通过找出传递给指标创建函数（如 TA-Lib 函数）的最大周期（以蜡烛为单位）来设置此值。在示例中，`startup_candle_count` 为 20，因为这是 `indicators_periods_candles` 中的最大值。

:::{note}
有些 TA-Lib 函数实际上需要比传递的 `period` 更多的数据，否则特征集会被 NaN 填充。经验上，将 `startup_candle_count` 乘以 2 总能获得无 NaN 的训练集。因此，通常最安全的做法是将预期的 `startup_candle_count` 乘以 2。你可以通过如下日志确认数据是否干净：

```
2022-08-31 15:14:04 - freqtrade.freqai.data_kitchen - INFO - dropped 0 training points due to NaNs in populated dataset 4319.
```
:::

## 创建动态目标阈值

决定何时入场或出场可以采用动态方式以反映当前市场状况。FreqAI 允许你从模型训练中返回额外信息（详见[此处](freqai-feature-engineering.md#returning-additional-info-from-training)）。例如，`&*_std/mean` 返回值描述了*最近一次训练期间*目标/标签的统计分布。将某一预测与这些值比较，可以了解预测的罕见程度。在 `templates/FreqaiExampleStrategy.py` 中，`target_roi` 和 `sell_roi` 被定义为距离均值 1.25 个标准差，这样更接近均值的预测会被过滤掉。

```python
dataframe["target_roi"] = dataframe["&-s_close_mean"] + dataframe["&-s_close_std"] * 1.25
dataframe["sell_roi"] = dataframe["&-s_close_mean"] - dataframe["&-s_close_std"] * 1.25
```

如果你想用*历史预测*的分布来创建动态目标，而不是用训练信息，可以在配置中设置 `fit_live_predictions_candles` 为你希望用于生成目标统计的历史预测蜡烛数。

```json
    "freqai": {
        "fit_live_predictions_candles": 300,
    }
```

如果设置了该值，FreqAI 会先用训练数据的预测，随后引入实际预测数据。FreqAI 会保存这些历史数据，如果你用相同的 `identifier` 停止并重启模型，这些数据会被重新加载。

## 使用不同的预测模型

FreqAI 已内置多种可直接通过 `--freqaimodel` 标志使用的预测模型库。这些库包括 `CatBoost`、`LightGBM` 和 `XGBoost` 的回归、分类和多目标模型，位于 `freqai/prediction_models/`。

回归模型和分类模型预测的目标不同——回归模型预测连续值（如明天 BTC 的价格），而分类器预测离散值（如明天 BTC 是否会上涨）。因此，你需要根据所用模型类型以不同方式指定目标（详见[下文](#setting-model-targets)）。

上述所有模型库都实现了梯度提升决策树算法。它们都基于集成学习原理，将多个简单学习器的预测组合起来，获得更稳定、泛化能力更强的最终预测。这里的简单学习器是决策树。梯度提升指的是学习方法，每个简单学习器按顺序构建，后一个学习器用于改进前一个的误差。想了解更多可参考各自的官方文档：

* CatBoost: https://catboost.ai/en/docs/
* LightGBM: https://lightgbm.readthedocs.io/en/v3.3.2/#
* XGBoost: https://xgboost.readthedocs.io/en/stable/#

还有许多在线文章对这些算法进行描述和比较。例如 [CatBoost vs. LightGBM vs. XGBoost — 哪个算法最好？](https://towardsdatascience.com/catboost-vs-lightgbm-vs-xgboost-c80f40662924#:~:text=In%20CatBoost%2C%20symmetric%20trees%2C%20or,the%20same%20depth%20can%20differ.) 和 [XGBoost、LightGBM 还是 CatBoost — 我该用哪个提升算法？](https://medium.com/riskified-technology/xgboost-lightgbm-or-catboost-which-boosting-algorithm-should-i-use-e7fda7bb36bc)。请记住，每种模型的性能高度依赖于具体应用，任何报告的指标未必适用于你的场景。

除了 FreqAI 已有的模型，你还可以用 `IFreqaiModel` 类自定义和创建自己的预测模型。建议你重写 `fit()`、`train()` 和 `predict()`，以自定义训练过程的各个方面。你可以将自定义 FreqAI 模型放在 `user_data/freqaimodels`，freqtrade 会根据 `--freqaimodel` 名称（需与自定义模型类名一致）自动加载。请确保使用唯一名称，避免覆盖内置模型。

### 设置模型目标

#### 回归器

如果你用的是回归器，需要指定连续值目标。FreqAI 包含多种回归器，如通过 `--freqaimodel CatboostRegressor` 使用的 `CatboostRegressor`。例如，预测未来 100 根蜡烛的价格目标可这样设置：

```python
df['&s-close_price'] = df['close'].shift(-100)
```

如果你想预测多个目标，需要用上述语法定义多个标签。

#### 分类器

如果你用的是分类器，需要指定离散值目标。FreqAI 包含多种分类器，如通过 `--freqaimodel CatboostClassifier` 使用的 `CatboostClassifier`。如果你选择用分类器，类别需用字符串设置。例如，预测未来 100 根蜡烛价格是涨还是跌：

```python
df['&s-up_or_down'] = np.where( df["close"].shift(-100) > df["close"], 'up', 'down')
```

如果你想预测多个目标，必须在同一标签列中指定所有标签。例如，可以添加 `same` 标签来表示价格未变：

```python
df['&s-up_or_down'] = np.where( df["close"].shift(-100) > df["close"], 'up', 'down')
df['&s-up_or_down'] = np.where( df["close"].shift(-100) == df["close"], 'same', df['&s-up_or_down'])
```

## PyTorch 模块

### 快速开始

最快速运行 pytorch 模型的方法如下（回归任务）：

```bash
freqtrade trade --config config_examples/config_freqai.example.json --strategy FreqaiExampleStrategy --freqaimodel PyTorchMLPRegressor --strategy-path freqtrade/templates 
```

:::{note} 安装/docker
PyTorch 模块需要如 `torch` 这样的大型包，需在 `./setup.sh -i` 时明确选择，在 "Do you also want dependencies for freqai-rl or PyTorch (~700mb additional space required) [y/N]?" 问题时回答 "y"。

喜欢 docker 的用户应确保使用带 `_freqaitorch` 后缀的 docker 镜像。

我们在 `docker/docker-compose-freqai.yml` 中提供了专用的 docker-compose 文件，可通过 `docker compose -f docker/docker-compose-freqai.yml run ...` 使用，或复制替换原有 docker 文件。

该 docker-compose 文件还包含一个（默认禁用的）部分，用于在 docker 容器中启用 GPU 资源。显然，这要求系统本身有可用的 GPU 资源。

PyTorch 从 2.3 版本起不再支持 macOS x64（基于 Intel 的 Apple 设备）。因此 freqtrade 也不再支持该平台上的 PyTorch。
:::

### 结构

#### 模型

你可以通过在自定义 [`IFreqaiModel` 文件](#using-different-prediction-models)中定义 `nn.Module` 类，并在 `def train()` 函数中使用该类，来构建自己的 PyTorch 神经网络架构。以下是用 PyTorch 实现的逻辑回归模型示例（分类任务应配合 nn.BCELoss 损失函数使用）。

```python
class LogisticRegression(nn.Module):
    def __init__(self, input_size: int):
        super().__init__()
        # 定义层
        self.linear = nn.Linear(input_size, 1)
        self.activation = nn.Sigmoid()

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # 前向传播
        out = self.linear(x)
        out = self.activation(out)
        return out

class MyCoolPyTorchClassifier(BasePyTorchClassifier):
    """
    这是一个自定义 IFreqaiModel，展示了用户如何为训练设置自己的神经网络架构。
    """

    @property
    def data_convertor(self) -> PyTorchDataConvertor:
        return DefaultPyTorchDataConvertor(target_tensor_type=torch.float)

    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        """
        用户在此设置训练和测试数据以适配所需模型
        :param data_dictionary: 包含所有训练、测试、标签、权重数据的字典
        :param dk: 当前币种/模型的数据厨房对象
        """

        class_names = self.get_class_names()
        self.convert_label_column_to_int(data_dictionary, dk, class_names)
        n_features = data_dictionary["train_features"].shape[-1]
        model = LogisticRegression(
            input_dim=n_features
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.CrossEntropyLoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            model_meta_data={"class_names": class_names},
            device=self.device,
            init_model=init_model,
            data_convertor=self.data_convertor,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary, self.splits)
        return trainer
```

#### 训练器

`PyTorchModelTrainer` 执行标准的 PyTorch 训练循环：
定义模型、损失函数和优化器，然后将它们移动到合适的设备（GPU 或 CPU）。在循环中，遍历 dataloader 中的批次，将数据移动到设备，计算预测和损失，反向传播，并用优化器更新模型参数。

此外，训练器还负责：
 - 保存和加载模型
 - 将数据从 `pandas.DataFrame` 转换为 `torch.Tensor`

#### 与 Freqai 模块集成

与所有 freqai 模型一样，PyTorch 模型继承自 `IFreqaiModel`。`IFreqaiModel` 声明了三个抽象方法：`train`、`fit` 和 `predict`。我们在三层继承体系中实现这些方法。
从上到下：

1. `BasePyTorchModel` - 实现 `train` 方法。所有 `BasePyTorch*` 继承自它。负责通用数据准备（如数据归一化）并调用 `fit` 方法。设置子类用到的 `device` 属性和父类用到的 `model_type` 属性。
2. `BasePyTorch*` -  实现 `predict` 方法。`*` 代表一组算法，如分类器或回归器。负责数据预处理、预测和必要的后处理。
3. `PyTorch*Classifier` / `PyTorch*Regressor` - 实现 `fit` 方法。负责主要的训练流程，初始化训练器和模型对象。

![image](assets/freqai_pytorch-diagram.png)

#### 完整示例

用 MLP（多层感知机）模型、MSELoss 损失函数和 AdamW 优化器构建 PyTorch 回归器。

```python
class PyTorchMLPRegressor(BasePyTorchRegressor):
    def __init__(self, **kwargs) -> None:
        super().__init__(**kwargs)
        config = self.freqai_info.get("model_training_parameters", {})
        self.learning_rate: float = config.get("learning_rate",  3e-4)
        self.model_kwargs: dict[str, Any] = config.get("model_kwargs",  {})
        self.trainer_kwargs: dict[str, Any] = config.get("trainer_kwargs",  {})

    def fit(self, data_dictionary: dict, dk: FreqaiDataKitchen, **kwargs) -> Any:
        n_features = data_dictionary["train_features"].shape[-1]
        model = PyTorchMLPModel(
            input_dim=n_features,
            output_dim=1,
            **self.model_kwargs
        )
        model.to(self.device)
        optimizer = torch.optim.AdamW(model.parameters(), lr=self.learning_rate)
        criterion = torch.nn.MSELoss()
        init_model = self.get_init_model(dk.pair)
        trainer = PyTorchModelTrainer(
            model=model,
            optimizer=optimizer,
            criterion=criterion,
            device=self.device,
            init_model=init_model,
            target_tensor_type=torch.float,
            **self.trainer_kwargs,
        )
        trainer.fit(data_dictionary)
        return trainer
```

这里我们创建了一个实现 `fit` 方法的 `PyTorchMLPRegressor` 类。`fit` 方法指定了训练的各个组成部分：模型、优化器、损失函数和训练器。我们继承了 `BasePyTorchRegressor` 和 `BasePyTorchModel`，前者实现了适合回归任务的 `predict` 方法，后者实现了训练方法。

:::{note} 设置分类器类别名
使用分类器时，用户必须通过重写 `IFreqaiModel.class_names` 属性声明类别名（或目标）。这可以在 FreqAI 策略的 `set_freqai_targets` 方法中设置 `self.freqai.class_names` 实现。
    
例如，若用二分类器预测价格涨跌，可如下设置类别名：
```python
def set_freqai_targets(self, dataframe: DataFrame, metadata: dict, **kwargs) -> DataFrame:
    self.freqai.class_names = ["down", "up"]
    dataframe['&s-up_or_down'] = np.where(dataframe["close"].shift(-100) >
                                                dataframe["close"], 'up', 'down')

    return dataframe
```
完整示例见 [classifier test strategy class](https://github.com/freqtrade/freqtrade/blob/develop/tests/strategy/strats/freqai_test_classifier.py)。
:::

#### 用 `torch.compile()` 提升性能

Torch 提供了 `torch.compile()` 方法，可用于针对特定 GPU 硬件提升性能。更多细节见 [官方教程](https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html)。简而言之，只需用 `torch.compile()` 包裹你的 `model`：

```python
        model = PyTorchMLPModel(
            input_dim=n_features,
            output_dim=1,
            **self.model_kwargs
        )
        model.to(self.device)
        model = torch.compile(model)
```

然后像平常一样使用模型即可。注意，这样会关闭 eager 执行，错误和回溯信息会变得不够直观。
