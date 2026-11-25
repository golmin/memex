---
title: "通用券活动类型"
type: reference
summary: "优惠类型 8 只改活动内容。触点仍按原来的活动类型编号找活动，充电站列表和停车场列表存在不同字段。"
tags: [优惠券, 活动]
created: "2026-11-25"
edges:
  - path: /knowledge/road/coupon/general-coupon-grant.md
    label: used_by
    description: "发券时按类型 8 写钱包名称和停车场范围" 
sources:
  - title: "iroad_base/doc/通用券活动类型说明.md"
---

# 通用券活动（favorabletype=8）

## 背景与范围

注册送券、充值送券等活动原来只有优惠券（`favorabletype=1`）和折扣（`2`）。优惠券发出去的是阶梯赠券，充电核销认不出，路外自动用券也会因为没有券号对应的停车场范围而丢掉。

本次在新增、修改活动时增加优惠类型 **8-通用券**。触发场景不改：仍然是注册、充值等触点进入 `doActivity`。`typeid` 必须继续用该场景原来的活动类型（注册送券仍是 `20170418114953608297856533858201`）。触点按 `typeid` 找活动，改成充电券类型后注册匹配不到。

不做的事：

- 不把通用券做成 `favorabletype=3` 那种只建空壳、引用充电券包的活动。
- 不在注册成功处改调 `manualGiftCoupon`。那个接口会再发一张券，活动编号写死，并且会把每个 id 当成充电站去查名称。

## 类与调用

| 类 | 作用 |
|---|---|
| `std/actn/com/actn/base/ActivityBase.java` `addActivity` / `updateActivityById` | 校验通用券入参，并把充电站写入活动内容 |
| `std/actn/com/actn/internal/ActivityInternal.java` `getActivityInfoById` / `getCouponGiveRecord` | 发放数量、发放记录把 8 按优惠券统计 |

发券落库在景德镇 `iroad_jdz` 的 `result_step_coupon.Result`，见[通用券活动发放](/knowledge/road/coupon/general-coupon-grant.md)。

## 入参

与优惠券相同的字段仍要传：`typeid`、`couponlist`（阶梯、有效期、总量）、`conditionlist`、`rulelist`。

`list`：充电站 id，逗号分隔，例如 `3602030030,gzyx000174`。`favorabletype=8` 时必填。前端不传停车场。

保存时 `ActivityBase.snapshotGeneralCouponContent`：

- `actn_info.content` 里的 `list` 仍是前端传来的充电站 id。充电 `ChaConponService.getMebCoupon` 用 `content.list.contains(当前充电站id)` 过滤。
- `content.parklist` 由 `ActivityDao.getPloids` 查询 `bee_jdz.plo_info`（`isvalid=1`），按 `mantype` 1、2 拼成 `[{"mantype","ploids"}]`。
- 请求里的 `list` 在序列化后还原成充电站，景德镇 `ActivityService.addActivity` 对类型 8 不再把 `list` 解析成停车场。

## 列表中文名

活动列表的优惠类型名称来自参数组 `activity`、参数名 `favorabletype` 的 `memo`。需要补一条 `paravalue=8`、`memo=通用券`，否则列表上的类型名是空的。

## 验收

1. `favorabletype=8` 且不传 `list`，接口返回「通用券充电站不能为空」。
2. 传齐券规则和充电站 `list` 后，活动能保存；`content.list` 是充电站 id，`content.parklist` 是当时查到的有效停车场。
3. `typeid` 仍是注册赠券类型时，注册触点能命中该活动。
4. 活动详情、发放记录不再返回「活动类型有误」。
