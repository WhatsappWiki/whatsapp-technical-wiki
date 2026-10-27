# Multi-Device and Data Synchronization · 多设备与数据同步

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍账号与多台设备之间的关联关系，以及消息、历史记录和应用状态的同步，整理设备身份、会话、消息分发、离线恢复及平台差异。

### 介绍范围

- 主设备、关联设备和设备列表的关系。
- 设备关联、身份确认、会话建立与解除关联。
- 消息在多个设备之间的分发、确认和状态同步。
- 历史消息、联系人、群组和应用状态同步。
- 离线队列、冲突、重复事件与恢复流程。
- Web / Desktop 与移动端之间的行为差异。

### 文档状态与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接并标明状态。

<a id="en"></a>

## English

### Overview

This directory introduces account/device relationships and the synchronization of messages, history, and application state. It organizes device identities, sessions, message distribution, offline recovery, and platform differences.

### Scope

- Relationships between primary devices, companion devices, and device lists.
- Device linking, identity confirmation, session establishment, and unlinking.
- Message distribution, acknowledgments, and state synchronization across devices.
- History, contacts, groups, and application-state synchronization.
- Offline queues, conflicts, duplicate events, and recovery.
- Behavioral differences between mobile, Web, and Desktop clients.

### Documentation Status and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and record its status when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `device-model.md` | 主设备、关联设备、设备身份与列表 | Primary and companion devices, device identities, and device lists |
| `linking-and-unlinking.md` | 设备关联、身份确认、会话建立与解除关联 | Device linking, identity confirmation, session setup, and unlinking |
| `message-distribution.md` | 跨设备消息分发、确认与状态同步 | Cross-device message distribution, acknowledgments, and state sync |
| `history-sync.md` | 历史消息、联系人与群组同步 | Message history, contacts, and group synchronization |
| `app-state-sync.md` | 应用状态、变更事件与同步处理 | Application state, change events, and synchronization |
| `offline-recovery.md` | 离线队列、冲突、重复事件与恢复 | Offline queues, conflicts, duplicate events, and recovery |
| `platform-differences.md` | 移动端、Web 与 Desktop 的行为对照 | Behavior comparisons across mobile, Web, and Desktop |

## 相关主题 / Related Topics

[登录认证与会话 / Login, Authentication, and Sessions](../authentication/) · [消息发送与接收 / Sending and Receiving Messages](../messaging/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/)
