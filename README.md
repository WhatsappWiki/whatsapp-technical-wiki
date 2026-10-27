# WhatsApp Wiki · WhatsApp 技术知识库

<a id="contact"></a>

## Telegram Contact: [@ttjjda](https://t.me/ttjjda)

**WhatsApp 注册、登录、通信协议、消息收发与语音通话技术文档。**

**Technical documentation on WhatsApp registration, authentication, protocols, messaging, and voice calls.**

[中文](#zh) · [English](#en) · [技术目录 / Technical Index](#technical-index) · [参与贡献 / Contributing](#contributing) · [Contact](#contact)

<a id="zh"></a>

## 中文

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

| 章节 | 内容范围 | English keywords |
| --- | --- | --- |
| <a id="zh-registration"></a>[01 · 账号注册与验证码](registration/) | 号码检查、验证码申请与提交、注册状态及请求字段 | Registration, Verification, OTP |
| <a id="zh-authentication"></a>[02 · 登录认证与会话](authentication/) | 身份材料、会话持久化、设备关联与重新登录 | Login, Authentication, Session, Device Linking |
| <a id="zh-protocol"></a>[03 · 通信协议与数据格式](protocol/) | 连接建立、握手、帧结构、编码和请求封装 | Protocol, Handshake, Serialization, Protobuf |
| <a id="zh-encryption"></a>[04 · 加密与密钥管理](encryption/) | 身份密钥、会话密钥、预密钥与加密边界 | E2EE, Signal Protocol, Key Management |
| <a id="zh-messaging"></a>[05 · 消息发送与接收](messaging/) | 消息构建、接收、回执、重试、去重与群聊 | Send / Receive Messages, ACK, Receipts |
| <a id="zh-media"></a>[06 · 图片、文件与语音消息](media/) | 媒体上传下载、加解密、文件元数据与语音消息 | Media, Upload, Download, Voice Messages |
| <a id="zh-calls"></a>[07 · 语音与视频通话](calls/) | 呼入呼出、通话信令、媒体传输与网络质量 | Voice Calls, Video Calls, VoIP, Signaling |
| <a id="zh-multi-device"></a>[08 · 多设备与数据同步](multi-device/) | 设备关联、消息分发、历史同步与离线恢复 | Multi-Device, Companion Devices, Sync |

### 目录结构 / Directory Structure

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

Directory names use lowercase English words and hyphens. Each topic README introduces its scope; detailed articles or subdirectories with their own README can be added within that topic.

### 阅读方式与文档约定

建议按“注册与登录 → 通信协议与加密 → 消息与媒体 → 通话与多设备”的顺序阅读。首页负责导航，各主题目录负责功能简介，具体流程、字段或实验放在该目录下的文章与子目录中。

每篇技术文章应尽量注明：目标平台与版本、问题背景、流程或调用链、字段及编码、证据来源、验证方法、结果和仍未确认的部分。静态分析、运行时观测、离线复算和服务端实际接受应分别说明。

示例统一使用合成或脱敏数据。公开文档不包含真实验证码、访问令牌、私钥、完整认证材料或可识别个人身份的原始记录。

<a id="en"></a>

## English

### About

WhatsApp Wiki is an unofficial technical knowledge base for developers, protocol researchers, and mobile engineers. It organizes research on WhatsApp account registration, login and authentication, communication protocols, sending and receiving messages, media transfer, voice and video calls, and multi-device synchronization.

The project aims to turn scattered research notes into searchable, traceable, and reproducible documentation. Articles explain workflows, field origins, encodings, state transitions, implementation approaches, and verification limits through technical notes, diagrams, field references, experiments, and reference code.

The repository is organized into English-named topic directories. Each directory has a `README.md` with an overview, scope, and planned articles. Open a chapter below to browse its directory; detailed articles will be added incrementally. A directory or planned article does not imply an implemented feature, a fully recovered protocol, or validation across every platform and version.

### Intended Audience

- Developers studying WhatsApp registration, verification codes, authentication, and account sessions.
- Engineers researching instant messaging, end-to-end encryption, delivery reliability, and multi-device systems.
- Researchers analyzing Android, iOS, Web, and Desktop clients.
- Contributors improving documentation, experiments, reference implementations, or translations.

### Technical Index

| Chapter | Scope | Navigation |
| --- | --- | --- |
| <a id="en-registration"></a>01 · Registration and Verification | Phone-number checks, OTP requests, verification, and request fields | [Open topic](registration/) |
| <a id="en-authentication"></a>02 · Login and Authentication | Credentials, sessions, device linking, logout, and recovery | [Open topic](authentication/) |
| <a id="en-protocol"></a>03 · Protocols and Data Formats | Connections, handshakes, framing, serialization, and encoding | [Open topic](protocol/) |
| <a id="en-encryption"></a>04 · Encryption and Keys | E2EE, Signal Protocol concepts, key storage, and rotation | [Open topic](encryption/) |
| <a id="en-messaging"></a>05 · Sending and Receiving Messages | Message pipelines, acknowledgments, receipts, and retries | [Open topic](messaging/) |
| <a id="en-media"></a>06 · Media and Voice Messages | Uploads, downloads, media encryption, and audio messages | [Open topic](media/) |
| <a id="en-calls"></a>07 · Voice and Video Calls | Incoming / outgoing calls, signaling, media, and call quality | [Open topic](calls/) |
| <a id="en-multi-device"></a>08 · Multi-Device and Synchronization | Companion devices, message distribution, and history sync | [Open topic](multi-device/) |

### Reading Guide and Documentation Conventions

A suggested reading order is registration and authentication → protocols and encryption → messaging and media → calls and multi-device systems. This homepage provides navigation, topic READMEs introduce their scope, and detailed workflows, fields, or experiments belong in articles and subdirectories within the relevant topic.

Technical articles should identify the target platform and version, problem, workflow or call chain, fields and encodings, evidence, verification method, results, and unresolved questions. Static analysis, runtime observation, offline recomputation, and actual server acceptance should be described separately.

Examples use synthetic or redacted data. Public documentation excludes real verification codes, access tokens, private keys, complete authentication material, and identifiable personal records.

<a id="contributing"></a>

## 参与贡献 / Contributing

欢迎通过 Issue 提出问题、报告文档错误或建议研究主题，通过 Pull Request 补充文章、实验、参考代码和翻译。提交时请注明目标平台及版本、复现步骤、证据和验证范围；推测或未验证结论请明确标记。

Use Issues for questions, documentation corrections, and topic suggestions. Pull requests are welcome for articles, experiments, reference code, and translations. Include the target platform and version, reproduction steps, evidence, and verification scope. Clearly label hypotheses and unverified conclusions.

中英文内容应保持含义一致；专有字段名、类名、接口名和协议标识保留原文。添加新章节时，请同步更新两种语言的目录；新增文章后，也请更新所属目录的 `README.md`。

Keep Chinese and English content aligned. Preserve original field names, class names, interface names, and protocol identifiers. Update both indexes when adding chapters and update the parent topic README when adding articles.

## 项目声明 / Project Notice

本项目为独立、非官方技术文档项目，与 WhatsApp 或 Meta 不存在官方隶属、合作或背书关系。WhatsApp 及相关名称和标识归各自权利人所有。研究结论适用于文中明确记录的版本与条件。

This is an independent, unofficial technical documentation project. It is not affiliated with, sponsored by, or endorsed by WhatsApp or Meta. Product names and trademarks belong to their respective owners. Research findings apply to the versions and conditions explicitly documented.

## 搜索主题 / Search Topics

WhatsApp 技术文档、WhatsApp 注册、验证码、登录认证、通信协议、消息发送、消息接收、语音消息、语音通话、视频通话、端到端加密、多设备同步、Android 逆向、协议分析。

WhatsApp wiki, WhatsApp registration, phone verification, OTP, login, authentication, session management, WhatsApp protocol, message sending and receiving, messaging, delivery receipts, voice messages, voice calls, video calls, VoIP, end-to-end encryption, E2EE, Signal Protocol, multi-device synchronization, Android reverse engineering, protocol analysis.
