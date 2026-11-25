---
title: "路内扫码当前单与历史欠费合并缴费"
type: reference
summary: "泊位扫码把当前进行中订单和勾选的历史欠费收成一笔金服消费。未勾选仍走原来的单笔缴费。合计金额不能塞进旧的 payGoingOrders。"
tags: [路内, 扫码, 欠费]
created: "2026-11-25"
edges:
  - path: /knowledge/road/scan/merge-pay-not-old-consume.md
    label: about
    description: "合计金额为什么不能进旧缴费，回调为什么要先结当前单"
  - path: /knowledge/road/ground-lock/ground-lock-pay-keep-parking.md
    label: about
    description: "付款不结单已回退，合并缴费线上仍是付款结单"
sources:
  - title: "iroad_jdz/doc/路内扫码当前单与历史欠费合并缴费说明.md"
---

# 路内扫码：当前单 + 历史欠费合并缴费

## 1. 背景与范围

- **场景**：用户扫泊位码后，先看到**当前进行中订单**，再按该单车牌列出**未缴欠费**，前端勾选若干笔，一次支付同时结算当前单并对历史欠费补缴。
- **参考**：路外 `com.scanpay.service.ScanPayService#payOrder` 的 `roadOrderIds` + `_FA_BACK_` 缓存、回调再补缴。
- **约束**：不改造现有方法，只加新方法。禁止改：
  - `JZBatchPayment.payGoingOrders` / `publicOnlineConsume`
  - `NotifyBase.payBatchArrearNotifyListener`
  - `BatchPayment.getGoingOrderInfo`
  - `JingDeZhenCalc`
- **不做**：不改旧扫码详情/单笔缴费接口；未勾选欠费时转回原渠道缴费（金额仍只校验当前单）。
- **对齐原扫码**：
  - 微信：`Command('wechat','StdWechat','getGoingOrderInfoByWeChat')` → `BatchPayment#getGoingOrderInfo`；缴费 `payGoingOrdersByWeChat` → **`JZBatchPayment#payGoingOrders`**。
  - 支付宝：`Command('alipay','StdAlipay','getGoingOrderInfoByAlipay')` → `BatchPayment#getGoingOrderInfo`；缴费 `payGoingOrdersByAlipay` → **`BatchPayment#payGoingOrders`**。
  - 新接口只在这两步外包一层泊位查欠费 / 合并下单。未选欠费按 openid / alipayopenid 分流到上面两条旧链路。
- **付款不结单**：曾实现过地锁泊位「付完不结单、驶离再结」，**2026-11-17 已回退**；恢复步骤见 [地锁泊位付款不结单](/knowledge/road/ground-lock/ground-lock-pay-keep-parking.md)。

## 2. 类 / 接口清单与调用关系

| 类 | 职责 | 不要做什么 |
|----|------|------------|
| `iroad_jdz` `com.wechat.internal.GoingArrearScanPay` | 查询、合并下单、回调业务 | 不改旧支付/回调方法 |
| `StdWechatService` | 微信新接口转发 | 不改 `payGoingOrdersByWeChat` |
| `StdAlipayService` | 支付宝新接口转发 | 不改 `payGoingOrdersByAlipay` |
| `NotifyService#payGoingWithArrearsNotifyListener` | 金服新回调入口 | 不改 `payBatchArrearNotifyListener` |

```text
扫码页
  → StdWechat/StdAlipay.getGoingOrderWithArrearsByBerth
      → OrderCenter.getOrderInfoByBerth(comid, berthcode)
      → com.wechat.internal.BatchPayment.getGoingOrderInfo（同 getGoingOrderInfoByAlipay）
      → BillBase.getArrearsByPlate（滤 billstate!=2，排除当前主单）

缴费
  → StdWechat/StdAlipay.payGoingOrderWithArrears
      → 未选欠费：微信 JZBatchPayment.payGoingOrders（同 payGoingOrdersByWeChat）
                 支付宝 BatchPayment.payGoingOrders（同 payGoingOrdersByAlipay）
      → 已选欠费：金服 Order.consume（合计金额 + batch）
                  缓存 jdz:scanpay:going_arrears:{goingOrderId}

金服回调
  → Notify.payGoingWithArrearsNotifyListener
      → GoingArrearScanPay.handleNotify
          → 用 sub_no / 欠费映射折算当前主单（batch 会通知两行）
          → 同一 sub_no 已处理则 success
          → updateOrderStatus(当前单, ispay=1)；state=1 即成功（不要求结束订单）
          → onPaid + 按缓存逐笔补缴欠费
```

