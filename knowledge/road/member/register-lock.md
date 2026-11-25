---
title: "注册先查后插的互斥"
type: reference
summary: "注册是先查手机号再插入。没有唯一约束时，互斥只存在于实际加载了这段代码的进程里。"
tags: [注册, 并发]
created: "2026-11-25"
edges:
  - path: /knowledge/road/member/register-old-bytecode.md
    label: about
    description: "锁写进源码之后，请求仍可能跑在旧字节码上" 
sources:
  - title: "iroad_jdz/doc/App注册并发防重复会员说明.md"
---

# App 注册并发防重复会员说明

## 背景与范围

### 做了什么

App 端 `registerMeb` 存在「先 `checkPhoneNum` 再 `INSERT meb_info`」的竞态：客户端连点/重试时，多个请求在毫秒级内都查到「未注册」，各自插入，导致同一手机号出现多条有效会员（线上曾出现 2 条、甚至 7 条）。

本次在 **不改表结构、不改 `regstatus` 分支含义、不改 insert SQL** 的前提下，仅在注册入口按手机号增加短时互斥锁，挡住重复插入。

### 明确不做的

- 不为 `meb_info.tel` 加/恢复唯一索引
- 不清理历史重复数据（需运维另行合并）
- 不改微信/支付宝等其它注册链路（若旁路也会插 `meb_info`，需另行套同一锁 key）
- 不改为严格 Redis `SET NX`（沿用现有 `ParkLockUtils` 约定）

## 类/接口清单与调用关系

```
App → StdService.registerMeb
         → MebServiceExt.registerMeb   ← 本次加锁点
              → checkPhoneNum（原逻辑，读库）
              → register / activateMeb（原逻辑）
         → ParkLockUtils（Redis 缓存锁）
```

| 类/方法 | 职责 |
|---------|------|
| `com.app.service.MebServiceExt#registerMeb` | App 注册/激活；加锁、原分支、按结果决定是否保留锁 |
| `com.parking.utils.ParkLockUtils` | `checkSyncLock` / `addSyncLock` / `removeSyncLock` |
| `StdService.registerMeb` | 入口转发，未改 |

锁 key：`parkLock_registerMeb_{手机号}`（`ParkLockUtils` 自动加 `parkLock_` 前缀）。

## 关键流程

1. 校验 `userPhoneNum` 非空（原逻辑）。
2. `synchronized (phoneNum.intern())` 内：
   - 若 `ParkLockUtils.checkSyncLock` 已占用 → 返回 `state=2`，文案「您操作太频繁，请稍候再试」。
   - 否则 `addSyncLock(key, 10)` 加锁。
3. 执行原 `checkPhoneNum` + `switch(regstatus)`（0/3 走 `register` 插入，1 激活，2/4 拒绝）。
4. 锁释放策略：
   - **本次真正 insert 成功**（case 0 或 3 且 `state=1`）：**不立刻删锁**，保留至 TTL=10 秒，挡住连点并兜住读库主从延迟。
   - **激活 / 已注册 / 注销中 / 插入失败**：立即 `removeSyncLock`，避免误伤正常重试。
   - **抛异常**：立即释放锁。

时序（连点场景）：

```
请求A ──加锁── check ── INSERT 成功 ── 保留锁(10s) ── 返回成功
请求B ──发现已锁── 立即返回「操作太频繁」
请求C（10s 后）──加锁── check 已能查到会员 ── 返回已注册
```

## 配置项、数据约定

| 项 | 约定 |
|----|------|
| 锁 TTL | 常量 `REGISTER_LOCK_TTL_SECONDS = 10`（秒） |
| 锁前缀 | `REGISTER_LOCK_PREFIX = "registerMeb_"` |
| 依赖缓存 | 与现有泊位/支付锁相同，走 `CacheUtils`（Redis） |
| 表结构 | 无变更 |
| 失败码 | 被互斥挡住：`state=2`，`message=您操作太频繁，请稍候再试`（非「已注册」） |

## 联调 / 验收要点

1. **正常注册**：单次请求仍成功，返回 `mebid`，行为与改前一致（含虚拟币、注册活动）。
2. **并发连点**：同一手机号在约 1 秒内打 5～10 次 `registerMeb`，库中仅 1 条 `meb_info`；其余返回「操作太频繁」。
3. **已注册用户再点注册**：仍返回原「已注册」类提示（拿锁后走 case 2，并释放锁）。
4. **注册失败重试**：插入失败后锁会释放，可立即再试。
5. **日志**：写入 `mebInfo.txt`（`Logging.getLogging("mebInfo")`，与 `RegisterBase` 同文件）。部署后搜：
   - `MebServiceExt.registerMeb 进入` → 确认新 class 已加载
   - `注册互斥` → 加锁成功 / 拒绝重复 / 释放锁
   若仍无「进入」日志，说明部署的不是本改动包。

## 常见坑

- **部署未生效**：若 `mebInfo.txt` 里没有 `MebServiceExt.registerMeb 进入`，说明线上 class 仍是旧包，锁逻辑未加载。请确认编译产物中的 `com/app/service/MebServiceExt.class` 已更新到全部 roadsystem 节点并重启。
- **`checkPhoneNum` 使用 `ds=read`**：若 insert 成功后立刻释放锁，从库未同步时第二请求仍可能再插。因此成功路径必须保留 TTL，不要改成「成功也立刻 unlock」。
- **多 Tomcat**：依赖 Redis；若某节点缓存连不上，锁失效，竞态可能再现。
- **`ParkLockUtils` 非原子 SETNX**：单机靠 `synchronized(phoneNum.intern())` 收紧；极端多机同毫秒仍有极小窗口，完整兜底需唯一索引（本次不做）。
- **历史脏数据**：本改动不合并已有重复行，需按业务保留一条、作废其余。

## 与旧逻辑的差异

| 项 | 旧 | 新 |
|----|----|----|
| 并发连点 | 可插多条 | 同号短时仅一条成功，其余被挡 |
| `regstatus` 分支 | 0/1/2/3/4 | **不变** |
| 表 / SQL | — | **不变** |
| 禁止改动的类 | — | 未改 `RegisterBase`、`meb.xml` 等；仅改 `MebServiceExt` |
