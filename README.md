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

- **理解注册流程**：从[账号注册与验证码](registration/README.md#zh)开始，再读[exist 参数](registration/exist.md#zh)与[code 参数](registration/code.md#zh)。
- **查阅字段与代码**：进入[Android 字段获取代码](registration/android-inputs.md#zh)或[安全字段总览](registration/security-fields/README.md#zh)。
- **理解请求如何发出**：接着读[参数序列化与加密封装](registration/request-encryption.md#zh)和[发送流程与响应处理](registration/sending-requests.md#zh)。
- **理解连接与握手**：阅读 [Noise 握手与加密传输](protocol/noise.md#zh)，了解 XX、IK、fallback、后量子分支与登录结果。
- **理解二进制节点**：阅读 [Protocol Tree Node](protocol/protocol-tree-node.md#zh)，了解树结构、token 字典、JID、Python 编解码与精确回写。
- **解析 Argo 业务载荷**：阅读 [Argo](protocol/argo.md#zh)，了解 schema 选择、Header、类型、Python 读写与精确回写。
- **浏览其他主题**：下方技术目录链接各主题简介与文章规划。

<a id="搜索主题--search-topics"></a>
<a id="搜索主题"></a>
<a id="technical-index"></a>

### 技术目录

| 章节 | 内容范围 |
| --- | --- |
| <a id="zh-registration"></a>[01 · 账号注册与验证码](registration/README.md#zh) | exist/code参数、Android输入、安全字段、封装和发送 |
| <a id="zh-authentication"></a>[02 · 登录认证与会话](authentication/README.md#zh) | 身份材料、会话持久化、设备关联与重新登录 |
| <a id="zh-protocol"></a>[03 · 通信协议与数据格式](protocol/README.md#zh) | 连接建立、[Noise 握手](protocol/noise.md#zh)、帧结构、[二进制节点](protocol/protocol-tree-node.md#zh)、[Argo 载荷](protocol/argo.md#zh)和请求封装 |
| <a id="zh-encryption"></a>[04 · 加密与密钥管理](encryption/README.md#zh) | 身份密钥、会话密钥、预密钥与加密边界 |
| <a id="zh-messaging"></a>[05 · 消息发送与接收](messaging/README.md#zh) | 消息构建、接收、回执、重试、去重与群聊 |
| <a id="zh-media"></a>[06 · 图片、文件与语音消息](media/README.md#zh) | 媒体上传下载、加解密、文件元数据与语音消息 |
| <a id="zh-calls"></a>[07 · 语音与视频通话](calls/README.md#zh) | 呼入呼出、通话信令、媒体传输与网络质量 |
| <a id="zh-multi-device"></a>[08 · 多设备与数据同步](multi-device/README.md#zh) | 设备关联、消息分发、历史同步与离线恢复 |

<a id="registration-articles"></a>

### 注册文章直达

阅读顺序、请求之间的状态延续和主题分组见[注册目录](registration/README.md#zh)。exist/code列出各自的已知字段目录与条件；合成JSON/Java展示对象还需经过所选通道的序列化、加密与发送。code申请验证路线，后续验证请求提交材料；HTTP成功、业务sent与完成注册分开判断。

| 文章 | 内容 |
| --- | --- |
| [exist 参数](registration/exist.md#zh) | 71项已知内层字段、条件省略、号码备份选择与缓存，附JSON/Java合成示意。 |
| [code 参数](registration/code.md#zh) | 70项已知内层字段、验证路线、已保存备份与交互状态，附JSON/Java合成示意。 |
| [Android 字段获取代码](registration/android-inputs.md#zh) | 四组Java辅助类：系统输入、示例存储、包与证书读取、字节编码；区分原始API与字段生命周期。 |
| [安全字段总览](registration/security-fields/README.md#zh) | 七字段的职责与表示，gpia回调、对象结果槽、context字典及库级状态的生命周期。 |
| [gpia · Play结果与应用信息](registration/security-fields/gpia.md#zh) | Play结果码、requestHash、provider准备与限流、超时范围和CBC包装。 |
| [_gg · Play结果精简包装](registration/security-fields/gg.md#zh) | `_ic/_it`内部结构、独立IV与发布，以及公共Map保留旧槽的条件。 |
| [_gi · 应用完整性信息](registration/security-fields/gi.md#zh) | 主安装文件、DEX、签名与目录摘要，CBC包装及源槽/context/结果缓存。 |
| [_gp · 声明权限摘要](registration/security-fields/gp.md#zh) | 声明权限名称的modified UTF-8、排序与摘要，空值/失败材料及缓存。 |
| [_gs · 系统信号与加密包络](registration/security-fields/gs.md#zh) | 13次路径检查、19整数CSV、prepared状态、独立随机源与X25519/GCM包络。 |
| [_ge · 模块与路径检查](registration/security-fields/ge.md#zh) | 模块与条件su路径、布尔JSON、配置转换及缓存命中行为。 |
| [_ga · 路径年龄与运行环境](registration/security-fields/ga.md#zh) | 路径年龄、Monkey与boot输入，仅bi单独CBC，公共材料快照及整份结果缓存。 |
| [参数序列化与加密封装](registration/request-encryption.md#zh) | 普通文本/预编码字节、Kotlin/原生序列化、ENC、H、证书链及失败分支。 |
| [发送流程与响应处理](registration/sending-requests.md#zh) | 最终合并、Kotlin/WAMsys通道、应用头、响应分层、等待与重试分支。 |

<a id="protocol-articles"></a>

### 协议文章直达

| 文章 | 内容 |
| --- | --- |
| [TCP 帧协议](protocol/tcp-framing.md#zh) | 三字节长度、握手与密文帧、标志与压缩、Protocol Tree Node 解析及 TCP 拆包。 |
| [Noise 握手与加密传输](protocol/noise.md#zh) | 经典 XX/IK、同连接 fallback、X25519/ML-KEM-512、Protobuf 与帧结构、服务器认证及登录状态。 |
| [Protocol Tree Node · 二进制节点与编解码](protocol/protocol-tree-node.md#zh) | 树结构、token 字典、长度与字符串打包、JID、zlib、Python read/write 与逐字节回写。 |
| [Argo · 二进制格式与编解码](protocol/argo.md#zh) | 节点业务载荷、schema 与 doc ID、Header、类型与错误、Python 读写和精确回写。 |

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
│   ├── security-fields/
│   │   ├── README.md
│   │   ├── gpia.md
│   │   ├── gg.md
│   │   ├── gi.md
│   │   ├── gp.md
│   │   ├── gs.md
│   │   ├── ge.md
│   │   └── ga.md
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
├── messaging/README.md
├── media/README.md
├── calls/README.md
└── multi-device/README.md
```

顶层目录使用小写英文与连字符命名。各目录README介绍主题并链接文章；上表提供注册13篇文章和通信协议文章的直接入口。

### 阅读方式与文档约定

具体文章按调用条件说明技术行为；API原始值、派生字段、缓存结果与最终HTTP正文分别介绍。

标识、号码、令牌和密码学载荷使用同格式合成示例；普通状态提供具体示例值。条件字段、空串、null、缺键、0和false分别处理。字段内部包装、ENC、H与Authorization保持各自职责，通道差异不由一个请求模板覆盖。

公开文档不包含真实验证码、访问令牌、私钥、完整认证材料或可识别个人身份的原始记录。

<a id="参与贡献--contributing"></a>

<a id="contributing"></a>

### 参与贡献

可通过Issue提出问题或文义修正，通过Pull Request补充文章、参考代码和翻译。说明具体触发条件、API要求、字段转换、状态变化及错误分支；示例使用合成材料。

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

- **Understand registration**: Begin with [Registration and Verification](registration/README.md#en), then read [exist parameters](registration/exist.md#en) and [code parameters](registration/code.md#en).
- **Look up fields and code**: Open [Android input examples](registration/android-inputs.md#en) or the [Security-field overview](registration/security-fields/README.md#en).
- **Follow a request to transmission**: Continue with [Serialization and request encryption](registration/request-encryption.md#en) and [Sending and response handling](registration/sending-requests.md#en).
- **Understand connections and handshakes**: Read [Noise Handshakes and Encrypted Transport](protocol/noise.md#en) for XX, IK, fallback, post-quantum branches, and login outcomes.
- **Understand binary nodes**: Read [Protocol Tree Node](protocol/protocol-tree-node.md#en) for tree structure, token dictionaries, JIDs, Python encoding/decoding, and exact rewriting.
- **Parse Argo business payloads**: read [Argo](protocol/argo.md#en) for schema selection, headers, types, Python read/write and exact rewriting.
- **Explore other topics**: The index below links to topic introductions and planned articles.

<a id="search-topics"></a>
<a id="technical-index-english"></a>

### Topic Index

| Chapter | Scope |
| --- | --- |
| <a id="en-registration"></a>[01 · Registration and Verification](registration/README.md#en) | exist/code parameters, Android inputs, security fields, packaging, and sending |
| <a id="en-authentication"></a>[02 · Login, Authentication, and Sessions](authentication/README.md#en) | Identity material, session persistence, device linking, and login recovery |
| <a id="en-protocol"></a>[03 · Communication Protocols and Data Formats](protocol/README.md#en) | Connections, [Noise handshakes](protocol/noise.md#en), framing, [binary nodes](protocol/protocol-tree-node.md#en), [Argo payloads](protocol/argo.md#en), and request packaging |
| <a id="en-encryption"></a>[04 · Encryption and Key Management](encryption/README.md#en) | Identity keys, session keys, prekeys, and encryption boundaries |
| <a id="en-messaging"></a>[05 · Sending and Receiving Messages](messaging/README.md#en) | Message construction, reception, receipts, retries, deduplication, and groups |
| <a id="en-media"></a>[06 · Images, Files, and Voice Messages](media/README.md#en) | Media upload/download, encryption, file metadata, and voice messages |
| <a id="en-calls"></a>[07 · Voice and Video Calls](calls/README.md#en) | Incoming/outgoing calls, signaling, media transport, and network quality |
| <a id="en-multi-device"></a>[08 · Multi-Device and Data Synchronization](multi-device/README.md#en) | Device linking, message distribution, history synchronization, and offline recovery |

<a id="registration-articles-en"></a>

### Direct Registration Articles

See the [registration directory](registration/README.md#en) for reading order, retained state between requests, and topic groups. exist/code list their own known field catalogs and conditions. Synthetic JSON/Java display objects still require the chosen transport’s serialization, encryption, and sending. code requests a verification route; later verification submits material. HTTP success, business sent, and completed registration are separate.

| Article | Scope |
| --- | --- |
| [exist parameters](registration/exist.md#en) | 71 known inner fields, conditional omission, number-bound backup selection, and caches, with synthetic JSON/Java illustrations. |
| [code parameters](registration/code.md#en) | 70 known inner fields, verification routes, retained backup and interaction state, with synthetic JSON/Java illustrations. |
| [Android input examples](registration/android-inputs.md#en) | Four Java helpers for system inputs, example storage, package/certificate reads, and byte encoding; distinguish raw APIs from field lifetimes. |
| [Security-field overview](registration/security-fields/README.md#en) | Roles and representations of seven fields; lifetimes of the gpia callback, owner slots, context dictionaries, and library state. |
| [gpia · Play outcome and application metadata](registration/security-fields/gpia.md#en) | Play outcome codes, requestHash, provider preparation/rate limits, timeout scope, and CBC wrapping. |
| [_gg · Compact Play outcome](registration/security-fields/gg.md#en) | Internal `_ic/_it` structure, independent IV/publication, and retained older slots in the public Map. |
| [_gi · Application integrity](registration/security-fields/gi.md#en) | Primary installation file, DEX, certificate and directory digests; CBC and source-slot/context/result caches. |
| [_gp · Declared-permission digest](registration/security-fields/gp.md#en) | Modified UTF-8, sorting, and hashing of declared permission names; empty/failure material and caching. |
| [_gs · System signals and encrypted envelope](registration/security-fields/gs.md#en) | 13 path checks, a 19-integer CSV, prepared state, separate randomness, and an X25519/GCM envelope. |
| [_ge · Module and path checks](registration/security-fields/ge.md#en) | Module and conditional su-path checks, boolean JSON, configuration conversion, and cache-hit behavior. |
| [_ga · Path ages and runtime environment](registration/security-fields/ga.md#en) | Path ages, Monkey and boot inputs; CBC only for bi, a public-material snapshot, and whole-result caching. |
| [Serialization and request encryption](registration/request-encryption.md#en) | Ordinary text/preencoded bytes, Kotlin/native serialization, ENC, H, certificate chains, and failure branches. |
| [Sending and response handling](registration/sending-requests.md#en) | Final merging, Kotlin/WAMsys paths, application headers, response layers, waits, and retry branches. |

<a id="protocol-articles-en"></a>

### Direct Protocol Articles

| Article | Scope |
| --- | --- |
| [TCP Framing](protocol/tcp-framing.md#en) | Three-byte lengths, handshake and encrypted frames, flags and compression, Protocol Tree Node parsing, and TCP stream handling. |
| [Noise Handshakes and Encrypted Transport](protocol/noise.md#en) | Classical XX/IK, fallback on the same connection, X25519/ML-KEM-512, Protobuf and framing, server authentication, and login states. |
| [Protocol Tree Node and Binary Encoding](protocol/protocol-tree-node.md#en) | Tree structure, token dictionaries, lengths and packed strings, JIDs, zlib, Python read/write, and exact rewriting. |
| [Argo · Binary Format and Encoding](protocol/argo.md#en) | Node business payloads, schemas and doc IDs, headers, types/errors, Python read/write and exact rewriting. |

### Directory Structure

```text
whatsapp-technical-wiki/
├── README.md
├── registration/
│   ├── README.md
│   ├── exist.md
│   ├── code.md
│   ├── android-inputs.md
│   ├── security-fields/
│   │   ├── README.md
│   │   ├── gpia.md
│   │   ├── gg.md
│   │   ├── gi.md
│   │   ├── gp.md
│   │   ├── gs.md
│   │   ├── ge.md
│   │   └── ga.md
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
├── messaging/README.md
├── media/README.md
├── calls/README.md
└── multi-device/README.md
```

Top-level directories use lowercase English names and hyphens. Topic READMEs introduce their scope and link articles; the tables above directly open all 13 registration articles and the communication protocol articles.

### Reading Guide and Documentation Conventions

Articles explain behavior under concrete calling conditions and distinguish raw API values, derived fields, caches, and final HTTP bodies.

Identifiers, numbers, tokens, and cryptographic payloads use synthetic examples in the same representation; ordinary states have concrete example values. Distinguish conditional presence, empty text, null, omission, 0, and false. Field envelopes, ENC, H, and Authorization have separate roles; one request template does not define every transport.

Public documentation excludes real verification codes, access tokens, private keys, complete authentication material, and identifiable personal records.

<a id="en-contributing"></a>

### Contribution Guide

Use Issues for questions/corrections and Pull Requests for articles, reference code, and translations. Describe concrete triggers, API requirements, field transformations, state changes, and error branches; use synthetic example material.

Place a complete Chinese section before a complete English section, keeping indexes, prose, example explanations, and related links aligned. Preserve field/class/interface names and protocol identifiers. When adding or changing articles, update homepage/topic navigation and links in both languages.

### Project Notice

This is independent, unofficial technical documentation, without affiliation, partnership, or endorsement from WhatsApp or Meta. Product names and trademarks belong to their respective owners.
