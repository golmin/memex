---
title: "滴滴充电对接开发操作"
type: reference
summary: "滴滴互联互通的角色、接口和配置项名称。密钥值不写在文档里。"
tags: [充电, 滴滴]
created: "2026-11-25"
edges:
  - path: /knowledge/charge/didi/didi-landed-code.md
    label: about
    description: "已经落地的代码范围" 
sources:
  - title: "icharge_jdz/doc/滴滴充电对接开发操作文档.md"
---

# 滴滴充电对接开发操作文档

本文供开发按步骤落地。原则：**不改原 APP 充电主流程**；**复制一份滴滴专用业务**；本次只做 **开枪 / 关枪 / 状态**；**取消用户余额限制**；本平台只负责 **生成订单、结束订单**，费用由滴滴侧结算。

---

## 0. 先搞清角色（必读）

现有 `EVCSService` / `TLDService` 是 **我们调对方**（用户走我们 APP，我们向特来电等平台发 `query_start_charge`）。

滴滴本次对接方向相反：

| 方向 | 谁实现 | 谁调用 |
|---|---|---|
| `/query_equip_auth` 等 `query_*` | **我方（运营商）** | 滴滴充电 |
| `/notification_*` | **滴滴充电** | **我方主动推** |

用户在 **滴滴 App 扫码**，不走我们 `IndependChargingPileService.beginCharging`。

内部枪控仍走原厂适配器，不要为滴滴再写一套桩协议：

```
滴滴 App
  → 滴滴平台  POST 我方 /query_*
      → DidiChargeService（新建，解密/鉴权/建单）
          → DidiChargeBizService（复制自 beginCharging / endCharge，去掉余额）
              → NetUtils.getService(chaid).addCharge() / addEndCharge()
                  → 原厂适配器（盛弘 / 直连 / EVCS …）不变
      → 桩上报 TcpBase.rechargeSuccess / 实时电量 / 结算
          → 识别滴滴订单后推送滴滴 /notification_*
```

**禁止**把滴滴密钥、URL 写进现有 `EVCSService`。那是另一条业务。

---

## 1. 隔离策略（保证原业务不受影响）

### 1.1 允许改 / 禁止改

| 范围 | 做法 |
|---|---|
| `IndependChargingPileService` / `IndependChargingPileBase` | **禁止改** 开始充电、扫码、钱包校验 |
| 原厂 `PileService` 子类（盛弘、云杉等） | **禁止改** 开枪关枪协议 |
| 新建 `DidiChargeService` + `DidiChargeBizService` + `DidiNotifyClient` | **全部新代码** |
| `ApplicationMain.DirectFilter.msmList` | **追加** 滴滴免登录白名单 |
| `TcpBase.rechargeSuccess` / 实时电量 / 结算 | **只加分支**：`isDidiOrder` 则推滴滴，不走钱包 |
| `ChargingOrderPaymentMQRece` | **只加分支**：滴滴订单直接 `orderstatus=7` 无需缴费，不扣钱包 |

### 1.2 订单如何识别为滴滴单

建单时固定写入：

- `startuptype = 5`（在 `DataDict.StartupWay` **新增** `DIDI("5")`，原有 1~4 不动）
- `trawater = StartChargeSeq`（滴滴 27 位订单号，全链路用它关联）
- `mebid` = 预置「滴滴渠道虚拟会员」ID（避免 `TcpBase` 里 `mebid` 空指针）
- `tel` = 虚拟会员手机号或滴滴 `UserId` 截断存放
- 可选：`remark` / 扩展字段存滴滴 `UserId`、`CarPlateNo`

判断：`startuptype == 5` 或 `trawater` 以双方约定的运营商 ID 开头。

### 1.3 虚拟会员

生产预置一条会员（如账号 `didi_channel`），配置到 `cha_source_param`：`didi.virtual.mebid`。所有滴滴单挂这个会员，**不校验其余额**。

### 1.4 余额 / 扣费全部取消（滴滴单）

复制业务时 **不要拷贝** 以下逻辑：

- 扫码里 `useentwallet == 0 && useownwallet == 0` 则不可用
- 开始充电前查钱包余额、最低起充金额
- `TcpBase.rechargeSuccess` 里把钱包余额写入 `CacheUtils.set(mebid + U_MEBID)` 后，实时电量用余额掐停
- 结算后 `MQSendUtils.send(MQNames.order_payment)` 扣钱包

滴滴单结算后：

