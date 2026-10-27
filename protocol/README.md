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
- Protobuf 等结构化数据的 schema、字段类型和版本变化。
- 请求标识、序列号、时间戳、回执和错误表示。
- UTF-8、字节序、URL 编码、Base64 与 Base64url 的具体使用位置。
- 原始字节、解码结果、运行时观测和协议推断之间的对应关系。

### 文档组织与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `protocol-boundaries.md` | 注册请求、消息连接与通话信令的范围 |
| `connection-lifecycle.md` | 建立连接、心跳、断开与重连 |
| `handshake.md` | 握手、连接认证及状态变化 |
| `binary-format/` | 二进制帧、节点、属性和消息封装 |
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
- Structured formats such as Protobuf: schemas, field types, and version changes.
- Request identifiers, sequence numbers, timestamps, acknowledgments, and errors.
- Specific uses of UTF-8, byte order, URL encoding, Base64, and Base64url.
- Mapping raw bytes to decoded data, runtime observations, and protocol hypotheses.

### Documentation Organization and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `protocol-boundaries.md` | Boundaries between registration, messaging connections, and call signaling |
| `connection-lifecycle.md` | Connection establishment, heartbeats, disconnection, and reconnection |
| `handshake.md` | Handshakes, connection authentication, and state transitions |
| `binary-format/` | Binary frames, nodes, attributes, and message envelopes |
| `protobuf/` | Structured messages, schemas, and field versions |
| `encoding.md` | UTF-8, byte order, URL encoding, Base64, and Base64url |
| `identifiers-and-errors.md` | Request IDs, sequence numbers, timestamps, receipts, and errors |

### Related Topics

[Account Registration and Verification](../registration/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Sending and Receiving Messages](../messaging/#en) · [Voice and Video Calls](../calls/#en)
