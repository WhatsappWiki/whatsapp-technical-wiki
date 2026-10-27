# End-to-End Encryption and Key Management · 端到端加密与密钥管理

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍消息加密与密钥管理相关概念，整理身份密钥、预密钥、会话、密钥轮换及验证方法，同时明确区分传输保护、请求签名与消息端到端加密。

### 介绍范围

- 端到端加密（E2EE）与传输层保护的职责和边界。
- Signal Protocol 相关概念：Identity Key、PreKey、Signed PreKey 和会话状态。
- 密钥生成、存储、选择、轮换与失效流程。
- 一对一消息、群聊与关联设备场景的密钥分发研究。
- 消息加密、解密、签名与完整性验证的输入输出。
- 应用层字段加密、请求签名和消息端到端加密的区别。
- 原始输入复算、字节比较和参考实现的验证范围。

### 文档状态与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接并标明状态。

<a id="en"></a>

## English

### Overview

This directory introduces message encryption and key-management concepts. It organizes identity keys, prekeys, sessions, key rotation, and verification methods, with clear distinctions between transport protection, request signing, and message end-to-end encryption.

### Scope

- Responsibilities and boundaries of end-to-end encryption (E2EE) and transport protection.
- Signal Protocol concepts: identity keys, prekeys, signed prekeys, and session state.
- Key generation, storage, selection, rotation, and expiration.
- Key distribution research for one-to-one conversations, groups, and companion devices.
- Inputs and outputs of message encryption, decryption, signing, and integrity verification.
- Differences between field encryption, request signatures, and message E2EE.
- Verification limits of raw-input recomputation, byte comparisons, and reference implementations.

### Documentation Status and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and record its status when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `encryption-boundaries.md` | 传输保护、字段加密、请求签名与 E2EE 的区别 | Transport protection, field encryption, request signing, and E2EE |
| `identity-and-prekeys.md` | 身份密钥、PreKey 与 Signed PreKey | Identity keys, prekeys, and signed prekeys |
| `sessions/` | 会话建立、密钥状态及状态持久化 | Session establishment, key state, and persistence |
| `key-lifecycle.md` | 密钥生成、存储、选择、轮换与失效 | Key generation, storage, selection, rotation, and expiration |
| `group-and-device-keys.md` | 群聊及关联设备的密钥分发研究 | Key-distribution research for groups and companion devices |
| `verification.md` | 原始输入复算、签名验证与字节比较 | Raw-input recomputation, signature verification, and byte comparisons |

## 相关主题 / Related Topics

[登录认证与会话 / Login, Authentication, and Sessions](../authentication/) · [通信协议与数据格式 / Communication Protocols and Data Formats](../protocol/) · [消息发送与接收 / Sending and Receiving Messages](../messaging/) · [多设备与数据同步 / Multi-Device and Data Synchronization](../multi-device/)