1. 订单状态改为 `3` 充电完成（算电量、算金额，仅记账）
2. 立刻改为 `7` 无需缴费（或跳过 MQ，直接 7）
3. **不发** `order_payment`，或 MQ 内发现 `startuptype=5` 直接 return 并置 7

金额仍要算对，因为要推给滴滴 `notification_charge_order_info`（滴滴按电量算用户账单）。

---

## 2. 建议目录与类

放在 `icharge_jdz`（定制），不要塞进公版 `EVCSService`。

```
icharge_jdz/
  net/com/net/didi/
    DidiChargeService.java      // 接收滴滴 query_*，加解密、验签、4秒返回
    DidiNotifyClient.java       // 调用滴滴 notification_*
    DidiCrypto.java             // AES + Sig，复用 com.evcs.util.AESUtil / HMacMD5
    DidiChargeBizService.java   // 复制开枪/关枪/建单/结单（无余额）
    DidiOrderHelper.java        // 按 StartChargeSeq / ConnectorID 查单、isDidiOrder
  common/com/sys/service/ApplicationMain.java  // 仅追加白名单
```

若框架强制 `module_Service_method` 命名：类名 `DidiChargeService` 对应 module=`net` 或新建 module=`didi`。推荐：

```
direct-module=didi_service=DidiCharge_method=queryEquipAuth
```

对外再由 Nginx 把中电联路径转进来（见第 8 节）。

---

## 3. 加解密与报文外壳（所有接口共用）

与现有 EVCS 一致，**不要自己发明算法**。参考：

- `icharge_base/common/com/evcs/util/AESUtil.java`（AES-128-CBC）
- `icharge_base/common/com/evcs/util/HMacMD5.java`
- `EVCSService.getResJSONMap` / `Decrypt` / `send`

外壳字段：

```json
{
  "OperatorID": "我方9位运营商ID",
  "Data": "AES加密后的业务JSON",
  "TimeStamp": "yyyyMMddHHmmss",
  "Seq": "0001",
  "Sig": "HMacMD5签名"
}
```

签名字段顺序以滴滴正式协议为准（通常 `OperatorID + Data + TimeStamp + Seq`，密钥为 SigSecret）。密钥配在 `cha_source_param`，禁止写死。

建议参数 key：

| key | 含义 |
|---|---|
| `didi.operator.id` | 我方 OperatorID（9 位） |
| `didi.operator.secret` | Token 用 |
| `didi.data.secret` | AES DataSecret（16 位） |
| `didi.data.iv` | AES IV（16 位） |
| `didi.sig.secret` | SigSecret |
| `didi.request.url` | 滴滴网关根路径（推送 notification 用） |
| `didi.virtual.mebid` | 虚拟会员 |
| `didi.token.ttl` | Token 有效期 |

中电联一般还有 `query_token`。滴滴联调包若要求，在 `DidiChargeService` 增加 `queryToken`（接收）和客户端取 Token（推送前）。现有参考：`EVCSService.sendToken` / `queryToken`。

---

## 4. 接口实现清单

### 4.1 我方实现（滴滴 → 我方）

统一：POST JSON，先验签再解密 `Data`，业务失败时 **HTTP 仍可 Ret=0**，用 `SuccStat=1` + `FailReason` 表达（与示例一致）。外层 `Ret≠0` 仅用于签名/解密/系统错误。

---

#### A. `POST /query_equip_auth`  设备认证（扫码主失败原因）

**超时：** 尽快返回。正式环境扫码必验。

**入参：** `EquipAuthSeq`，`ConnectorID`

**处理步骤：**

1. 用 `ConnectorID` 查桩：`ChargeBase.queryChaidBysn`（`chasn = ConnectorID`）。查不到 → `SuccStat=1`，`FailReason` 自定义（建议 3=设备不存在）。
2. 读心跳 `HeartUtils.getHeart(chasn)`：
   - 心跳异常 / 离线 → `SuccStat=1`，`FailReason=2`（检测失败）
   - 故障 `PileState.FAULTY` → `FailReason=2`
   - `HeartUtils.getConnectStatus` 未插枪（`ConnectState.DISCONNECT`）→ `SuccStat=1`，`FailReason=1`（此设备尚未插枪）
3. 插枪且空闲/可充 → `SuccStat=0`，`FailReason=0`
4. 原样回传 `EquipAuthSeq`、`ConnectorID`

**不要**在这里建单、不要调 `addCharge`。

