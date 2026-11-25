---
title: "滴滴充电已落地代码说明"
type: reference
summary: "滴滴作为运营商调用我方。启动不扣车主钱包，账单把订单标成无需缴费。不要把它配成桩厂家协议。"
tags: [充电, 滴滴]
created: "2026-11-25"
edges:
  - path: /knowledge/charge/didi/no-wallet-deduct.md
    label: about
    description: "不走钱包扣费的原因"
  - path: /knowledge/charge/didi/didi-dev-guide.md
    label: uses
    description: "对接原则和后续任务" 
sources:
  - title: "icharge_jdz/doc/滴滴充电对接已落地代码说明.md"
---

# 滴滴充电对接已落地代码说明

本文说明**当前仓库里已经写好的代码**，供未参加本次开发的人阅读、联调和继续实现。  
更完整的对接原则与后续任务见：[滴滴充电对接开发操作](/knowledge/charge/didi/didi-dev-guide.md)。

**更新日期：** 2026-08-27  
**代码位置：** 以 `icharge_jdz` 为主，Tcp 钩子与白名单在 `icharge_base`。

---

## 1. 背景与范围

### 1.1 做了什么（本轮已落地）

- 滴滴互联互通（中电联 T/CEC 102 风格）运营商侧入口：加解密、验签、Bearer Token。
- Token 双向：滴滴申请我方 Token；我方向滴滴申请 Token（推送用）。
- 业务策略 `query_equip_business_policy`（本桩电价 → PolicyInfos）。
- **设备认证** `query_equip_auth`（插枪 / 在线 / 故障）。
- **启动充电** `query_start_charge`（建单 startuptype=5 + addCharge，回 Stat=1）。
- **过程查询** `query_equip_charge_status`（按 StartChargeSeq，电量单调不降）。
- **停止充电** `query_stop_charge`（addEndCharge，回 Stat=3）。
- **二维码解析** `query_terminal_code`（支持 hlht:// 标准码）。
- TcpBase 分支：保留 StartChargeSeq、跳过钱包缓存、推启动结果 / 失败 / 账单。
- 扣费 MQ 分支：滴滴单置 `orderstatus=7` 无需缴费，不扣钱包。

### 1.2 明确没做 / 不要改的

- **没有改** `IndependChargingPileService` / `IndependChargingPileBase`（APP 开始充电、钱包校验）。
- **没有改** 盛弘、直连等 `PileService` 子类的开枪关枪协议。
- **没有把** `DidiChargeService` 配成枪的 `cha_source.dockingpath`（滴滴不是桩厂家）。
- 过程状态**周期推送** `notification_equip_charge_status` 尚未挂实时电量回调（查询接口已可用）。
- 账单分段目前用「整段一条 ChargeDetails」兜底，峰平谷细分段可后续增强。

### 1.3 角色

滴滴 App 扫码 → 滴滴平台调用**我方** `query_*`。  
我方内部用 `NetUtils.getService(chaid).addCharge()` 控真实桩。  
与现有 `EVCSService`（我们 APP 调特来电）方向相反，因此独立一套类，不改 `EVCSService`。

---

## 2. 类清单与调用关系

```
滴滴 POST JSON（加密外壳 + Bearer）
        │
        ▼
DidiChargeService          入口：拆包、验签、Token、调业务、加密响应
        │
        ├─ DidiCrypto      AES + HMAC-MD5 外壳
        ├─ DidiConfig      读 appConfig / cha_source_param
        ├─ DidiTokenService
        │      ├─ issueToken / validateIssuedToken
        │      └─ fetch/refresh 向滴滴申请
        ├─ DidiChargeBizService
        │      ├─ authEquip / queryPolicy / startCharge
        │      ├─ queryChargeStatus / stopCharge / parseTerminalCode
        │      ├─ DidiPolicyHelper   查桩 + 计费规则 → PolicyInfos
        │      └─ DidiOrderHelper    流水索引、状态映射、报文组装
        └─ DidiNotifyClient         推送 notification_*
               └─ DidiHttp

桩侧回调
TcpBase ──反射──► DidiHook(base) ──► DidiTcpBridge(jdz) ──► DidiNotifyClient
扣费 MQ
ChargingOrderPaymentMQRece ──► DidiOrderHelper.isDidiOrder → 置 7 并 return
```

