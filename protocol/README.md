# Communication Protocols and Data Formats · 通信协议与数据格式

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 请求与连接数据的组织方式，整理连接建立、握手、数据帧、序列化、编码、确认与错误表示，并区分注册、消息和通话的协议边界。

### 介绍范围

- 注册请求、长连接消息协议和通话信令的边界。
- 连接建立、握手、身份认证、心跳及断线重连。
- 二进制帧、节点与属性、消息封装和序列化格式研究。
- Argo 业务载荷、Header、Wire Schema、类型与精确回写。
- Protobuf 等结构化数据的 schema、字段类型和版本变化。
- 请求标识、序列号、时间戳、回执和错误表示。
- UTF-8、字节序、URL 编码、Base64 与 Base64url 的具体使用位置。
- 原始字节、解码结果、运行时观测和协议推断之间的对应关系。

### 文档组织与扩展方式

本页提供主题简介与文章导航；后续文章规划列出可继续展开的连接、格式与编码主题。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

### 文章导航

| 文章 | 内容 |
| --- | --- |
| [TCP 帧协议](tcp-framing.md#zh) | 三字节大端长度、连接前导、握手与传输帧、解密、压缩及节点解析顺序。 |
| [Noise 握手与加密传输](noise.md#zh) | XX、IK、同连接 fallback、ML-KEM-512 混合握手、帧结构、服务器认证与登录结果。 |
| [Protocol Tree Node · 二进制节点与编解码](protocol-tree-node.md#zh) | 节点模型、列表与 token 字典、长度头、字符串打包、JID、zlib、Python read/write 与逐字节回写。 |
| [Argo · 二进制格式与编解码](argo.md#zh) | 节点业务载荷、Header、标量块与 Core、schema 选择、类型与错误、Python 读写和精确回写。 |

理解完整接收链路时，可依次阅读 TCP 帧协议、Noise 握手与加密传输、Protocol Tree Node：先确定帧边界，再认证解密，最后处理明文封装并恢复节点。业务结果声明 Argo 时，再阅读 [Argo](argo.md#zh) 解析节点内的字节载荷。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `protocol-boundaries.md` | 注册请求、消息连接与通话信令的范围 |
| `connection-lifecycle.md` | 建立连接、心跳、断开与重连 |
| `handshake.md` | 握手、连接认证及状态变化 |
| `binary-format/` | 复合消息封装与其他二进制格式专题 |
| `protobuf/` | 结构化消息、schema 与字段版本 |
| `encoding.md` | UTF-8、字节序、URL 编码及 Base64 系列 |
| `identifiers-and-errors.md` | 请求标识、序列号、时间戳、回执与错误 |

<a id="相关主题--related-topics"></a>

### 相关主题

[账号注册与验证码](../registration/#zh) · [端到端加密与密钥管理](../encryption/#zh) · [消息发送与接收](../messaging/#zh) · [语音与视频通话](../calls/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces the organization of WhatsApp requests and connection data. It covers connection setup, handshakes, framing, serialization, encoding, acknowledgments, and errors while separating registration, messaging, and calling boundaries.

### Scope

- Boundaries between registration requests, persistent messaging connections, and call signaling.
- Connection establishment, handshakes, authentication, heartbeats, and reconnection.
- Binary frames, nodes and attributes, message envelopes, and serialization formats.
- Argo business payloads, headers, Wire Schemas, types and exact rewriting.
- Structured formats such as Protobuf: schemas, field types, and version changes.
- Request identifiers, sequence numbers, timestamps, acknowledgments, and errors.
- Specific uses of UTF-8, byte order, URL encoding, Base64, and Base64url.
- Mapping raw bytes to decoded data, runtime observations, and protocol hypotheses.

### Documentation Organization and Expansion

This page provides the topic overview and article navigation. Planned articles identify further connection, format, and encoding topics.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Articles

| Article | Scope |
| --- | --- |
| [TCP Framing](tcp-framing.md#en) | Three-byte big-endian lengths, connection prefixes, handshake and transport frames, decryption, compression, and node parsing. |
| [Noise Handshakes and Encrypted Transport](noise.md#en) | XX, IK, fallback on the same connection, ML-KEM-512 hybrid handshakes, framing, server authentication, and login outcomes. |
| [Protocol Tree Node and Binary Encoding](protocol-tree-node.md#en) | Node model, lists and token dictionaries, length headers, packed strings, JIDs, zlib, Python read/write, and exact rewriting. |
| [Argo · Binary Format and Encoding](argo.md#en) | Node business payloads, headers, scalar blocks and Core, schema selection, types/errors, Python read/write and exact rewriting. |

To follow the receiving pipeline, read TCP Framing, Noise Handshakes and Encrypted Transport, then Protocol Tree Node: establish frame boundaries, authenticate and decrypt, then process the plaintext envelope and reconstruct nodes. When the business result declares Argo, continue with [Argo](argo.md#en) to parse its node payload.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `protocol-boundaries.md` | Boundaries between registration, messaging connections, and call signaling |
| `connection-lifecycle.md` | Connection establishment, heartbeats, disconnection, and reconnection |
| `handshake.md` | Handshakes, connection authentication, and state transitions |
| `binary-format/` | Compound message envelopes and other binary formats |
| `protobuf/` | Structured messages, schemas, and field versions |
| `encoding.md` | UTF-8, byte order, URL encoding, Base64, and Base64url |
| `identifiers-and-errors.md` | Request IDs, sequence numbers, timestamps, receipts, and errors |

### Related Topics

[Account Registration and Verification](../registration/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Sending and Receiving Messages](../messaging/#en) · [Voice and Video Calls](../calls/#en)