对应原逻辑：`IndependChargingPileService` 扫码里的「未插枪 / 网络故障 / 故障桩」判断，复制到 `DidiChargeBizService.authEquip`。

App 失败文案由滴滴侧展示，我方只要 FailReason 准确。

---

#### B. `POST /query_equip_business_policy`  业务策略（扫码次要失败原因）

**注意：** 滴滴会缓存约半小时；时段必须 **完整循环**（从 `000000` 覆盖到次日 0 点）。

**入参：** `EquipBizSeq`，`ConnectorID`

**处理步骤：**

1. 桩不存在 → `SuccStat=1`，`FailReason=1`
2. 用 `ChaStationBase.getChargeRuleByChaid` 取本桩规则（与 APP 同一套价）
3. 转成滴滴 `PolicyInfos[]`：
   - `StartTime`：`HHmmss` 六位
   - `ElecPrice` / `SevicePrice`：小数点后 4 位（注意协议拼写是 **SevicePrice**）
   - 第一段必须从 `000000` 起
   - 时段连续，最后一段接到 24:00
   - `SumPeriod` = 数组长度，范围 0~32
4. 无规则 → `SuccStat=1`，`FailReason=1`

**不要**走 `EVCSService.getPrice` 再去调特来电的 `query_equip_business_policy`。我们是运营商，价格出自我方库。

---

#### C. `POST /query_start_charge`  请求启动充电

**硬性：4 秒内返回。** 超时滴滴会当失败并重发，可能重复建单。

**入参：** `StartChargeSeq`（必填）、`ConnectorID`（必填）、`QRCode`（必填）、`CarPlateNo`（可选）、示例还有 `UserId`

**立即返回（同步）：**

```json
{
  "StartChargeSeq": "...",
  "StartChargeSeqStat": 1,
  "ConnectorID": "...",
  "SuccStat": 0,
  "FailReason": 0
}
```

- 成功受理：**必须** `StartChargeSeqStat=1`（启动中）
- **禁止**返回 `5`（未知）→ 滴滴会当启动失败
- 设备不存在 `FailReason=1`；离线 `FailReason=2`；此时 `SuccStat=1`

**异步真正开枪（返回后再做，或返回前只发指令不等桩结果）：**

复制 `IndependChargingPileService.beginCharging`，改成：

1. 幂等：已有 `trawater=StartChargeSeq` 的单 → 直接成功返回 1，不再建单
2. `queryNoStopOrder`：枪上已有非本流水的执行中订单 → 失败（自定义 FailReason）
3. **不查钱包**
4. `addOrder`：`orderstatus=0`，`startuptype=5`，`trawater=StartChargeSeq`，`carno=CarPlateNo`，`mebid=虚拟会员`
5. `NetUtils.getService(chaid)` → `setUserid` / `setOrderid` → `addCharge()`
6. 下发失败：删单或作废，本接口若已返回成功则必须在 60s 内推 `notification_start_charge_result` 且 `StartChargeSeqStat=4`
7. **不要等** `rechargeSuccess` 再返回本接口

原厂启动成功后走 `TcpBase.rechargeSuccess`（订单 0→1）。在该方法末尾：

```
if (DidiOrderHelper.isDidiOrder(order)) {
    DidiNotifyClient.notificationStartChargeResult(seq, 2, connectorId, startTime, identCode);
}
```

启动失败走 `rechargeFauil` 时同样推 Stat=**4**。

**60 秒内**必须推送启动结果。滴滴以推送为准，不以本接口的 1 为准。

---

#### D. `POST /query_equip_charge_status`  查询充电状态

**按订单号查，禁止按枪号拿「当前枪上别的单」。**

**入参：** 仅 `StartChargeSeq`

**处理：**

1. `trawater = StartChargeSeq` 查订单（含历史）
2. 找不到 → 可 `StartChargeSeqStat=5` 但文档建议尽量不要用 5；更稳妥是按已结束返回电量 0 或最后快照
3. 电量、金额、SOC **只取该订单缓存**（`CacheUtils.get(orderid)` / 实时电量字段）
4. 该单电量是 0 就必须回 0
5. `TotalPower` 单调不降（与上次推送/查询取 max）
6. `StartTime` / `EndTime` 格式 `yyyy-MM-dd HH:mm:ss`
7. 状态映射：

