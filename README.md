# WhatsApp Wiki · WhatsApp 技术知识库

<a id="contact"></a>

## Telegram Contact: [@ttjjda](https://t.me/ttjjda)

[中文](#zh) · [English](#en) · [技术目录](#technical-index) · [Technical Index](#technical-index-english) · [Contact](#contact)

<a id="zh"></a>

## 中文

**WhatsApp 注册、登录、通信协议、消息收发与语音通话技术文档。**

### 项目介绍

WhatsApp Wiki 是一个面向开发者、协议研究者和移动端工程师的非官方技术知识库，围绕 WhatsApp 账号注册、登录认证、通信协议、消息发送与接收、媒体传输、语音及视频通话、多设备同步等主题组织文档。

项目希望把分散的分析笔记整理为可检索、可追溯、可复现的技术资料：既解释业务流程，也记录字段来源、编码方式、状态变化、实现思路与验证边界。内容以技术文档、流程说明、字段表、研究记录和参考代码为主要形式。

仓库按技术主题划分英文目录，每个目录的 `README.md` 提供功能简介、介绍范围和后续文章规划。点击下方章节名称即可进入相应目录；详细文章将逐步补充。目录与规划不表示相关功能已经实现、协议已经完整恢复，或所有平台及版本均已验证。

### 适合谁阅读

- 希望理解 WhatsApp 注册、验证码和账号会话流程的开发者。
- 研究即时通信、端到端加密、消息可靠性和多设备架构的工程师。
- 分析 Android、iOS、Web 或 Desktop 客户端的移动端与协议研究者。
- 希望提交文档、修正文义、补充实验或提供中英文翻译的贡献者。

<a id="technical-index"></a>

### 技术目录

| 章节 | 内容范围 |
| --- | --- |
| <a id="zh-registration"></a>[01 · 账号注册与验证码](registration/#zh) | exist/code/register 明文参数、安全字段、加密封装与请求发送 |
| <a id="zh-authentication"></a>[02 · 登录认证与会话](authentication/#zh) | 身份材料、会话持久化、设备关联与重新登录 |
| <a id="zh-protocol"></a>[03 · 通信协议与数据格式](protocol/#zh) | 连接建立、握手、帧结构、编码和请求封装 |
| <a id="zh-encryption"></a>[04 · 加密与密钥管理](encryption/#zh) | 身份密钥、会话密钥、预密钥与加密边界 |
| <a id="zh-messaging"></a>[05 · 消息发送与接收](messaging/#zh) | 消息构建、接收、回执、重试、去重与群聊 |
| <a id="zh-media"></a>[06 · 图片、文件与语音消息](media/#zh) | 媒体上传下载、加解密、文件元数据与语音消息 |
| <a id="zh-calls"></a>[07 · 语音与视频通话](calls/#zh) | 呼入呼出、通话信令、媒体传输与网络质量 |
| <a id="zh-multi-device"></a>[08 · 多设备与数据同步](multi-device/#zh) | 设备关联、消息分发、历史同步与离线恢复 |

<a id="目录结构--directory-structure"></a>

### 目录结构

```text
wiki/
├── README.md
├── registration/README.md
├── authentication/README.md
├── protocol/README.md
├── encryption/README.md
├── messaging/README.md
├── media/README.md
├── calls/README.md
└── multi-device/README.md
```

顶层目录使用小写英文与连字符命名。各目录 `README.md` 介绍主题，后续在对应目录内增加文章或带 `README.md` 的子目录。

### 阅读方式与文档约定

建议按“注册与登录 → 通信协议与加密 → 消息与媒体 → 通话与多设备”的顺序阅读。首页负责导航，各主题目录负责功能简介，具体流程、字段或实验放在该目录下的文章与子目录中。

技术文章直接介绍用途、触发条件、流程、字段含义、取值方式、编码与加密、状态变化和错误处理。涉及平台 API 时注明必要的调用条件；字段来源说明获取方式及所属模块。

标识、号码、令牌和密码学载荷使用同格式随机示例；普通状态提供具体示例值。公开文档不包含真实验证码、访问令牌、私钥、完整认证材料或可识别个人身份的原始记录。

<a id="参与贡献--contributing"></a>

<a id="contributing"></a>

### 参与贡献

欢迎通过 Issue 提出问题、报告文档错误或建议研究主题，通过 Pull Request 补充文章、实验、参考代码和翻译。提交时请注明目标平台及版本、复现步骤、证据和验证范围；推测或未验证结论请明确标记。

每篇文章先完整中文区，再完整英文区；目录表、正文、示例说明和相关链接分别归入对应语言，不在同一表格中并列两种语言的说明。中英文含义保持一致；字段名、类名、接口名和协议标识保留原文。添加章节或文章时，同步更新两种语言的目录及所属目录的 `README.md`。

<a id="项目声明--project-notice"></a>

### 项目声明

本项目为独立、非官方技术文档项目，与 WhatsApp 或 Meta 不存在官方隶属、合作或背书关系。WhatsApp 及相关名称和标识归各自权利人所有。研究结论适用于文中明确记录的版本与条件。

<a id="搜索主题--search-topics"></a>

### 搜索主题

WhatsApp 技术文档、WhatsApp 注册、验证码、登录认证、通信协议、消息发送、消息接收、语音消息、语音通话、视频通话、端到端加密、多设备同步、Android 逆向、协议分析。

<a id="en"></a>

## English

**Technical documentation on WhatsApp registration, authentication, protocols, messaging, and voice calls.**

### About

WhatsApp Wiki is an unofficial technical knowledge base for developers, protocol researchers, and mobile engineers. It organizes research on WhatsApp account registration, login and authentication, communication protocols, sending and receiving messages, media transfer, voice and video calls, and multi-device synchronization.

The project aims to turn scattered research notes into searchable, traceable, and reproducible documentation. Articles explain workflows, field origins, encodings, state transitions, implementation approaches, and verification limits through technical notes, diagrams, field references, experiments, and reference code.

The repository is organized into English-named topic directories. Each directory has a `README.md` with an overview, scope, and planned articles. Open a chapter below to browse its directory; detailed articles will be added incrementally. A directory or planned article does not imply an implemented feature, a fully recovered protocol, or validation across every platform and version.

### Intended Audience

- Developers studying WhatsApp registration, verification codes, authentication, and account sessions.
- Engineers researching instant messaging, end-to-end encryption, delivery reliability, and multi-device systems.
- Researchers analyzing Android, iOS, Web, and Desktop clients.
- Contributors improving documentation, experiments, reference implementations, or translations.

<a id="technical-index-english"></a>

### Topic Index

| Chapter | Scope | Navigation | Keywords |
| --- | --- | --- | --- |
| <a id="en-registration"></a>01 · Registration and Verification | exist/code/register parameters, security fields, encryption, and transmission | [Open topic](registration/#en) | Registration, Verification, Request Parameters, Encryption |
| <a id="en-authentication"></a>02 · Login and Authentication | Credentials, sessions, device linking, logout, and recovery | [Open topic](authentication/#en) | Login, Authentication, Session, Device Linking |
| <a id="en-protocol"></a>03 · Protocols and Data Formats | Connections, handshakes, framing, serialization, and encoding | [Open topic](protocol/#en) | Protocol, Handshake, Serialization, Protobuf |
| <a id="en-encryption"></a>04 · Encryption and Keys | E2EE, Signal Protocol concepts, key storage, and rotation | [Open topic](encryption/#en) | E2EE, Signal Protocol, Key Management |
| <a id="en-messaging"></a>05 · Sending and Receiving Messages | Message pipelines, acknowledgments, receipts, and retries | [Open topic](messaging/#en) | Send / Receive Messages, ACK, Receipts |
| <a id="en-media"></a>06 · Media and Voice Messages | Uploads, downloads, media encryption, and audio messages | [Open topic](media/#en) | Media, Upload, Download, Voice Messages |
| <a id="en-calls"></a>07 · Voice and Video Calls | Incoming / outgoing calls, signaling, media, and call quality | [Open topic](calls/#en) | Voice Calls, Video Calls, VoIP, Signaling |
| <a id="en-multi-device"></a>08 · Multi-Device and Synchronization | Companion devices, message distribution, and history sync | [Open topic](multi-device/#en) | Multi-Device, Companion Devices, Sync |

### Directory Structure

```text
wiki/
├── README.md
├── registration/README.md
├── authentication/README.md
├── protocol/README.md
├── encryption/README.md
├── messaging/README.md
├── media/README.md
├── calls/README.md
└── multi-device/README.md
```

Directory names use lowercase English words and hyphens. Each topic README introduces its scope; detailed articles or subdirectories with their own README can be added within that topic.

### Reading Guide and Documentation Conventions

A suggested reading order is registration and authentication → protocols and encryption → messaging and media → calls and multi-device systems. This homepage provides navigation, topic READMEs introduce their scope, and detailed workflows, fields, or experiments belong in articles and subdirectories within the relevant topic.

Technical articles directly explain purpose, triggers, workflows, field meanings and acquisition, encodings and encryption, state changes, and error handling. State the necessary conditions for platform API calls; describe field sources by their acquisition method and owning module.

Identifiers, numbers, tokens, and cryptographic payloads use random examples in the same representation; ordinary state fields have concrete example values. Public documentation excludes real verification codes, access tokens, private keys, complete authentication material, and identifiable personal records.

<a id="en-contributing"></a>

### Contribution Guide

Use Issues for questions, documentation corrections, and topic suggestions. Pull requests are welcome for articles, experiments, reference code, and translations. Include the target platform and version, reproduction steps, evidence, and verification scope. Clearly label hypotheses and unverified conclusions.

Place a complete Chinese section before a complete English section. Keep navigation tables, prose, example explanations, and related links within their respective language sections; do not put Chinese and English descriptions side by side in a table. Keep both languages aligned, preserve field/class/interface names and protocol identifiers, and update both indexes and the parent topic README when adding chapters or articles.

### Project Notice

This is an independent, unofficial technical documentation project. It is not affiliated with, sponsored by, or endorsed by WhatsApp or Meta. Product names and trademarks belong to their respective owners. Research findings apply to the versions and conditions explicitly documented.

### Search Topics

WhatsApp wiki, WhatsApp registration, phone verification, OTP, login, authentication, session management, WhatsApp protocol, message sending and receiving, messaging, delivery receipts, voice messages, voice calls, video calls, VoIP, end-to-end encryption, E2EE, Signal Protocol, multi-device synchronization, Android reverse engineering, protocol analysis.