## 3. 关键流程

### 3.1 查询

1. `comid` + `berthcode` 取进行中订单；无单 `state=2`。
2. 用当前单 `orderid` 调已有 `getGoingOrderInfo`，得到金额/折扣/路段（字段与旧扫码详情一致，另补 `orderid`、`carplate`）。
3. `getArrearsByPlate` 拉同车牌欠费：`billstate==2` 跳过；`orderid` 等于当前主单跳过；`comid` 不一致跳过。
4. 返回 `currentOrder`、`arrears`、`arrearTotal`、`goingShouldpay`、`payableTotal`（当前 + **全部**欠费，勾选前参考）。

### 3.2 支付

1. `arrearOrderIds` 可空，逗号或下划线分隔（对齐路外 `roadOrderIds` 的 `_`）。
2. 为空：按渠道转原缴费；`total_fee` 只含当前单。微信 `JZBatchPayment#payGoingOrders`，支付宝 `BatchPayment#payGoingOrders`。
3. 非空：再查 `getArrearsInfo`，已补缴（`billstate==2`）抛错；`total_fee` 必须等于「当前应付 + 所选欠费未付」。
4. 金服一笔：`no=当前单`，`total_fee/actual_fee=合计`，`operate=3`（补交欠款），`batch` 挂当前单 + 各欠费。
5. 下单成功后缓存欠费列表，TTL **10800 秒**（对齐路外 `_FA_BACK_`）。

### 3.3 回调

1. 验签，`status==1`。
2. 金服 batch 会对当前单行、欠费行各通知一次。`consumeno` 可能是欠费主单，须用缓存折回进行中主单，禁止对欠费单走进行中 `apply`。
3. 同一 `sub_no` 已入账：直接 `success`（幂等）。
4. 当前单 `updateOrderStatus` + `ispay=1`（金服已收款，禁止再扣余额）。
5. 成功条件对齐单独缴进行中单：`state=1` 且 `isFinishOrder` 非 false；结单失败回 `fail` 让金服重试。
6. 入账成功则 `onPaid`，再按缓存补缴历史欠费；单笔异常只打日志，不回滚当前单。
7. （已回退）付款不结单方案见 [地锁泊位付款不结单](/knowledge/road/ground-lock/ground-lock-pay-keep-parking.md)，线上当前仍为付款结单。

## 4. 前端调用约定

微信：`module=wechat`，`service=StdWechat`  
支付宝：`module=alipay`，`service=StdAlipay`

### 4.1 查询 `getGoingOrderWithArrearsByBerth`

| 入参 | 必填 | 说明 |
|------|------|------|
| comid | 是 | 管理公司 |
| berthcode | 是 | 泊位号 |
| paymentway | 否 | 支付方式，算折扣 |

| 出参 | 说明 |
|------|------|
| state | 1 成功，2 无进行中单/参数错 |
| currentOrder | 同 `getGoingOrderInfo`，含 `shouldpay`、`orderid`、`carplate` |
| arrears[] | `arrearid`/`orderid`/`ordercode`/`carplate`/`berthcode`/`intime`/`outtime`/`shouldpay`/`actualpay`/`billstart`/`mebid` |
| arrearCount / arrearTotal | 未缴笔数与合计 |
| goingShouldpay | 当前单应付 |
| payableTotal | 当前 + 全部欠费（勾选前参考，实付以勾选为准） |

### 4.2 缴费 `payGoingOrderWithArrears`

| 入参 | 必填 | 说明 |
|------|------|------|
| orderid | 是 | 当前进行中订单 id |
| comid | 是 | 管理公司 |
| total_fee | 是 | **当前应付 + 所选欠费**；未选欠费则只传当前应付 |
| arrearOrderIds | 否 | 勾选的历史**主订单 id**，逗号或 `_` 分隔 |
| openid | 微信必传 | |
| alipayopenid | 支付宝必传 | |
| returnUrl | 聚合支付时 | `jh_set.on_off=1` 时必传 |
| userid / mebid / paymentway / way | 否 | `way=11` 支付宝 JSAPI |

成功：`state=1`，`payUrl`（JSAPI 另有 `tradeNo`）。  
失败：`2` 已结算/参数，`4` 金服失败，`5` 金额变化（需重新查单）。

## 5. 配置与数据约定

Redis key、TTL、回调锁秒数均在 `GoingArrearScanPay` **类内写死**，不读 xml / 参数表，发布即可用。

