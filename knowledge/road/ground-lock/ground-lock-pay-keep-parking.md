---
title: "地锁泊位付款不结单"
type: reference
summary: "付款后降锁但订单不结束、继续计费的方案已回退；需要时按本文步骤加回。"
tags: [地锁, 支付]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "这条说明被路内地锁开发记忆引用"
  - path: /knowledge/road/scan/merge-pay-not-old-consume.md
    label: about
    description: "合并缴费回调里 isFinishOrder 为 false 仍表示结单没完成"
sources:
  - title: "iroad_jdz/doc/地锁泊位付款不结单说明.md"
---

# 地锁泊位：付款入账不结单（已回退，待启用）

> **状态（2026-11-17）**：线上暂不需要，相关代码已从仓库工作区回退。  
> 本文档保留完整落地记忆，需要时按下文「如何重新启用」加回。

## 1. 目标行为（业务）

付款回调成功 → **降锁**；车未驶离则 **订单不结束、继续计费**；定时捞取发现「现算总价 − actualpay − 抵扣 &gt; 0」→ **升锁**；**车辆驶离**（地磁）再 `finishParkOrder` 结单。

前提约定（当时确认）：地锁泊位一定有地磁；只走新微信/支付宝扫码（`payGoingOrderWithArrears`），无 PDA 现金场景。

## 2. 为何不能只改回调里的 isFinishOrder=false

景德镇 `countdownset=false`，旧/新回调里正常付款走 `APPLYTYPE_POSTPAID`，`apply` 里会先结单。`isFinishOrder=false` 表示 **结单失败**，不是「故意不结单」。只把回调改成 success 达不到「继续计费」，还会跳过重试。

也不要打开全局 `countdownset` / 复用 `APPLYTYPE_RENEW`：会带西安锁单冻结、自动驶离，且续费计费对景德镇免费前缀会算错。

## 3. 推荐实现（最小侵入）

### 3.1 新增 `GoingKeepParkingPay`（iroad_jdz）

路径建议：`std/weixin/com/wechat/internal/GoingKeepParkingPay.java`

- 仅由 `GoingArrearScanPay#handleNotify` 调用。
- `orderPayment` + `orderUpdate` 累加 `actualpay`（及可选 `shouldpay`），**不调** `finishParkOrder`。
- 按金服 `sub_no` / `consumesubno` 对 `tra_pay_road.id` **幂等**。
- 同订单 Redis/ParkLock 短锁防重入。
- 金额优先读下单缓存 `jdz:scanpay:going_amt:{orderId}`，避免 batch 合计误入进行中单。
- 写 `orderendtime_{orderId}=payTimeMs,15`（TTL≈20 分钟），供驶离宽限内按付款时刻结单。
- 成功返回 `state=1`、`keepParking=true`，**不要带** `isFinishOrder=false`。

### 3.2 改 `GoingArrearScanPay`（不要动旧 payGoingOrders）

1. **下单**：无论是否勾选欠费，一律本类 `createCombinedConsume` + `payGoingWithArrearsNotifyListener`；**不要**再 `payGoingOrdersByOrigin`（旧回调会结单）。
2. 下单成功缓存 `going_amt` + 可选 `going_arrears`。
3. **回调**：进行中单 → `payWithoutFinish` → `onPaid`；历史欠费仍 `updateOrderStatus`。
4. 补强：`arrearOrderId → goingOrderId` 反查缓存（注释里写过、上次实现未做完），防止 batch 先通知欠费行导致进行中单漏入账。

### 3.3 地锁编排（iroad_base，可选小改）

- `onPaid` 仍在停：`due = max(规则推算, now + leaveGraceMs)`；可给 `GroundLockRaiseDueUtils` 加 `getLeaveGraceMs()`。
- `tryRaiseForOrder` 的「未付&gt;0 才升」可保持不动。
- **不要改** `createOrder` / `descend` / 旧 `payBatchArrearNotifyListener` 结单条件。

### 3.4 明确不改

- `JZBatchPayment#payGoingOrders`、`NotifyBase#payBatchArrearNotifyListener` 主体
- `JZStdOrderService#updateOrderStatus` 主体
- 全局 `countdownset`

## 4. 回退后的线上行为（当前）

| 路径 | 行为 |
|------|------|
| 新扫码选了欠费 | 合并 consume + 新回调 → `updateOrderStatus` → **结单** + `onPaid` |
| 新扫码未选欠费 | 转旧 `payGoingOrders` → `payBatchArrearNotifyListener` → **结单** + `onPaid` |
| 旧扫码 | 不变，付款结单 |
| 驶离 | `parkOut` → `finishParkOrder` + `descend`（未改） |

## 5. 如何重新启用

1. 按 §3.1 恢复 `GoingKeepParkingPay.java`（可从本对话/Agent 历史或 Git 未提交草稿找回）。
2. 按 §3.2 改 `GoingArrearScanPay` 下单与 `handleNotify`。
3. （建议）`onPaid` 加上 `leaveGraceMs` 下限。
4. 同步更新本文状态为「已启用」，并改 [路内扫码当前单与历史欠费合并缴费](/knowledge/road/scan/going-arrear-merge-pay.md) 的 3.2/3.3。
5. 联调：付完单仍进行中、降锁、宽限内驶离无欠费、超时未付再升锁、同 `sub_no` 幂等、旧扫码仍结单。

## 6. 已知坑（启用时必看）

1. `resolveGoingOrderId` 必须能从欠费单号反查进行中主单。
2. `going_amt` 丢失时勿直接用 batch 合计 `actualfee`。
3. 累加库内 `shouldpay` 可能影响报表口径；扫码查询以 `getGoingOrderInfo` 的「现算−已付」为准。
4. `nextRaiseDueByRule(fromMs=now)` 会把免费前缀再推一遍，升锁可能偏晚；长期应用「从驶入累算第一次超过已付的时刻」。
