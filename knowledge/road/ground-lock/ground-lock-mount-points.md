---
title: "路内地锁升降锁挂载点"
type: reference
summary: "无视频桩时，升锁、降锁和支付回调应该挂在哪些方法上，哪些入口明确不挂。"
tags: [地锁, 路内]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "这条说明被路内地锁开发记忆引用"
  - path: /knowledge/road/ground-lock/ground-lock-session-schedule.md
    label: uses
    description: "挂载点对应的会话和到期队列约定"
  - path: /knowledge/road/ground-lock/ground-lock-bind-session-ddl.md
    label: uses
    description: "挂载点读写的两张表"
sources:
  - title: "iroad_jdz/doc/路内地锁升降锁挂载点说明.md"
---

# 路内地锁升降锁挂载点说明

## 1. 背景与范围

- **场景**：景德镇路内，**无视频桩**；地磁 `uploadBerthStatus` 记泊位变化，**PDA 录单建单**。
- **目标**：免费/计费起点后预约升锁；缴费降锁并可再预约；泊位空强制降锁；定时对账。
- **不做**：不在 `uploadBerthStatus` / `parkIn` / MQ Rece 内直接调厂商升锁；不重写 `JingDeZhenCalc`；不用 `ParkLockUtils`（Redis 互斥）。

## 2. 业务链路（无视频）

```text
有车: uploadBerthStatus → berthchange → BerthChangeMQRece → parkIn（不建单）
录单: PDA startParking → StdService.apply → JZParkingService.apply → createOrder
无车: uploadBerthStatus → berthchange → BerthChangeMQRece → parkOut（泊位置空，可结单）
缴费: 扫码回调 / updateOrderStatus / 余额 consume
```

## 3. 已标注 TODO(地锁) 位置

| 动作 | 仓库 | 文件与位置 | TODO 含义 |
|------|------|------------|-----------|
| 预约升锁 | iroad_base | `ParkingService.createOrder` 新建单成功 | `onOrderCreated`（`JingDeZhenChargeStart` 推计费起点 + ZADD） |
| 到期升锁 | iroad_jdz | `GroundLockRaiseDueTask` / `JDZApplicationMain` | `processDueRaises`：未付>0 才升 |
| 强制降锁 | iroad_base | `ParkingService.parkOut`，泊位置空写库成功后 | `descend`：降锁+ZREM+删会话，不依赖结单 |
| 强制驶离降锁 | iroad_jdz | `JZParkingService.parkOutForce`，泊位写成无车且写库成功 | `descend`（覆盖了基类，必须在这里补；仍有进行中单则不降） |
| 强制降锁（旁注） | iroad_base | `BerthChangeMQRece` 无车分支 | 指向 `parkOut` 内实现 |
| 入口说明 | iroad_base | `StdBase.uploadBerthStatus` 投递 MQ 前 | 禁止在此升锁/ZADD |
| 强制降锁 | iroad_jdz | ~~`JZParkingService.jzParkOut`~~ | **不挂**（无视频桩） |
| 缴费降锁+再预约 | iroad_jdz | ~~`JZStdOrderService.updateOrderStatus`~~ | **不挂**（进行中扫码不走此方法） |
| 缴费降锁+再预约 | iroad_jdz | ~~`JZParkingService` 余额 consume~~ | **不做** |
| 缴费降锁+再预约 | iroad_base | `NotifyBase.payBatchArrearNotifyListener` 成功 | `onPaid`（微信/支付宝进行中扫码） |
| 捞取+对账 | iroad_jdz | `JDZApplicationMain.taskInit` 末尾 | ZSET 捞取 + 约 30s 对账任务 |

统一编排类：`iroad_base` `com.tra.utils.GroundLockOrchestrator`（`onOrderCreated` / `processDueRaises` / `descend` / `onPaid`）。

## 4. 关键流程（目标态）

```text
createOrder 成功（进行中）
  → ZADD due=JingDeZhenChargeStart 计费起点

ZSET 到期
  → 仍在停 && 未付>0 → 升锁

缴费成功
  → 降锁 → ZREM → 仍在停则按计费规则从现在推下次计费起点再 ZADD

泊位无车（parkOut）
  → 强制降锁 + ZREM（优先保证下一辆）

定时对账
  → 该升未升 / 该降未降 补指令
```

## 5. 配置与数据

- **协议出站**：`com.tra.utils.GroundLockUtils`；**预约 ZSET**：`GroundLockRaiseDueUtils`（见 [路内地锁会话与预约升锁](/knowledge/road/ground-lock/ground-lock-session-schedule.md)）
- **DDL**：[地锁绑定与进行中会话表](/knowledge/road/ground-lock/ground-lock-bind-session-ddl.md)（`tra_ground_lock_bind` / `tra_ground_lock_session`）
- **支付降锁**：`GroundLockOrchestrator.onPaid`（`NotifyBase.payBatchArrearNotifyListener`；按 session.orderid；**余额不处理**）
- `groundLock.enabled` / `leaveGraceMs` / `operatorId` / `operatorSecret`
- 已落地：会话 Dao、`createOrder` 预约、`parkOut`/`parkOutForce` 强制降锁、支付 `onPaid`、定时 pollDue
- 仍待落地：对账补降
- 勿用 `ParkLockUtils`（Redis 互斥，非设备协议）

## 6. 联调要点

1. 地磁有车 alone：无 ZADD、无升锁。  
2. PDA 建单后：ZSET 有记录；免费期内不升。  
3. 过计费起点且未付>0：升锁。  
4. 缴费未驶离：降锁，宽限后再升。  
5. 地磁无车：无论结单成败，锁须降。  
6. 对账：无车仍升起 → 补降。

## 7. 与旧逻辑差异

- 升锁不跟 `uploadBerthStatus` / `parkIn`。  
- 视频 `jzPark` MQ **非本场景主路径**（无视频桩）。  
- 公版仅薄挂 TODO；实现优先放 jdz 编排。
