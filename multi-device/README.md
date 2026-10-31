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

### 文档组织与扩展方式

本页介绍多设备与数据同步主题；文内链接指向已有的相关内容，下方规划路径尚未创建。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `device-model.md` | 主设备、关联设备、[设备身份](../messaging/storage/axolotl.md#identities-zh)与列表 |
| `linking-and-unlinking.md` | 设备关联、身份确认、会话建立与解除关联 |
| `message-distribution.md` | 跨设备消息分发、确认与状态同步 |
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

### Documentation Organization and Expansion

This page introduces multi-device synchronization. Inline links lead to existing related content; the planned paths below have not been created.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `device-model.md` | Primary and companion devices, [device identities](../messaging/storage/axolotl.md#identities-en), and device lists |
| `linking-and-unlinking.md` | Device linking, identity confirmation, session setup, and unlinking |
| `message-distribution.md` | Cross-device message distribution, acknowledgments, and state sync |
| `history-sync.md` | [Message history](../messaging/storage/msgstore.md#en), contacts, and group synchronization |
| `app-state-sync.md` | Application state, change events, and synchronization |
| `offline-recovery.md` | [Offline queues](../messaging/storage/axolotl.md#en), conflicts, duplicate events, and recovery |
| `platform-differences.md` | Behavior comparisons across mobile, Web, and Desktop |

### Related Topics

[Login, Authentication, and Sessions](../authentication/#en) · [Sending and Receiving Messages](../messaging/#en) · [End-to-End Encryption and Key Management](../encryption/#en)
