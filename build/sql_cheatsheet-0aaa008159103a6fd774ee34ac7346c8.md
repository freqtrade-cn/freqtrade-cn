---
title: SQL 助手
subject: SQL 查询指南
subtitle: SQLite 数据库查询参考
short_title: SQL 助手
description: 本文档提供了使用 SQLite 数据库进行查询的详细指南和常用命令参考。
---

# SQL 助手

本页包含一些帮助，如果你想查询你的 sqlite 数据库。

:::{tip} 其他数据库系统
要使用其他数据库系统，如 PostgreSQL 或 MariaDB，你可以使用相同的查询，但你需要使用相应数据库系统的客户端。[点击这里](advanced-setup.md#use-a-different-database-system)了解如何使用 freqtrade 设置不同的数据库系统。
:::

:::{warning}
如果你不熟悉 SQL，在数据库上运行查询时应该非常小心。

在运行任何查询之前，始终确保有数据库的备份。
:::

## 安装 sqlite3

Sqlite3 是一个基于终端的 sqlite 应用程序。
如果你觉得更舒服，可以随意使用像 SqliteBrowser 这样的可视化数据库编辑器。

### Ubuntu/Debian 安装

```bash
sudo apt-get install sqlite3
```

### 通过 docker 使用 sqlite3

freqtrade docker 镜像确实包含 sqlite3，所以你可以在不需要在主机系统上安装任何东西的情况下编辑数据库。

```bash
docker compose exec freqtrade /bin/bash
sqlite3 <database-file>.sqlite
```

## 打开数据库

```bash
sqlite3
.open <filepath>
```

## 表结构

### 列出表

```bash
.tables
```

### 显示表结构

```bash
.schema <table_name>
```

### 获取表中的所有交易

```sql
SELECT * FROM trades;
```

## 破坏性查询

写入数据库的查询。
这些查询通常不应该需要，因为 freqtrade 试图自己处理所有数据库操作 - 或通过 API 或 telegram 命令暴露它们。

:::{warning}
在运行以下任何查询之前，请确保你有数据库的备份。
:::

:::{danger}
当机器人连接到数据库时，你也应该**永远不要**运行任何写入查询（`update`、`insert`、`delete`）。

这可能导致数据损坏 - 很可能无法恢复。
:::

### 修复在交易所手动出场后仍然开放的交易

:::{warning}
在交易所手动卖出交易对不会被机器人检测到，它会尝试卖出。只要可能，应该使用 /forceexit <tradeid> 来完成同样的事情。

强烈建议在进行任何手动更改之前备份你的数据库文件。
:::

:::{note}
在 /forceexit 之后，这应该不是必需的，因为 force_exit 订单会在下一次迭代时被机器人自动关闭。
:::

```sql
UPDATE trades
SET is_open=0,
  close_date=<close_date>,
  close_rate=<close_rate>,
  close_profit = close_rate / open_rate - 1,
  close_profit_abs = (amount * <close_rate> * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason=<exit_reason>
WHERE id=<trade_ID_to_update>;
```

#### 示例

```sql
UPDATE trades
SET is_open=0,
  close_date='2020-06-20 03:08:45.103418',
  close_rate=0.19638016,
  close_profit=0.0496,
  close_profit_abs = (amount * 0.19638016 * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason='force_exit'  
WHERE id=31;
```

### 从数据库中删除交易

:::{tip} 使用 RPC 方法删除交易
考虑通过 telegram 或 rest API 使用 `/delete <tradeid>`。这是删除交易的推荐方式。
:::

如果你仍然想直接从数据库中删除交易，你可以使用下面的查询。

:::{danger}
某些系统（Ubuntu）在其 sqlite3 包中禁用了外键。使用 sqlite 时 - 请确保通过在上面的查询之前运行 `PRAGMA foreign_keys = ON` 来启用外键。
:::

```sql
DELETE FROM trades WHERE id = <tradeid>;

DELETE FROM trades WHERE id = 31;
```

:::{warning}
这将从数据库中删除此交易。请确保你获得了正确的 id，并且**永远不要**在没有 `where` 子句的情况下运行此查询。
:::
