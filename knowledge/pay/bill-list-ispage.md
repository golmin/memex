---
title: "页码写成 0 仍会按第一页查"
type: note
summary: "账单列表是否分页只看 isPage。页码小于 1 会被改成 1，页大小小于等于 0 会被改成 10。要全量得把 isPage 关掉。"
tags: [支付, 分页]
created: "2026-11-25"
updated: "2026-11-25"
---

## 问题

调用账单列表时，把 `pageIndex` 和 `pageSize` 改成 0，希望一次拿出全部记录，结果仍是前 10 条。

## 卡点

是否加 `LIMIT` 由 `isPage` 决定。这个键不存在，或者值被当成真，都会走分页。分页函数里，页码小于等于 0 会改成 1，页大小小于等于 0 会改成 10。0 在这里是「没填」，会被默认值替换，不会变成「不要分页」。

所以两个 0 配上仍然开启的 `isPage`，查询的是第 1 页、每页 10 条。

## 解决思路

要全量结果，把 `isPage` 设成 0 或 false，让查询走不分页的那条。页码和页大小只在分页已经打开时才有意义。

## 关键位置

`ifinancial4_jdz` 的 `ConsumeBlcDao.getConsumeList` 用 `queryIsPage`。页码和页大小的下限在 `CommonUtils.getCacheQueryPageData`：`pageIndex <= 0` 改为 1，`pageSize <= 0` 改为 10。入口是 `Bill.getConsumeBillList`。

## 以后怎么用

「改了页大小仍是 10 条」时，先看分页开关是不是开着，再看 0 和负数有没有被默认值替换。想关闭一种行为，要找那个开关，不要把数量参数写成 0。
