---
title: "地锁绑定与进行中会话表"
type: reference
summary: "tra_ground_lock_bind 与 tra_ground_lock_session 的建表语句。一泊位一条会话，驶出删除。"
tags: [地锁, 表结构]
created: "2026-11-25"
updated: "2026-11-25"
edges:
  - path: /knowledge/road/ground-lock/road-ground-lock.md
    label: about
    description: "这条说明被路内地锁开发记忆引用"
sources:
  - title: "iroad_jdz/doc/地锁绑定与进行中会话表.sql"
---

# 地锁绑定与进行中会话表

建表语句从 `iroad_jdz/doc/地锁绑定与进行中会话表.sql` 移入。适用库与 `tra_road_order` 相同。插入会话时不要在 SQL 里再写 `createtime`、`updatetime`，框架会自动补。

```sql
-- =============================================================================
-- 路内地锁：泊位绑定表 + 进行中会话表
-- 适用库：parkcloud_jdz（与 tra_road_order 同库）
-- 说明：
--   1) 绑定表：泊位 ↔ 车位锁平台设备身份（协议 beId / numbers / deviceSerial）
--   2) 进行中会话表：当前泊位地锁归属哪笔路内订单（支付降锁按 orderid 比对，防补缴误降）
--   3) 升锁预约不靠本表倒计时，用 Redis ZSET（见 GroundLockRaiseDueUtils）
--   4) 余额缴费本业务不支持，不在会话/编排范围内
-- =============================================================================

-- -----------------------------------------------------------------------------
-- 1. 泊位与地锁绑定
-- -----------------------------------------------------------------------------
CREATE TABLE IF NOT EXISTS `tra_ground_lock_bind` (
  `id` varchar(32) NOT NULL COMMENT '主键',
  `comid` varchar(32) NOT NULL COMMENT '业主/管理公司ID',
  `berth_id` varchar(32) DEFAULT NULL COMMENT '本方泊位主键 com_berth.id，可空',
  `berthcode` varchar(32) NOT NULL COMMENT '本方泊位号',
  `be_id` bigint(20) DEFAULT NULL COMMENT '车位锁平台泊位ID（协议 beId）',
  `numbers` varchar(64) DEFAULT NULL COMMENT '车位锁平台泊位号（协议 numbers）',
  `device_serial` varchar(64) DEFAULT NULL COMMENT '地锁设备序列号（协议 deviceSerial）',
  `device_type` int(11) DEFAULT NULL COMMENT '锁体类型 0-6（协议 deviceType）',
  `isvalid` int(11) NOT NULL DEFAULT '1' COMMENT '是否有效 0无效 1有效',
  `creator` varchar(32) DEFAULT NULL COMMENT '创建人',
  `createtime` bigint(20) DEFAULT NULL COMMENT '创建时间毫秒',
  `updatetime` bigint(20) DEFAULT NULL COMMENT '更新时间毫秒',
  `remark` varchar(200) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_comid_berthcode` (`comid`,`berthcode`),
  KEY `idx_be_id` (`be_id`),
  KEY `idx_device_serial` (`device_serial`),
  KEY `idx_isvalid` (`isvalid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='路内地锁-泊位绑定表';

-- -----------------------------------------------------------------------------
-- 2. 地锁进行中会话（一泊位一条）
-- 写入：PDA 录单 createOrder 成功且存在绑定
-- 关闭：驶出强制降锁 / 会话失效；支付成功按 orderid 命中后降锁，会话可保留到驶出
-- -----------------------------------------------------------------------------
CREATE TABLE IF NOT EXISTS `tra_ground_lock_session` (
  `id` varchar(32) NOT NULL COMMENT '主键',
  `comid` varchar(32) NOT NULL COMMENT '业主/管理公司ID',
  `berth_id` varchar(32) DEFAULT NULL COMMENT '本方泊位主键，可空',
  `berthcode` varchar(32) NOT NULL COMMENT '本方泊位号',
  `be_id` bigint(20) DEFAULT NULL COMMENT '车位锁平台泊位ID快照',
  `numbers` varchar(64) DEFAULT NULL COMMENT '车位锁平台泊位号快照',
  `device_serial` varchar(64) DEFAULT NULL COMMENT '设备序列号快照',
  `device_type` int(11) DEFAULT NULL COMMENT '锁体类型快照',
  `orderid` varchar(32) NOT NULL COMMENT '路内进行中订单ID（tra_road_order.id）',
  `carplate` varchar(20) DEFAULT NULL COMMENT '车牌号',
  `lock_state` int(11) NOT NULL DEFAULT '1' COMMENT '锁状态：0已升 1已降/未升 2降锁失败',
  `startparktime` bigint(20) DEFAULT NULL COMMENT '真实驶入毫秒（试算未付/回退用；下次升锁看 next_raise_due）',
  `next_raise_due` bigint(20) DEFAULT NULL COMMENT '下次应升锁时间毫秒（与 ZSET score 一致，可空）',
  `isvalid` int(11) NOT NULL DEFAULT '1' COMMENT '是否有效 1进行中（关闭会话请 DELETE 本行，勿堆历史）',
  `createtime` bigint(20) DEFAULT NULL COMMENT '创建时间毫秒',
  `updatetime` bigint(20) DEFAULT NULL COMMENT '更新时间毫秒',
  `remark` varchar(200) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_comid_berthcode` (`comid`,`berthcode`),
  KEY `idx_orderid` (`orderid`),
  KEY `idx_next_raise_due` (`next_raise_due`),
  KEY `idx_lock_state` (`lock_state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='路内地锁-进行中会话表（一泊位一条，驶出删除）';
```
