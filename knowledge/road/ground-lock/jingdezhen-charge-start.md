---
title: "景德镇计费开始时间推算"
type: reference
summary: "按收费窗和 0 元费率档推算真正开始产生费用的时刻，供地锁预约升锁，不使用试算返回的 startparktime。"
tags: [地锁, 计费]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "这条说明被路内地锁开发记忆引用"
  - path: /knowledge/road/ground-lock/ground-lock-session-schedule.md
    label: used_by
    description: "预约升锁的到期时间调用本推算"
sources:
  - title: "iroad_base/doc/景德镇计费开始时间推算说明.md"
---

# 景德镇计费开始时间推算

## 背景与范围

- **做了什么**：按 `JingDeZhenCalc#getPayMoney` 的**收费标准明细**，算出「真正开始产生费用」的毫秒，给地锁预约升锁用。
- **明确不做**：不算金额；不改 `getPayMoney`；不读库表 `billstart`；不处理新能源减免、月卡/白名单。
- **为何另写**：`getNeedPayMoney` 出参 `startparktime` 注释是「开始计费时间」，实测常等于驶入时间。

## 真正起作用的是明细，不是头上两项

后台「编辑收费策略」里：

| 位置 | 字段 | 这张「正式环」截图 | 对计费起点的影响 |
|------|------|-------------------|------------------|
| 策略头 | 免费时长 `freeduration` | **0 分钟** | 没有「前 N 分钟墙钟免费」 |
| 策略头 | 免费时长是否收费 `freecharging` | **是（=2）** | 头上免费分钟为 0 时**不起作用** |
| 策略头 | 跨时段是否连续 `timespan` | **连续（=1）** | 0 元档分钟跨夜累计，不重置 |
| 明细 | 收费类型 + 起止 | **按时 07:30–21:00** | 窗外（21:00–07:30）是隐含免费，不升锁 |
| 明细 | 计费规则第一档 | **0–120 分钟，0 元/120 分钟** | **真正的免费**：进窗后再停满 120 分钟才出第一笔钱 |
| 明细 | 第二档起 | 120–240 分钟 5 元… | 进入这档的时刻才是 `chargeStart` |

所以 2026-10-13 10:23 小型车驶入：10:23 已在 07:30–21:00 窗内，再加 120 分钟 → **12:23** 才是计费起点。之前只看 `freecharging/freeduration` 会错判成等于驶入时间。

新能源第一档是 1.5 元/120 分钟（不是 0），必须传对 `vehicletype`，默认 1=小型车。

## 类 / 接口清单

| 类 | 路径 | 职责 |
|----|------|------|
| `JingDeZhenChargeStart` | `iroad_base` `com.rule.internal` | 推算函数本体 |
| `RuleBase#getChargeStartTime` | `iroad_base` `com.rule.base` | 对外入口 |
| `GroundLockOrchestrator#resolveChargeStartMs` | `iroad_base` `com.tra.utils` | 地锁预约改走本函数 |

```text
createOrder / pollDue 改约
  → GroundLockOrchestrator
       → JingDeZhenChargeStart.resolve
            → RuleDao 策略 / 时段 / 费率 / 节假日
            → 收费窗 + 0 元费率档往前走

其它业务
  → RuleBase.getChargeStartTime
       → JingDeZhenChargeStart.resolve
```

## 关键流程

```text
查泊位策略 + 时段 + 费率
  → 收费窗 = 按时/按次（如 07:30–21:00），再挖掉 chargingmethod=3
  → 窗外直接跳到下一收费窗起点
  → 窗内依次扣：时段 freetime → 头上 freeduration（仅 freecharging=1 且 >0）→ 费率 rateprice=0 的前缀分钟
  → 落到第一档 rateprice>0 的时刻 = chargeStart
  → 若 freecharging=2 且 freeduration>0：再与「驶入+墙钟」取较晚者
```

正式环、小型车、`timespan=连续` 示例：

| 驶入 | 计费起点 | 原因 |
|------|----------|------|
| 10:23 | 12:23 | 窗内，0 元档 120 分钟 |
| 06:00 | 09:30 | 先等到 07:30，再加 120 分钟 |
| 20:30 | 次日 09:00 | 当天只剩 30 分钟计入 0 元档，次日 07:30 再走 90 分钟 |
| 21:30 | 次日 09:30 | 窗外，次日 07:30 + 120 分钟 |

`timespan=重置` 时，20:30 驶入变为次日 09:30（新窗重新走满 120 分钟）。

## 配置 / 数据约定

- 时段：`getPricingTimeById`（`com_pricingmethod`）
- 费率：`getPricingRateById`（`com_pricingrate`：`ratestartminute/rateendminute/rateunit/rateprice`）
- 日期：`getAllHolidayList` + `RoadUtils.getDateType`（工作日/非工作日）
- `RoadUtils.select` 按空格切条件，**不能**写带括号的 where
- 跨夜钟点按 +24 小时；`23:59` 补到当天最后一毫秒
- 费率 `rateendminute=0` 且 `rateprice=0`：后面永远不出钱，最多往前走 7 天
- 出参失败时 `chargeStart` 回退驶入时间
- 日志：`chargeWins=07:30-21:00` `zeroRateMin=120` `vehicletype` `datetype` `timespan`

## 不要挂到哪里

- **禁止**在 `ParkingService.getOrderNeedPayPrice` / `getNeedPayMoney` 里调 `getChargeStartTime`：查价是热路径。
- 联调请单独调 `RuleBase.getChargeStartTime` 或看地锁 `onOrderCreated` 日志。

## 联调 / 验收

1. 正式环、工作日小型车、10:23 驶入：`chargeStart` 约为 12:23，日志 `zeroRateMin=120` `chargeWins=07:30-21:00`。  
2. 06:00 驶入：约为 09:30。  
3. 20:30 驶入且跨时段连续：约为次日 09:00。  
4. 新能源车（第一档不是 0 元）：进窗即出钱，`zeroRateMin=0`。  
5. 对比 `getNeedPayMoney.startparktime`：后者往往仍是驶入，**以本函数为准**。  
6. 地锁：建单后 ZSET score 应等于本函数；0 元档内 `pollDue` 不应升锁。  
7. 泊位 320022 若无地锁绑定，`onOrderCreated` 会跳过，与计费起点无关。

## 与旧逻辑差异

- **禁止**再把 `getNeedPayMoney` 的 `startparktime` 当计费起点。  
- **禁止**改 `JingDeZhenCalc#getPayMoney`。  
- **不要**只拿策略头 `freecharging/freeduration` 当免费依据；正式环这两项为 0，免费在明细 0 元档。  
- `RuleInfo#getBillStartTimeByBerthCode` 只做「驶入+freeduration」，不含明细费率，本函数不调用它。
