---
title: "办理停车场月卡后缓存未更新"
type: insight
summary: "App 办理月卡时订单 cardcode 写成 0，缓存键落到 account_0，进出场仍读取原来的月卡账号。"
tags: [parking, month-card]
created: "2026-11-25"
edges:
  - path: /knowledge/cursor-session-index.md
    label: source
    description: "结论来自 2026-09-05 月卡缓存排查会话"
---

## Observation

App 办理停车场月卡成功后，进出场缓存仍是旧数据。问题出在缓存键，不是支付没成功。

## Evidence

调用链是 App `applyPloMonthCardOnline`，经 VIP 块写订单，支付激活后再 `cacheCardData`。

订单上的 `cardcode` 被写成 `0` 时，缓存落到 `account_0`。进出场读取的仍是 `account_MO...`，所以看起来像缓存没更新。

会走进办理接口而不是续费，是因为 App 没有走 `paytype=2`。特殊人群还会命中 VIP 快捷续费。

说明在：

- `iparkcloud_jdz/doc/App办理停车场月卡缓存未更新排查说明.md`
- `iroad_jdz/doc/App办理停车场月卡缓存未更新排查说明.md`

## Implications

先核对订单 `cardcode` 和缓存键是否一致，再看 Redis 有没有刷新。后续改动应避免 `getRecordString` 把卡号写成 `0`，并让已有月卡的用户走续费入口。