| 我方 orderstatus | 滴滴 StartChargeSeqStat | ConnectorStatus 建议 |
|---|---|---|
| 0 初始化 | 1 启动中 | 2 占用未充电 |
| 1 充电中 | 2 充电中 | 3 占用充电中 |
| 2 暂停 | 2 或自定义 | 2 |
| 3/4/7 结束 | 4 已结束 | 1 空闲 |
| 5 作废 | 4 已结束 | 1 |

8. 电压电流 SOC 从心跳/实时缓存取，没有则 0
9. `ChargeDetails` 尽量带上（与结算同一套分段），字段注意协议里有 `SeviceMoney` / 示例里又有 `ServiceMoney`，**与滴滴确认最终拼写**，两边都传可减少争议

滴滴异常单会查 **24 小时**，挂单也必须按该 `StartChargeSeq` 的真实电量回。

---

#### E. `POST /query_stop_charge`  请求停止充电

**入参：** `StartChargeSeq`，`ConnectorID`

**立即返回：** `StartChargeSeqStat=3`（停止中），`SuccStat=0`

失败：设备不存在 1，离线 2，已停止 3。

**异步：** 复制 `IndependChargingPileBase.endCharge`：

1. 按 `trawater` 找充电中订单
2. `pileService.addEndCharge()`
3. **30 秒内**推 `notification_stop_charge_result`，成功时 `StartChargeSeqStat=4`，`SuccStat=0`
4. 随后 30 秒内推 `notification_charge_order_info`（没赶上 24 小时内补推）

文档写「停止过程在 notification_start_charge_result 处理」是笔误，停止结果走 **`notification_stop_charge_result`**。

---

#### F. `POST /query_terminal_code`  二维码解析（按需）

仅当枪二维码 **不含 ConnectorID** 时实现。有中电联标准码 `hlht://{ConnectorID}.{OperatorID}/` 则可不做。

一桩多枪必须每枪一张码。

---

### 4.2 我方调用（我方 → 滴滴）

封装 `DidiNotifyClient.post(path, dataMap)`：加密、签名、带 Token、记日志、失败重试（建议 3 次，间隔 2s/5s/10s）。

| 路径 | 时机 | 关键字段 |
|---|---|---|
| `/notification_start_charge_result` | `query_start_charge` 后 60s 内 | 成功 Stat=**2**；失败 Stat=**4**；必填 StartTime；IdentCode 可选 |
| `/notification_equip_charge_status` | 充电中周期推（建议 30~60s，可挂在实时电量回调） | 与查询状态同结构；Stat=2；电量不降 |
| `/notification_stop_charge_result` | `query_stop_charge` 后 30s 内 | Stat=**4**；SuccStat 与是否停成功一致 |
| `/notification_charge_order_info` | 结束后 30s（可 24h 补推） | 电费/服务费拆开；**必须** ChargeDetails |

账单推送硬性要求：

- 分段 ≤ 48
- 相邻分段时间连续，首=开始充电，尾=结束充电，开始 < 结束
- 分段电量之和 = `TotalPower`
- 每段：电量、电费、服务费
- 若过程查询已有电量、账单推 0，滴滴会以查询为准 → **禁止结单电量 0 而过程电量 > 0**

`StopReason` 映射建议：

| 我方结束原因 | 滴滴 StopReason |
|---|---|
| 用户/滴滴停止 | 0 或 1 |
| 充满 BMS | 2 |
| 设备故障 | 3 |
| 拔枪 | 4 |
| 其他 | 6~99 自定义并备案 |

---

## 5. 时序（开发对照）

```
扫码
  滴滴 → query_equip_auth          插枪？在线？
  滴滴 → query_equip_business_policy  电价时段
启动
  滴滴 → query_start_charge        ≤4s 回 Stat=1，同时 addOrder + addCharge
  桩启动成功 TcpBase.rechargeSuccess
  我方 → notification_start_charge_result  Stat=2  （≤60s）
过程
  滴滴 → query_equip_charge_status  按订单回电量
  我方 → notification_equip_charge_status  可选周期推
停止
  滴滴 → query_stop_charge         回 Stat=3，addEndCharge
  我方 → notification_stop_charge_result  Stat=4  （≤30s）
  我方结算（不算钱包）
  我方 → notification_charge_order_info   （≤30s，可补推）
```

用户在我们自己 App 充电：完全不进上述类。

---

## 6. 与原流程对照（复制时改什么）

以 `IndependChargingPileService.beginCharging` 为蓝本：

