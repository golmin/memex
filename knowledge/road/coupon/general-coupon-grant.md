---
title: "通用券活动发放"
type: reference
summary: "活动类型 8 发出的券，名称要能被充电过滤认出来，停车场范围写在券号上，失败不能回滚已经进钱包的券。"
tags: [优惠券, 活动]
created: "2026-11-25"
edges:
  - path: /knowledge/road/coupon/general-coupon-type.md
    label: requires
    description: "活动如何保存成类型 8"
  - path: /knowledge/road/coupon/manual-gift-validity.md
    label: alternative
    description: "手动赠券是另一条入口，有效期在调用时写好" 
sources:
  - title: "iroad_jdz/doc/通用券活动发放说明.md"
---

# 通用券活动发放

## 背景与范围

活动在 `iroad_base` 以 `favorabletype=8` 保存后，注册、充值等场景仍走原来的 `doActivity`。本仓库只改发券结果：条件、阶梯、限次、有效期不变，发出去的券要能被充电和路外核销。

不做的事：

- 不改 `doRegActivity`、支付回调这些入口。
- 不调用 `ActivityService.manualGiftCoupon`。通用券购买、积分兑换仍走那个接口；活动触发的通用券不走。
- 不调用 `ChargingPileCall.getStationList`。那个循环把每个 id 当成自营充电站，查空会抛下标异常。

建活动入参见 [通用券活动类型](/knowledge/road/coupon/general-coupon-type.md)。

## 类与调用

`std/actn/result_step_coupon/Result.java` 的 `doAction`。

`joinActivity` 条件通过后才会进入这里。方法内按活动 id 读 `actn_info.content`：

- `favorabletype` 不是 8：钱包 `resultname` 仍是结果类型名（阶梯赠券），不写停车场范围。
- `favorabletype` 是 8：
  1. `resultname` 使用券名称，且必须包含「充电券」。充电 `ChaConponService.getMebCoupon` 按这个字过滤。
  2. `rechargeGift` 成功后调用 `bindGeneralCouponPark`，`act_park_ticket.actid` 写本次券号。

## 停车场范围

路外自动用券 `StdService.getJDZCouponNoTest` 按券号查 `act_park_ticket`。没有记录会把券丢掉。有记录且 `useway` 不是 2（路内）才保留。这里不写 `actn_useway`，避免被当成仅路内。

范围来源：

- `content.parklist` 是建活动时按 `plo_info` 查出的 `mantype`、`ploids`，按这个范围写入。
- `parklist` 为空时不补车场，只记日志。

写停车场失败只打日志，不把已经成功的钱包券改成失败。

## 充电站

充电核销读的是活动 `content.list`（建活动时前端传入的充电站 id），不是本券的 `actn_useway`。因为钱包上的 `actid` 是真实活动 id，不是手动发券那个固化 id `20170817140303101673440000000000`。

## 验收

1. 注册命中 `favorabletype=8` 的活动后，钱包券名称含「充电券」，面额和有效期仍来自活动阶梯。
2. `act_park_ticket` 有以券号为 `actid` 的记录，路外当前车场 id 在 `ploids` 里时可以被自动用券选中。
3. 当前充电站 id 出现在该活动 `content.list` 里时，`getMebCoupon` 能返回这张券。
4. `favorabletype=1` 的旧活动发券结果与改前一致。
