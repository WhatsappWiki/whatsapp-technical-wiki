# Database Storage · 数据库存储

[消息目录](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 阅读顺序

本专题分别介绍加密状态与消息历史的存储。先读总览中的存储分类和术语，再查各库的地址、字段及存取行为。

| 顺序 | 文章 | 内容 |
| --- | --- | --- |
| 1 | [本地数据库与存储](../local-storage.md#zh) | 数据库职责、连接密钥、WAL、一致快照和事务边界 |
| 2 | [axolotl.db：密钥、会话与队列](axolotl.md#zh) | 15 张应用表及显式索引，identity、预密钥、会话和群状态 |
| 3 | [msgstore.db：消息、回执与媒体](msgstore.md#zh) | 6 张核心表及字段，JID/聊天/消息关系、状态、回执和媒体 |

### 结构范围

两篇数据库文章都按功能解释字段和生命周期，并给出对应表及索引的完整 DDL。消息库的 6 张表不是整个消息库；依赖配置创建的索引单列说明。密钥库的应用表也不包含数据库管理元数据。

### 相关主题

[端到端加密与密钥管理](../../encryption/README.md#zh) · [登录认证与会话](../../authentication/README.md#zh) · [图片、文件与语音消息](../../media/README.md#zh) · [多设备与数据同步](../../multi-device/README.md#zh)

<a id="en"></a>

## English

### Reading Order

This topic explains storage of cryptographic state and message history separately. Start with the storage categories and terminology in the overview, then look up each database’s addresses, fields, and storage behavior.

| Order | Article | Contents |
| --- | --- | --- |
| 1 | [Local Databases and Storage](../local-storage.md#en) | Database responsibilities, connection keys, WAL, consistent snapshots, and transaction boundaries |
| 2 | [axolotl.db: Keys, Sessions, and Queues](axolotl.md#en) | 15 application tables and explicit indexes; identities, prekeys, sessions, and group state |
| 3 | [msgstore.db: Messages, Receipts, and Media](msgstore.md#en) | 6 core tables and their fields; JID/chat/message relationships, statuses, receipts, and media |

### Structural Scope

Both database articles explain fields and lifecycles by function and include complete DDL for the covered tables and indexes. The 6 message tables are not the entire message database; indexes created conditionally by configuration are listed separately. The key database's application tables likewise exclude database-management metadata.

### Related Topics

[End-to-End Encryption and Key Management](../../encryption/README.md#en) · [Login, Authentication, and Sessions](../../authentication/README.md#en) · [Media Transfer and Voice Messages](../../media/README.md#en) · [Multi-Device and Data Synchronization](../../multi-device/README.md#en)
