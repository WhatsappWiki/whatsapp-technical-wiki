# Communication Protocols and Data Formats · 通信协议与数据格式

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 请求与连接数据的组织方式，整理[连接建立](noise.md#zh)、[握手](noise.md#zh)、[数据帧](tcp-framing.md#zh)、序列化、编码、确认与错误表示，并区分注册、消息和通话的协议边界。

### 介绍范围

- [注册请求](../registration/README.md#zh)、长连接消息协议和通话信令的边界。
- [连接建立](noise.md#zh)、[握手](noise.md#zh)、[身份认证](noise.md#zh)、心跳及[断线重连](noise.md#zh-fallback)。
- [二进制帧](tcp-framing.md#zh)、[节点与属性](protocol-tree-node.md#zh-model)、消息封装和序列化格式研究。
- [Argo](argo.md#zh) [业务载荷](argo.md#zh)、[Header](argo.md#zh-header)、[Wire Schema](argo.md#zh-schema)、类型与[精确回写](argo.md#zh-rewrite)。
- Protobuf 等结构化数据的 schema、字段类型和版本变化。
- [请求标识](../registration/sending-requests.md#zh)、序列号、时间戳、[回执](../messaging/storage/msgstore.md#receipts-zh)和错误表示。
- UTF-8、字节序、[URL 编码](../registration/request-encryption.md#zh)、[Base64](../registration/android-inputs.md#zh) 与 [Base64url](../registration/android-inputs.md#zh) 的具体使用位置。
- 原始字节、解码结果、运行时观测和协议推断之间的对应关系。

### 文档组织与扩展方式

本页提供主题简介与文章导航；后续文章规划列出可继续展开的连接、格式与编码主题。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

### 文章导航

| 文章 | 内容 |
| --- | --- |
| [TCP 帧协议](tcp-framing.md#zh) | 三字节大端长度、连接前导、[握手](noise.md#zh)与传输帧、解密、压缩及节点解析顺序。 |
| [Noise 握手与加密传输](noise.md#zh) | [XX](noise.md#zh-xx)、[IK](noise.md#zh-fallback)、同连接 [fallback](noise.md#zh-fallback)、[ML-KEM-512 混合握手](noise.md#zh-pq)、[帧结构](noise.md#zh-wire)、[服务器认证](noise.md#zh)与[登录结果](noise.md#zh-login)。 |
| [Protocol Tree Node · 二进制节点与编解码](protocol-tree-node.md#zh) · [Python 代码](https://github.com/WhatsappWiki/whatsapp-protocol-tree-node) | [节点模型](protocol-tree-node.md#zh-model)、[列表与 token 字典](protocol-tree-node.md#zh-tokens)、[长度头](protocol-tree-node.md#zh-binary)、[字符串打包](protocol-tree-node.md#zh-packed)、[JID](protocol-tree-node.md#zh-jid)、[zlib](protocol-tree-node.md#zh-envelope)、[Python read/write](protocol-tree-node.md#zh-python) 与[逐字节回写](protocol-tree-node.md#zh-preservation)。 |
| [Argo · 二进制格式与编解码](argo.md#zh) · [Python 代码](https://github.com/WhatsappWiki/whatsapp-argo) | 节点[业务载荷](argo.md#zh)、[Header](argo.md#zh-header)、[标量块与 Core](argo.md#zh-header)、[schema 选择](argo.md#zh-schema)、[类型与错误](argo.md#zh-types)、[Python 读写](argo.md#zh-python)和[精确回写](argo.md#zh-rewrite)。 |

理解完整接收链路时，可依次阅读 [TCP 帧协议](tcp-framing.md#zh)、[Noise 握手与加密传输](noise.md#zh)、[Protocol Tree Node](protocol-tree-node.md#zh)：先确定帧边界，再认证解密，最后处理明文封装并恢复节点。业务结果声明 [Argo](argo.md#zh) 时，再阅读 [Argo](argo.md#zh) 解析节点内的字节载荷。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `protocol-boundaries.md` | [注册请求](../registration/README.md#zh)、消息连接与通话信令的范围 |
| `connection-lifecycle.md` | [建立连接](noise.md#zh)、心跳、断开与[重连](noise.md#zh-fallback) |
| `handshake.md` | [握手](noise.md#zh)、连接认证及状态变化 |
| `binary-format/` | 复合消息封装与其他二进制格式专题 |
| `protobuf/` | 结构化消息、schema 与字段版本 |
| `encoding.md` | UTF-8、字节序、[URL 编码](../registration/request-encryption.md#zh)及 [Base64](../registration/android-inputs.md#zh) 系列 |
| `identifiers-and-errors.md` | [请求标识](../registration/sending-requests.md#zh)、序列号、时间戳、[回执](../messaging/storage/msgstore.md#receipts-zh)与错误 |

<a id="相关主题--related-topics"></a>

### 相关主题

[账号注册与验证码](../registration/#zh) · [端到端加密与密钥管理](../encryption/#zh) · [消息发送与接收](../messaging/#zh) · [语音与视频通话](../calls/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces the organization of WhatsApp requests and connection data. It covers [connection setup](noise.md#en), [handshakes](noise.md#en), [framing](tcp-framing.md#en), serialization, encoding, acknowledgments, and errors while separating registration, messaging, and calling boundaries.

### Scope

- Boundaries between [registration requests](../registration/README.md#en), persistent messaging connections, and call signaling.
- [Connection establishment](noise.md#en), [handshakes](noise.md#en), [authentication](noise.md#en), heartbeats, and [reconnection](noise.md#en-fallback).
- [Binary frames](tcp-framing.md#en), [nodes and attributes](protocol-tree-node.md#en-model), message envelopes, and serialization formats.
- [Argo](argo.md#en) [business payloads](argo.md#en), [headers](argo.md#en-header), [Wire Schemas](argo.md#en-schema), types and [exact rewriting](argo.md#en-rewrite).
- Structured formats such as Protobuf: schemas, field types, and version changes.
- [Request identifiers](../registration/sending-requests.md#en), sequence numbers, timestamps, [acknowledgments](../messaging/storage/msgstore.md#receipts-en), and errors.
- Specific uses of UTF-8, byte order, [URL encoding](../registration/request-encryption.md#en), [Base64](../registration/android-inputs.md#en), and [Base64url](../registration/android-inputs.md#en).
- Mapping raw bytes to decoded data, runtime observations, and protocol hypotheses.

### Documentation Organization and Expansion

This page provides the topic overview and article navigation. Planned articles identify further connection, format, and encoding topics.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Articles

| Article | Scope |
| --- | --- |
| [TCP Framing](tcp-framing.md#en) | Three-byte big-endian lengths, connection prefixes, [handshake](noise.md#en) and transport frames, decryption, compression, and node parsing. |
| [Noise Handshakes and Encrypted Transport](noise.md#en) | [XX](noise.md#en-xx), [IK](noise.md#en-fallback), [fallback on the same connection](noise.md#en-fallback), [ML-KEM-512 hybrid handshakes](noise.md#en-pq), [framing](noise.md#en-wire), [server authentication](noise.md#en), and [login outcomes](noise.md#en-login). |
| [Protocol Tree Node and Binary Encoding](protocol-tree-node.md#en) · [Python code](https://github.com/WhatsappWiki/whatsapp-protocol-tree-node) | [Node model](protocol-tree-node.md#en-model), [lists and token dictionaries](protocol-tree-node.md#en-tokens), [length headers](protocol-tree-node.md#en-binary), [packed strings](protocol-tree-node.md#en-packed), [JIDs](protocol-tree-node.md#en-jid), [zlib](protocol-tree-node.md#en-envelope), [Python read/write](protocol-tree-node.md#en-python), and [exact rewriting](protocol-tree-node.md#en-preservation). |
| [Argo · Binary Format and Encoding](argo.md#en) · [Python code](https://github.com/WhatsappWiki/whatsapp-argo) | Node [business payloads](argo.md#en), [headers](argo.md#en-header), [scalar blocks and Core](argo.md#en-header), [schema selection](argo.md#en-schema), [types/errors](argo.md#en-types), [Python read/write](argo.md#en-python) and [exact rewriting](argo.md#en-rewrite). |

To follow the receiving pipeline, read [TCP Framing](tcp-framing.md#en), [Noise Handshakes and Encrypted Transport](noise.md#en), then [Protocol Tree Node](protocol-tree-node.md#en): establish frame boundaries, authenticate and decrypt, then process the plaintext envelope and reconstruct nodes. When the business result declares [Argo](argo.md#en), continue with [Argo](argo.md#en) to parse its node payload.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `protocol-boundaries.md` | Boundaries between [registration](../registration/README.md#en), messaging connections, and call signaling |
| `connection-lifecycle.md` | [Connection establishment](noise.md#en), heartbeats, disconnection, and [reconnection](noise.md#en-fallback) |
| `handshake.md` | Handshakes, connection [authentication](noise.md#en), and state transitions |
| `binary-format/` | Compound message envelopes and other binary formats |
| `protobuf/` | Structured messages, schemas, and field versions |
| `encoding.md` | UTF-8, byte order, [URL encoding](../registration/request-encryption.md#en), [Base64](../registration/android-inputs.md#en), and [Base64url](../registration/android-inputs.md#en) |
| `identifiers-and-errors.md` | [Request IDs](../registration/sending-requests.md#en), sequence numbers, timestamps, [receipts](../messaging/storage/msgstore.md#receipts-en), and errors |

### Related Topics

[Account Registration and Verification](../registration/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Sending and Receiving Messages](../messaging/#en) · [Voice and Video Calls](../calls/#en)
