---
title: "App 办理停车场月卡缓存链路"
type: reference
summary: "办理月卡后进出场缓存没更新时，写入键和读取键不是同一把：结果集不带行号取到空，订单 cardcode 变成 0，缓存写到 account_0。"
tags: [月卡, 缓存, 停车场]
created: "2026-11-25"
edges:
  - path: /knowledge/parking/month-card/cache-cardcode-zero.md
    label: about
    description: "这条链路说明缓存键写偏的机制"
  - path: /knowledge/parking/month-card/app-cache-road-entry.md
    label: about
    description: "路内只转发，不写这把缓存"
  - path: /knowledge/parking/month-card/renew-plate-pointer.md
    label: about
    description: "续费还要改车牌到卡号的指针" 
sources:
  - title: "iparkcloud_jdz/doc/App办理停车场月卡缓存未更新排查说明.md"
---

# App 办理停车场月卡：缓存未更新排查说明

## 1. 背景与范围

### 做了什么

针对现象：**App 办理/续费停车场月卡后，进出场用的月卡缓存没有更新**。以 2026-08-29 车牌 `某车牌` 一笔真实单为样本，把「查哪个日志 → 缓存写到哪 → 为什么 cardcode 是 0 → 为什么续费用户会进办理接口」整条链路还原清楚。

### 明确不做的

- 不覆盖路内月卡（`iroad_jdz` 的 `JDZMonthCardInternal.updateCacheForCardInfo`）。
- 进出场 `isCard` 主流程不因本问题改动。

### 已落地修复（2026-09-24）

VIP 快捷申请写订单时，三处统一改为 `getRecordString(0, "cardcode")`，避免 `meb_cardorderlist` / `gnl_meb_cardorderlist.cardcode` 落 null/0。详见 §4.3、§7。

### 样本单

| 项 | 值 |
|----|----|
| 时间 | 2026-08-29 10:38:44 申请，10:38:58 支付写缓存 |
| 车牌 | 某车牌 |
| 场库 ploid | 某车场 |
| 月卡 id | 某月卡 |
| 订单 id | 某订单 |
| 正确月卡编号 MO | 某月卡编号 |
| 客户端 | Android，appVersion=78 |
| 手机号 | 某手机号 |
| 返回文案 | 申请成功，待支付 |

---

## 2. 类 / 接口清单与调用关系

```
Android App
  └─ PloMonthCard.applyPloMonthCardOnline          （iroad_jdz，办理入口）
        └─ LotsUtils → iparkcloud MonthCard.applyPloMonthCardOnline
              └─ VIP 块：addMonthCardOrderOnline（写入 meb_cardorderlist.cardcode）
                    返回 monthcardid（旧卡）+ monthcardorderid（新订单）

约 14 秒后支付
  └─ PloMonthCard.payMonthCardOnline               （iroad_jdz）
        └─ MonthCard.activeMonthCardOnline         （paytype=1 开卡）
              └─ 查不到未激活卡，但订单待支付
                    → 改 paytype=2 + viptotalprice，再调自己
              └─ consumeAmount4cardorder
                    └─ account4card
                          └─ CardCarNumberBase.cacheCardData
                                → CacheUtils.set("account_" + cardcode)
```

| 仓库 | 类 | 职责 | 不要做什么 |
|------|----|------|------------|
| iroad_jdz | `PloMonthCardService` | App 办理/支付转发场库 | 不写 `account_` 缓存 |
| iparkcloud_jdz | `MonthCardService.applyPloMonthCardOnline` | 线上申请；VIP 特殊人群走快捷订单 | 不要当成正常续费入口 |
| iparkcloud_jdz | `MonthCardService.activeMonthCardOnline` | 开卡/续费激活 | paytype=2 才是真正续费写订单 |
| iparkcloud_base | `AccountBase.consumeAmount4cardorder` | 订单改已支付、建钱包、调缓存 | 从**订单表**读 cardcode，不从主表读 |
| iparkcloud_base | `CardCarNumberBase.cacheCardData` | 写 `account_{cardcode}` | Logger 名为 `CardLog` |
| 进出场 | `OrderBase.isCard` | 先 `车牌_ploid` 取 MO，再读 `account_MO` | 不会读 `account_0` |

