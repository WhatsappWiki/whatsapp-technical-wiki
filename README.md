# WhatsApp Technical Wiki · WhatsApp 技术知识库

<a id="whatsapp-wiki--whatsapp-技术知识库"></a>

<a id="contact"></a>

## Telegram Contact: [@ttjjda](https://t.me/ttjjda)

[中文](#zh) · [English](#en) · [从这里开始](#start-here) · [Start Here](#start-here-en) · [技术目录](#technical-index) · [Topic Index](#technical-index-english) · [Contact](#contact)

<a id="zh"></a>

## 中文

**中英双语的非官方 WhatsApp 技术文档。**

面向开发者与协议研究者，围绕账号注册、登录认证、通信协议、消息、媒体、通话、端到端加密和多设备同步组织内容。文章说明字段来源、编码与加密、缓存生命周期、调用条件及错误处理。

<a id="start-here"></a>
<a id="项目介绍"></a>
<a id="适合谁阅读"></a>

### 从这里开始

- **理解注册流程**：从[账号注册与验证码](registration/README.md#zh)开始，再读[exist 参数](registration/exist.md#zh)、[code 参数](registration/code.md#zh)和[register 参数](registration/register.md#zh)。
- **理解请求如何发出**：接着读[参数序列化与加密封装](registration/request-encryption.md#zh)和[发送流程与响应处理](registration/sending-requests.md#zh)。
- **理解连接与握手**：阅读 [Noise 握手与加密传输](protocol/noise.md#zh)，了解 XX、IK、fallback、后量子分支与登录结果。
- **理解二进制节点**：阅读 [Protocol Tree Node](protocol/protocol-tree-node.md#zh)，了解树结构、token 字典、JID、Python 编解码与精确回写。
- **解析 Argo 业务载荷**：阅读 [Argo](protocol/argo.md#zh)，了解 schema 选择、Header、类型、Python 读写与精确回写。
- **理解本地数据库**：阅读[存储总览](messaging/local-storage.md#zh)，再查 [axolotl.db](messaging/storage/axolotl.md#zh) 的密钥会话与 [msgstore.db](messaging/storage/msgstore.md#zh) 的消息、回执和媒体。
- **理解多设备消息**：阅读[多设备消息分发与状态同步](multi-device/message-distribution.md#zh)，了解目标筛选、逐设备加密、重试、回执合并及历史迁移。
- **浏览其他主题**：下方[技术目录](#technical-index)链接各主题简介与文章规划。

<a id="搜索主题--search-topics"></a>
<a id="搜索主题"></a>
<a id="technical-index"></a>

### 技术目录

| 章节 | 内容范围 |
| --- | --- |
| <a id="zh-authentication"></a>[02 · 登录认证与会话](authentication/README.md#zh) | [身份材料](messaging/local-storage.md#keystore-zh)、[会话持久化](messaging/storage/axolotl.md#sessions-zh)、[设备关联](multi-device/README.md#zh)与[重新登录](authentication/README.md#zh) |
| <a id="zh-protocol"></a>[03 · 通信协议与数据格式](protocol/README.md#zh) | [连接建立](protocol/noise.md#zh-state)、[Noise 握手](protocol/noise.md#zh)、[帧结构](protocol/tcp-framing.md#zh)、[二进制节点](protocol/protocol-tree-node.md#zh)、[Argo 载荷](protocol/argo.md#zh)和[请求封装](registration/request-encryption.md#zh) |
| <a id="zh-encryption"></a>[04 · 加密与密钥管理](encryption/README.md#zh) | [身份密钥](messaging/storage/axolotl.md#identities-zh)、[会话密钥](messaging/storage/axolotl.md#sessions-zh)、[预密钥](messaging/storage/axolotl.md#prekeys-zh)、[密钥数据库](messaging/storage/axolotl.md#zh)与[加密边界](protocol/noise.md#zh-layers) |
| <a id="zh-messaging"></a>[05 · 消息发送与接收](messaging/README.md#zh) | [消息构建](messaging/README.md#zh)、[接收](messaging/README.md#zh)、[回执](messaging/storage/msgstore.md#receipts-zh)、[重试](messaging/README.md#zh)、[去重](messaging/README.md#zh)、[群聊](messaging/README.md#zh)与[本地数据库](messaging/local-storage.md#zh) |
| <a id="zh-media"></a>[06 · 图片、文件与语音消息](media/README.md#zh) | [媒体上传下载](media/README.md#zh)、[加解密](media/README.md#zh)、[文件元数据](messaging/storage/msgstore.md#media-zh)与[语音消息](media/README.md#zh) |
| <a id="zh-calls"></a>[07 · 语音通话能力](calls/README.md#zh) | 已实现能力、[一对一语音](calls/voice-calls.md#zh)、[群组语音](calls/group-calls.md#zh)与演示范围 |
| <a id="zh-multi-device"></a>[08 · 多设备与数据同步](multi-device/README.md#zh) | [设备关联](multi-device/README.md#zh)、[消息分发](multi-device/message-distribution.md#zh)、[历史同步](multi-device/message-distribution.md#history-zh)与[离线处理](multi-device/message-distribution.md#connection-zh) |

内容范围中的链接优先指向现有文章或具体小节；未单独成篇的主题指向所属目录。

<a id="protocol-articles"></a>

### 协议文章直达

| 文章 | 内容 |
| --- | --- |
| [TCP 帧协议](protocol/tcp-framing.md#zh) | 三字节长度、握手与密文帧、标志与压缩、Protocol Tree Node 解析及 TCP 拆包。 |
| [Noise 握手与加密传输](protocol/noise.md#zh) | 经典 XX/IK、同连接 fallback、X25519/ML-KEM-512、Protobuf 与帧结构、服务器认证及登录状态。 |
| [Protocol Tree Node · 二进制节点与编解码](protocol/protocol-tree-node.md#zh) · [Python 代码](https://github.com/WhatsappWiki/whatsapp-protocol-tree-node) | 树结构、token 字典、长度与字符串打包、JID、zlib、Python read/write 与逐字节回写。 |
| [Argo · 二进制格式与编解码](protocol/argo.md#zh) · [Python 代码](https://github.com/WhatsappWiki/whatsapp-argo) | 节点业务载荷、schema 与 doc ID、Header、类型与错误、Python 读写和精确回写。 |

<a id="storage-articles"></a>

### 数据库文章直达

| 文章 | 内容 |
| --- | --- |
| [本地数据库与存储](messaging/local-storage.md#zh) | 加密状态与消息历史、连接密钥、WAL、一致快照及事务边界。 |
| [axolotl.db：密钥、会话与队列](messaging/storage/axolotl.md#zh) | 15 张应用表、20 个显式索引、预密钥消费、五维会话与群状态。 |
| [msgstore.db：消息、回执与媒体](messaging/storage/msgstore.md#zh) | 6 张核心表、150 列、20 个无条件创建索引、状态、回执与媒体关联。 |

<a id="multi-device-articles-zh"></a>

### 多设备文章直达

| 文章 | 内容 |
| --- | --- |
| [多设备消息分发与状态同步](multi-device/message-distribution.md#zh) | 设备目录、本机排除、逐设备与群加密、设备增删、重试与密文复用、ACK 去重、回执合并及 ChatTransfer。 |
| [回执计数与状态回退](multi-device/message-distribution.md#regression-zh) | recipient_count、PN/LID、乱序条件与普通气泡状态。 |
| [历史迁移与回执一致性](multi-device/message-distribution.md#history-zh) | 90 天回执窗口、导入计数、明细写入与孤立回执回放。 |

<a id="目录结构--directory-structure"></a>

### 目录结构

```text
whatsapp-technical-wiki/
├── README.md
├── registration/
│   ├── README.md
│   ├── exist.md
│   ├── code.md
│   ├── android-inputs.md
│   ├── request-encryption.md
│   └── sending-requests.md
├── authentication/README.md
├── protocol/
│   ├── README.md
│   ├── noise.md
│   ├── tcp-framing.md
│   ├── protocol-tree-node.md
│   └── argo.md
├── encryption/README.md
├── messaging/
│   ├── README.md
│   ├── local-storage.md
│   └── storage/
│       ├── README.md
│       ├── axolotl.md
│       └── msgstore.md
├── media/README.md
├── calls/README.md
└── multi-device/
    ├── README.md
    └── message-distribution.md
```

顶层目录使用小写英文与连字符命名。各目录README介绍主题并链接文章；上表提供注册14篇文章、通信协议文章、数据库3篇文章及多设备消息分发文章与章节的直接入口。

### 阅读方式与文档约定

具体文章按调用条件说明技术行为；API原始值、派生字段、缓存结果与最终HTTP正文分别介绍。

标识、号码、令牌和密码学载荷使用同格式合成示例；普通状态提供具体示例值。条件字段、空串、null、缺键、0和false分别处理。字段内部包装、ENC、H与Authorization保持各自职责，通道差异不由一个请求模板覆盖。

公开文档不包含真实验证码、访问令牌、私钥、完整认证材料或可识别个人身份的原始记录。

通话专题结合能力说明与独立的技术侧写，介绍已实现功能、交互表现、声音体验和演示范围。

<a id="参与贡献--contributing"></a>

<a id="contributing"></a>

### 参与贡献

可通过 [Issue](https://github.com/WhatsappWiki/whatsapp-technical-wiki/issues) 提出问题或文义修正，通过 [Pull Request](https://github.com/WhatsappWiki/whatsapp-technical-wiki/pulls) 补充文章、参考代码和翻译。说明具体触发条件、API要求、字段转换、状态变化及错误分支；示例使用合成材料。

每篇文章先完整中文区，再完整英文区；目录、正文、示例说明和相关链接保持对应。字段名、类名、接口名和协议标识保留原文。增加或调整文章时，同步首页、所属主题导航和两种语言的链接。

<a id="项目声明--project-notice"></a>

### 项目声明

本项目为独立、非官方技术文档，与WhatsApp或Meta不存在官方隶属、合作或背书关系。WhatsApp及相关名称和标识归各自权利人所有。

<a id="en"></a>

## English

**Unofficial WhatsApp technical documentation in Chinese and English.**

For developers and protocol researchers, with topics organized around registration, authentication, protocols, messaging, media, calls, end-to-end encryption, and multi-device synchronization. Articles explain field sources, encoding and encryption, cache lifetimes, calling conditions, and error handling.

<a id="start-here-en"></a>
<a id="about"></a>
<a id="intended-audience"></a>

### Start Here

- **Understand registration**: Begin with [Registration and Verification](registration/README.md#en), then read [exist parameters](registration/exist.md#en), [code parameters](registration/code.md#en), and [register parameters](registration/register.md#en).
- **Follow a request to transmission**: Continue with [Serialization and request encryption](registration/request-encryption.md#en) and [Sending and response handling](registration/sending-requests.md#en).
- **Understand connections and handshakes**: Read [Noise Handshakes and Encrypted Transport](protocol/noise.md#en) for XX, IK, fallback, post-quantum branches, and login outcomes.
- **Understand binary nodes**: Read [Protocol Tree Node](protocol/protocol-tree-node.md#en) for tree structure, token dictionaries, JIDs, Python encoding/decoding, and exact rewriting.
- **Parse Argo business payloads**: read [Argo](protocol/argo.md#en) for schema selection, headers, types, Python read/write and exact rewriting.
- **Understand local databases**: Read the [storage overview](messaging/local-storage.md#en), then [axolotl.db](messaging/storage/axolotl.md#en) for keys/sessions and [msgstore.db](messaging/storage/msgstore.md#en) for messages, receipts, and media.
- **Understand multi-device messages**: Read [Multi-Device Message Distribution and State Synchronization](multi-device/message-distribution.md#en) for target selection, per-device encryption, retries, receipt aggregation, and history transfer.
- **Explore other topics**: The [index below](#technical-index-english) links to topic introductions and planned articles.

<a id="search-topics"></a>
<a id="technical-index-english"></a>

### Topic Index

| Chapter | Scope |
| --- | --- |
| <a id="en-authentication"></a>[02 · Login, Authentication, and Sessions](authentication/README.md#en) | [Identity material](messaging/local-storage.md#keystore-en), [session persistence](messaging/storage/axolotl.md#sessions-en), [device linking](multi-device/README.md#en), and [reauthentication](authentication/README.md#en) |
| <a id="en-protocol"></a>[03 · Communication Protocols and Data Formats](protocol/README.md#en) | [Connections](protocol/noise.md#en-state), [Noise handshakes](protocol/noise.md#en), [framing](protocol/tcp-framing.md#en), [binary nodes](protocol/protocol-tree-node.md#en), [Argo payloads](protocol/argo.md#en), and [request packaging](registration/request-encryption.md#en) |
| <a id="en-encryption"></a>[04 · Encryption and Key Management](encryption/README.md#en) | [Identity keys](messaging/storage/axolotl.md#identities-en), [session keys](messaging/storage/axolotl.md#sessions-en), [prekeys](messaging/storage/axolotl.md#prekeys-en), the [key database](messaging/storage/axolotl.md#en), and [encryption boundaries](protocol/noise.md#en-layers) |
| <a id="en-messaging"></a>[05 · Sending and Receiving Messages](messaging/README.md#en) | [Message construction](messaging/README.md#en), [reception](messaging/README.md#en), [receipts](messaging/storage/msgstore.md#receipts-en), [retries](messaging/README.md#en), [deduplication](messaging/README.md#en), [groups](messaging/README.md#en), and [local databases](messaging/local-storage.md#en) |
| <a id="en-media"></a>[06 · Images, Files, and Voice Messages](media/README.md#en) | [Media upload/download](media/README.md#en), [encryption](media/README.md#en), [file metadata](messaging/storage/msgstore.md#media-en), and [voice messages](media/README.md#en) |
| <a id="en-calls"></a>[07 · Voice Calling Capabilities](calls/README.md#en) | Implemented capabilities, [one-to-one voice](calls/voice-calls.md#en), [group voice](calls/group-calls.md#en) and demonstration scope |
| <a id="en-multi-device"></a>[08 · Multi-Device and Data Synchronization](multi-device/README.md#en) | [Device linking](multi-device/README.md#en), [message distribution](multi-device/message-distribution.md#en), [history synchronization](multi-device/message-distribution.md#history-en), and [offline handling](multi-device/message-distribution.md#connection-en) |

Scope links point to existing articles or specific sections where available; topics without a dedicated article link to their topic directory.

<a id="protocol-articles-en"></a>

### Direct Protocol Articles

| Article | Scope |
| --- | --- |
| [TCP Framing](protocol/tcp-framing.md#en) | Three-byte lengths, handshake and encrypted frames, flags and compression, Protocol Tree Node parsing, and TCP stream handling. |
| [Noise Handshakes and Encrypted Transport](protocol/noise.md#en) | Classical XX/IK, fallback on the same connection, X25519/ML-KEM-512, Protobuf and framing, server authentication, and login states. |
| [Protocol Tree Node and Binary Encoding](protocol/protocol-tree-node.md#en) · [Python code](https://github.com/WhatsappWiki/whatsapp-protocol-tree-node) | Tree structure, token dictionaries, lengths and packed strings, JIDs, zlib, Python read/write, and exact rewriting. |
| [Argo · Binary Format and Encoding](protocol/argo.md#en) · [Python code](https://github.com/WhatsappWiki/whatsapp-argo) | Node business payloads, schemas and doc IDs, headers, types/errors, Python read/write and exact rewriting. |

<a id="storage-articles-en"></a>

### Direct Database Articles

| Article | Scope |
| --- | --- |
| [Local Databases and Storage](messaging/local-storage.md#en) | Cryptographic state versus history, connection keys, WAL, consistent snapshots, and transaction boundaries. |
| [axolotl.db: Keys, Sessions, and Queues](messaging/storage/axolotl.md#en) | 15 application tables, 20 explicit indexes, prekey consumption, five-dimensional sessions, and group state. |
| [msgstore.db: Messages, Receipts, and Media](messaging/storage/msgstore.md#en) | 6 core tables, 150 columns, 20 unconditionally created indexes, statuses, receipts, and media relationships. |

<a id="multi-device-articles-en"></a>

### Direct Multi-Device Articles

| Article | Scope |
| --- | --- |
| [Multi-Device Message Distribution and State Synchronization](multi-device/message-distribution.md#en) | Directories, sender exclusion, pairwise/group encryption, device changes, retries/ciphertext reuse, ACK deduplication, receipt aggregation, and ChatTransfer. |
| [Receipt Counts and State Regression](multi-device/message-distribution.md#regression-en) | recipient_count, PN/LID, reordering conditions, and ordinary bubble state. |
| [History Transfer and Receipt Consistency](multi-device/message-distribution.md#history-en) | The 90-day receipt window, imported counts, detail writes, and orphan replay. |

### Directory Structure

```text
whatsapp-technical-wiki/
├── README.md
├── registration/
│   ├── README.md
│   ├── exist.md
│   ├── code.md
│   ├── android-inputs.md
│   ├── request-encryption.md
│   └── sending-requests.md
├── authentication/README.md
├── protocol/
│   ├── README.md
│   ├── noise.md
│   ├── tcp-framing.md
│   ├── protocol-tree-node.md
│   └── argo.md
├── encryption/README.md
├── messaging/
│   ├── README.md
│   ├── local-storage.md
│   └── storage/
│       ├── README.md
│       ├── axolotl.md
│       └── msgstore.md
├── media/README.md
├── calls/README.md
└── multi-device/
    ├── README.md
    └── message-distribution.md
```

Top-level directories use lowercase English names and hyphens. Topic READMEs introduce their scope and link articles; the tables above directly open all 14 registration articles, the communication protocol articles, 3 database articles, and the multi-device distribution article and sections.

### Reading Guide and Documentation Conventions

Articles explain behavior under concrete calling conditions and distinguish raw API values, derived fields, caches, and final HTTP bodies.

Public documentation excludes real verification codes, access tokens, private keys, complete authentication material, and identifiable personal records.

The calling topic combines capability descriptions with independent technical perspectives on implemented features, interactions, audio experience and demonstration scope.

<a id="en-contributing"></a>

### Contribution Guide

Use [Issues](https://github.com/WhatsappWiki/whatsapp-technical-wiki/issues) for questions/corrections and [Pull Requests](https://github.com/WhatsappWiki/whatsapp-technical-wiki/pulls) for articles, reference code, and translations. Describe concrete triggers, API requirements, field transformations, state changes, and error branches; use synthetic example material.

Place a complete Chinese section before a complete English section, keeping indexes, prose, example explanations, and related links aligned. Preserve field/class/interface names and protocol identifiers. When adding or changing articles, update homepage/topic navigation and links in both languages.

### Project Notice

This is independent, unofficial technical documentation, without affiliation, partnership, or endorsement from WhatsApp or Meta. Product names and trademarks belong to their respective owners.
