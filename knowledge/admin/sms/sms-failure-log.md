---
title: "短信失败返回空串会吞掉原因"
type: reference
summary: "网关调用失败若只返回空串，上一层解析 JSON 时只剩空 message，真实异常留在另一个输出里。"
tags: [短信, 日志]
created: "2026-11-25"
edges:
  - path: /knowledge/admin/sms/signature-already-present.md
    label: about
    description: "同一发送方法里，签名是否追加看正文里有没有结束符" 
sources:
  - title: "iroad_base/doc/短信发送失败排查日志说明.md"
---

# 短信发送失败排查日志

## 背景与范围

微信注册验证码走 `SendSmsBase.sendSms`，再调用 `Client.ms` 向电信 `capService` 发 POST。网关或 TLS 出错时，`Client.ms` 原来只 `printStackTrace()`，方法返回空字符串。`SendSmsBase` 对空字符串做 JSON 解析，空指针的 `getMessage()` 是 null，`SendSMS.txt` 里只能看到「短信发送结果：」和 `{"message":""}`。接口则是「验证码发送失败」。

本次只加日志，不改发送地址、签名算法、成功时的 `state`。`JdzMsgSender` 未改。curl 能通、程序失败时，用新增日志区分证书、HTTP 状态和签名换行。

## 类与调用

| 类 | 作用 |
|---|---|
| `sys/com/cmpp/util/Client.java` `ms` | 打出实际请求地址、签名是否含换行、HTTP 状态、非 200 的错误体、完整异常栈 |
| `sys/com/sms/base/SendSmsBase.java` `sendSms` | 网关无返回或没有 `retCode` 时抛出带原因的异常，并把栈打进同一 logger |

调用关系：`SafecodeSms.generateSecurityCode` → `SendSmsBase.sendSms` → `JdzMsgSender.SENDER_CLINETS` 里的 `Telecom2024` → `Client.ms`。

日志 logger 名是 `SendSMS`，文件是 `SendSMS.txt`。异常栈不再只出现在 `catalina.out`。

## 关键日志

按一次失败请求的先后顺序：

1. `短信签名长度：…，含换行：true/false`。`sun.misc.BASE64Encoder` 常在签名末尾加换行。curl 手写的签名没有换行。这里为 `true` 时，先核对网关是否拒签。
2. `短信开始请求，地址：…`。地址来自这台 JVM 里 `Telecom2024` 客户端的 `HttpURL`，用来确认不是打到别的主机。
3. `短信请求写出失败`。`PrintWriter` 不抛异常，写出失败只走这条。
4. `短信HTTP状态：`。有状态码说明 TCP 和 TLS 已经完成。非 200 时紧跟 `短信HTTP错误体：`。
5. `短信HTTP调用失败，…异常：`。没有状态码、直接出现这条，是连接或证书失败。栈里常见 `PKIX path building failed`、`SSLHandshakeException`。
6. `SendSmsBase发送异常`。正文会写「短信网关无返回」或「没有 retCode」。`safecodesms` 日志里的「验证码发送失败原因」会带上这句话，不再是空 `message`。

成功时仍会打 HTTP 状态 200 和网关 JSON。`retCode=0000` 的 `state` 仍是 `1`。

## 部署

改动在 `iroad_base` 的 `com.cmpp.util.Client`、`com.sms.base.SendSmsBase`。路内系统加载的是这份类（或打进去的 jar）。只重启 `iroad_jdz` 而类还是旧包时，日志不会出现。

## 验收

1. 在原先失败的那台机器上再发一次注册验证码。
2. 同一条请求在 `SendSMS.txt` 里能看到地址、HTTP 状态或异常栈，三者至少有一项。
3. `safecodesms` 的失败原因不再是 `{"message":""}`。
4. 网关返回 `0000` 时，验证码接口仍是成功。
