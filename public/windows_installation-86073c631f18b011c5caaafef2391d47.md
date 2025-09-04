---
title: Windows 安装指南
subject: Freqtrade Windows 安装文档
subtitle: 在 Windows 系统上安装 Freqtrade 的详细说明
short_title: Windows 安装
description: 本文档详细介绍了如何在 Windows 系统上安装 Freqtrade 交易机器人,包括自动安装和手动安装两种方式。
tags: [Windows, 安装, 配置, 指南]
categories: [基础教程]
prerequisites: [Python 3.10+, Git]
difficulty: beginner
estimated_time: 30分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 11
toc: true
---

# Windows 安装

我们**强烈**建议 Windows 用户使用 [Docker](docker_quickstart.md)，因为这将更容易和更顺畅地工作（也更安全）。

如果不可能，请尝试使用 Windows Linux 子系统（WSL）- 对于 Ubuntu 的说明应该有效。

否则，请按照以下说明进行操作。

所有说明都假设已安装并可用 `Python 3.10+`。

## 克隆 git 仓库

首先，通过运行以下命令克隆仓库：

```powershell
git clone https://github.com/freqtrade/freqtrade.git
```

现在，选择您的安装方法，要么通过脚本自动安装（推荐），要么按照相应的说明手动安装。

## 自动安装 freqtrade

### 运行安装脚本

脚本将询问您几个问题，以确定应安装哪些部分。

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass
cd freqtrade
. .\setup.ps1
```

## 手动安装 freqtrade

:::{warning} 64位 Python 版本
请确保使用 64 位 Windows 和 64 位 Python，以避免由于 32 位应用程序在 Windows 下的内存限制而导致回测或超参数优化问题。
32 位 Python 版本在 Windows 下不再受支持。
:::

:::{tip} 建议 Windows 中使用 Anaconda
在 Windows 下使用 [Anaconda Distribution](https://www.anaconda.com/distribution/) 可以大大帮助解决安装问题。查看文档中的 [Anaconda 安装部分](installation.md#installation-with-conda) 以获取更多信息。
:::

### Windows 安装过程中的错误

```bash
error: Microsoft Visual C++ 14.0 is required. Get it with "Microsoft Visual C++ Build Tools": http://landinghub.visualstudio.com/visual-cpp-build-tools
```

不幸的是，许多需要编译的包没有提供预构建的 Wheels。因此，必须为您的 Python 环境安装并可用 C/C++ 编译器。

您可以从[这里](https://visualstudio.microsoft.com/visual-cpp-build-tools/)下载 Visual C++ 构建工具，并在其默认配置中安装"使用 C++ 的桌面开发"。不幸的是，这是一个很重的下载/依赖项，因此您可能想先考虑 WSL2 或 [docker compose](docker_quickstart.md)。

![Windows 安装](assets/windows_install.png)
