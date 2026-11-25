---
title: "流水线认出一类之后，后面的包不会再跑"
type: note
summary: "入场黑白名单把车认成白名单就 break。还能看到预约日志，说明这一步把车分类成了继续往下走的临停。"
tags: [停车场, 入场, 预约]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/parking/entry/blacklist-link-isvalid.md
    label: uses
    description: "分类结果取决于车场关联是否仍然有效"
---

## 问题

入场同时配了黑白名单和预约。车主说已经预约，岗亭仍提示临停车辆未预约。

## 卡点

入场包按顺序执行，前面的包可以 `break` 掉后面的包。黑白名单在预约之前。它返回白名单（`outType=3`）时，预约包不会执行，也就不会有预约日志。

还能看到预约日志或「临停车辆未预约」，说明名单这一步返回的是继续往下走的分类（`outType=1`），车被当成临时车。预约包执行过，和「预约包没排到」是相反的控制流。

## 解决思路

先看名单这一步的 `outType`，再决定要不要查预约数据。`3` 表示流水线已经在这里结束。`1` 表示预约包会跑，接下来查它为什么没把这辆车认成预约车。名单没认出的原因见 [入场认名单看的是车场关联是否有效](/knowledge/parking/entry/blacklist-link-isvalid.md)。

## 关键位置

入场流程里的 `jdz_entryin_checkblacklist`，以及其后的 `entryin_parkAppointment`。

## 以后怎么用

多段流水线里，后一段的日志出现，就说明前一段没有提前结束。先读前一段的分类码，再查后一段的业务数据。