---

## 3. 关键流程

### 3.1 正常续费（历史 6/5、6/13、7/14 三笔）

用户从「我的月卡 → 续费」进入：

1. `getMonthCardInfoAndRenewDate` 只算时间段，不写库。
2. `payMonthCardOnline(paytype=2)`。
3. `activeMonthCardOnline(paytype=2)` 用 `getRecordString(0, "cardcode")` 复用旧 MO。
4. `addMonthCardRenewOrder` 插入订单，`cardcode = MO...`。
5. `cacheCardData` 更新 `account_MO...`。进出场读同一 key，缓存正确。

**不经过** `applyPloMonthCardOnline`，也就**不会进 VIP 块**。

### 3.2 本次异常（8/29）

用户从「办理月卡」进入：

1. App 调 `applyPloMonthCardOnline`（日志已证实）。
2. 场库查到该车牌该场库已有**未过期、已激活**月卡 → `renewType = 3`。
3. 手机号在特殊人群表 `getCustomerTypeListPlus` 命中 → 进 VIP 块。
4. VIP 块**复用旧 monthcardid**，只插待支付订单，返回「申请成功，待支付」。
5. 订单 `meb_cardorderlist.cardcode` 被写成 **`0`**（见第 4 节）。
6. 支付时 `activeMonthCardOnline(paytype=1)`：查不到未激活卡，但订单 `orderstatus=1`，于是改成 `paytype=2` + `viptotalprice`，**复用这笔 cardcode=0 的订单**，不再 `addMonthCardRenewOrder`。
7. `consumeAmount4cardorder` 从订单读出 `cardcode=0`。
8. `cacheCardData` 写入 **`account_0`**。进出场仍读 **`account_MO...`**，表现为「缓存没更新」。

### 3.3 办理入口里「没进 VIP」会怎样

`applyPloMonthCardOnline` 中 `renewType=3` 但不在特殊人群名单时，会**新建一张月卡**（新 monthcardid + 新 MO），不是原卡续费。支付按开卡（paytype=1）走，并可能注销旧卡。

| 路径 | 入口 | 月卡 MO | 订单 MC | 月卡 id |
|------|------|---------|---------|---------|
| 正常续费 paytype=2 | 续费页 | 复用旧 | 新 | 不变 |
| VIP 快捷申请 | 办理页 + 特殊人群 | 应复用旧（本次写成 0） | 新 | 不变 |
| 办理页 renewType=3 非 VIP | 办理页 | **新 MO** | 新 | **新 id** |

---

## 4. 根因

### 4.1 直接原因：缓存 key 写错

`AccountLog` 显示钱包/订单成功：

```
更新月卡续费订单状态和添加月卡钱包信息成功。... cardid=20260516... carnumber=某车牌
输出结果：{"message":"充值成功","state":1}
```

`CardLog` 同时有更新日志，但：

```
更新月卡缓存时间： cardcode=0 ... map={"cord":[多场库钱包混在一起]}
```

成功日志说明 **一定调用了 `cacheCardData`**。`account4card` 在返回「充值成功」前必调缓存：

- 入参无 `isbind` 时只调 `cacheCardData`（不更新 `carnumber_`）。
- `cacheCardData` 的 key 是 `"account_" + cardcode`。
- cardcode 来自 **`meb_cardorderlist.cardcode`**，不是 `meb_cardorder.cardcode`。

进出场：

```
cardcode = CacheUtils.get(车牌 + "_" + ploid)   // 得到正确 MO
cacheCard = CacheUtils.get("account_" + cardcode)  // 读 account_MO...
```

因此写到 `account_0` 对进出场无效。`cardcode=0` 还会把库里所有 `cardcode='0'` 的月卡钱包拼进同一份缓存（样本里 9 段不同 ploid）。

### 4.2 数据证据

| 表 | 记录 | cardcode |
|----|------|----------|
| `meb_cardorder` 主表 | 某月卡 | **MO202605...**（正确） |
| 历史订单 6/5、6/13、7/14 | 三笔续费 | **MO202605...** |
| 问题订单 8/29 | 某订单 | **0** |

主表一直正确，只有这一笔订单写错。