| 类 | 路径 | 职责 |
|---|---|---|
| `DidiChargeService` | `icharge_jdz/net/com/net/service/DidiChargeService.java` | 运营商侧 HTTP 入口。**继承 `BaseService`，不继承 `PileService`** |
| `DidiChargeBizService` | `icharge_jdz/net/com/net/didi/DidiChargeBizService.java` | 认证/策略/开枪/状态/停枪业务副本 |
| `DidiPolicyHelper` | `…/DidiPolicyHelper.java` | ConnectorID 查桩、规则转滴滴时段 |
| `DidiOrderHelper` | `…/DidiOrderHelper.java` | 识别滴滴单、按流水查单、组装状态/账单 |
| `DidiTcpBridge` | `…/DidiTcpBridge.java` | Tcp 回调推送（给反射用） |
| `DidiConfig` / `DidiCrypto` / `DidiHttp` / `DidiTokenService` / `DidiNotifyClient` | 同目录 | 配置、加解密、HTTP、Token、推送 |
| `DidiHook` | `icharge_base/app/com/tcp/util/DidiHook.java` | TcpBase 可编译依赖的反射桥 |

`icharge_base` 改动：

| 文件 | 改动 |
|---|---|
| `ApplicationMain.java` | DirectFilter 增加滴滴免登录方法 |
| `DataDict.java` | `StartupWay.DIDI("5")` |
| `TcpBase.java` | 滴滴分支：保留 trawater、跳过钱包缓存、推送、跳过扣费 MQ |
| `DidiHook.java` | 新增反射桥 |

---

## 3. 为什么入口不继承 PileService

`PileService` 是**枪适配器**，由 `NetUtils.getService(chaid)` 按 `cha_source.dockingpath` 反射出来，给 APP `addCharge()` 用。

若把 `DidiChargeService` 配成某枪的 `dockingpath`：用户走我们 APP 扫这把枪会去调滴滴，原厂协议被换掉。

滴滴是**渠道入口**（对方调我们），不是桩协议。开枪时再 `NetUtils.getService(chaid)` 走原来的盛弘/直连等。

---

## 4. 框架访问路径与白名单

框架约定：`direct-module=net_service=DidiCharge_method=方法名`。

已加入免登录白名单（`ApplicationMain.redirectInit`）：

| method | 对应中电联路径 | 当前实现 |
|---|---|---|
| `queryToken` | `/query_token` | **已实现** 下发 Token |
| `fetchDidiToken` | （我方内部联调） | **已实现** 向滴滴申请 Token |
| `queryEquipAuth` | `/query_equip_auth` | **已实现** |
| `queryEquipBusinessPolicy` | `/query_equip_business_policy` | **已实现** |
| `queryStartCharge` | `/query_start_charge` | **已实现** |
| `queryEquipChargeStatus` | `/query_equip_charge_status` | **已实现** |
| `queryStopCharge` | `/query_stop_charge` | **已实现** |
| `queryTerminalCode` | `/query_terminal_code` | **已实现**（hlht 标准码） |

Nginx 反代示例见操作文档第 8 节。

---

## 5. 加解密与 Token

与现网 `EVCSService` 相同：Data=AES-128-CBC，Sig=HMAC-MD5。  
业务失败一般 **Ret=0**，用 Data 里 `SuccStat`/`FailReason`；Ret≠0 用于签名错误（4001）、Token 错误（4002）、参数（4003）、系统（500）。

- `query_token` / `fetchDidiToken`：**不校验** Bearer。
- 其余 `query_*`：校验 `Authorization: Bearer {AccessToken}`，失败 Ret=4002。

密钥与 Redis key 见第 9 节。

---

## 6. 业务接口要点

### 6.1 authEquip

1. ConnectorID → 查桩（chasn / chacode）。
2. `HeartUtils`：故障/维护/心跳异常 → FailReason=2；未插枪 → 1；成功 → 0。
3. 桩不存在 → FailReason=3。

### 6.2 startCharge（≤4s 回 Stat=1）

