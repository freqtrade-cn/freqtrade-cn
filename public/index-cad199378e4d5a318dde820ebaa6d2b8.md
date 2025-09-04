---
title: Freqtrade 交易机器人
subject: Freqtrade 使用指南
subtitle: 开源加密货币交易机器人
short_title: Freqtrade 指南
description: Freqtrade 是一个功能强大的开源加密货币交易机器人,支持多个交易所、回测、策略优化等功能。本文档详细介绍了其安装、配置和使用方法。
tags: [Freqtrade, 交易机器人, 加密货币, 量化交易, 开源]
categories: [交易工具, 指南]
keywords: [Freqtrade, 加密货币交易, 量化交易, 交易机器人, 回测, 策略优化]
author: Freqtrade 社区
version: 1.0
last_updated: 2024-01-15
nav_order: 1
toc: true
homepage: true
search: true
---
![freqtrade](assets/freqtrade_poweredby.svg)

## 简介

Freqtrade 是一个用 `Python` 编写的免费开源加密货币交易机器人。

* 它支持所有主流交易所，并可通过 Telegram 或 WebUI 控制。
* 它包含回测、绘图和资金管理工具，以及基于机器学习的策略优化功能。

:::{danger} 免责声明
本软件仅供教育用途。请勿拿你无法承受损失的资金进行风险投资。使用本软件风险自负。作者及所有相关方对你的交易结果不承担任何责任。

请务必先以 `Dry-run`（模拟盘）模式运行交易机器人，在完全理解其工作原理及可能的盈亏预期前，不要投入真实资金。

我们强烈建议你具备基本的编程技能和 `Python` 知识。请不要犹豫，阅读源代码，理解本机器人实现的机制、算法和技术。
:::

![freqtrade 截图](assets/freqtrade-screenshot.png)

## 功能特性

* **开发你的策略**
  * 用 [pandas](https://pandas.pydata.org/) 在 `Python` 中编写你的交易策略。
  * 可在 [策略仓库](https://github.com/freqtrade/freqtrade-strategies) 中找到示例策略以供参考。
* **下载市场数据**
  * 下载你可能想交易的交易所和市场的历史数据。
* **回测**
  * 在下载的历史数据上测试你的策略。
* **优化**
  * 通过超参数优化（采用机器学习方法）为你的策略寻找最佳参数。你可以优化买入、卖出、止盈（ROI）、止损和追踪止损等参数。
* **选择市场**
  * 创建你的静态市场列表，或基于交易量和/或价格自动选择（回测时不可用）。你也可以明确黑名单不想交易的市场。
* **运行**
  * 用模拟资金（Dry-Run 模式）测试你的策略，或用真实资金（Live-Trade 模式）部署。
* **使用 Edge（可选模块）运行**
  * 该概念是通过止损变化找到各市场的最佳历史[交易期望值](./edge.md#expect)，然后允许/拒绝市场进行交易。交易规模基于你资金的风险百分比。
* **控制/监控**
  * 通过 Telegram 或 WebUI 控制/监控（启动/停止机器人，显示盈亏、每日总结、当前持仓结果等）。
* **分析**
  * 可对回测数据或 Freqtrade 交易历史（SQL 数据库）进行进一步分析，包括自动标准图表，以及将数据加载到[交互式环境](./data-analysis.md)的方法。

## 支持的交易所

请阅读[交易所特别说明](exchanges.md)，了解每个交易所可能需要的特殊配置。

* [X] [Binance](https://www.binance.com/)
* [X] [BingX](https://bingx.com/invite/0EM9RX)
* [X] [Bitmart](https://bitmart.com/)
* [X] [Bybit](https://bybit.com/)
* [X] [Gate.io](https://www.gate.io/ref/6266643)
* [X] [HTX](https://www.htx.com/)
* [X] [Hyperliquid](https://hyperliquid.xyz/)（去中心化交易所 DEX）
* [X] [Kraken](https://kraken.com/)
* [X] [OKX](https://okx.com/)
* [X] [MyOKX](https://okx.com/)（OKX EEA）
* [ ] [还有许多其他,你可以通过 CCXT](https://github.com/ccxt/ccxt/)来参考或启用 _(我们无法保证它们都能正常工作)_

### 支持的合约交易所（实验性）

* [X] [Binance](https://www.binance.com/)
* [X] [Bybit](https://bybit.com/)
* [X] [Gate.io](https://www.gate.io/ref/6266643)
* [X] [Hyperliquid](https://hyperliquid.xyz/)（去中心化交易所 DEX）
* [X] [OKX](https://okx.com/)

请务必阅读[交易所特别说明](./exchanges.md)以及[杠杆交易文档](./leverage.md)后再深入使用。

### 社区测试通过

由社区确认可用的交易所：

* [X] [Bitvavo](https://bitvavo.com/)
* [X] [Kucoin](https://www.kucoin.com/)

## 社区展示

```{include} includes/showcase.md
```

## 系统需求

### 硬件要求

我们建议你在 Linux 云主机上运行本机器人，最低配置为：

* 2GB 内存
* 1GB 磁盘空间
* 2vCPU

### 软件要求

* Docker（推荐）

或可选

* Python 3.10+
* pip（pip3）
* git
* TA-Lib
* virtualenv（推荐）

## 支持

### 帮助 / Discord

如有文档未涵盖的问题、需要进一步了解机器人，或想与志同道合者交流，欢迎加入 Freqtrade [Discord 服务器](https://discord.gg/p7nuUNVfP7)。

## 准备好开始了吗？

建议先阅读[Docker 安装指南](./docker_quickstart.md)（推荐）或 [非 Docker 安装指南](./installation.md)。