Java 入参名是 `monthcardcode`，DAO 第三个参数写入 SQL 列 **`cardcode`**，是同一值的两层命名，不是两个字段。

### 4.3 订单写成 0 / null 的代码位置

VIP 块把查询结果里的 `cardcode`（`MO...` 字符串）赋给 `monthcardcode`。

| 错误写法 | 落库表现 | 说明 |
|----------|----------|------|
| `getRecordInt(0, "cardcode")` | `0` | 非数字字符串转 int 得到 0（8/29 样本） |
| `getString("cardcode")` | `null` / 空 | 查询结果集字段不在 ParaMap 顶层（9/21 某车牌） |

**正确写法（已落地）**：`monthCardMap.getRecordString(0, "cardcode")`，与正常续费路径一致。

已改三处 VIP 快捷订单写入：

| 类 | 方法 | 表 |
|----|------|-----|
| `MonthCardService` | `applyPloMonthCardOnline` | `meb_cardorderlist` |
| `GeneralMonthCardService` | `applyPloMonthCardOnline` | `gnl_meb_cardorderlist` |
| `GeneralMonthCardService` | `applyOnedayPloMonthCardOnline` | `gnl_meb_cardorderlist` |

### 4.4 为什么续费用户会进办理接口

后端**不会**根据「已有月卡」自动改走续费接口。入口由 App 决定：

| App 操作 | 接口 | 会否进 VIP 块 |
|----------|------|----------------|
| 办理月卡 | `applyPloMonthCardOnline` | 已有未过期卡 + 特殊人群 → 会 |
| 我的月卡 → 续费 | `payMonthCardOnline(paytype=2)` | 不会 |

VIP 块是办理流程里给特殊人群的**快捷续费**（复用旧卡只插订单），不是替代正常续费。

8/29 路内请求日志已证实：

- `method=applyPloMonthCardOnline`
- 响应 `申请成功，待支付` + 旧 `monthcardid` + 新 `monthcardorderid`
- 无 `getMonthCardInfoAndRenewDate` / `paytype=2`

`renewType == 3` 条件：该车牌该场库已有 `cardstatus=1` 且 `finaltime > now`。再叠加 `getCustomerTypeListPlus(comid, tel)` 有记录（`meb_type` + `meb_type_tel`，`remark is null`）。

---

## 5. 该查哪些日志

`CardLog` / `AccountLog` 在仓库 `log4j.properties` 里**没有独立 appender** 时，默认进 `iparkcloud_jdz/logs/all.txt`（按天滚动可能是 `all.txt.2026-08-29`）。生产若拆了 `CardLog.txt`，以现场为准。

### App / 路内（iroad_jdz）

| 文件/Logger | 关键字 |
|-------------|--------|
| `app/applyPloMonthCardOnline` | `applyPloMonthCardOnline`、会员手机号 |
| `lotsrequest/.../applyPloMonthCardOnline.txt` | 请求 URL、`申请成功，待支付` |
| `app/payMonthCardOnline` | `paytype`、`activeMonthCardOnline` |
| `alipay/payMonthCardOnlineConsumeNotifyListener` | 在线支付回调 |

### 场库（iparkcloud_jdz）

| Logger | 关键字 | 含义 |
|--------|--------|------|
| `MonthCardService` | `applyPloMonthCardOnline inMap` | 收到办理申请 |
| `AccountLog` | `更新月卡续费订单状态和添加月卡钱包信息成功` | 钱包/订单成功 |
| `CardLog` | `更新月卡缓存时间： cardcode=` | **看 cardcode 是否为 0** |
| `CardLog` | `月卡缓存更新成功，但是月卡钱包信息为空` | 查不到钱包，会删缓存 |
| `MonthCardService` | `activeMonthCardOnline,inMap`、`续费 inMap` | 支付激活；看 paytype |

建议命令：

```bash
# 路内：是否走了办理而不是续费
grep applyPloMonthCardOnline iroad_jdz/logs/all.txt*

# 场库：缓存 key
grep -E "更新月卡缓存时间|account_0|cardcode=0" iparkcloud_jdz/logs/all.txt*

# 按订单/车牌
grep "某订单\|某车牌" iparkcloud_jdz/logs/all.txt*
```

