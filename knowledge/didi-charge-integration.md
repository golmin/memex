---
title: "滴滴充电对接落地要点"
type: insight
summary: "滴滴充电主链路按虚拟会员启动，不走车主钱包扣费，订单以 StartChargeSeq 作为流水。"
tags: [charging, didi]
created: "2026-11-25"
edges:
  - path: /knowledge/cursor-session-index.md
    label: source
    description: "结论来自 2026-08-27 补全滴滴对接的会话收尾"
---

## Observation

滴滴对接已经从骨架补到可联调的主链路。启动不校验车主余额，扣费也不走原来的钱包 MQ。

## Evidence

会话收尾时落地的行为：

- `authEquip`：查桩和心跳。未插枪为 1，故障或离线为 2，桩不存在为 3。
- `startCharge`：不校验余额，`startuptype=5`，`trawater=StartChargeSeq`，调用 `addCharge`，返回 Stat=1，禁止返回 5。
- `queryChargeStatus`：按流水查单，电量只增不减。
- `stopCharge`：`addEndCharge`，返回 Stat=3。
- `parseTerminalCode`：解析 `hlht://` 标准码。
- 除 `queryToken` 外，业务接口校验 `Authorization` Bearer Token。
- TCP 侧保留滴滴流水、跳过钱包缓存，推送启动成功、失败和账单，跳过扣费 MQ，订单状态置 `7`。扣费 MQ 里滴滴单直接标成无需缴费并返回。

说明写在 `icharge_jdz/doc/滴滴充电对接已落地代码说明.md`。联调前要配置密钥、`didi.request.url`、`didi.virtual.mebid`，并把 `/query_*` 转到 `DidiCharge` 对应方法。

## Implications

没有预置虚拟会员时无法启动。不要把滴滴单重新接回车主钱包扣费，否则会和「无需缴费」的约定冲突。
