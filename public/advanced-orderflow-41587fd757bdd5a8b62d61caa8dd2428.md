---
title: 高级委托流分析
subject: FreqTrade 委托流指南
subtitle: 深入了解委托流数据分析功能
short_title: 委托流分析
description: 本文档介绍了 FreqTrade 的委托流分析功能,包括如何配置和使用公共成交数据进行高级委托流分析。
---

# 委托流（Orderflow）数据

本指南将带你了解如何在 Freqtrade 中利用公共成交数据进行高级委托流分析。

:::{warning} 实验性功能
委托流功能目前处于测试阶段，未来版本可能会有变动。如有问题或建议，请在 [Freqtrade GitHub 仓库](https://github.com/freqtrade/freqtrade/issues) 反馈。

目前尚未与 freqAI 联合测试——将这两者结合使用暂不在官方支持范围内。
:::

:::{warning} 性能提示
委托流需要原始成交数据。这类数据体量较大，freqtrade 在下载最近 X 根 K 线的成交数据时，初次启动可能会变慢。此外，启用该功能会增加内存占用。请确保有足够的系统资源。
:::

## 快速开始

### 启用公共成交数据

在你的 `config.json` 文件中，将 `exchange` 部分的 `use_public_trades` 选项设置为 true。

```json
"exchange": {
   ...
   "use_public_trades": true,
}
```

### 配置委托流处理

在 config.json 的 orderflow 部分定义你想要的委托流处理设置。你可以调整如下参数：

- `cache_size`：缓存多少根历史委托流 K 线，避免每根新 K 线都重新计算
- `max_candles`：限制获取成交数据的最大 K 线数量
- `scale`：控制 footprint 图的价格分箱大小
- `stacked_imbalance_range`：定义连续失衡价位的最小数量
- `imbalance_volume`：过滤成交量低于该阈值的失衡
- `imbalance_ratio`：过滤买卖量差低于该值的失衡

```json
"orderflow": {
    "cache_size": 1000, 
    "max_candles": 1500, 
    "scale": 0.5, 
    "stacked_imbalance_range": 3, // 至少需要这么多连续失衡
    "imbalance_volume": 1, // 过滤低于该值的失衡
    "imbalance_ratio": 3 // 过滤低于该比值的失衡
  },
```

## 下载回测用成交数据

如需下载历史成交数据用于回测，请在 freqtrade download-data 命令中加上 --dl-trades 参数。

```bash
freqtrade download-data -p BTC/USDT:USDT --timerange 20230101- --trading-mode futures --timeframes 5m --dl-trades
```

:::{warning} 数据可用性
并非所有交易所都提供公共成交数据。对于支持的交易所，如果你用 `--dl-trades` 下载数据而数据不可用，freqtrade 会发出警告。
:::

## 访问委托流数据

启用后，你的数据表（dataframe）会新增如下列：

```python

dataframe["trades"] # 包含每笔成交的详细信息。
dataframe["orderflow"] # footprint 图字典（见下文）
dataframe["imbalances"] # 委托流失衡信息。
dataframe["bid"] # 总买单量
dataframe["ask"] # 总卖单量
dataframe["delta"] # 买卖量差。
dataframe["min_delta"] # 本 K 线内最小 delta
dataframe["max_delta"] # 本 K 线内最大 delta
dataframe["total_trades"] # 总成交笔数
dataframe["stacked_imbalances_bid"] # 堆叠买方失衡区间起点价位列表
dataframe["stacked_imbalances_ask"] # 堆叠卖方失衡区间起点价位列表
```

你可以在策略代码中访问这些列进行进一步分析。例如：

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    # 计算累计 delta
    dataframe["cum_delta"] = cumulative_delta(dataframe["delta"])
    # 访问总成交笔数
    total_trades = dataframe["total_trades"]
    ...

def cumulative_delta(delta: Series):
    cumdelta = delta.cumsum()
    return cumdelta

```

### Footprint 图（`dataframe["orderflow"]`）

该列为不同价位的买卖订单提供详细分布，有助于洞察委托流动态。配置中的 `scale` 参数决定价格分箱大小。

`orderflow` 列为如下结构的字典：

``` output
{
    "price": {
        "bid_amount": 0.0,
        "ask_amount": 0.0,
        "bid": 0,
        "ask": 0,
        "delta": 0.0,
        "total_volume": 0.0,
        "total_trades": 0
    }
}
```

#### orderflow 列说明

- key：价格分箱（以 `scale` 为间隔分箱）
- `bid_amount`：每个价位的买入总量
- `ask_amount`：每个价位的卖出总量
- `bid`：每个价位的买单数
- `ask`：每个价位的卖单数
- `delta`：每个价位的买卖量差
- `total_volume`：每个价位的总成交量（买入+卖出）
- `total_trades`：每个价位的总成交笔数（买单+卖单）

利用这些特性，你可以基于委托流分析获得市场情绪和潜在交易机会。

### 原始成交数据（`dataframe["trades"]`）

该列为本 K 线期间发生的每笔成交组成的列表，可用于更细粒度的委托流分析。

每条记录为如下结构的字典：

- `timestamp`：成交时间戳
- `date`：成交日期
- `price`：成交价格
- `amount`：成交量
- `side`：买入或卖出
- `id`：成交唯一标识
- `cost`：成交总额（价格 * 数量）

### 失衡（`dataframe["imbalances"]`）

该列为字典，记录委托流中的失衡信息。当某价位的买卖量差异显著时，视为失衡。

每行数据如下——以价格为索引，对应的 bid/ask 失衡值为列：

``` output
{
    "price": {
        "bid_imbalance": False,
        "ask_imbalance": False
    }
}
```