---

## 6. 数据约定与核验 SQL

```sql
-- 主表 cardcode 是否正常
SELECT id, cardcode, ploid, finaltime, cardstatus
FROM meb_cardorder
WHERE id = '月卡id';

-- 问题订单是否写成 0
SELECT id, cardid, cardcode, startdate, enddate, orderstatus, payway, paytype, ordertime, paytime
FROM meb_cardorderlist
WHERE id = '订单id';

-- 历史订单对比
SELECT id, cardcode, startdate, enddate, ordertime
FROM meb_cardorderlist
WHERE cardid = '月卡id'
ORDER BY ordertime DESC;

-- 是否特殊人群（VIP 块条件）
SELECT t.id, m.comid, t.tel, t.remark
FROM meb_type m
JOIN meb_type_tel t ON m.id = t.typeid
WHERE m.comid = '场库comid' AND t.tel = '会员手机号' AND m.isvalid = 1;
```

`payway`：1=APP，2=人工。人工续费 `MonthCardRenewService.renewMonthCard` 用 `getRecordString`，与本次无关。

缓存 key：

- `account_{MO编号}`：进出场读的月卡钱包。
- `{车牌}_{ploid}`：车牌到 MO 的映射。
- `carnumber_{MO编号}`：开卡 `isbind=1` 才写；续费默认不传 isbind，不更新。

---

## 7. 联调 / 验收要点与常见坑

### 验收

1. 特殊人群 + 已有未过期月卡，从 App **办理页**申请：订单 `cardcode` 必须等于主表 MO，不能为 `0`。
2. 支付后 `CardLog` 为 `cardcode=MO...`，Redis 有 `account_MO...` 且无脏数据写入 `account_0`。
3. 从 **续费页** paytype=2：不经过 apply，订单 cardcode 仍为旧 MO。
4. 进出场按月卡放行，不按临停。

### 常见坑

1. 只看 `AccountLog` 成功、去翻独立 `CardLog.txt` 找不到 → 多数在 `all.txt`；有 CardLog 也要看 **cardcode 值**。
2. 把 `monthcardordercode`（MC 订单流水号）当成月卡编号。MC 每次新建；MO 才是缓存 key。
3. 把办理页 `renewType=3` 当成正常续费。非 VIP 会**换新卡新 MO**。
4. 生产 jar 与仓库不一致：VIP 块仍可能是 `getRecordInt`。
5. 修了订单 cardcode 后必须再刷一次 `account_MO...`，否则进出场仍读旧缓存。

### 数据应急

把问题订单 `cardcode` 改成主表 MO，再对正确 cardcode 调一次 `cacheCardData`（或等价刷新）。`account_0` 上混有多场库钱包，不要当单卡数据用。

### 代码修复状态（2026-09-24）

1. **已修**：上述三处 VIP 块统一 `getRecordString(0, "cardcode")`，新订单不再写 null/0。
2. **未做（可选加固）**：`consumeAmount4cardorder` 发现订单 cardcode 为空/`0` 时，回退读主表 `meb_cardorder.cardcode`，避免历史脏单再写错缓存。
3. **未做（App）**：已有有效月卡应引导「续费」接口，不要再走 `applyPloMonthCardOnline`。

历史脏单（如 `某历史订单`）仍需按「数据应急」回填 cardcode 并刷 `account_MO...`。

---

## 8. 与旧逻辑的差异

| 点 | 正常续费 | 本次 VIP 办理申请 |
|----|----------|-------------------|
| 入口 | `payMonthCardOnline(paytype=2)` | `applyPloMonthCardOnline` |
| 是否新建月卡 | 否 | 否（复用旧 id） |
| 订单 cardcode | 旧 MO | 写成 0 |
| 支付 paytype | 直接 2 | 先 1，再转 2 + viptotalprice |
| 缓存 | `account_MO...` | `account_0` |
| 禁止改的类 | 不要为修缓存去改进出场 `isCard` 主流程 | 根因在申请写订单 + 激活读订单 cardcode |

路内侧 `PloMonthCardService.applyPloMonthCardOnline` 只转发，**不要改转发协议来「修缓存」**；缓存写在场库 `AccountBase` / `CardCarNumberBase`。
