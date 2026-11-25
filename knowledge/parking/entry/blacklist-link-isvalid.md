---
title: "列表看主表，运行时看关联行是否有效"
type: note
summary: "后台名单还能看到这辆车，只说明主表 isvalid 仍是 1。入场认不认，看车场关联 plo_groupblackwhite 的 isvalid。"
tags: [停车场, 入场, 黑白名单]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/parking/entry/blacklist-skips-appointment.md
    label: about
    description: "名单把车分成白名单后，后面的预约包不会执行"
---

## 问题

后台名单里一直能看到这辆车，入场却不按名单处理。

## 卡点

列表查询的是名单主表 `group_blackwhiteinfo`。主表 `isvalid` 仍是 1，列表就还在。入场读取的是车场关联 `plo_groupblackwhite`，并且要求关联行 `isvalid=1`。主表有效、关联失效，两边结果不同。

失效语句只把关联的 `isvalid` 改成 0，不改 `updatetime`。用更新时间当「何时失效」会对不上，因为这一列没有参与这次更新。

## 解决思路

把「展示用的主表」和「运行时用的关联」分开查。入场对不上时，查这辆车在该车场的关联行是不是已经失效。失效时间要找真正被 update 的列，或者更新日志，不能默认用 `updatetime`。

## 关键位置

入场缓存读取在 `BlackWhiteInfoBase.getCacheBlackWhiteByCarNumber`。关联表是 `plo_groupblackwhite`。

## 以后怎么用

后台看得到、运行时认不出，先对两张表各自的有效标志。凡是失效语句没有 set 到的时间列，都不能当失效时刻。
