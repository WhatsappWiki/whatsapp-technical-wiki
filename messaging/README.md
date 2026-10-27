# Sending and Receiving Messages · 消息发送与接收

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 消息从创建到发送、从接收到展示的处理链路，整理消息结构、状态、回执、可靠性、群聊事件与本地数据一致性。

### 介绍范围

- 发送路径：创建消息、生成标识、序列化、加密、发送和状态更新。
- 接收路径：收到数据、解析、解密、存储、通知与回执。
- 文本、引用、回复、转发、表情回应及消息编辑或撤回的结构研究。
- 服务端确认、送达回执、已读回执与它们各自代表的状态。
- 超时、重试、重复消息、顺序处理和离线消息恢复。
- 群聊消息的发送、接收、成员变化与相关事件。
- 本地数据库、发送队列和 UI 显示之间的一致性。

### 文档组织与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="en"></a>

## English

### Overview

This directory introduces WhatsApp message processing from creation to transmission and from reception to display. It organizes message structures, states, receipts, reliability, group events, and local-data consistency.

### Scope

- Outbound pipeline: creation, identifiers, serialization, encryption, transmission, and state updates.
- Inbound pipeline: reception, parsing, decryption, storage, notifications, and receipts.
- Research on text messages, quotes, replies, forwarding, reactions, edits, and deletion events.
- Server acknowledgments, delivery receipts, read receipts, and their distinct meanings.
- Timeouts, retries, duplicate handling, ordering, and offline-message recovery.
- Group messaging, membership changes, and related events.
- Consistency between local databases, outbound queues, and UI state.

### Documentation Organization and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `sending.md` | 消息创建、标识、序列化、加密与发送 | Message creation, identifiers, serialization, encryption, and sending |
| `receiving.md` | 消息接收、解析、解密、存储与通知 | Reception, parsing, decryption, storage, and notifications |
| `message-types/` | 文本、引用、回复、转发及表情回应 | Text, quotes, replies, forwarding, and reactions |
| `receipts.md` | 服务端确认、送达回执与已读回执 | Server acknowledgments, delivery receipts, and read receipts |
| `reliability.md` | 超时、重试、顺序、去重与离线恢复 | Timeouts, retries, ordering, deduplication, and offline recovery |
| `groups/` | 群聊消息、成员变化与相关事件 | Group messages, membership changes, and related events |
| `edits-and-deletion.md` | 消息编辑、撤回及状态更新 | Message edits, deletion events, and state updates |
| `local-storage.md` | 本地数据库、发送队列与 UI 一致性 | Local databases, outbound queues, and UI consistency |

## 相关主题 / Related Topics

[通信协议与数据格式 / Communication Protocols and Data Formats](../protocol/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/) · [图片、文件与语音消息 / Media Transfer and Voice Messages](../media/) · [多设备与数据同步 / Multi-Device and Data Synchronization](../multi-device/)
