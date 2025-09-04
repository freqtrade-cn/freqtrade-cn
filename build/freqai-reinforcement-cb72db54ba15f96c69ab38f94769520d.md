---
title: FreqAI 强化学习指南
subject: FreqAI 强化学习文档
subtitle: 强化学习模型的详细说明
short_title: 强化学习
description: 本文档详细介绍了 FreqAI 的强化学习功能,包括基本概念、运行方法、环境配置等内容。这些功能可以帮助用户构建和训练强化学习模型进行自动交易。
---

# 强化学习

:::{note} 安装体积
强化学习依赖项包含如 `torch` 这样的大型包，需在执行 `./setup.sh -i` 时，在"Do you also want dependencies for freqai-rl (~700mb additional space required) [y/N]?”问题上选择"y"以显式安装。

喜欢使用 docker 的用户应确保使用带有 `_freqairl` 后缀的 docker 镜像。
:::

## 背景与术语

### 什么是 RL，FreqAI 为什么需要它？

强化学习涉及两个重要组成部分：*智能体（agent）*和训练*环境（environment）*。在智能体训练期间，智能体逐根遍历历史蜡烛数据，每次做出一组动作中的一个：多头开仓、多头平仓、空头开仓、空头平仓、中立。在此训练过程中，环境会跟踪这些动作的表现，并根据自定义的 `calculate_reward()`（我们为用户提供了一个默认奖励函数，详情见[此处](#creating-a-custom-reward-function)）对智能体进行奖励。奖励用于训练神经网络中的权重。

FreqAI RL 实现的另一个重要组成部分是*状态（state）*信息的使用。每一步都会将状态信息（如当前利润、当前持仓、当前交易持续时间）输入网络。这些信息用于训练环境中的智能体，并在 dry/live 时强化智能体（此功能在回测中不可用）。*FreqAI + Freqtrade 是这种强化机制的完美结合，因为这些信息在实时部署中随时可用。*

强化学习是 FreqAI 的自然进化，因为它为市场自适应和反应性增加了新的层次，这是分类器和回归器无法比拟的。然而，分类器和回归器也有 RL 不具备的优势，比如稳健的预测。训练不当的 RL 智能体可能会找到"漏洞"或"技巧"来最大化奖励，但实际上并未获得任何交易收益。因此，RL 更加复杂，需要比典型分类器和回归器更高的理解水平。

### RL 接口

在当前框架下，我们旨在通过通用的"预测模型"文件暴露训练环境，该文件是用户继承的 `BaseReinforcementLearner` 对象（如 `freqai/prediction_models/ReinforcementLearner`）。在此用户类中，RL 环境可通过 `MyRLEnv` 进行自定义（见[下文](#creating-a-custom-reward-function)）。

我们设想大多数用户会将精力集中在创造性设计 `calculate_reward()` 函数（详情见[此处](#creating-a-custom-reward-function)），而对环境的其他部分保持不变。其他用户甚至不会修改环境，只会调整配置和 FreqAI 已有的强大特征工程。与此同时，我们也允许高级用户完全自定义自己的模型类。

该框架基于 stable_baselines3（torch）和 OpenAI gym 构建基础环境类。但总体而言，模型类隔离良好，因此可以轻松集成其他竞争库。环境继承自 `gym.Env`，因此如需切换到其他库，需编写全新的环境。

### 重要注意事项

如上所述，智能体在一个"人工"交易"环境"中被"训练"。在我们的案例中，这个环境看起来与真实的 Freqtrade 回测环境很相似，但它*不是*。实际上，RL 训练环境要简单得多。它不包含任何复杂的策略逻辑，如 `custom_exit`、`custom_stoploss`、杠杆控制等回调。RL 环境是对真实市场的非常"原始"表示，智能体可以自由学习由 `calculate_reward()` 强制执行的策略（如止损、止盈等）。因此，必须注意，智能体训练环境并不等同于真实世界。

## 运行强化学习

设置和运行强化学习模型与运行回归器或分类器相同。命令行上必须定义同样的两个参数：`--freqaimodel` 和 `--strategy`：

```bash
freqtrade trade --freqaimodel ReinforcementLearner --strategy MyRLStrategy --config config.json
```

其中 `ReinforcementLearner` 将使用 `freqai/prediction_models/ReinforcementLearner` 中的模板类（或 `user_data/freqaimodels` 下的自定义类）。策略部分则与典型回归器一样，采用[特征工程](freqai-feature-engineering.md)的 `feature_engineering_*`。不同之处在于目标的创建，强化学习不需要目标标签。然而，FreqAI 要求在动作列中设置一个默认（中立）值：

```python
    def set_freqai_targets(self, dataframe, **kwargs) -> DataFrame:
        """
        *仅适用于启用 FreqAI 的策略*
        设置模型目标的必需函数。
        所有目标必须以 `&` 开头，以便 FreqAI 内部识别。

        更多特征工程细节见：

        https://www.freqtrade.io/en/latest/freqai-feature-engineering

        :param df: 将接收目标的策略数据框
        使用示例：dataframe["&-target"] = dataframe["close"].shift(-1) / dataframe["close"]
        """
        # 对于 RL，无需设置直接目标。此处为占位（中立），直到智能体发送动作。
        dataframe["&-action"] = 0
        return dataframe
```

大部分函数与典型回归器相同，但下方函数展示了策略如何将原始价格数据传递给智能体，以便其在训练环境中访问原始 OHLCV：

```python
    def feature_engineering_standard(self, dataframe: DataFrame, **kwargs) -> DataFrame:
        # 以下特征对 RL 模型是必要的
        dataframe[f"%-raw_close"] = dataframe["close"]
        dataframe[f"%-raw_open"] = dataframe["open"]
        dataframe[f"%-raw_high"] = dataframe["high"]
        dataframe[f"%-raw_low"] = dataframe["low"]
    return dataframe
```

最后，没有显式的"标签"需要设置——而是需要分配 `&-action` 列，该列在 `populate_entry/exit_trends()` 中被访问时包含智能体的动作。在本例中，中立动作为 0。此值应与所用环境一致。FreqAI 提供的两个环境均以 0 作为中立动作。

用户会很快意识到无需设置标签，智能体会"自主"做出进出场决策。这使得策略构建变得相当简单。进出场信号由智能体以整数形式给出，直接用于策略中的进出场判断：

```python
    def populate_entry_trend(self, df: DataFrame, metadata: dict) -> DataFrame:

        enter_long_conditions = [df["do_predict"] == 1, df["&-action"] == 1]

        if enter_long_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_long_conditions), ["enter_long", "enter_tag"]
            ] = (1, "long")

        enter_short_conditions = [df["do_predict"] == 1, df["&-action"] == 3]

        if enter_short_conditions:
            df.loc[
                reduce(lambda x, y: x & y, enter_short_conditions), ["enter_short", "enter_tag"]
            ] = (1, "short")

        return df

    def populate_exit_trend(self, df: DataFrame, metadata: dict) -> DataFrame:
        exit_long_conditions = [df["do_predict"] == 1, df["&-action"] == 2]
        if exit_long_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_long_conditions), "exit_long"] = 1

        exit_short_conditions = [df["do_predict"] == 1, df["&-action"] == 4]
        if exit_short_conditions:
            df.loc[reduce(lambda x, y: x & y, exit_short_conditions), "exit_short"] = 1

        return df
```

需要注意的是，`&-action` 取决于所选环境。上述示例展示了 5 个动作，其中 0 为中立，1 为多头开仓，2 为多头平仓，3 为空头开仓，4 为空头平仓。

## 配置强化学习器

要配置 `Reinforcement Learner`，需在 `freqai` 配置中包含如下字典：

```json
        "rl_config": {
            "train_cycles": 25,
            "add_state_info": true,
            "max_trade_duration_candles": 300,
            "max_training_drawdown_pct": 0.02,
            "cpu_count": 8,
            "model_type": "PPO",
            "policy_type": "MlpPolicy",
            "model_reward_parameters": {
                "rr": 1,
                "profit_aim": 0.025
            }
        }
```

参数详情见[参数表](freqai-parameter-table.md)。一般来说，`train_cycles` 决定智能体在其人工环境中遍历蜡烛数据以训练模型权重的次数。`model_type` 是一个字符串，选择 [stable_baselines](https://stable-baselines3.readthedocs.io/en/master/)（外部链接）中可用的模型之一。

:::{note}
如果你想尝试 `continual_learning`，应在主 `freqai` 配置字典中将其设为 `true`。这会让强化学习库在每次再训练时，从前一模型的最终状态继续训练新模型，而不是每次都从头训练。
:::

:::{note}
请记住，通用的 `model_training_parameters` 字典应包含特定 `model_type` 的所有模型超参数。例如，`PPO` 参数见[此处](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html)。
:::

## 创建自定义奖励函数

:::{danger} 不适用于生产环境
警告！

Freqtrade 源码中提供的奖励函数仅用于展示功能，旨在展示/测试尽可能多的环境控制特性，并在小型计算机上快速运行。它是基准，不适用于生产环境。请注意，你需要自己创建 custom_reward() 函数，或使用其他用户在 Freqtrade 源码之外构建的模板。
:::

当你开始修改策略和预测模型时，会很快发现强化学习器与回归器/分类器有一些重要区别。首先，策略不设置目标值（无标签！）。相反，你需要在 `MyRLEnv` 类中设置 `calculate_reward()` 函数（见下文）。`prediction_models/ReinforcementLearner.py` 中提供了一个默认的 `calculate_reward()`，用于演示奖励构建的必要模块，但*不适用于生产*。用户*必须*创建自己的自定义强化学习模型类，或使用 Freqtrade 源码之外的预构建模型并保存到 `user_data/freqaimodels`。在 `calculate_reward()` 中可以表达你对市场的创造性理论。例如，你可以在智能体获利时奖励它，亏损时惩罚它，或奖励其开仓、惩罚其持仓过久。下方展示了这些奖励的计算方式：

:::{note} 提示
最好的奖励函数是连续可微且缩放良好的。换句话说，对罕见事件施加一次性大负惩罚不是好主意，神经网络无法学会这种函数。更好的做法是对常见事件施加小负惩罚，这有助于智能体更快学习。你还可以通过让奖励/惩罚随某些线性/指数函数按严重程度缩放来提升连续性。例如，随着持仓时间增加，逐步增加惩罚，这比在某一时刻一次性施加大惩罚更好。
:::

```python
from freqtrade.freqai.prediction_models.ReinforcementLearner import ReinforcementLearner
from freqtrade.freqai.RL.Base5ActionRLEnv import Actions, Base5ActionRLEnv, Positions


class MyCoolRLModel(ReinforcementLearner):
    """
    用户自定义 RL 预测模型。

    将此文件保存到 `freqtrade/user_data/freqaimodels`

    然后用如下命令调用：

    freqtrade trade --freqaimodel MyCoolRLModel --config config.json --strategy SomeCoolStrat

    用户可重写 `IFreqaiModel` 继承树中的任意函数。对 RL 来说，最重要的是在此重写 `MyRLEnv`（见下文），以定义自定义 `calculate_reward()`，或重写环境的其他部分。

    此类还允许用户重写 IFreqaiModel 树的其他部分。例如，可以重写 `def fit()`、`def train()` 或 `def predict()` 以精细控制这些过程。

    另一个常见重写是 `def data_cleaning_predict()`，用于精细控制数据处理管道。
    """
    class MyRLEnv(Base5ActionRLEnv):
        """
        用户自定义环境。此类继承自 BaseEnvironment 和 gym.Env。
        用户可重写父类的任意函数。以下为自定义 `calculate_reward()` 的示例。

        警告！
        此函数仅用于展示功能，旨在展示尽可能多的环境控制特性，并在小型计算机上快速运行。它是基准，不适用于生产环境。
        """
        def calculate_reward(self, action: int) -> float:
            # 首先，若动作无效则惩罚
            if not self._is_valid(action):
                return -2
            pnl = self.get_unrealized_profit()

            factor = 100

            pair = self.pair.replace(':', '')

            # 可使用 dataframe 中的特征值
            # 假设策略中已生成移位的 RSI 指标。
            rsi_now = self.raw_features[f"%-rsi-period_10_shift-1_{pair}_"
                            f"{self.config['timeframe']}"]\
                            .iloc[self._current_tick]

            # 奖励智能体开仓
            if (action in (Actions.Long_enter.value, Actions.Short_enter.value)
                    and self._position == Positions.Neutral):
                if rsi_now < 40:
                    factor = 40 / rsi_now
                else:
                    factor = 1
                return 25 * factor

            # 惩罚智能体未开仓
            if action == Actions.Neutral.value and self._position == Positions.Neutral:
                return -1
            max_trade_duration = self.rl_config.get('max_trade_duration_candles', 300)
            trade_duration = self._current_tick - self._last_trade_tick
            if trade_duration <= max_trade_duration:
                factor *= 1.5
            elif trade_duration > max_trade_duration:
                factor *= 0.5
            # 惩罚持仓不动
            if self._position in (Positions.Short, Positions.Long) and \
            action == Actions.Neutral.value:
                return -1 * trade_duration / max_trade_duration
            # 多头平仓
            if action == Actions.Long_exit.value and self._position == Positions.Long:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            # 空头平仓
            if action == Actions.Short_exit.value and self._position == Positions.Short:
                if pnl > self.profit_aim * self.rr:
                    factor *= self.rl_config['model_reward_parameters'].get('win_reward_factor', 2)
                return float(pnl * factor)
            return 0.
```

## 使用 Tensorboard

强化学习模型受益于训练指标的跟踪。FreqAI 集成了 Tensorboard，允许用户跨所有币种和所有再训练过程跟踪训练和评估表现。可通过以下命令激活 Tensorboard：

```bash
tensorboard --logdir user_data/models/unique-id
```

其中 `unique-id` 是 `freqai` 配置文件中设置的 `identifier`。该命令需在单独的 shell 中运行，浏览器访问 127.0.0.1:6006（6006 为 Tensorboard 默认端口）查看输出。

![tensorboard](assets/tensorboard.jpg)

## 自定义日志

FreqAI 还内置了一个名为 `self.tensorboard_log` 的分集摘要日志器，用于将自定义信息添加到 Tensorboard 日志。默认情况下，该函数在环境内每步调用一次以记录智能体动作。单集内所有步的值在每集结束时汇总报告，然后所有指标重置为 0，为下一个 episode 做准备。

`self.tensorboard_log` 也可在环境内任意位置使用，例如可在 `calculate_reward` 函数中添加，以收集奖励各部分被调用的频率：

```python
    class MyRLEnv(Base5ActionRLEnv):
        """
        用户自定义环境。此类继承自 BaseEnvironment 和 gym.Env。
        用户可重写父类的任意函数。以下为自定义 `calculate_reward()` 的示例。
        """
        def calculate_reward(self, action: int) -> float:
            if not self._is_valid(action):
                self.tensorboard_log("invalid")
                return -2

```

:::{note}
`self.tensorboard_log()` 仅用于跟踪计数对象（如事件、动作）。如需记录浮点数，可作为第二参数传入，如 `self.tensorboard_log("float_metric1", 0.23)`。此时指标值不会累加。
:::

## 选择基础环境

FreqAI 提供三种基础环境：`Base3ActionRLEnvironment`、`Base4ActionEnvironment` 和 `Base5ActionEnvironment`。顾名思义，这些环境分别适用于可选择 3、4、5 种动作的智能体。`Base3ActionEnvironment` 最简单，智能体可选择持有、多头或空头。该环境也可用于仅做多的机器人（自动跟随策略的 `can_short` 标志），其中多头为开仓，空头为平仓。`Base4ActionEnvironment` 中，智能体可多头开仓、空头开仓、中立持有或平仓。`Base5ActionEnvironment` 则与 Base4 类似，但将平仓动作区分为多头平仓和空头平仓。环境选择的主要影响包括：

* `calculate_reward` 中可用的动作
* 用户策略中消费的动作

FreqAI 提供的所有环境均继承自动作/持仓无关的 `BaseEnvironment`，该对象包含所有共享逻辑。架构设计易于自定义。最简单的自定义是 `calculate_reward()`（详见[此处](#creating-a-custom-reward-function)）。当然，也可进一步扩展环境内的任意函数，只需在预测模型文件的 `MyRLEnv` 中重写这些函数即可。更高级的自定义建议直接继承 `BaseEnvironment` 创建全新环境。

:::{note}
仅 `Base3ActionRLEnv` 支持仅做多训练/交易（将用户策略属性 `can_short = False`）。
:::
