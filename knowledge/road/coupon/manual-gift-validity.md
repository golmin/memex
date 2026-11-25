---
title: "手动赠券的有效期在进钱包前就定好了"
type: note
summary: "手动赠券的开始时间是调用当时，结束时间是入参。这两个值放进赠送参数再交给金服，钱包保存的是收到的区间。"
tags: [优惠券, 有效期]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/coupon/general-coupon-grant.md
    label: alternative
    description: "活动触发的通用券走另一条发券，不经过这个手动接口"
---

## 问题

手动赠出的券，有效期和页面上理解的区间对不上。需要分清是发放时没带，还是钱包自己改了。

## 卡点

这个接口没有单独的「开始时间」入参。开始时间在方法里取调用当时的毫秒，结束时间用入参 `endtime`。两者放进赠送结构的 `begin_time`、`end_time`，再按张调用金服。钱包保存调用方送来的区间，不会按活动规则再算一遍。

结束时间为空时方法直接失败，不会发一张没有结束时间的券。

活动触发的通用券不走这个接口，有效期来自活动阶梯。两条入口的时间来源不同，见 [通用券活动发放](/knowledge/road/coupon/general-coupon-grant.md)。

## 解决思路

对开始时间时，对的是服务器收到请求的时刻，不是页面上另一个未传入的字段。对结束时间时，对的是入参有没有原样放进赠送参数。钱包侧若和赠送参数一致，就不是钱包改写。

## 关键位置

`iroad_jdz` 的 `ActivityService.manualGiftCoupon`。`manualGiftCouponPlus` 用同一套开始、结束时间。

## 以后怎么用

券的时间与预期不符时，先找构造 `begin_time`、`end_time` 的那一层，再看钱包存的是不是同一对值。开始时间被写成 `now` 的接口，不会保留表单里的开始日期。
