---
title: 使用 Jupyter 笔记本分析机器人数据
description: 使用 Jupyter 笔记本分析 Freqtrade 机器人的回测结果和交易历史数据
---

# 使用 Jupyter 笔记本分析机器人数据

你可以使用 Jupyter 笔记本轻松分析回测结果和交易历史。初始化用户目录后，示例笔记本位于 `user_data/notebooks/`，可通过命令 `freqtrade create-userdir --userdir user_data` 创建。

## 使用 Docker 快速开始

Freqtrade 提供了一个 docker-compose 文件，可以启动 jupyter lab 服务器。
你可以通过以下命令运行该服务器：`docker compose -f docker/docker-compose-jupyter.yml up`

这将创建一个运行 jupyter lab 的 docker 容器，可通过 `https://127.0.0.1:8888/lab` 访问。
请使用启动后控制台中打印的链接进行简化登录。

更多信息请参见 [使用 Docker 进行数据分析](docker_quickstart.md#data-analysis-using-docker-compose) 部分。

### 专业提示

* 参见 [jupyter.org](https://jupyter.org/documentation) 获取使用说明。
* 不要忘记在 conda 或 venv 环境中启动 Jupyter notebook 服务器，或使用 [nb_conda_kernels](https://github.com/Anaconda-Platform/nb_conda_kernels)*
* 在使用前请复制示例笔记本，这样你的更改不会在下次 freqtrade 更新时被覆盖。

### 使用系统级 Jupyter 安装的虚拟环境

有时你可能希望使用系统范围安装的 Jupyter notebook，并使用虚拟环境中的 jupyter 内核。
这样可以避免在系统中多次安装完整的 jupyter 套件，并便于在不同任务（freqtrade/其他分析任务）间切换。

为此，首先激活你的虚拟环境并运行以下命令：

```bash
# 激活虚拟环境
source .venv/bin/activate

pip install ipykernel
ipython kernel install --user --name=freqtrade
# 重启 jupyter（lab / notebook）
# 在 notebook 中选择内核 "freqtrade"
```

:::{note}
本节仅为完整性提供，Freqtrade 团队不会为此类设置提供完整支持，并建议直接在虚拟环境中安装 Jupyter，因为这是启动 jupyter 笔记本最简单的方式。如需帮助，请参阅 [Project Jupyter](https://jupyter.org/) 的 [文档](https://jupyter.org/documentation) 或 [帮助渠道](https://jupyter.org/community)。
:::

:::{warning}
某些任务在笔记本中并不适用。例如，任何使用异步执行的内容在 Jupyter 中都可能有问题。此外，freqtrade 的主要入口是 shell cli，因此在笔记本中直接用纯 python 运行会绕过一些为辅助函数提供必要对象和参数的参数。你可能需要手动设置这些值或创建所需对象。
:::

## 推荐工作流

| 任务 | 工具 |
  --- | ---
机器人操作 | CLI
重复性任务 | Shell 脚本
数据分析与可视化 | Notebook

1. 使用 CLI 进行：

    * 下载历史数据
    * 运行回测
    * 使用实时数据运行
    * 导出结果

2. 将这些操作收集到 shell 脚本中：

    * 保存带参数的复杂命令
    * 执行多步操作
    * 自动化测试策略和准备分析数据

3. 使用 notebook 进行：

    * 数据可视化
    * 数据处理和绘图以获得洞见

## 示例实用代码片段

### 切换目录到项目根目录

Jupyter 笔记本默认在 notebook 目录下执行。以下代码片段可搜索项目根目录，使相对路径保持一致。

```python
import os
from pathlib import Path

# 切换目录
# 修改此单元格以确保输出显示正确路径。
# 所有路径均应相对于单元格输出显示的项目根目录定义
project_root = "somedir/freqtrade"
i=0
try:
    os.chdir(project_root)
    assert Path('LICENSE').is_file()
except:
    while i<4 and (not Path('LICENSE').is_file()):
        os.chdir(Path(Path.cwd(), '../'))
        i+=1
    project_root = Path.cwd()
print(Path.cwd())
```

### 加载多个配置文件

此方法可用于检查传入多个配置文件的结果。
这也会完整运行配置初始化，因此配置会被完全初始化，可传递给其他方法。

```python
import json
from freqtrade.configuration import Configuration

# 从多个文件加载配置
config = Configuration.from_files(["config1.json", "config2.json"])

# 显示内存中的配置
print(json.dumps(config['original_config'], indent=2))
```

对于交互式环境，建议额外指定 `user_data_dir` 并最后传入，这样你无需在运行机器人时切换目录。
最好避免使用相对路径，因为 notebook 的存储位置就是起始目录，除非你已切换目录。

```json
{
    "user_data_dir": "~/.freqtrade/"
}
```

### 更多数据分析文档

* [策略调试](strategy_analysis_example.md) - 也可作为 Jupyter 笔记本使用（`user_data/notebooks/strategy_analysis_example.ipynb`）
* [绘图](plotting.md)
* [标签分析](advanced-backtesting.md)

如果你有更好的数据分析方法，欢迎提交 issue 或 Pull Request 来完善本文档。