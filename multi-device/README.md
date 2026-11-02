# Multi-Device and Data Synchronization · 多设备与数据同步

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍账号与多台设备之间的关联关系，以及[消息、历史记录](../messaging/storage/msgstore.md#zh)和应用状态的同步，整理[设备身份](../messaging/storage/axolotl.md#identities-zh)、会话、消息分发、离线恢复及平台差异。

### 介绍范围

- 主设备、关联设备和设备列表的关系。
- 设备关联、身份确认、会话建立与解除关联。
- 消息在多个设备之间的分发、确认和状态同步。
- [历史消息](../messaging/storage/msgstore.md#zh)、联系人、群组和应用状态同步。
- [离线队列](../messaging/storage/axolotl.md#zh)、冲突、重复事件与恢复流程。
- Web / Desktop 与移动端之间的行为差异。

### 已有文章

| 文章与章节 | 内容 |
| --- | --- |
| [多设备消息分发与状态同步](message-distribution.md#zh) | 设备目录、目标筛选、逐设备加密、任务重试与回执合并 |
| [目标筛选与本机排除](message-distribution.md#targets-zh) | PN/LID 当前设备、本人副本与消息目标快照 |
| [设备增删与会话更新](message-distribution.md#device-changes-zh) | key-index、dead_devices、session 与 Sender Key |
| [发送队列与连接恢复](message-distribution.md#queue-zh) | 执行预算、有效期、[重连与 ACK 超时](message-distribution.md#connection-zh)、[密文复用](message-distribution.md#ciphertext-zh) |
| [回执合并与状态回退](message-distribution.md#receipts-zh) | 设备/用户/群层级、[recipient_count 与乱序](message-distribution.md#regression-zh) |
| [历史迁移与回执一致性](message-distribution.md#history-zh) | ChatTransfer、90 天回执窗口、导入计数与孤立回执 |

### 文档组织与扩展方式

已有文章及章节从上表进入；下方规划表仅列尚未创建的文章。历史迁移和离线恢复也可先阅读消息分发文章中的对应章节。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `device-model.md` | 主设备、关联设备、[设备身份](../messaging/storage/axolotl.md#identities-zh)与列表 |
| `linking-and-unlinking.md` | 设备关联、身份确认、会话建立与解除关联 |
| `history-sync.md` | [历史消息](../messaging/storage/msgstore.md#zh)、联系人与群组同步 |
| `app-state-sync.md` | 应用状态、变更事件与同步处理 |
| `offline-recovery.md` | [离线队列](../messaging/storage/axolotl.md#zh)、冲突、重复事件与恢复 |
| `platform-differences.md` | 移动端、Web 与 Desktop 的行为对照 |

<a id="相关主题--related-topics"></a>

### 相关主题

[登录认证与会话](../authentication/#zh) · [消息发送与接收](../messaging/#zh) · [端到端加密与密钥管理](../encryption/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces account/device relationships and the synchronization of [messages, history](../messaging/storage/msgstore.md#en), and application state. It organizes [device identities](../messaging/storage/axolotl.md#identities-en), sessions, message distribution, offline recovery, and platform differences.

### Scope

- Relationships between primary devices, companion devices, and device lists.
- Device linking, identity confirmation, session establishment, and unlinking.
- Message distribution, acknowledgments, and state synchronization across devices.
- [History](../messaging/storage/msgstore.md#en), contacts, groups, and application-state synchronization.
- [Offline queues](../messaging/storage/axolotl.md#en), conflicts, duplicate events, and recovery.
- Behavioral differences between mobile, Web, and Desktop clients.

### Available Articles

| Article and section | Contents |
| --- | --- |
| [Multi-Device Message Distribution and State Synchronization](message-distribution.md#en) | Device directories, target selection, per-device encryption, job retries, and receipt aggregation |
| [Target Selection and Sender Exclusion](message-distribution.md#targets-en) | Current PN/LID device, own copies, and message target snapshots |
| [Device Changes and Session Updates](message-distribution.md#device-changes-en) | key-index, dead_devices, sessions, and Sender Keys |
| [Send Jobs and Connection Recovery](message-distribution.md#queue-en) | Execution budgets, expiration, [reconnection/ACK timeouts](message-distribution.md#connection-en), and [ciphertext reuse](message-distribution.md#ciphertext-en) |
| [Receipts and State Regression](message-distribution.md#receipts-en) | Device/user/group layers and [recipient_count/reordering](message-distribution.md#regression-en) |
| [History Transfer and Receipt Consistency](message-distribution.md#history-en) | ChatTransfer, the 90-day receipt window, imported counts, and orphan receipts |

### Documentation Organization and Expansion

Use the table above for available articles and sections. The planning table below lists articles that do not yet exist. History transfer and offline recovery are also covered in the corresponding message-distribution sections.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `device-model.md` | Primary and companion devices, [device identities](../messaging/storage/axolotl.md#identities-en), and device lists |
| `linking-and-unlinking.md` | Device linking, identity confirmation, session setup, and unlinking |
| `history-sync.md` | [Message history](../messaging/storage/msgstore.md#en), contacts, and group synchronization |
| `app-state-sync.md` | Application state, change events, and synchronization |
| `offline-recovery.md` | [Offline queues](../messaging/storage/axolotl.md#en), conflicts, duplicate events, and recovery |
| `platform-differences.md` | Behavior comparisons across mobile, Web, and Desktop |

### Related Topics

[Login, Authentication, and Sessions](../authentication/#en) · [Sending and Receiving Messages](../messaging/#en) · [End-to-End Encryption and Key Management](../encryption/#en)
