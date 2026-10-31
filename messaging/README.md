# Sending and Receiving Messages · 消息发送与接收

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 消息从创建到发送、从接收到展示的处理链路，整理消息结构、状态、[回执](storage/msgstore.md#receipts-zh)、可靠性、群聊事件与本地数据一致性。

### 介绍范围

- 发送路径：创建消息、生成标识、序列化、加密、发送和[状态更新](storage/msgstore.md#messages-zh)。
- 接收路径：收到数据、解析、解密、[存储](local-storage.md#zh)、通知与[回执](storage/msgstore.md#receipts-zh)。
- 文本、引用、回复、转发、表情回应及消息编辑或撤回的结构研究。
- [服务端确认](storage/msgstore.md#messages-zh)、[送达回执](storage/msgstore.md#receipts-zh)、[已读回执](storage/msgstore.md#receipts-zh)与它们各自代表的状态。
- 超时、重试、重复消息、顺序处理和离线消息恢复。
- 群聊消息的发送、接收、成员变化与相关事件。
- [本地数据库](local-storage.md#zh)、发送队列和 [UI 显示](storage/msgstore.md#zh)之间的一致性。

### 已有文章

| 文章 | 内容 |
| --- | --- |
| [本地数据库与存储](local-storage.md#zh) | [数据库职责](local-storage.md#zh)、[WAL](local-storage.md#zh)、[一致快照](local-storage.md#zh)与[连接密钥](local-storage.md#keystore-zh) |
| [数据库专题](storage/README.md#zh) | [axolotl.db](storage/axolotl.md#zh) [密钥与会话](storage/axolotl.md#zh)；[msgstore.db](storage/msgstore.md#zh) [消息、回执及媒体](storage/msgstore.md#zh) |

### 文档组织与扩展方式

已有文章从上表进入；下方规划表仅列尚未创建的文章或子目录。各文章直接说明相应功能、字段和条件行为。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `sending.md` | 消息创建、标识、序列化、加密与发送 |
| `receiving.md` | 消息接收、解析、解密、[存储](local-storage.md#zh)与通知 |
| `message-types/` | 文本、引用、回复、转发及表情回应 |
| `receipts.md` | [服务端确认](storage/msgstore.md#messages-zh)、[送达回执](storage/msgstore.md#receipts-zh)与[已读回执](storage/msgstore.md#receipts-zh) |
| `reliability.md` | 超时、重试、顺序、去重与离线恢复 |
| `groups/` | 群聊消息、成员变化与相关事件 |
| `edits-and-deletion.md` | 消息编辑、撤回及[状态更新](storage/msgstore.md#messages-zh) |

<a id="相关主题--related-topics"></a>

### 相关主题

[通信协议与数据格式](../protocol/#zh) · [端到端加密与密钥管理](../encryption/#zh) · [图片、文件与语音消息](../media/#zh) · [多设备与数据同步](../multi-device/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces WhatsApp message processing from creation to transmission and from reception to display. It organizes message structures, states, [receipts](storage/msgstore.md#receipts-en), reliability, group events, and local-data consistency.

### Scope

- Outbound pipeline: creation, identifiers, serialization, encryption, transmission, and [state updates](storage/msgstore.md#messages-en).
- Inbound pipeline: reception, parsing, decryption, [storage](local-storage.md#en), notifications, and [receipts](storage/msgstore.md#receipts-en).
- Research on text messages, quotes, replies, forwarding, reactions, edits, and deletion events.
- [Server acknowledgments](storage/msgstore.md#messages-en), [delivery receipts](storage/msgstore.md#receipts-en), [read receipts](storage/msgstore.md#receipts-en), and their distinct meanings.
- Timeouts, retries, duplicate handling, ordering, and offline-message recovery.
- Group messaging, membership changes, and related events.
- Consistency between [local databases](local-storage.md#en), outbound queues, and [UI state](storage/msgstore.md#en).

### Available Articles

| Article | Contents |
| --- | --- |
| [Local Databases and Storage](local-storage.md#en) | [Database responsibilities](local-storage.md#en), [WAL](local-storage.md#en), [consistent snapshots](local-storage.md#en), and [connection keys](local-storage.md#keystore-en) |
| [Database Storage](storage/README.md#en) | [axolotl.db](storage/axolotl.md#en) [keys/sessions](storage/axolotl.md#en); [msgstore.db](storage/msgstore.md#en) [messages, receipts, and media](storage/msgstore.md#en) |

### Documentation Organization and Expansion

Use the table above for available articles. The planning table below lists articles or subdirectories that do not yet exist. Each article explains its functions, fields, and conditional behavior directly.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `sending.md` | Message creation, identifiers, serialization, encryption, and sending |
| `receiving.md` | Reception, parsing, decryption, [storage](local-storage.md#en), and notifications |
| `message-types/` | Text, quotes, replies, forwarding, and reactions |
| `receipts.md` | [Server acknowledgments](storage/msgstore.md#messages-en), [delivery receipts](storage/msgstore.md#receipts-en), and [read receipts](storage/msgstore.md#receipts-en) |
| `reliability.md` | Timeouts, retries, ordering, deduplication, and offline recovery |
| `groups/` | Group messages, membership changes, and related events |
| `edits-and-deletion.md` | Message edits, deletion events, and [state updates](storage/msgstore.md#messages-en) |

### Related Topics

[Communication Protocols and Data Formats](../protocol/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Media Transfer and Voice Messages](../media/#en) · [Multi-Device and Data Synchronization](../multi-device/#en)