| 项 | 值 | 来源 |
|----|----|------|
| 回调 URL | `{notify_url}redirect-module_alipay__service_Notify__method_payGoingWithArrearsNotifyListener`（可带 `__userid_`） | 沿用现网 `notify_url` |
| 欠费缓存 | `jdz:scanpay:going_arrears:{goingOrderId}`，JSON 数组 `orderid/arrearid/shouldpay/mebid` | 类内写死，TTL=10800 |
| 欠费→当前单 | `jdz:scanpay:arrear_going:{欠费主单id}` | 类内写死 |
| sub_no→当前单 | `jdz:scanpay:going_subno:{sub_no}` | 类内写死 |
| 入账幂等 | `jdz:scanpay:going_done:{sub_no}` | 类内写死 |
| 回调锁 | `parkLock_jdz_scanpay_notify_{goingOrderId}`，60 秒 | 类内写死 |
| 结束时间缓存 | `scanpay_endtime_{orderid}`，TTL 172800 | 与旧扫码一致 |
| 金服 | `Order.consume`，`is_online=2`，`operate=3` | 代码写死 |
| 日志 | 业务 `GoingArrearScanPay`；回调 `paylog/going_arrear_notify` | 代码写死 |

无新增白名单、无新表、无新配置项。

## 6. 联调 / 验收

1. 泊位无进行中单：查询 `state=2`。
2. 有当前单、无欠费：`arrears=[]`，缴费不传 `arrearOrderIds`。微信走 `JZBatchPayment#payGoingOrders`（同 `payGoingOrdersByWeChat`），支付宝走 `BatchPayment#payGoingOrders`（同 `payGoingOrdersByAlipay`），回调仍是旧 `payBatchArrearNotifyListener`。
3. 有欠费：列表不含当前主单、不含已补缴（`billstate=2`）。
4. 勾选 1～N 笔：`total_fee` = 当前 `shouldpay` + 所选未付；金服金额为合计。
5. `total_fee` 故意偏 0.01：`state=5`。
6. 勾选已补缴单：接口抛错（已补缴）。
7. 支付成功：当前单写入付款单（车可仍在场、订单仍进行中）+ 地锁 `onPaid`；勾选欠费 `billstate=2`。金服收到 `success`，不再 30/60 秒重试。
8. 车未驶离再停一段时间后扫码：只付新增差额，不把已付金额再收一遍。
9. 微信 / 支付宝各走一遍；`way=11` 看 `tradeNo`。

### 常见坑

- **不要**把合计金额塞进旧 `payGoingOrders`（微信 `JZBatchPayment`、支付宝 `BatchPayment`）：`publicOnlineConsume` 用当前单金额校验 `total_fee`，会 `state=5`。未选欠费才转旧链路。
- `arrearOrderIds` 传**主订单 id**（`arrears[].orderid`），不是 `arrearid`。
- 未选欠费时 `total_fee` 只能是当前单，否则旧链路金额校验失败。
- 回调必须 `ispay=1`，否则会再走余额。
- **不要**在 `isFinishOrder=false` 时回 `fail`：那是车还在场，不是入账失败。单独缴费旧回调只认 `state=1`。
- 金服已扣款但路内 `actualpay` 仍为 0 时，再扫码会按全额再下单。回调必须先写付款单。
- 当前单已入账后不要对整笔回 `fail` 让金服重试。
- `payableTotal` 是「全部欠费」参考值，实付按勾选重算。

## 7. 与旧逻辑的差异

| | 旧扫码 `payGoingOrders` | 本新接口 |
|--|-------------------------|----------|
| 查询 | 订单 id → `getGoingOrderInfo` | 泊位 → 当前单 + 车牌欠费 |
| 未选欠费缴费 | 微信 `JZBatchPayment` / 支付宝 `BatchPayment` | 有欠费才走新合并下单；未选则按渠道回旧方法 |
| 回调 | 微信单独缴进行中：`payOnlineConsumeNotifyListener`（`state=1` 即 success） | `payGoingWithArrearsNotifyListener`（同样 `state=1` 即 success，车在场不结束订单） |
| 欠费记忆 | 无 | Redis，对齐路外 `_FA_BACK_` |
| 地锁 | 旧回调 `onPaid` | 仅当前单 `onPaid`，欠费不降锁 |

复制本链路时：**不要改** `payGoingOrders`、`publicOnlineConsume`、`payBatchArrearNotifyListener`、`getGoingOrderInfo`。
