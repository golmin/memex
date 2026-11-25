---
title: "路内办理月卡不写进出场缓存"
type: reference
summary: "路内办理和支付只转发到场库。进出场读的 account_ 缓存不在这条进程里写。"
tags: [月卡, 缓存, 路内]
created: "2026-11-25"
edges:
  - path: /knowledge/parking/month-card/app-cache-trace.md
    label: about
    description: "场库侧才是缓存写入和读出的位置" 
sources:
  - title: "iroad_jdz/doc/App办理停车场月卡缓存未更新排查说明.md"
---

# App 办理停车场月卡：缓存未更新排查说明（路内侧）

路内只负责办理/支付转发，**不写**进出场用的 `account_` 月卡缓存。完整根因、SQL、场库日志见：

[场库缓存链路](/knowledge/parking/month-card/app-cache-trace.md)

本文只记路内侧入口、日志和与「正常续费」的差异，避免只查 iroad 时对不上场库现象。

---

## 背景与范围

- **现象**：App 办理/续费停车场月卡后，场库进出场仍按旧缓存（或临停）。
- **样本**：2026-08-29 10:38，某车牌，Android appVersion=78。
- **不做**：不改路内主流程、不覆盖路内月卡缓存（`queryCardInfo_SE`）、不覆盖通用月卡（`PloGeneralMonthCardService`）。

---

## 类与调用

| 类 | 职责 | 不要做什么 |
|----|------|------------|
| `PloMonthCardService.applyPloMonthCardOnline` | 办理申请，转发场库 `MonthCard.applyPloMonthCardOnline` | 不要在这里写 Redis 月卡缓存 |
| `PloMonthCardService.payMonthCardOnline` | 支付；`paytype` 1=开卡 2=续费 | 不要把办理接口当续费 |
| `PloMonthCardService.getMonthCardInfoAndRenewDate` | 续费页拉时间段 | 办理页不会调它 |

---

## 两条入口（后端不会自动改走续费）

| App | 路内方法 | 场库 | 缓存 |
|-----|----------|------|------|
| 办理月卡 | `applyPloMonthCardOnline` | 可能进 VIP 块，订单 cardcode 写成 0 | 写到 `account_0` |
| 我的月卡 → 续费 | `payMonthCardOnline(paytype=2)` | `activeMonthCardOnline` 复用旧 MO | 写到 `account_MO...` |

样本日志（iroad）：

```
app/PloMonthCardService.applyPloMonthCardOnline  会员手机号 某手机号
lotsrequest/.../applyPloMonthCardOnline.txt
  请求 method=applyPloMonthCardOnline carnumber=某车牌
  响应 申请成功，待支付
       monthcardid=某月卡（旧卡）
       monthcardorderid=某订单
```

约 14 秒后场库 `AccountLog` + `CardLog cardcode=0`。没有 `getMonthCardInfoAndRenewDate`，说明不是续费页。

VIP 条件在场库：已有未过期卡（`renewType=3`）+ `getCustomerTypeListPlus` 特殊人群。路内只看到「申请成功，待支付」且 **monthcardid 仍是旧卡 id**。

---

## 路内该查的日志

| 文件 | 关键字 |
|------|--------|
| `app/applyPloMonthCardOnline.txt` | 办理申请 |
| `lotsrequest/{comid}/applyPloMonthCardOnline.txt` | 场库请求/响应 |
| `framework/app_PloMonthCard_applyPloMonthCardOnline.txt` | 入参 ClientType、appVersion |
| `app/payMonthCardOnline` | 支付时 `paytype` 是 1 还是 2 |
| `alipay/payMonthCardOnlineConsumeNotifyListener` | 微信/支付宝回调后再激活 |

```bash
grep applyPloMonthCardOnline iroad_jdz/logs/all.txt*
grep -E "payMonthCardOnline|paytype" iroad_jdz/logs/all.txt*
```

缓存是否更新、cardcode=0，在 **iparkcloud** 的 `AccountLog` / `CardLog`（或 `logs/all.txt`）。

---

## 联调要点

1. 已有有效月卡应从续费入口走 `paytype=2`，不应再调 `applyPloMonthCardOnline`。
2. 办理接口返回「申请成功，待支付」且 monthcardid 与旧卡相同 → 场库 VIP 快捷续费，需核对订单 `cardcode`。
3. 禁止为修缓存去改路内转发协议或进出场主流程；改场库写订单 / `cacheCardData` 的 cardcode。

禁止改的类：`PloMonthCardService` 的转发参数（comid/module/service/method）不要为修缓存乱动。
