---
title: 使用 Docker 运行 Freqtrade
description: 了解如何使用 Docker 容器运行和管理 Freqtrade 交易机器人,包括安装、配置和常见问题解决
subject: Docker 快速入门
subtitle: 使用 Docker 容器运行 Freqtrade
short_title: Docker 快速入门
tags: [Docker, 容器, 快速入门, 部署]
categories: [安装部署]
prerequisites: [Docker, Docker Compose]
difficulty: beginner
estimated_time: 20分钟
last_updated: 2024-01-15
version: 1.0
nav_order: 3
toc: true
---

# 使用 Docker 运行 Freqtrade

本页介绍如何使用 Docker 运行机器人。它并非开箱即用，你仍需阅读文档并了解如何正确配置。

## 安装 Docker

首先为你的平台下载安装 Docker / Docker Desktop：

* [Mac](https://docs.docker.com/docker-for-mac/install/)
* [Windows](https://docs.docker.com/docker-for-windows/install/)
* [Linux](https://docs.docker.com/install/)

:::{hint} Docker compose 安装
Freqtrade 文档假定你使用 Docker Desktop（或 docker compose 插件）。  

虽然独立安装的 docker-compose 依然可用，但你需要将所有 `docker compose` 命令改为 `docker-compose`（如 `docker compose up -d` 改为 `docker-compose up -d`）。
:::

:::{warning} Windows 上的 Docker
如果你刚在 Windows 系统上安装了 Docker，请务必重启系统，否则可能会遇到与 Docker 容器网络连接相关的莫名问题。
:::

```{include} includes/docker_evns.md
```

## 使用 docker 运行 Freqtrade

Freqtrade 在 [Dockerhub](https://hub.docker.com/r/freqtradeorg/freqtrade/) 上提供了官方 Docker 镜像，并有一份[docker compose 文件](https://github.com/freqtrade/freqtrade/blob/stable/docker-compose.yml)可直接使用。

:::{hint} 运行提示

* 以下内容假设 `docker` 已安装并可被当前用户使用。
* 所有命令均使用相对路径，需在包含 `docker-compose.yml` 文件的目录下执行。

:::

(docker-quick-start)=

### Docker 快速入门

新建一个目录，并将 [docker-compose 文件](https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml)下载到该目录。

```bash
mkdir ft_userdata
cd ft_userdata/

# 从仓库下载 docker-compose 文件
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# 拉取 freqtrade 镜像
docker compose pull

# 创建用户目录结构
docker compose run --rm freqtrade create-userdir --userdir user_data

# 创建配置文件 - 需要交互式回答问题
docker compose run --rm freqtrade new-config --config user_data/config.json
```

上述命令片段会创建一个名为 `ft_userdata` 的新目录，下载最新的 compose 文件并拉取 freqtrade 镜像。
最后两步会创建带有 `user_data` 的目录，并根据你的选择（交互式）生成默认配置。

:::{tip} 如何编辑机器人配置？
你可以随时编辑配置文件，配置文件位于 `ft_userdata` 目录下的 `user_data/config.json`。

你也可以通过编辑 `docker-compose.yml` 文件中的 command 部分来更改策略和命令。
:::

#### 添加自定义策略

1. 现在配置文件已生成在 `user_data/config.json`
2. 将自定义策略复制到 `user_data/strategies/` 目录下
3. 在 `docker-compose.yml` 文件中添加策略类名

默认会运行 `SampleStrategy`。

:::{danger} `SampleStrategy` 只是演示用！
`SampleStrategy` 仅供参考，帮助你构思自己的策略。

请务必先回测你的策略，并用 `dry-run` 模式运行一段时间后再投入真实资金！

有关策略开发的更多信息请参见[策略文档](strategy-customization.md)。
:::

完成上述步骤后，你就可以启动机器人进入交易模式（`Dry-run` 或实盘，取决于你之前的选择）。

```bash
docker compose up -d
```

:::{warning} 默认配置
虽然生成的配置大多可用，但你仍需确认所有选项（如定价、交易对列表等）是否符合你的需求后再启动机器人。
:::

(#accessing-the-ui)

#### 访问 UI

如果你在 `new-config` 步骤中选择启用了 FreqUI，则可通过 `localhost:8080` 端口访问 FreqUI。

你可以在浏览器中输入 **http://localhost:8080** 访问 UI。

:::{important} 远程服务器上的 UI 访问
如果你在 VPS 上运行，建议使用 ssh 隧道或设置 VPN（openVPN、wireguard）连接到你的机器人。

这样可以确保 freqUI 不直接暴露在互联网，出于安全考虑（freqUI 默认不支持 https），不推荐直接暴露。

这些工具的设置不在本教程范围内，网上有很多相关教程可供参考。

另请阅读 [API 配置与 docker](rest-api.md#configuration-with-docker) 章节了解更多相关配置。
:::

#### 监控机器人

你可以用 `docker compose ps` 查看正在运行的实例。

这会显示 `freqtrade` 服务为 `running`。如果不是，建议查看日志（见下文）。

#### Docker compose 日志

日志会写入：`user_data/logs/freqtrade.log`。  

你也可以用 `docker compose logs -f` 查看最新日志。

#### 数据库

数据库位于：`user_data/tradesv3.sqlite`

#### 使用 docker 更新 freqtrade

使用 `docker` 更新 freqtrade 只需运行以下两条命令：

```bash
# 下载最新镜像
docker compose pull
# 重启镜像
docker compose up -d
```

这会先拉取最新镜像，然后用新镜像重启容器。

:::{warning} 请检查更新日志
每次更新前请务必查看更新日志，关注破坏性变更/需手动干预的内容，并确保机器人在更新后能正常启动。
:::

### 编辑 docker-compose 文件

高级用户可以进一步编辑 `docker-compose` 文件，包含所有可用选项或参数。

所有 `freqtrade` 参数都可通过 `docker compose run --rm freqtrade <命令> <可选参数>` 运行。

:::{warning} `docker compose` 用于交易命令
交易命令（`freqtrade trade <...>`）不应通过 `docker compose run` 运行，而应使用 `docker compose up -d`。

这样可确保容器正确启动（包括端口转发），并在系统重启后自动重启。

如果你打算使用 freqUI，请确保[相应配置已调整](rest-api.md#configuration-with-docker)，否则 UI 将无法访问。
:::

:::{tip} `docker compose run --rm`
加上 `--rm` 会在命令完成后删除容器，强烈推荐除交易模式（`freqtrade trade` 命令）外的所有模式都加上。
:::

:::{tip} 不使用 `docker compose` 的 `docker` 用法

`docker compose run --rm` 需要提供 `compose` 文件。

某些不需要认证的 freqtrade 命令（如 `list-pairs`）可直接用 "`docker run --rm`" 运行。  

例如：`docker run --rm freqtradeorg/freqtrade:stable list-pairs --exchange binance --quote BTC --print-json`。  

这对于获取交易所信息并添加到 `config.json` 而不影响正在运行的容器很有用。
:::

#### 示例：用 `docker` 下载数据

从 Binance 下载 ETH/BTC 交易对 1 小时线的 5 天回测数据，数据将存储在主机的 `user_data/data/` 目录下。

```bash
docker compose run --rm freqtrade download-data \
    --pairs ETH/BTC \
    --exchange binance \
    --days 5 \
    -t 1h
```

更多数据下载细节请参见[数据下载文档](data-download.md)。

#### 示例：用 `docker` 回测

在 docker 容器中用 `SampleStrategy` 和指定历史数据区间(`20190801-20191001`)、做 `5` 分钟线的回测：

```bash
docker compose run --rm freqtrade backtesting \
    --config user_data/config.json \
    --strategy SampleStrategy \
    --timerange 20190801-20191001 \
    -i 5m
```

更多内容请参见[回测文档](backtesting.md)。

### 在 docker 中增加的额外依赖(加入自己的 Python 依赖库)

如果你的策略依赖**默认镜像未包含的依赖**，则需在本地构建镜像。

为此，请创建一个包含额外依赖安装步骤的 Dockerfile（可参考 [docker/Dockerfile.custom](https://github.com/freqtrade/freqtrade/blob/develop/docker/Dockerfile.custom)）。

你还需修改 `docker-compose.yml` 文件，取消 `build` 步骤的注释，并重命名镜像以避免命名冲突。

``` yaml
    image: freqtrade_custom
    build:
      context: .
      dockerfile: "./Dockerfile.<你自定义的扩展名>"
```

然后运行 `docker compose build --pull` 构建镜像，并用上述命令运行。

### docker 下绘图

将 `freqtrade plot-profit` 和 `freqtrade plot-dataframe`（[文档](plotting.md)）命令的镜像在 `docker-compose.yml` 文件中改为 `*_plot`，即可使用这些命令：

```bash
docker compose run --rm freqtrade plot-dataframe \
    --strategy AwesomeStrategy \
    -p BTC/ETH \
    --timerange=20180801-20180805
```

输出会保存在 `user_data/plot` 目录下，可用任意现代浏览器打开。

### 用 docker compose 进行数据分析

Freqtrade 提供了一个 `docker-compose` 文件，可启动 `jupyter lab` 服务器。
你可以用以下命令启动该服务器：

```bash
docker compose -f docker/docker-compose-jupyter.yml up
```

这会创建一个运行 `jupyter lab` 的 `docker` 容器，可通过 `https://127.0.0.1:8888/lab` 访问。

请使用启动后**控制台打印的链接**简化登录。

由于该镜像部分在本地构建，建议不时重建镜像以保持 freqtrade（及依赖）为最新。

```bash
docker compose -f docker/docker-compose-jupyter.yml build --no-cache
```

## 故障排查

### Windows 下的 Docker

* 错误：`"Timestamp for this request is outside of the recvWindow."`

市场 API 请求需要同步时钟，但 docker 容器内的时间会逐渐偏移。

临时解决方法是运行 `wsl --shutdown` 并重启 docker（Windows 10 会弹窗提示）。

永久解决方案是将容器部署在 Linux 主机，或定期用计划任务重启 wsl。

```bash
taskkill /IM "Docker Desktop.exe" /F
wsl --shutdown
start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
```

* 无法连接 API（Windows）  

如果你在 Windows 上刚安装 Docker（Desktop），请务必重启系统。否则 Docker 可能会有网络连接问题。

当然也要确保你的[设置](#accessing-the-ui)正确。

:::{warning}
基于上述原因，我们不推荐在生产环境下在 Windows 上使用 docker，仅建议用于实验、数据下载和回测。

可靠运行 freqtrade 最好使用 Linux VPS。
:::