| 原步骤 | 滴滴副本 |
|---|---|
| 校验 mebid/tel/chaid | 校验 ConnectorID + StartChargeSeq |
| queryNoStopOrder 他人占用 | 保留 |
| 故障/维护/心跳异常 | 保留，映射 FailReason |
| 钱包 / useentwallet | **删除** |
| addOrder startuptype 来自 APP | 固定 `5`，trawater=StartChargeSeq |
| pileService.addCharge | **原样调用** |
| 等 APP 轮询 realTimeCharge | 改推滴滴 + 响应 query_status |
| endCharge 用户点结束 | 由 query_stop_charge 触发 |
| 结算发 order_payment | 滴滴单不扣费，状态 7 |

`TcpBase` 只加识别分支，不要复制整个 TcpBase。

实时电量：在现有 `realTimeData` 更新缓存后，若是滴滴单则调用 `DidiNotifyClient.notificationEquipChargeStatus`。注意节流，避免每帧都推。

---

## 7. 白名单与入口

`icharge_base/common/com/sys/service/ApplicationMain.java` 的 `DirectFilter.msmList` 追加（名称以实际 module/service/method 为准）：

```
didi_DidiCharge_queryToken
didi_DidiCharge_queryEquipAuth
didi_DidiCharge_queryEquipBusinessPolicy
didi_DidiCharge_queryStartCharge
didi_DidiCharge_queryEquipChargeStatus
didi_DidiCharge_queryStopCharge
didi_DidiCharge_queryTerminalCode
```

滴滴要求路径是 `/query_equip_auth`，我方框架是：

```
/system/direct-module=didi_service=DidiCharge_method=queryEquipAuth
```

必须用 Nginx（或网关 Filter）做路径映射，见下一节。

---

## 8. Nginx 路径映射（联调前配好）

```nginx
# 滴滴 → 我方
location /query_token {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryToken;
}
location /query_equip_auth {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryEquipAuth;
}
location /query_equip_business_policy {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryEquipBusinessPolicy;
}
location /query_start_charge {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryStartCharge;
}
location /query_equip_charge_status {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryEquipChargeStatus;
}
location /query_stop_charge {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryStopCharge;
}
location /query_terminal_code {
    proxy_pass http://127.0.0.1:8080/system/direct-module=didi_service=DidiCharge_method=queryTerminalCode;
}
```

推送滴滴的完整 URL = `didi.request.url` + 路径，例如：

`https://{滴滴网关}/notification_start_charge_result`

---

## 9. 数据与配置

### 9.1 枪编码

`ConnectorID` 必须等于互联互通文档中的接口编码，并与库表 `cha_baseinfo.chasn`（或单独映射表）一致。

若现网 `chasn` 不是 16 位中电联编码：建映射表 `didi_connector_map(connector_id, chaid, chasn)`，认证/启动都先查映射。

二维码建议标准：`hlht://{ConnectorID}.{我方OperatorID}/`

### 9.2 供应商

**不要**给滴滴单独建 `cha_source.dockingpath=DidiChargeService`。  
滴滴不是桩厂家。桩仍指向盛弘/直连等原 `sourceid`。  
滴滴参数单独建一条 `cha_source`（如名称「滴滴充电渠道」）只存密钥 URL，由 `DidiCrypto` 按 source 名读取，与枪的 `sourceid` 无关。

### 9.3 扩展 DataDict（仅新增）

```java
// StartupWay 增加
DIDI("5");
```

订单状态沿用现有 0/1/3/7，不要新造一套。

---

## 10. 开发任务拆分（建议顺序）

按此顺序提交，便于联调。

1. **加解密工具 + query_token**  
   用滴滴测试环境密钥对文档示例加解密，Sig 对得上再写业务。

2. **Nginx 映射 + 白名单 + 空接口**  
   每个 query_* 先回固定成功报文，让滴滴连通。

3. **query_equip_auth**  
   接 `HeartUtils` 插枪/在线。用真实枪扫码，故意不插枪应对 FailReason=1。

4. **query_equip_business_policy**  
   接本桩计费规则，检查 000000 起完整循环、半小时缓存下价格变更策略（变更后最多 30 分钟生效）。

5. **DidiChargeBizService 建单 + addCharge**  
   无余额；startuptype=5；4 秒返回 Stat=1。

6. **TcpBase 分支推 notification_start_charge_result**  
   成功 2 / 失败 4；60 秒。

7. **query_equip_charge_status + 周期推状态**  
   严格按 trawater；电量不降；带 SOC。

8. **query_stop_charge + 停枪推送 + 账单推送**  
   30 秒；ChargeDetails；不扣钱包；orderstatus=7。

