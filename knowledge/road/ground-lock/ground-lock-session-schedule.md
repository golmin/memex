---
title: "路内地锁会话与预约升锁"
type: reference
summary: "绑定表、进行中会话和 Redis 到期队列的约定，以及建单预约、支付降锁、驶出降锁的挂法。"
tags: [地锁, 路内]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "这条说明被路内地锁开发记忆引用"
  - path: /knowledge/road/ground-lock/jingdezhen-charge-start.md
    label: uses
    description: "到期时间用计费起点推算，不用驶入时间"
  - path: /knowledge/road/ground-lock/ground-lock-bind-session-ddl.md
    label: uses
    description: "绑定表和会话表的建表语句"
  - path: /knowledge/road/ground-lock/ground-lock-pay-keep-parking.md
    label: about
    description: "付款不结单已回退"
sources:
  - title: "iroad_jdz/doc/路内地锁会话与预约升锁说明.md"
---

# 路内地锁：表结构与 ZSET 预约升锁

## 背景与范围

- **做了什么**：绑定/会话 DDL；ZSET 工具；`createOrder` 后预约；定时捞取到期升锁；`parkOut`/`parkOutForce` 强制降锁；扫码支付 `onPaid` 降锁+宽限再预约。
- **不做**：余额缴费编排；对账补降本轮未挂。
- **支付降锁入口**：微信/支付宝进行中扫码 → `payBatchArrearNotifyListener` → `GroundLockOrchestrator.onPaid`；余额 `consume` 不处理。
- **计费起点**：`JingDeZhenChargeStart` / `RuleBase.getChargeStartTime`（按收费窗 + 0 元费率档推算，例如正式环 07:30–21:00 内前 120 分钟 0 元）。**不要**只用策略头 `freecharging/freeduration`，也不要用 `getNeedPayMoney.startparktime`。详见 [景德镇计费开始时间推算](/knowledge/road/ground-lock/jingdezhen-charge-start.md)。

## 表

SQL：[地锁绑定与进行中会话表](/knowledge/road/ground-lock/ground-lock-bind-session-ddl.md)

| 表 | 用途 |
|----|------|
| `tra_ground_lock_bind` | 泊位 ↔ 地锁（beId/numbers/deviceSerial/deviceType） |
| `tra_ground_lock_session` | 进行中会话：一泊位一行，含 orderid、车牌、锁状态、startparktime |

约定：

- 录单 `createOrder` 成功且有绑定 → INSERT/REPLACE session  
- 驶出强制降锁后 → DELETE session  
- 支付成功按 **session.orderid == 本单** 才降锁（防欠费补缴误降）

## ZSET 会话（预约升锁）

| 项 | 值 |
|----|-----|
| 类 | `iroad_base` `com.tra.utils.GroundLockRaiseDueUtils` |
| key | `jdz:ground_lock:raise_due` |
| member | `orderid` |
| score | 应升锁毫秒时间戳 |

| 方法 | 场景 |
|------|------|
| `scheduleRaise(orderId, startparktime)` | 录单后按计费起点预约 |
| `scheduleRaiseAfterPaid(orderId)` | 仍可用；当前 onPaid **不用**，改为按计费规则推下次起点 |
| `cancelRaise(orderId)` | 驶出 / 无需再升 |
| `pollDue(now, limit)` | 定时捞取到期（只查不删，处理成功后再 cancel） |
| `getDueMs(orderId)` | 查询当前预约 |

到期处理（后续任务，不在本类）：

```text
pollDue → 查 session(orderid) → 仍进行中且(试算price-actualpay-deductionfee)>0 → raiseLock
         → 否则改约或 cancelRaise
```

## 配置

| Key | 说明 |
|-----|------|
| `groundLock.enabled` | 总开关 |
| `groundLock.leaveGraceMs` | 缴费后再升锁宽限，默认 900000（15 分钟） |
| `redisIP` / `redisPort` / `redisTimeout` / `redisAuth` | Jedis（与现网 Cache 一致） |

## 类清单（已落地）

