---
title: 开发者指南
subject: FreqTrade 开发者指南
subtitle: 为开发者提供的详细指南
short_title: 开发者指南
description: 本文档面向 FreqTrade 的开发者,介绍如何为项目做出贡献、配置开发环境、编写测试以及其他开发相关的重要信息。
---

# 开发帮助

本页面面向 Freqtrade 的开发者、想要为 Freqtrade 代码库或文档做出贡献的人，或者想要了解他们正在运行的应用程序源代码的人。

我们欢迎所有贡献、错误报告、错误修复、文档改进、增强和想法。我们在 [GitHub](https://github.com) 上[跟踪问题](https://github.com/freqtrade/freqtrade/issues)，并且在 [discord](https://discord.gg/p7nuUNVfP7) 上有一个开发频道，你可以在那里提问。

## 文档

文档可在 [https://freqtrade.io](https://www.freqtrade.io/) 获取，每个新功能 PR 都需要提供文档。

文档的特殊字段（如注释框等）可以在[这里](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)找到。

要在本地测试文档，请使用以下命令。

```bash
pip install -r docs/requirements-docs.txt
mkdocs serve
```

这将启动一个本地服务器（通常在端口 8000 上），这样你就可以查看是否一切看起来都符合你的预期。

## 开发者设置

要配置开发环境，你可以使用提供的 [DevContainer](#devcontainer-setup)，或者使用 `setup.sh` 脚本并在询问"Do you want to install dependencies for dev [y/N]? "时回答"y"。
或者（例如，如果你的系统不受 setup.sh 脚本支持），按照手动安装过程并运行 `pip3 install -r requirements-dev.txt` - 然后运行 `pip3 install -e .[all]`。

这将安装开发所需的所有工具，包括 `pytest`、`ruff`、`mypy` 和 `coveralls`。

然后通过运行 `pre-commit install` 安装 git hook 脚本，这样你的更改将在提交前在本地进行验证。
这已经避免了很多等待 CI 的时间，因为一些基本的格式检查是在你的机器上本地完成的。

在打开拉取请求之前，请熟悉我们的[贡献指南](https://github.com/freqtrade/freqtrade/blob/develop/CONTRIBUTING.md)。

### Devcontainer 设置

最快和最简单的方法是使用带有 Remote container 扩展的 [VSCode](https://code.visualstudio.com/)。
这使开发者能够启动机器人，而*不需要*在本地机器上安装任何 freqtrade 特定的依赖项。

#### Devcontainer 依赖项

* [VSCode](https://code.visualstudio.com/)
* [docker](https://docs.docker.com/install/)
* [Remote container 扩展文档](https://code.visualstudio.com/docs/remote)

有关 [Remote container 扩展](https://code.visualstudio.com/docs/remote) 的更多信息，最好查阅文档。

### 测试

新代码应该由基本的单元测试覆盖。根据功能的复杂性，审查者可能会要求更深入的单元测试。
如有必要，Freqtrade 团队可以协助并提供编写良好测试的指导（但请不要期望有人为你编写测试）。

#### 如何运行测试

在根文件夹中使用 `pytest` 运行所有可用的测试用例，并确认你的本地环境设置正确

:::{note} 功能分支
测试应该在 `develop` 和 `stable` 分支上通过。其他分支可能是正在进行的工作，测试可能尚未工作。
:::

#### 在测试中检查日志内容

Freqtrade 使用 2 种主要方法来检查测试中的日志内容，`log_has()` 和 `log_has_re()`（用于使用正则表达式检查，在动态日志消息的情况下）。
这些可以从 `conftest.py` 导入，并可以在任何测试模块中使用。

示例检查如下：

```python
from tests.conftest import log_has, log_has_re

def test_method_to_test(caplog):
    method_to_test()

    assert log_has("This event happened", caplog)
    # 检查带有尾随数字的正则表达式 ...
    assert log_has_re(r"This dynamic event happened and produced \d+", caplog)

```

### 调试配置

要调试 freqtrade，我们推荐使用 VSCode（带有 Python 扩展）和以下启动配置（位于 `.vscode/launch.json`）。
细节显然会因设置而异 - 但这应该可以帮助你开始。

```json
{
    "name": "freqtrade trade",
    "type": "debugpy",
    "request": "launch",
    "module": "freqtrade",
    "console": "integratedTerminal",
    "args": [
        "trade",
        // 可选：
        // "--userdir", "user_data",
        "--strategy", 
        "MyAwesomeStrategy",
    ]
},
```

命令行参数可以在 `"args"` 数组中添加。
此方法也可用于调试策略，通过在策略中设置断点。

Pycharm 也可以使用类似的设置 - 使用 `freqtrade` 作为模块名称，并将命令行参数设置为"parameters"。

:::{tip} 正确使用虚拟环境
使用虚拟环境时（你应该这样做），确保你的编辑器使用正确的虚拟环境，以避免问题或"未知导入"错误。
:::

#### Vscode

你可以使用命令"Python: Select Interpreter"在 VSCode 中选择正确的环境 - 这将显示扩展检测到的环境。
如果你的环境未被检测到，你也可以手动选择路径。

#### Pycharm

在 pycharm 中，你可以在"Run/Debug Configurations"窗口中选择适当的环境。
![Pycharm 调试配置](assets/pycharm_debug.png)

:::{note} 启动目录
这假设你已经检出仓库，并且编辑器在仓库根级别启动（所以 pyproject.toml 在你的仓库的顶层）。
:::

## 错误处理

Freqtrade 异常都继承自 `FreqtradeException`。
但是，这个通用错误类不应该直接使用。相反，存在多个专门的子异常。

以下是异常继承层次结构的概述：

```
+ FreqtradeException
|
+---+ OperationalException
|   |
|   +---+ ConfigurationError
|
+---+ DependencyException
|   |
|   +---+ PricingError
|   |
|   +---+ ExchangeError
|       |
|       +---+ TemporaryError
|       |
|       +---+ DDosProtection
|       |
|       +---+ InvalidOrderException
|           |
|           +---+ RetryableOrderError
|           |
|           +---+ InsufficientFundsError
|
+---+ StrategyError
```

---

## 插件

### 交易对列表

你有一个想要尝试的新交易对选择算法的好主意？太好了。
希望你也想将其贡献回上游。

无论你的动机是什么 - 这应该能让你开始开发一个新的交易对列表处理器。

首先，查看 [VolumePairList](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/plugins/pairlist/VolumePairList.py) 处理器，最好复制这个文件并命名为你的新交易对列表处理器。

这是一个简单的处理器，但它可以作为开始开发的好例子。

接下来，修改处理器的类名（最好与模块文件名保持一致）。

基类提供了交易所实例（`self._exchange`）、交易对列表管理器（`self._pairlistmanager`），以及主配置（`self._config`）、交易对列表专用配置（`self._pairlistconfig`）和交易对列表中的绝对位置。

```python
        self._exchange = exchange
        self._pairlistmanager = pairlistmanager
        self._config = config
        self._pairlistconfig = pairlistconfig
        self._pairlist_pos = pairlist_pos
```

:::{tip}
别忘了在 `constants.py` 中的 `AVAILABLE_PAIRLISTS` 变量下注册你的交易对列表 - 否则它将不可选择。
:::

现在，让我们逐步了解需要操作的方法：

#### 交易对列表配置

交易对列表处理器链的配置在机器人配置文件的 `"pairlists"` 元素中完成，这是链中每个交易对列表处理器的配置参数数组。

按照惯例，`"number_assets"` 用于指定要在交易对列表中保留的最大交易对数量。请遵循这一点以确保一致的用户体验。

可以根据需要配置其他参数。例如，`VolumePairList` 使用 `"sort_key"` 来指定排序值 - 但请随意指定你的优秀算法成功和动态所需的任何内容。

#### short_desc

返回用于 Telegram 消息的描述。

这应该包含交易对列表处理器的名称，以及包含资产数量的简短描述。请遵循格式 `"PairlistName - top/bottom X pairs"`。

#### gen_pairlist

如果交易对列表处理器可以用作链中的主导交易对列表处理器，定义初始交易对列表，然后由链中的所有交易对列表处理器处理，则覆盖此方法。例如 `StaticPairList` 和 `VolumePairList`。

这在机器人的每次迭代中调用（仅当交易对列表处理器位于第一个位置时）- 所以考虑为计算/网络密集型计算实现缓存。

它必须返回结果交易对列表（然后可以传递给交易对列表处理器链）。

验证是可选的，父类提供了 `verify_blacklist(pairlist)` 和 `_whitelist_for_active_markets(pairlist)` 来进行默认过滤。如果你将结果限制为特定数量的交易对，请使用此功能 - 这样最终结果不会比预期的短。

#### filter_pairlist

此方法由交易对列表管理器为链中的每个交易对列表处理器调用。

这在机器人的每次迭代中调用 - 所以考虑为计算/网络密集型计算实现缓存。

它接收一个交易对列表（可以是之前交易对列表的结果）以及 `tickers`，这是 `get_tickers()` 的预获取版本。

基类中的默认实现只是为交易对列表中的每个交易对调用 `_validate_pair()` 方法，但你可以覆盖它。所以你应该在交易对列表处理器中实现 `_validate_pair()` 或覆盖 `filter_pairlist()` 来做其他事情。

如果被覆盖，它必须返回结果交易对列表（然后可以传递给链中的下一个交易对列表处理器）。

验证是可选的，父类提供了 `verify_blacklist(pairlist)` 和 `_whitelist_for_active_markets(pairlist)` 来进行默认过滤。如果你将结果限制为特定数量的交易对，请使用此功能 - 这样最终结果不会比预期的短。

在 `VolumePairList` 中，这实现了不同的排序方法，进行早期验证，因此只返回预期数量的交易对。

##### 示例

```python
    def filter_pairlist(self, pairlist: list[str], tickers: dict) -> List[str]:
        # 生成动态白名单
        pairs = self._calculate_pairlist(pairlist, tickers)
        return pairs
```

### 保护

最好阅读[保护文档](plugins.md#protections)以了解保护。
本指南面向想要开发新保护的开发者。

任何保护都不应该直接使用 datetime，而是使用提供的 `date_now` 变量进行日期计算。这保留了回测保护的能力。

:::{tip} 编写新的保护
最好复制一个现有的保护作为好例子。
:::

#### 实现新的保护

所有保护实现都必须以 `IProtection` 作为父类。
因此，它们必须实现以下方法：

* `short_desc()`
* `global_stop()`
* `stop_per_pair()`。

`global_stop()` 和 `stop_per_pair()` 必须返回一个 ProtectionReturn 对象，它包含：

* lock pair - 布尔值
* lock until - datetime - 交易对应该被锁定到什么时候（将向上取整到下一个新蜡烛）
* reason - 字符串，用于日志记录和数据库存储
* lock_side - long、short 或 '*'。

`until` 部分应该使用提供的 `calculate_lock_end()` 方法计算。

所有保护都应该使用 `"stop_duration"` / `"stop_duration_candles"` 来定义交易对（或所有交易对）应该被锁定多长时间。
这个内容作为 `self._stop_duration` 提供给每个保护。

如果你的保护需要回溯期，请使用 `"lookback_period"` / `"lockback_period_candles"` 以保持所有保护一致。

#### 全局与本地停止

保护可以有 2 种不同的方式来暂时停止交易：

* 每个交易对（本地）
* 所有交易对（全局）

##### 保护 - 每个交易对

实现每个交易对方法的保护必须设置 `has_local_stop=True`。
每当交易关闭（出场订单完成）时，都会调用 `stop_per_pair()` 方法。

##### 保护 - 全局保护

这些保护应该对所有交易对进行评估，因此也会锁定所有交易对的交易（称为全局交易对锁定）。
全局保护必须设置 `has_global_stop=True` 才能被评估为全局停止。
每当交易关闭（出场订单完成）时，都会调用 `global_stop()` 方法。

##### 保护 - 计算锁定结束时间

保护应该根据它考虑的最后一个交易计算锁定结束时间。
这避免了在回溯期长于实际锁定期时重新锁定。

`IProtection` 父类在 `calculate_lock_end()` 中为此提供了一个辅助方法。

---

## 实现新的交易所（进行中）

:::{note}
本节正在进行中，不是关于如何使用 Freqtrade 测试新交易所的完整指南。
:::

:::{note}
在运行以下任何测试之前，请确保使用最新版本的 CCXT。

你可以通过在激活的虚拟环境中运行 `pip install -U ccxt` 来获取最新版本的 ccxt。

这些测试不支持原生 docker，但是可用的 dev-container 将支持所有必需的操作和最终必要的更改。
:::

大多数 CCXT 支持的交易所应该可以直接工作。

要快速测试交易所的公共端点，将你的交易所配置添加到 `tests/exchange_online/conftest.py` 并使用 `pytest --longrun tests/exchange_online/test_ccxt_compat.py` 运行这些测试。
成功完成这些测试是一个良好的基础点（实际上这是一个要求），但这些并不能保证交易所功能正确，因为这只测试公共端点，而不是私有端点（如生成订单或类似操作）。

还要尝试使用 `freqtrade download-data` 下载扩展时间范围（几个月）的数据，并验证数据是否正确下载（没有漏洞，实际下载了指定的时间范围）。

这些是将交易所列为支持或社区测试（在主页上列出）的先决条件。
以下是"额外"内容，这将使交易所更好（功能完整）- 但对于这两个类别中的任何一个都不是绝对必要的。

要完成的额外测试/步骤：

* 验证 `fetch_ohlcv()` 提供的数据 - 并最终为此交易所调整 `ohlcv_candle_limit`
* 检查 L2 订单簿限制范围（API 文档）- 并根据需要设置
* 检查余额是否正确显示（*）
* 创建市价单（*）
* 创建限价单（*）
* 取消订单（*）
* 完成交易（入场 + 出场）（*）
  * 比较交易所和机器人之间的结果计算
  * 确保正确应用费用（根据交易所检查数据库）

（*）需要 API 密钥和交易所余额。

### 交易所止损

检查新交易所是否通过其 API 支持交易所止损订单。

由于 CCXT 尚未为交易所止损提供统一，我们需要自己实现交易所特定的参数。最好查看 `binance.py` 作为此实现的示例。你需要深入研究交易所 API 的文档，了解如何做到这一点。[CCXT Issues](https://github.com/ccxt/ccxt/issues) 也可能提供很大帮助，因为其他人可能已经为他们的项目实现了类似的东西。

### 不完整的蜡烛

在获取蜡烛（OHLCV）数据时，我们可能会得到不完整的蜡烛（取决于交易所）。
为了演示这一点，我们将使用日线蜡烛（`"1d"`）来保持简单。
我们查询 api（`ct.fetch_ohlcv()`）获取时间框架，并查看最后一个条目的日期。如果这个条目发生变化或显示"不完整"蜡烛的日期，那么我们应该删除它，因为有不完整的蜡烛是有问题的，因为指标假设只传递完整的蜡烛给它们，并且会生成大量错误的买入信号。因此，默认情况下，我们假设最后一个蜡烛不完整并将其删除。

要检查新交易所的行为，你可以使用以下代码片段：

```python
import ccxt
from datetime import datetime, timezone
from freqtrade.data.converter import ohlcv_to_dataframe
ct = ccxt.binance()  # 使用你正在测试的交易所
timeframe = "1d"
pair = "BTC/USDT"  # 确保使用在该交易所存在的交易对！
raw = ct.fetch_ohlcv(pair, timeframe=timeframe)

# 转换为数据框
df1 = ohlcv_to_dataframe(raw, timeframe, pair=pair, drop_incomplete=False)

print(df1.tail(1))
print(datetime.now(timezone.utc))
```

``` output
                         date      open      high       low     close  volume  
499 2019-06-08 00:00:00+00:00  0.000007  0.000007  0.000007  0.000007   26264344.0  
2019-06-09 12:30:27.873327
```

输出将显示交易所的最后一个条目以及当前 UTC 日期。
如果日期显示同一天，那么最后一个蜡烛可以被认为是不完整的，应该被删除（保持交易所类中的 `"ohlcv_partial_candle"` 设置不变 / True）。否则，将 `"ohlcv_partial_candle"` 设置为 `False` 以不删除蜡烛（如上例所示）。
另一种方法是连续多次运行此命令，观察成交量是否变化（而日期保持不变）。

### 更新币安缓存的杠杆层级

更新杠杆层级应该定期进行 - 并且需要一个启用了期货的认证账户。

```python
import ccxt
import json
from pathlib import Path

exchange = ccxt.binance({
    'apiKey': '<apikey>',
    'secret': '<secret>',
    'options': {'defaultType': 'swap'}
    })
_ = exchange.load_markets()

lev_tiers = exchange.fetch_leverage_tiers()

# 假设这是在仓库的根目录运行。
file = Path('freqtrade/exchange/binance_leverage_tiers.json')
json.dump(dict(sorted(lev_tiers.items())), file.open('w'), indent=2)

```

然后应该将此文件贡献给上游，这样其他人也可以从中受益。

## 更新示例笔记本

为了保持 jupyter 笔记本与文档一致，在更新示例笔记本后应该运行以下命令。

```bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace freqtrade/templates/strategy_analysis_example.ipynb
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --to markdown freqtrade/templates/strategy_analysis_example.ipynb --stdout > docs/strategy_analysis_example.md
```

## 回测文档结果

要生成回测输出，请使用以下命令：

``` bash
# 假设为此输出创建专用的用户目录
freqtrade create-userdir --userdir user_data_bttest/
# 设置 can_short = True
sed -i "s/can_short: bool = False/can_short: bool = True/" user_data_bttest/strategies/sample_strategy.py

freqtrade download-data --timerange 20250625-20250801 --config tests/testdata/config.tests.usdt.json --userdir user_data_bttest/ -t 5m

freqtrade backtesting --config tests/testdata/config.tests.usdt.json -s SampleStrategy --userdir user_data_bttest/ --cache none --timerange 20250701-20250801
```

## 持续集成

本文档记录了一些为 CI 流水线做出的决定。

* CI 在所有操作系统变体上运行，Linux（ubuntu）、macOS 和 Windows。
* Docker 镜像为 `stable` 和 `develop` 分支构建，并作为多架构构建构建，通过同一标签支持多个平台。
* 包含绘图依赖项的 Docker 镜像也可用，作为 `stable_plot` 和 `develop_plot`。
* Docker 镜像包含一个文件 `/freqtrade/freqtrade_commit`，其中包含此镜像所基于的提交。
* 完整的 docker 镜像重建每周通过计划运行一次。
* 部署在 ubuntu 上运行。
* 所有测试必须通过才能将 PR 合并到 `stable` 或 `develop`。

## 创建发布

本文档的这一部分面向维护者，并展示了如何创建发布。

### 创建发布分支

:::{note}
确保 `stable` 分支是最新的！
:::

首先，选择一个大约一周前的提交（以不包括最新添加到发布中的内容）。

```bash
# 创建新分支
git checkout -b new_release <commitid>
```

确定在此提交和当前状态之间是否进行了关键的错误修复，并最终选择这些修复。

* 将发布分支（stable）合并到此分支。
* 编辑 `freqtrade/__init__.py` 并添加与当前日期匹配的版本（例如 `2019.7` 表示 2019 年 7 月）。次要版本可以是 `2019.7.1`，如果我们需要在该月进行第二次发布。版本号必须遵循 PEP0440 允许的版本，以避免推送到 pypi 时失败。
* 提交这部分。
* 将该分支推送到远程，并创建一个针对**stable 分支**的 PR。
* 将 develop 版本更新为下一个版本，遵循模式 `2019.8-dev`。

### 从 git 提交创建更新日志

```bash
# 需要在合并/拉取该分支之前完成。
git log --oneline --no-decorate --no-merges stable..new_release
```

为了保持发布日志简短，最好将完整的 git 更新日志包装在一个可折叠的详细信息部分中。

```markdown
<details>
<summary>展开完整更新日志</summary>

... 完整的 git 更新日志

</details>
```

### FreqUI 发布

如果 FreqUI 已大幅更新，确保在合并发布分支之前创建发布。
确保在合并发布之前，发布上的 freqUI CI 已完成并通过。

### 创建 github 发布/标签

一旦针对 stable 的 PR 被合并（最好在合并后立即）：

* 在 Github UI 中使用"起草新发布"按钮（发布子部分）。
* 使用指定的版本号作为标签。
* 使用"stable"作为参考（这一步在合并上述 PR 之后）。
* 使用上述更新日志作为发布评论（作为代码块）。
* 使用以下代码片段作为新发布

:::{tip} 发布模板
```{include} includes/release_template.md
````
:::

## 发布

### pypi

:::{warning} 手动发布
此过程作为 Github Actions 的一部分自动化。
不应该需要手动 pypi 推送。
:::

:::{tip} 手动发布
要手动创建 pypi 发布，请运行以下命令：

额外要求：`wheel`、`twine`（用于上传）、具有适当权限的 pypi 账户。

```bash
pip install -U build
python -m build --sdist --wheel

# 对于 pypi 测试（检查安装的某些更改是否有效）
twine upload --repository-url https://test.pypi.org/legacy/ dist/*

# 对于生产：
twine upload dist/*
```

请不要将非发布版本推送到生产/真实的 pypi 实例。
:::