9. **幂等与超时重试**  
   同一 StartChargeSeq 重复启动只建一单；推送失败补推（账单 24h）。

10. **联调清单**（下一节）通过后再上生产。

---

## 11. 联调与验收清单

- [ ] 不插枪扫码：auth FailReason=1，滴滴提示熄火关门/拔枪重插类文案
- [ ] 无策略或时段不完整：policy 失败，提示未获取到电桩信息
- [ ] 插枪空闲：auth+policy 成功，能点启动
- [ ] query_start_charge 抓包 < 4s，Stat=1，无 Stat=5
- [ ] 60s 内收到 Stat=2 的启动结果推送
- [ ] 我方 App 同时扫同一枪：仍走原逻辑，互不影响（滴滴占用后 APP 应提示他人在充）
- [ ] 滴滴启动后，原 APP 用户余额为 0 也能被滴滴启动（虚拟会员无余额限制）
- [ ] 过程查询电量与推送电量属同一 StartChargeSeq
- [ ] 停止 < 30s 推 Stat=4，再推账单；分段电量之和 = TotalPower
- [ ] 故意超时不推账单，24h 补推成功
- [ ] 重复 query_start_charge 同一流水不产生第二张单
- [ ] 结算后用户钱包无扣款，订单为无需缴费
- [ ] 原 APP 充电、扣费、余额不足停充 **行为与改前一致**

---

## 12. 常见坑（来自滴滴注意事项 + 现网代码）

1. `query_start_charge` 超时 → 重复订单。受理成功立刻落库 unique：`trawater` 唯一索引（若现网无索引，滴滴单用 Redis `SETNX didi:seq:{StartChargeSeq}`）。
2. 必填字段缺失（StartTime、ConnectorID、Stat）→ 启动失败。推送前做非空校验。
3. 示例 JSON 字段拼写不统一：`SevicePrice` / `ServicePrice`，`SumPeriod` / `SumPreiod`。**以滴滴最新 PDF 为准**，建议与对方确认后写常量，不要混用。
4. 按枪查电量会串单，必须按 `StartChargeSeq`。
5. 不要用 `IndependChargingPileService` 加 if (didi) —— 入口都不是 APP。
6. 不要把滴滴做成新的 `PileService` 去替换枪的 dockingpath，否则 APP 扫该枪会去调滴滴。
7. `TcpBase` 结算默认发 MQ 扣费，滴滴单必须拦住，否则会扣虚拟会员或报错。
8. 启动结果推 Stat=4 表示失败结束；充电中推 Stat=2。不要在成功启动时推 4。

---

## 13. 参考代码位置

| 用途 | 路径 |
|---|---|
| APP 开始充电（复制蓝本） | `icharge_jdz/app/com/independ/service/IndependChargingPileService.java` `beginCharging` |
| APP 结束充电（复制蓝本） | `icharge_base/app/com/independ/service/IndependChargingPileBase.java` `endCharge` |
| 下发枪指令 | `NetUtils.getService` → `PileService.addCharge/addEndCharge` |
| 启动成功/失败 | `TcpBase.rechargeSuccess` / `rechargeFauil` |
| 结算并发 MQ | `TcpBase` 结算末尾 `MQNames.order_payment` |
| 扣钱包（滴滴要跳过） | `icharge_jdz/common/com/sys/mq/ChargingOrderPaymentMQRece.java` |
| 插枪/心跳 | `HeartUtils` |
| 计费规则 | `ChaStationBase.getChargeRuleByChaid` |
| 国标加解密/推送外壳 | `EVCSService`（只参考，不改） |
| 免登录白名单 | `ApplicationMain` DirectFilter |
| 订单状态枚举 | `DataDict.ChargeOrderStatus`：0 初始化 1 充电中 3 完成 4 已缴费 7 无需缴费 |

更完整的原平台流程见：`icharge_jdz/doc/充电业务全流程与新平台对接说明.md`。

---

## 14. 本次范围外（不要做）

- 滴滴侧用户注册、钱包、发票
- 预约锁枪、即插即充以外的增值
- 修改原 APP 余额不足停充
- 电站静态信息全量同步（若滴滴另要 `query_stations_info` 等，另开任务；本单只做开枪关枪状态）

若滴滴准入还要求电站/枪状态推送（`notification_stationStatus`），在开枪联调通过后再加，仍放 `DidiNotifyClient`，不要改 `EVCSQueryStationsTask`。
