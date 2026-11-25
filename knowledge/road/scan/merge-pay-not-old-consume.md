---
title: "合并缴费的合计金额不能进旧的单笔校验"
type: note
summary: "路内扫码把当前单和勾选欠费收成一笔时，旧缴费只按当前单校验 total_fee。合计要走新消费：主单号是当前单，batch 里每笔各带自己的金额。未勾选才转回旧方法。"
tags: [路内, 扫码, 欠费]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/scan/going-arrear-merge-pay.md
    label: uses
    description: "查询、下单、回调和缓存键的说明"
  - path: /knowledge/road/ground-lock/ground-lock-pay-keep-parking.md
    label: about
    description: "isFinishOrder 为 false 表示结单没完成；付款不结单那版已回退"
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "当前单入账成功后才对这一单降锁"
---

## 问题

路内扫泊位码，页面上同时有当前进行中订单和同车牌的历史欠费。用户勾选几笔后一次支付。把「当前应付 + 所选欠费」传给原来的进行中缴费，会返回金额变化（`state=5`）。

## 卡点

原来的缴费按当前这一单校验 `total_fee`。合计比这一单大，校验就失败。这不是金额算错，是这笔请求进了只认识一张单的方法。

勾选列表用的是欠费的主订单号。欠费记录自己的 id 不能拿来下单，也排除不了当前这一单。查询里的应付合计是当前单加上全部未缴欠费，实收只加勾中的那几笔。两个数不是同一个。

金服已经扣款之后，回调若再走余额，会扣第二次。当前单的结算结果里 `isFinishOrder=false` 表示这一单没有结完，要让金服重试。它不表示「车还在、故意不结单」。付款后不结单、继续计费的那一版已经回退。

说明里还写了欠费反查当前单、`sub_no` 幂等这几把缓存。当前 `GoingArrearScanPay` 实际写入的是 `jdz:scanpay:going_arrears:{当前单}` 和 `scanpay_endtime_{当前单}`。回调用通知里的单号直接当当前单去取前一把缓存。通知若带来的是欠费单号，这把缓存对不上，历史欠费不会按缓存补缴。

## 解决思路

没有勾选欠费时，微信仍进原来的 `JZBatchPayment`，支付宝仍进原来的 `BatchPayment`，`total_fee` 只含当前单。

勾选了欠费时另起一笔消费：主单号是当前进行中订单，`total_fee` 和 `actual_fee` 都是合计，`operate=3`。`batch` 里第一行是当前单和它自己的金额，后面每笔欠费各一行，行上的单号和金额都是那一笔的未付。下单成功后把勾选结果缓存 10800 秒，键挂在当前单上。

回调先验签，再带 `ispay=1` 结当前单，避免再扣余额。`state=1` 且订单已经结束，才对这一单做地锁的支付成功处理，并按缓存逐笔补历史欠费。某一笔欠费抛错只记日志，不把已经结完的当前单退回去。当前单没结完则回 `fail`，等金服重试。

旧的进行中缴费和它的回调保持不动。未勾选的请求还要走它们。

## 关键位置

- 说明：[路内扫码当前单与历史欠费合并缴费](/knowledge/road/scan/going-arrear-merge-pay.md)
- 业务类：`iroad_jdz` 的 `GoingArrearScanPay`
- 未勾选：微信 `JZBatchPayment.payGoingOrders`，支付宝 `BatchPayment.payGoingOrders`
- 勾选：`Order.consume`，回调 `NotifyService.payGoingWithArrearsNotifyListener`
- 结单失败的含义：[地锁泊位付款不结单](/knowledge/road/ground-lock/ground-lock-pay-keep-parking.md)

## 以后怎么用

一笔支付里有多张单时，先看旧方法校验的是哪一张的金额。合计只能进「主单号 + 分行金额」这种消费，不能改旧方法的校验去迁就合计。

页面上的「全部欠费合计」和「勾选后的实收」要分开对。回调先确认通知里的单号是当前单，再去取挂在当前单上的欠费缓存。`isFinishOrder=false` 先按结单没完成处理。
