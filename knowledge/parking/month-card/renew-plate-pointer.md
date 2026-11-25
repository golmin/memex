---
title: "月卡续费要同时改车牌指针"
type: reference
summary: "出场先用车牌加车场取出卡号，再读 account_卡号。只更新卡号上的内容、不改指针，读到的仍是另一张卡。"
tags: [月卡, 缓存, 停车场]
created: "2026-11-25"
edges:
  - path: /knowledge/parking/month-card/cache-cardcode-zero.md
    label: about
    description: "卡号本身写错时，指针和内容会指向不同的键" 
sources:
  - title: "iparkcloud_jdz/doc/停车场月卡续费同步车牌缓存说明.md"
---

# 停车场月卡续费：同步 `车牌_ploid` 缓存说明

## 1. 背景与范围

### 做了什么

App / 一体化后台续费成功后，除原有 `cacheCardData`（写 `account_{MO}`）外，**补调 `cacheCardCode`**，把 `车牌_ploid` 指到**本次续费的 MO**。

用于消除：续费已成功、短信有效期已更新，但出场仍读历史另一张月卡有效期的问题（样本：某车牌，续费 `MO202504…`，出场指针仍在过期的 `MO202506…`）。

### 明确不做的

- **不用**续费路径传 `isbind=1`（见第 5 节隐患）。
- 不自动注销同车牌同场库的其它有效/过期月卡（数据治理另做）。
- 不改路内月卡、通用月卡（`gnl_`）缓存。
- 不改开卡（`paytype=1`）已有 `isbind=1` 逻辑。

---

## 2. 类 / 接口与调用关系

```
App payMonthCardOnline → activeMonthCardOnline(paytype=2)
  └─ AccountBase.consumeAmount4cardorder   // 写钱包 + cacheCardData(account_MO)
  └─ CardCarNumberBase.cacheCardCode       // 【新增】写 车牌_ploid = 本次 MO

后台 MonthCardRenewService.续费
  └─ consumeAmount4cardorder
  └─ cacheCardCode                         // 【新增】同上
```

| 仓库 | 类 / 方法 | 职责 |
|------|-----------|------|
| iparkcloud_jdz | `MonthCardService.activeMonthCardOnline` 续费分支 | App 续费后补 `cacheCardCode` |
| iparkcloud_jdz | `MonthCardRenewService` 人工续费 | 后台续费后补 `cacheCardCode` |
| iparkcloud_base | `CardCarNumberBase.cacheCardCode` | `CacheUtils.set(车牌_ploid, cardcode)`，支持逗号多车牌 |
| 出场 | `CardCarNumberBase.getCardCache` | 先 `车牌_ploid` 再 `account_MO` |

---

## 3. 关键流程（续费后）

1. 按 App/后台传入的 `monthcardid` 续费，订单 `cardcode` = 该卡 MO。
2. `consumeAmount4cardorder` 建钱包段并更新 `account_{MO}`。
3. **新增**：`cacheCardCode(carnumber, cardcode, ploid, id)`，把绑定车牌全部写成 `车牌_ploid → 本次 MO`。
4. 出场：`getCardCache` 读到本次 MO → 新有效期。

---

## 4. 能否避免本次错误？

**能避免同类「续费成功但出场仍旧」问题**，前提是：

- 用户续费的那张卡，就是业务上要用的当前卡（本次样本中 App 列表只展示未过期的 4 月卡，续费对象正确）。
- 补写缓存成功（失败只打日志，不回滚已成功续费；需盯 `续费同步车牌月卡缓存` 日志）。

不能单独解决：

- 同车牌同场多张 `cardstatus=1` 脏数据；
- 已发生历史错位、尚未再次续费的存量单（仍需运维改 Redis / 刷 `cacheCardCode`）。

---

## 5. 为何不用 `isbind=1`（隐患）

`account4card` 在 `isbind=1` 时还会：

1. **`bindCardCarnumber`**：场内有临停订单时返回失败 → **续费整笔失败**（用户已支付场景风险大）。
2. 按差集增删车牌绑定，续费场景不需要改绑定关系。
3. 再调 `cacheCarnumber` + `cacheCardCode`。

因此续费只补 **`cacheCardCode`**，不动绑定表、不拦场内车。

### 其它残余隐患（需知悉）

| 隐患 | 说明 | 建议 |
|------|------|------|
| 续错卡 | 若 App 续了非「出场应认」的卡，指针会跟着指过去 | 办理/续费前禁止双有效卡；列表与出场口径对齐 |
| 一卡多车 | `cacheCardCode` 会刷新该卡 `GROUP_CONCAT` 的所有车牌指针 | 符合「这张卡续费后出场都应认这张」 |
| 另一张未过期卡 | Redis 指针改后，另一张卡在库里仍可能 status=1 | 数据清理 / 续费前校验 |
| 缓存失败 | 补写失败不回滚续费 | 日志告警 + 运维重刷 |
| 其它只刷 `account_` 的路径 | 如部分线上续费列表、作废订单后只 `cacheCardData` | 按需同样补 `cacheCardCode` |

---

## 6. 联调 / 验收

1. 构造：`车牌_ploid` 先指向旧 MO，再对另一张有效卡续费。
2. 查 Redis：`车牌_ploid` 应变为续费卡 MO；`account_{新MO}` 含新 enddate。
3. 出场应走月卡，有效期与短信一致。
4. 场内有临停订单时续费仍应成功（验证未误用 `isbind=1`）。
5. 日志关键字：`续费同步车牌月卡缓存` / `后台续费同步车牌月卡缓存`。

---

## 7. 与旧逻辑差异

| 项 | 旧 | 新 |
|----|----|----|
| 续费后 Redis | 只更新 `account_MO` | + 更新 `车牌_ploid` |
| `isbind` | 续费不传 | 仍不传（刻意） |
| 开卡 paytype=1 | 已 `isbind=1` | **禁止改** |
