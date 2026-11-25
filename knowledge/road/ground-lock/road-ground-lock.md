---
title: "路内地锁从方案到落地的卡点"
type: note
summary: "景德镇路内地锁最终是绑定表加会话表、按真正计费起点预约升锁；支付和驶出分开降锁。计费起点、挂钩类、集群开关都曾经判断错。"
tags: [地锁, 路内, 计费起点]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/jingdezhen-charge-start.md
    label: uses
    description: "预约升锁用的计费起点推算"
  - path: /knowledge/road/ground-lock/ground-lock-session-schedule.md
    label: uses
    description: "绑定表、会话和到期队列约定"
  - path: /knowledge/road/ground-lock/ground-lock-mount-points.md
    label: uses
    description: "升降锁挂在哪些方法上"
  - path: /knowledge/road/ground-lock/ground-lock-bind-session-ddl.md
    label: uses
    description: "绑定表和会话表建表语句"
  - path: /knowledge/road/ground-lock/ground-lock-pay-keep-parking.md
    label: about
    description: "付款不结单已回退，需要时按这篇恢复"
  - path: /knowledge/road/scan/merge-pay-not-old-consume.md
    label: about
    description: "合并缴费只对当前单做支付成功后的降锁"
---

## 问题

景德镇路内没有视频桩。地磁只上报泊位有车或无车，订单要等巡检员 PDA 录单才会生成。泊位上有车位锁时：产生了应付费用才升锁，这单付清或者车已经离开要降锁。车离开时不能等结单成功，否则锁还立着，下一辆进不来。

最早的想法是定时扫有费用的进行中订单再升锁。后来改成：录单成功后按计费规则算出下一次可能产生费用的时刻，到点再试算，避免免费期内误锁。余额缴费不做。

## 卡点

**消息入口不能直接控锁，但建单就在消费线程里。** `uploadBerthStatus` 只把泊位状态丢进 `berthchange`。地磁有车走 `parkIn`，并不建单。一开始把「不要在 MQ 里做地锁」理解成整条消费线程都不能挂钩。`createOrder` 实际就跑在这条栈上。该避开的是消费入口当时还没有订单、也没有计费结果，看见有车就升、看见无车就降。

**计费起点不是驶入时间，也不能信试算返回的 `startparktime`。** 进行中订单没有单独存「开始计费时间」。夜间 21:00 到次日 07:00 免费，免费分钟还要看落在收费窗里还是免费窗里。`ParkingService.jdztime` 在白天、免费分钟为 0 时，算出的起点就等于驶入时间。`getNeedPayMoney` 返回里的 `startparktime` 更不能用：`JingDeZhenCalc` 很多提前返回仍把入参驶入时间写回去，应付为 0 时它几乎就是驶入时刻。拿它去预约，锁会马上到期空转。库表 `billstart` 也常常只是建单时抄下来的驶入时间。

后来按 `JingDeZhenCalc` 的切段规则单独写了 `JingDeZhenChargeStart`。`freecharging=1` 时免费分钟只在收费空隙里扣，例如 20:50 驶入、免 15 分钟、21:00 起免费，起点会推到次日 07:05 左右。`freecharging=2` 且 `freeduration=0` 时，日志里 `chargeStart` 等于驶入时间是规则结果，不是字段写错。这个函数不算金额，也不改原来的 `getPayMoney`。

**公版调用不到景德镇包。** 协议 2.1.2 第一次按文档加的类太多，已撤销，改成贴着现有出站方式写。`iroad_base` 不能依赖 `iroad_jdz`。反射钩子能隔离开，但支付回调和驶出都在公版里，直接调用更省事。最终把出站工具和编排都放进 `iroad_base`，用 `groundLock.enabled` 默认关闭，其它城市不会误调。密钥走配置，不写死。

**订单表当不了「这把锁属于哪一单」。** 按泊位加进行中状态去 `getOrderInfo`，拿不到要用的订单。所以另建两张表：泊位和锁的绑定，以及一泊位一行的进行中会话。会话里放绑定快照、订单号、车牌、锁状态、下次应升时间。未绑定的泊位整段跳过，不要把泊位号硬当成设备编号。插入会话时不要自己写 `createtime`、`updatetime`，框架 `insert` 会补，再写就会报 `Column 'updatetime' specified twice`。`update` 不会自动改 `updatetime`。

**驶出降锁曾经挂错位置，也会读错字段。** 降锁一度放在推送 PDA 成功之后。泊位已经写成无车，但前面占着业务锁提前返回时不会执行；没有订单号时还会直接返回，预约清不掉。绑定查询返回的是摊平的一行，用「第 0 条记录」读设备号是空的。调用方放的是小写 `orderid`，读成驼峰就变成没有订单。正确挂点是泊位状态写库成功之后立刻降，有没有订单都降。降锁指令失败也要清会话和预约，只是先把锁状态标成降锁失败，方便对日志。

**进行中扫码的降锁不在订单状态更新方法里。** 微信详情和缴费命令是 `StdWechat` 的 `getGoingOrderInfoByWeChat`、`payGoingOrdersByWeChat`，实现在 `JZBatchPayment`。支付宝详情同名方法在 `StdAlipay`，缴费实现在 `BatchPayment`。这两条的回调是 `NotifyBase.payBatchArrearNotifyListener`。结单失败时它回 `fail`，不降锁，等金服重试。`JZStdOrderService.updateOrderStatus` 是另一条在线消费回调会进的方法，不是这两条扫码的主路径，只改那里会漏。

