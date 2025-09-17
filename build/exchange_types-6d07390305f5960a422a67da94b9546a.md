# 交易所订单执行策略（TIF）与撮合约束备忘清单（ccxt / Freqtrade 实战版）

> 面向量化/程序化交易的快速参考。涵盖常见 **TIF（Time-in-Force，有效期/执行时机）** 与 **撮合/风控约束**，并附 **ccxt / Freqtrade** 的参数示例与踩坑提示。

---

## 1) 核心概念与对比

### 1.1 经典四项（最常见）

| 名称                           | 含义                     | 是否允许部分成交 | 是否要求立即成交 | 仅做挂单（Maker） |
| ---------------------------- | ---------------------- | -------: | -------: | ----------: |
| **GTC**（Good-Till-Canceled）  | 订单一直有效，直到完全成交或被你撤单     |        ✅ |        ❌ |           ❌ |
| **IOC**（Immediate-Or-Cancel） | 立刻成交能成交的部分，余量立刻取消      |        ✅ |        ✅ |           ❌ |
| **FOK**（Fill-Or-Kill）        | 要么立刻 100% 成交，要么整笔取消    |        ❌ |        ✅ |           ❌ |
| **PO**（Post-Only）            | 只能作为挂单进入订单簿；若会立刻成交则拒/撤 |        — |        — |           ✅ |

> 注：**PO**严格说是“撮合约束”而不是 TIF，但平台常与 TIF 一并提供，故列入经典四项。

### 1.2 其他常见 TIF

| 名称                                | 含义                                                | 常见平台/备注                 |
| --------------------------------- | ------------------------------------------------- | ----------------------- |
| **GTX**（Good-Till-Crossing）       | 本质等同 **Post-Only 的 TIF 表达**：若会立刻与对手单成交（cross）则拒/撤 | 常见于部分期货/合约平台            |
| **GTT / GTD**（Good-Til-Time/Date） | 有效至指定时间点/日期                                       | 常见于部分交易所的高级/条件单         |
| **DAY / Good-for-Day**            | 仅当日/当交易时段有效                                       | 传统证券市场常见，部分加密衍生/法币经纪也支持 |

---

## 2) 常与 TIF 搭配的撮合/风控约束

| 约束                                   | 作用                           | 典型使用场景               |
| ------------------------------------ | ---------------------------- | -------------------- |
| **Post-Only（PO）**                    | 只允许作为 **Maker** 进入簿；会立即成交则拒单 | 赚 maker 费率/做被动成交     |
| **Reduce-Only**                      | 该单只能**减少**仓位，不会开新仓           | 止盈/止损、网格收口           |
| **Close-On-Trigger / closePosition** | 触发后**一键平掉**该方向全部持仓           | 合约止损/风控强平场景          |
| **Iceberg / Hidden**                 | 冰山/隐藏单：只展示一小部分，实际大单分片执行      | 大额下单控冲击成本（通常需配合 GTC） |
| **STP（自成交防护）**                       | 避免同账户挂单互相成交（不同模式）            | 多策略/多端共用账户           |

---

## 3) 选择策略（如何用得对）

* **耐心排队、默认限价**：选 **GTC**。
* **尽快拿到能成交的量、余量不要**：选 **IOC**。
* **非全量不买/不卖**：选 **FOK**（下前先评估深度）。
* **只做被动挂单、赚 maker 费**：选 **PO / GTX**。
* **订单需要“到期时间”**：选 **GTT/GTD**（并设置到期时刻）。
* **不想意外加仓**：勾 **Reduce-Only**。
* **止损/止盈一键平仓**：加 **closePosition / Close-On-Trigger**。
* **大额分片、降低冲击**：用 **Iceberg**（通常与 **GTC** 绑定）。

---

## 4) 行为示例（更直观）

> 设卖一=60,000，卖二=60,100；你挂**限价买 60,050 / 2 BTC**

* **GTC**：成交卖一 1 BTC，剩余 1 BTC 继续挂在 60,050 等成交。
* **IOC**：成交卖一 1 BTC，**剩余 1 BTC 立刻取消**。
* **FOK**：若 60,050 以内凑不满 2 BTC，**整笔取消**。
* **PO/GTX**：因 60,050 会立刻成交 → **订单被拒/撤**。要 PO 成功，应挂在现价之外（如 59,950）。

