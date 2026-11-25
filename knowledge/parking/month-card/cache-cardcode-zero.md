---
title: "缓存写入成功但读的是另一把键"
type: note
summary: "月卡进出场是两级缓存。结果集不带行号取到空，订单编号变成 0，第二级键写成 account_0，读取仍走原来的编号。"
tags: [月卡, 缓存, 停车场]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/parking/month-card/app-cache-trace.md
    label: uses
    description: "办理、支付、写缓存、进出场读取的整条链路"
  - path: /knowledge/parking/month-card/renew-plate-pointer.md
    label: uses
    description: "第一级指针不改时，第二级写对了也读到旧卡"
  - path: /knowledge/parking/month-card/today-entry-not-blocked.md
    label: about
    description: "办理前的在场单过滤用的是另一条时间条件"
---

## 问题

App 办理或续费停车场月卡，支付已经成功，进出场仍按旧月卡或临停处理。日志里能看到缓存写入，键是 `account_0`。

## 卡点

进出场读两级缓存。第一级键是车牌加车场，值是月卡编号。第二级键是 `account_` 加上这个编号，值是有效期等内容。第二级写成功，只要第一级仍指向别的编号，出场就读不到这次写入。

订单上的编号来自查询结果。不带行号的取值读的是参数表顶层，行里的字段不在那里，得到空。空写进订单列会变成 0。`account_0` 是一把合法的键，写入不会失败，所以日志看起来像缓存已经更新。

续费会按指定行取出已有编号再写订单。办理在特殊人群分支里新插一行。已有月卡的请求若进了办理，就会走出这个空编号。

## 解决思路

先对齐三把值：日志里的写入键、第一级指针里的编号、第二级实际存放内容的键。拼键用的字段必须从结果集的指定行取出。0 和空不能再拿去拼第二级键。

只更新第二级、不改第一级时，出场仍顺着旧指针读另一张卡。续费成功后第一级要指向这次的编号，见 [月卡续费要同时改车牌指针](/knowledge/parking/month-card/renew-plate-pointer.md)。

这条覆盖停车场月卡。路内进程只转发办理和支付，不写这两级缓存。进出场判断是不是月卡的主流程不用为这个问题改。

## 关键位置

- 链路说明：[场库缓存链路](/knowledge/parking/month-card/app-cache-trace.md)、[路内只转发](/knowledge/parking/month-card/app-cache-road-entry.md)
- 第二级写入在场库支付后的 `cacheCardData`，键是 `account_` 加订单上的编号
- 进出场先读车牌加车场，再读 `account_` 加读到的编号

## 以后怎么用

缓存写成功但业务仍读到旧值时，先把写入键和读取键打出来。由查询结果拼出来的键，先看取的是第几行，空值落库后会不会变成 0。两级缓存要同时核对指针和内容。