| 类 | 路径 | 职责 |
|----|------|------|
| `GroundLockDao` | iroad_base `com.tra.utils` | 绑定/会话 CRUD |
| `GroundLockOrchestrator` | 同上 | `onOrderCreated` / `processDueRaises` / `descend` / `onPaid` |
| `JingDeZhenChargeStart` | iroad_base `com.rule.internal` | 真正计费起点（免费时长+免费时段） |
| `GroundLockRaiseDueUtils` | 同上 | ZSET |
| `GroundLockRaiseDueTask` | iroad_jdz `com.parking.task` | 定时捞取 |

挂载：

- `ParkingService.createOrder` 新建单成功 → `onOrderCreated`
- `ParkingService.parkOut` / `parkOutForce` 泊位 state=2 写库成功后 → `descend`
- `NotifyBase.payBatchArrearNotifyListener` 支付成功 → `onPaid`（微信/支付宝进行中扫码）
- `JDZApplicationMain.taskInit`（`groundLock.enabled`）→ `GroundLockRaiseDueTask`

## 支付降锁再预约（`onPaid`）

| 项 | 约定 |
|----|------|
| 业务入口 | `payGoingOrdersByWeChat` / `payGoingOrdersByAlipay` → `JZBatchPayment.payGoingOrders`；合并缴费新回调同样可 `onPaid` |
| 回调 | `payBatchArrearNotifyListener` / `payGoingWithArrearsNotifyListener` |
| 入参 | dataMap：`consumeno` / `cacheorderid` / `orderid` |
| 门槛 | 有会话且 `session.orderid == 本单`，否则跳过（防欠费补缴误降） |
| 流程 | bind/会话组 identity → `lowerLock` → 成功 `lock_state=1` / 失败 `=2` → `cancelRaise` → 仍进行中则从**现在**按规则推下次计费起点再 `scheduleRaise`；已结束则删会话 |
| 不做 | 余额缴费；付款不结单方案已回退（见 [地锁泊位付款不结单](/knowledge/road/ground-lock/ground-lock-pay-keep-parking.md)） |

常见坑：进行中扫码勿挂 `JZStdOrderService.updateOrderStatus`；勿只用 berthcode 当 numbers（优先绑定表 beId/deviceSerial）。

## 驶出强制降锁（`descend`）

| 项 | 约定 |
|----|------|
| 挂点 | `updateBerthStatus` 成功把泊位置无车**之后立刻**调用，不依赖结单/PDA 推送 |
| 入参 | `comid`、`berthcode` 必填；`orderid` 可选（`parkOut` 有进行中单时从 orderMap 带入） |
| 流程 | 查会话补 orderid → 绑定（或会话快照）组 identity → `lowerLock` → `cancelRaise` → `deleteSessionByBerth` |
| 失败 | 内部吞异常，只打 `groundlock` 日志，不影响驶出主流程 |
| 无绑定 | 仍按会话清 ZSET/删会话；无绑定且无会话则跳过 |

常见坑：

- 勿读 `orderId` 驼峰（调用方是小写 `orderid`）
- `getBind` 已返回扁平行，用 `getString`/`get`，勿 `getRecordString(0,…)`
- 勿把降锁挂在推送 try 末尾（占锁提前 return / 无单路径会漏降）

## 与主流程关系

```text
createOrder(APPLYTYPE_PAYMENT 成功)
  → 查 bind → 无绑定则跳过
  → JingDeZhenChargeStart 推 chargeStart → upsert session → ZADD(due=chargeStart)

定时 pollDue
  → 无会话/非进行中 → 取消预约
  → 未付=试算price-actualpay-deductionfee；<0.01 → 改约到 JingDeZhenChargeStart（已过则 +60s）
  → 未付>0 → raiseLock → 更新 lock_state=已升 → 取消预约

扫码支付成功（payBatchArrearNotifyListener）
  → onPaid：session.orderid 命中 → 降锁（失败则 lock_state=2）
  → 仍在停：从支付时刻按 JingDeZhenChargeStart 推下次计费起点再 ZADD
  → 已结束：删会话
驶出 parkOut/parkOutForce 泊位置空成功 → descend（降锁失败也清会话）
余额缴费 → 不处理
不做视频 jzParkOut 挂载（无视频桩）
```
