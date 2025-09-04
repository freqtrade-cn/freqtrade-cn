---
title: 安装 Freqtrade
description: 了解如何安装和配置 Freqtrade 交易机器人,包括脚本安装、手动安装和 Conda 安装等多种方式
subject: Freqtrade 安装文档
subtitle: 完整的安装和配置指南
short_title: 安装
tags: [安装, 配置, 指南, 入门]
categories: [基础教程]
prerequisites: [Python 3.10+, Git]
difficulty: beginner
estimated_time: 30分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 2
toc: true
---

# 安装

本页介绍如何为运行机器人准备环境。

freqtrade 文档描述了多种安装 freqtrade 的方式：

* [Docker 镜像](docker_quickstart.md)
* [脚本安装](#script-installation)
* [手动安装](#manual-installation)
* [Conda 安装](#installation-with-conda)

建议在评估 freqtrade 工作原理时，优先使用预构建的 [Docker 镜像](docker_quickstart.md) 快速开始。

------

## 信息

Windows 安装请参考 [Windows 安装指南](windows_installation.md)。

最简单的安装和运行 Freqtrade 的方式是克隆 Github 仓库，然后运行适用于你平台的 `./setup.sh` 脚本。

:::{important} 版本说明
克隆仓库时，默认分支为 `develop`。该分支包含所有最新特性（由于自动化测试，可认为相对稳定）。

`stable` 分支包含最新发布的代码（通常每月发布一次，基于 `develop` 分支约一周前的快照，以避免打包 bug，因此可能更稳定）。
:::

:::{hint} 安装前提
假定已安装 Python3.10 或更高版本及相应的 `pip`。

如果未满足，安装脚本会警告并停止。还需安装 `git` 以克隆 Freqtrade 仓库。  

此外，必须有 python 头文件（`python<yourversion>-dev` / `python<yourversion>-devel`）才能顺利完成安装。
:::

:::{warning} 时钟需同步
运行机器人的系统时钟必须准确，并频繁与 NTP 服务器同步，以避免与交易所通信时出现问题。
:::

------
(requirements)=

## 依赖要求

这些依赖适用于 [脚本安装](#script-installation) 和 [手动安装](#manual-installation)。

:::{note} ARM64 系统
如果你运行在 ARM64 系统（如 MacOS M1 或 Oracle VM），请使用 [docker](docker_quickstart.md) 运行 freqtrade。

虽然原生安装也可行，但目前不受支持。
:::

### 安装指南

* [Python >= 3.10](http://docs.python-guide.org/en/latest/starting/installation/)
* [pip](https://pip.pypa.io/en/stable/installing/)
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [virtualenv](https://virtualenv.pypa.io/en/stable/installation.html)（推荐）

### 安装代码

我们为 Ubuntu、MacOS 和 Windows 收集/整理了安装说明。这些仅供参考，其他发行版可能略有不同。

操作系统相关步骤在前，通用部分适用于所有系统。

:::{note} 务必前提
假定已安装 Python3.10 或更高版本及相应 pip。
:::

::::{tab-set}
:::{tab-item} Debian/Ubuntu
:sync: tab1

#### Linux 安装必要依赖

```bash
# 更新仓库
sudo apt-get update

# 安装软件包
sudo apt install -y python3-pip python3-venv python3-dev python3-pandas git curl
```

:::

:::{tab-item} MacOS
:sync: tab2

#### Mac 安装必要依赖

如果尚未安装 [Homebrew](https://brew.sh/)，请先安装。

```bash
# 安装软件包
brew install gettext libomp
```

`setup.sh` 脚本会自动为你安装这些依赖（前提是已安装 brew）。
:::

:::{tab-item} RaspberryPi/Raspbian
:sync: tab3
以下假定使用最新的 [Raspbian Buster lite 镜像](https://www.raspberrypi.org/downloads/raspbian/)。
该镜像自带 python3.11，便于快速部署 freqtrade。

在 Raspberry Pi 3 + Raspbian Buster lite 镜像 + 所有更新下测试通过。

```bash
sudo apt-get install python3-venv libatlas-base-dev cmake curl libffi-dev
# 使用 piwheels.org 加速安装
sudo echo "[global]\nextra-index-url=https://www.piwheels.org/simple" > tee /etc/pip.conf

git clone https://github.com/freqtrade/freqtrade.git
cd freqtrade

bash setup.sh -i
```

> 安装耗时

取决于网络速度和树莓派型号，安装可能需要数小时。

因此建议树莓派用户使用预构建的 docker 镜像，详见 [Docker 快速入门文档](docker_quickstart.md)

上述方法未安装 hyperopt 依赖。如需安装，请运行 `python3 -m pip install -e .[hyperopt]`。

不建议在树莓派上运行 hyperopt，因为该操作非常消耗资源，建议在性能更强的机器上运行。
:::
::::

------

(freqtrade-repository)=

## Freqtrade 仓库

Freqtrade 是一个开源加密货币交易机器人，代码托管在 `github.com`

```bash
# 下载 freqtrade 仓库 develop 分支
git clone https://github.com/freqtrade/freqtrade.git

# 进入下载目录
cd freqtrade

# 选项 (1)：新手用户
git checkout stable

# 选项 (2)：高级用户
git checkout develop
```

(1) 该命令将仓库切换到 `stable` 分支。如果你想保持在 (2) `develop` 分支，则无需切换。

你可以随时用 `git checkout stable`/`git checkout develop` 在分支间切换。

:::{note} 从 pypi 安装

另一种安装 Freqtrade 的方式是通过 [pypi](https://pypi.org/project/freqtrade/)。但这种方式要求先正确安装 `ta-lib`，因此目前不推荐。

```bash
pip install freqtrade
```

:::

------

(script-installation)=

## 脚本安装

Freqtrade 的一种安装方式是使用提供的 Linux/MacOS `./setup.sh` 脚本，该脚本会安装所有依赖并帮助你配置机器人。

请确保已满足[依赖要求](#requirements)并已下载 [Freqtrade 仓库](#freqtrade-repository)。

### 使用 `/setup.sh -install` (Linux/MacOS)

在 Debian、Ubuntu 或 MacOS 上，freqtrade 提供了安装脚本。

```bash
# --install，从零安装 freqtrade
./setup.sh -i
```

### 激活虚拟环境

每次打开新终端，都需运行 `source .venv/bin/activate` 激活虚拟环境。

```bash
# 激活虚拟环境
source ./.venv/bin/activate
```

[你现在可以运行机器人了](#you-are-ready)。

### /setup.sh 脚本的其他选项

你还可以用 `./script.sh` 更新、配置和重置机器人代码库。

```bash
# --update，git pull 更新代码
./setup.sh -u
# --reset，强制重置 develop/stable 分支
./setup.sh -r
```

**--install**

此选项会安装机器人及大部分依赖：

你需要预先安装 `git` 和 `python3.10+`。

* 必需软件如：`ta-lib`
* 在 `.venv/` 下设置虚拟环境

此选项结合了安装任务和 `--reset`

**--update**

此选项会拉取当前分支最新代码并更新虚拟环境。建议定期用此选项更新机器人。

**--reset**

此选项会强制重置分支（仅限 `stable` 或 `develop`），并重建虚拟环境。

------

(manual-installation)=

## 手动安装

请确保已满足[依赖要求](#requirements)并已下载 [Freqtrade 仓库](#freqtrade-repository)。

### 设置 Python 虚拟环境（virtualenv）

你将会在独立的 `virtual environment` 下运行 freqtrade

```bash
# 在 /freqtrade/.venv 目录下创建虚拟环境
python3 -m venv .venv

# 运行虚拟环境
source .venv/bin/activate
```

### 安装 python 依赖

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
# 安装 freqtrade
python3 -m pip install -e .
```

[你现在可以运行机器人了](#you-are-ready)。

### （可选）安装后任务

:::{note} 保持 Docker 运行
如果你在服务器上运行机器人，建议使用 [Docker](docker_quickstart.md) 或终端复用器如 `screen` 或 [`tmux`](https://en.wikipedia.org/wiki/Tmux)，以避免登出后机器人被停止。
:::

在带有 `systemd` 的 Linux 上，作为可选的安装后任务，你可以将机器人设置为 `systemd service`，或配置日志输出到 `syslog`/`rsyslog` 或 `journald`。详见 [高级日志](advanced-setup.md#advanced-logging)。

------

(installation-with-conda)=

## Conda 安装

Freqtrade 也可通过 Miniconda 或 Anaconda 安装。

**推荐使用 Miniconda**，因为其安装体积更小。Conda 会自动准备和管理 Freqtrade 的大量依赖库。

### 什么是 Conda？

Conda 是多语言的软件包、依赖和环境管理器：[conda 文档](https://docs.conda.io/projects/conda/en/latest/index.html)

(installation-with-conda)=

### 用 conda 安装

#### 安装 Conda

[Linux 安装方法](https://conda.io/projects/conda/en/latest/user-guide/install/linux.html#install-linux-silent)

[Windows 安装方法](https://conda.io/projects/conda/en/latest/user-guide/install/windows.html)

全部问题请按提示回答。

安装后，务必关闭并重新打开终端。

#### 下载 Freqtrade

下载并安装 freqtrade。

```bash
# 下载 freqtrade
git clone https://github.com/freqtrade/freqtrade.git

# 进入下载目录 'freqtrade'
cd freqtrade      
```

#### Freqtrade 安装：Conda 环境

```bash
conda create --name freqtrade python=3.12
```

:::{note} 创建 Conda 环境

conda 命令 `create -n` 会自动安装所选库的所有依赖，命令结构如下：

```bash
# 自定义包
conda env create -n [环境名] [python 版本] [包]
```

:::

#### 进入/退出 freqtrade 环境

查看可用环境：

```bash
conda env list
```

进入已安装环境：

```bash
# 进入 conda 环境
conda activate freqtrade

# 退出 conda 环境（此处无需执行）
conda deactivate
```

用 pip 安装最后的 python 依赖：

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .
```

[你现在可以运行机器人了](#you-are-ready)。

### 重要快捷命令

```bash
# 列出已安装 conda 环境
conda env list

# 激活 base 环境
conda activate

# 激活 freqtrade 环境
conda activate freqtrade

# 退出所有 conda 环境
conda deactivate                              
```

### anaconda 进一步信息

:::{tip} 安装大型 Python 包的小技巧
有时新建 Conda 环境并在创建时指定所需包，比在已建环境中单独安装大型库或应用更快。
:::

:::{warning} 在 conda 内使用 pip
conda 官方文档建议不要在 conda 环境内用 pip，因为可能会有内部问题。

不过这种情况很少见。

[Anaconda 博客](https://www.anaconda.com/blog/using-pip-in-a-conda-environment)

因此推荐优先使用 `conda-forge` 源：

* 可用库更多（更少用到 `pip`）
* `conda-forge` 与 `pip` 配合更好
* 库版本更新更快
:::

祝你交易顺利！

------

(you-are-ready)=

## 你已准备就绪

你已成功安装 freqtrade。

### 初始化配置

```bash
# 步骤 1 - 初始化用户文件夹
freqtrade create-userdir --userdir user_data

# 步骤 2 - 创建新配置文件
freqtrade new-config --config user_data/config.json
```

你已可以运行机器人，详见 [机器人配置](configuration.md)，请务必以 `dry_run: True` 开始，并确认一切正常。

配置方法详见 [机器人配置](configuration.md) 文档。

### 启动机器人

```bash
freqtrade trade --config user_data/config.json --strategy SampleStrategy
```

:::{warning} 交易前的重要提示
请务必阅读其余文档，回测你要用的策略，并在启用真实交易前用 dry-run 模式测试。
:::

------

## 故障排查

### 常见问题："command not found"

如果你用 (1)`脚本` 或 (2)`手动` 安装，需在虚拟环境下运行机器人。如果出现如下错误，请确保 `venv` 已激活。

```bash
# 如果：
bash: freqtrade: command not found

# 则激活虚拟环境
source ./.venv/bin/activate
```

### MacOS 安装错误

新版 MacOS 可能会因 `error: command 'g++' failed with exit status 1` 安装失败。

这通常需要手动安装 SDK 头文件，MacOS 10.14 可用如下命令：

```bash
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```

如果该文件不存在，说明你用的是其他版本的 MacOS，请上网查找具体解决方法。