---

## 5) ccxt 参数速查（Python）

```python
# GTC（多数默认）
ex.create_order('BTC/USDT', 'limit', 'buy', 0.5, 60000, {'timeInForce': 'GTC'})

# IOC / FOK
ex.create_order('BTC/USDT', 'limit', 'buy', 0.5, 60000, {'timeInForce': 'IOC'})
ex.create_order('BTC/USDT', 'limit', 'buy', 0.5, 60000, {'timeInForce': 'FOK'})

# Post-Only：两种常见写法（按交易所不同二选一）
ex.create_order('BTC/USDT', 'limit', 'buy', 0.5, 59950, {'postOnly': True})
ex.create_order('BTC/USDT', 'limit_maker', 'buy', 0.5, 59950)  # 某些所用专门类型

# GTX（部分合约所，把 PO 表达为 TIF）
ex.create_order('BTC/USDT:USDT', 'limit', 'buy', 0.5, 59950, {'timeInForce': 'GTX'})

# GTT/GTD（到期时间；字段名依交易所而定）
ex.create_order('BTC/USDT', 'limit', 'buy', 0.5, 60000, {
    'timeInForce': 'GTD',
    'expireTime': '2025-09-30T12:00:00Z'  # 示例字段
})

# Reduce-Only / Close-Position（合约）
ex.create_order('BTC/USDT:USDT', 'limit', 'sell', 0.5, 60500, {'reduceOnly': True})
ex.create_order('BTC/USDT:USDT', 'stop_market', 'sell', 0.0, None, {'closePosition': True, 'stopPrice': 59000})

# Iceberg / Hidden（依所支持）
ex.create_order('BTC/USDT', 'limit', 'buy', 5.0, 60000, {'icebergQty': '0.5', 'timeInForce': 'GTC'})
```

> **提示**：不同交易所字段名/可选值不完全一致（如 `limit_maker`、`expireTime`、`icebergQty` 等），以目标交易所 API 为准。

---

## 6) Freqtrade 配置与策略要点

**全局 TIF（配置文件）**

```json
{
  "order_time_in_force": {
    "buy": "gtc",
    "sell": "gtc"
  }
}
```

**策略里精细控制（传递到 ccxt 的 params）**

```python
# 例：策略下单时为买单使用 IOC，为卖单使用 PO
order = self.buy(price=price, amount=amount, time_in_force='ioc')  # 或在 order() 的 params 中附加
# 对应 Post-Only
params = {'postOnly': True}
order = self.sell(price=target, amount=amount, params=params)
```

**实战建议**

* 期货/永续优先用专用 **exchange id**（如 `binanceusdm`），避免现货/合约 API 混用。
* Maker 策略优先 **PO/GTX**；Taker/抢成交用 **IOC**。
* 大单评估深度：若要求“全有或全无”，再考虑 **FOK**。
* 止盈/止损与网格收口配 **Reduce-Only**，防止意外加仓。

---

## 7) 常见坑位与排查

1. **PO 下不去**：价格“穿”对手盘 → 会立即成交 → 被拒/撤；把价格挂到簿外侧。
2. **FOK 总被取消**：一次性吃不满目标数量；先检查盘口深度或改用 IOC。
3. **GTT/GTD 无效**：未按交易所要求提供到期字段/格式。
4. **Iceberg 报错**：不少平台要求 **配合 GTC**；且有最小分片/最小显示量限制。
5. **字段名差异**：`postOnly`/`limit_maker`、`closePosition`/`close_on_trigger`… 落地前查看目标交易所文档。

---

## 8) 速记卡（TL;DR）

* **GTC**：默认、耐心等。
* **IOC**：能成交就成，余量不要。
* **FOK**：非全量不成交。
* **PO / GTX**：只做 **Maker**。
* **GTT/GTD**：到期自动撤。
* **Reduce-Only**：只减仓。
* **Close-On-Trigger**：触发平全仓。
* **Iceberg/Hidden**：大单分片隐身执行（多与 GTC 绑定）。

---
