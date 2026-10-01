# SunshineCTF 2026 Writeups

SunshineCTF 2026 题解集。赛事由 Hack@UCF 主办、与 BSides Orlando 2026 同期，2026 年 9 月 26 日至 28 日线上举行，共 758 支队伍参赛。

## Web

### [kidding — JWT 的 kid 被当成文件路径](09-kidding-JWT-kid路径穿越.md)

站点的 JWT header 里，`kid` 值被当成文件名拼进路径，服务端读到什么文件就把内容当签名密钥。编辑器密钥硬编码在源码里，而源码又能从同一个读文件的口子拿到。

一句话概括：`kid` 被当成路径而不是编号，黑名单只挡住展示没挡住加载，密钥写死在能被读到的源码里。三条叠起来，读者证就变成了编辑证。

`sun{h0tw1r3d_4dm1n_jwt}`

## 复现环境

- 靶机：`https://kidding.web.2026.sunshinectf.games/`
- 抓包：Burp Suite
- 文中所有请求、响应和数值均为实测结果，未做推测性描述

## 说明

内容仅供安全研究与教学使用。所有测试均在赛事提供的公开靶机上完成，未涉及任何真实系统或用户数据。
