---
title: "进行中订单和今天之前入场是两个条件"
type: note
summary: "办理停车场月卡拦截的是今天 0 点之前入场且仍进行中的订单。当天入场的在场单不在这个谓词里。"
tags: [月卡, 停车场, 时间边界]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/parking/month-card/cache-cardcode-zero.md
    label: about
    description: "同一条办理链路上，缓存键写偏是另一个机制"
---

## 问题

办理停车场月卡时，有一张今天才入场、现在还在场的订单。需要确认它会不会挡住办理。

## 卡点

「有进行中订单」和「有今天 0 点之前入场的进行中订单」是两个谓词。注释和 SQL 写的是后者。把 `orderstatus=1` 单独拿出来看，会把当天入场的车也算进拦截。

边界是当天 0 点的毫秒，时区按东八区取本地日的开始，比较符是 `intime < 这个时刻`。等于今天 0 点的入场不在拦截集合里。它也不是「现在减去 24 小时」。

## 解决思路

拦截条件同时要：订单进行中、车牌和车场匹配、入场时间早于今天 0 点。三个都成立才挡住办理。当天入场的在场单可以继续办。

## 关键位置

`MonthCardService` 与 `GeneralMonthCardService` 里办理前的在场校验。SQL 在 `iparkcloud_jdz/config/std_comp_ext.xml` 的 `getOrdersCountInProgressBeforeTicks`：

```sql
select count(1) as c from tra_order
where orderstatus=1 and carnumber=? and ploid=? and intime < ?
```

参数是东八区当天 0 点的毫秒。

## 以后怎么用

用户说「在场所以办不了」时，先看比较的是日期起点还是任意进行中。把样本的入场时间和用来比较的那个毫秒打在一起，确认落在开区间的哪一侧。