1. 同 StartChargeSeq 已有单 → 幂等回 1（已结束则回 4）。
2. 桩不存在 → FailReason=1；离线/故障 → 2；枪被占用 → 4。
3. 必须配置 `didi.virtual.mebid`。
4. `addOrder`（startuptype=5）后立刻 `updateOrder(trawater=StartChargeSeq)`，并写 Redis `didi_seq_{流水}`。
5. `NetUtils.getService(chaid).addCharge()`；指令失败则删单。
6. **禁止**回 StartChargeSeqStat=5。真正成功看推送 Stat=2。

### 6.3 queryChargeStatus

按 `trawater`/Redis 索引查单，禁止按枪号串单。电量与上次回包取 max。查不到按已结束、电量 0 回。

### 6.4 stopCharge

回 Stat=3；对执行中订单 `addEndCharge`。已结束 → FailReason=3。

### 6.5 推送时机（Tcp）

| 事件 | 推送 |
|---|---|
| `rechargeSuccess` | `notification_start_charge_result` Stat=2 |
| `rechargeFauil` | 同上 Stat=4 |
| 拔枪/记录结算 | `notification_stop_charge_result` + `notification_charge_order_info`，并置 orderstatus=7 |

---

## 7. 配置

优先 `appConfig.properties`，没有再读 `cha_source_param`（`dockingpath=com.net.service.DidiChargeService`）。  
改库后清 Redis：`didi_source_param`。

**不要**把枪的 `sourceid` 指到这个 dockingpath。

```
didi.operator.id=
didi.operator.secret=
didi.data.secret=          # 16 位
didi.sig.secret=
didi.iv.parameter=         # 16 位
didi.request.url=

didi.peer.operator.id=
didi.peer.operator.secret=
didi.peer.data.secret=
didi.peer.sig.secret=
didi.peer.iv.parameter=

didi.token.ttl=7200
didi.virtual.mebid=        # 必配，否则无法启动
```

Redis：

| key | 含义 |
|---|---|
| `didi_source_param` | 供应商参数 |
| `didi_issue_{滴滴OperatorID}` | 我方下发的 Token |
| `didi_fetch_{我方OperatorID}` | 我方从滴滴申请到的 Token |
| `didi_seq_{StartChargeSeq}` | 流水 → orderid/chaid |
| `didi_power_{StartChargeSeq}` | 上次回给滴滴的电量（单调不降） |

---

## 8. 订单约定

- `DataDict.StartupWay.DIDI = "5"`
- `trawater` = 滴滴 `StartChargeSeq`（Tcp 启动成功时**不会**被桩流水覆盖）
- `mebid` = `didi.virtual.mebid`
- 结算后目标状态 `7` 无需缴费（Tcp 直接置 7，或 MQ 兜底）

判断：`DidiOrderHelper.isDidiOrder` / `DidiHook.isDidiOrder`。

---

## 9. 联调建议

1. 配齐密钥、`didi.request.url`、`didi.virtual.mebid`（预置会员）。
2. `queryToken` → 带 Bearer 调 `queryEquipAuth`（故意不插枪看 FailReason=1）。
3. `queryEquipBusinessPolicy` 看时段从 `000000` 起。
4. `queryStartCharge`：抓包 &lt;4s、Stat=1；库中 startuptype=5、trawater=流水。
5. 桩启动成功后，滴滴应收到 Stat=2 推送。
6. `queryEquipChargeStatus` 按流水回电量。
7. 停止后推账单；虚拟会员钱包无扣款；订单为 7。
8. 原 APP 扫同一枪：行为与改前一致。

日志名：`net-didi`。

---

## 10. 后续可增强

1. 实时电量回调里节流推 `notification_equip_charge_status`。
2. 账单 ChargeDetails 按峰平谷真实分段，且分段电量之和 = TotalPower。
3. 推送失败重试队列（账单 24h 补推）。
4. `trawater` 唯一索引或 SETNX 加强幂等。

---

## 11. 相关文档

- 对接原则与全量接口：[滴滴充电对接开发操作](/knowledge/charge/didi/didi-dev-guide.md)
- 原充电全流程：`icharge_jdz/doc/充电业务全流程与新平台对接说明.md`