付完且订单已经结束，这张单不会再升锁。路内没有道闸抬杆，付完做的是降锁。测试里出现过付完金额还在涨，原因是那次没结成单：入账中途空指针，或者 `finishParkOrder` 失败。订单仍进行中、未付大于 0 时，到期仍会升锁。这是结单失败的兜底，不是「付完原单继续计费」的设计。

后来做过一版「付款降锁但订单不结束，车还在就继续计费，未付再大于 0 再升锁」。`isFinishOrder=false` 表示结单失败，不是故意不结束，只改这个标志达不到继续计费，还会让金服重试。那版写完后把原来能付的路径也改坏了，已整段回退。再启用时看 `iroad_jdz/doc/地锁泊位付款不结单说明.md`，不要重开全局续费开关。

**强制驶离覆盖了父类，父类里的降锁不会跑。** 地磁驶出走 `new ParkingService()`，父类 `parkOut` 里的降锁会执行。强制驶离走 `JZParkingService.parkOutForce`。子类把方法盖掉之后，父类里后加的调用不存在。补在子类里：泊位写成无车并且写库成功才降；泊位上还有进行中订单时子类会把状态写成有车，这时不降。

**升锁成功后，到期队列不再看见这张单。** 捞取只查还在队列里的到期订单，成功后会从队列删掉。所以不能靠这个队列发现「锁已经立着，但订单没了」。曾经加过扫会话表、对已升且订单已结束的记录补降锁，改动面太大，已撤回。现在没有这层兜底。车走了但没走到驶出或强制驶离时，锁可能一直立着。

**两台机器会抢同一批到期升锁，但不能用总开关关掉其中一台。** 捞取大约 15 秒一次，只查询不删除，也没有跨进程认领。两台同时跑会各打一次升锁。Nginx 只分 HTTP，定时任务每台进程自己起。`groundLock.enabled` 同时管建单预约、支付降锁、驶出降锁和出站请求。另一台如果关掉这个开关，请求落到那台时地锁业务等于没有。两台都要开业务，升锁捞取只留一台。代码里没有单独的任务开关。

15 秒一轮、多数时候队列是空的，几乎只访问一次 Redis，不会因为查库把整站拖死。每轮最多处理 100 笔。真正会拖时间的是到期堆在一起时的计费计算，以及升锁 HTTP。升锁失败会改到约 60 秒后再试。

## 解决思路

用户自己梳过的六步是对的，落成下面这样：

1. 绑定表管泊位和设备身份，会话表管「这把锁当前属于哪张进行中订单」。录单 `createOrder` 成功并且有绑定，才写入会话并预约升锁。地磁有车不写。
2. 预约时间用 `JingDeZhenChargeStart`，不用驶入时间，也不用试算出参里的 `startparktime`。会话里的 `startparktime` 存真实驶入，下次升锁时刻放 `next_raise_due`。
3. 到期后订单仍进行中，并且试算金额减去已付和抵扣仍大于 0，才升锁。已付过但仍在停，从当前时刻按规则推下一次起点。没付过就从驶入时间推。月卡或优惠后应付为 0 不升。
4. 扫码支付成功且会话订单号等于这笔单，才降锁。人还在、订单仍进行中，再按规则预约下一次。余额不处理。欠费补缴对不上会话订单号就跳过，避免降错锁。
5. 泊位确认无车并且写库成功，立刻降锁并清会话。降锁失败也清，状态标成降锁失败。强制驶离在子类里按同样条件再挂一次。

未付的判断要放在「订单仍进行中」之后。订单已经结束就只清预约和会话，不再升。

## 关键位置

- 编排：`iroad_base/std/tra/com/tra/utils/GroundLockOrchestrator.java`
- 计费起点：`iroad_base` 的 `JingDeZhenChargeStart`，说明在 [景德镇计费开始时间推算](/knowledge/road/ground-lock/jingdezhen-charge-start.md)
- 出站：`GroundLockUtils`（协议 2.1.2 升锁、降锁）
- 会话写入：`GroundLockDao.upsertSession`，不要手写 `updatetime`
- 建单预约：`ParkingService.createOrder`
- 地磁驶出：`ParkingService.parkOut`，泊位写库成功后
- 强制驶离：`iroad_jdz` 的 `JZParkingService.parkOutForce`
- 扫码降锁：`NotifyBase.payBatchArrearNotifyListener`；合并缴费回调同样会调 `onPaid`
- 到期任务：`JDZApplicationMain` 在 `groundLock.enabled` 时启动 `GroundLockRaiseDueTask`
- 表和预约说明：[建表语句](/knowledge/road/ground-lock/ground-lock-bind-session-ddl.md)、[会话与预约升锁](/knowledge/road/ground-lock/ground-lock-session-schedule.md)、[升降锁挂载点](/knowledge/road/ground-lock/ground-lock-mount-points.md)
- 已回退的付款不结单：[地锁泊位付款不结单](/knowledge/road/ground-lock/ground-lock-pay-keep-parking.md)

## 以后怎么用

先确认运行时 `new` 的是哪个类、方法有没有被子类覆盖，再决定挂钩写在父类还是子类。调度时间用 `JingDeZhenChargeStart` 对一下日志里的 `chargeStart`，不要用名称相近的驶入时间或试算返回值。

集群上不要用 `groundLock.enabled` 单独停定时任务。锁已经升起后，不要再把它留在到期队列里空转；没走到驶出降锁的立着的锁，现有队列发现不了。

付款后如果金额还在涨，先看订单是否仍进行中、有没有支付记录，再决定是入账失败还是结单失败。不要把 `isFinishOrder=false` 当成继续计费的开关。
