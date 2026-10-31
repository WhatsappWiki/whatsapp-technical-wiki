# End-to-End Encryption and Key Management · 端到端加密与密钥管理

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍消息加密与密钥管理相关概念，整理[身份密钥](../messaging/storage/axolotl.md#identities-zh)、[预密钥](../messaging/storage/axolotl.md#prekeys-zh)、会话、密钥轮换及验证方法，同时明确区分[传输保护](../protocol/noise.md#zh-layers)、[请求签名](../registration/request-encryption.md#zh)与消息端到端加密。

### 介绍范围

- 端到端加密（E2EE）与[传输层保护](../protocol/noise.md#zh-layers)的职责和边界。
- Signal Protocol 相关概念：[Identity Key](../messaging/storage/axolotl.md#identities-zh)、[PreKey](../messaging/storage/axolotl.md#prekeys-zh)、[Signed PreKey](../messaging/storage/axolotl.md#prekeys-zh) 和[会话状态](../messaging/storage/axolotl.md#sessions-zh)。
- [密钥生成、存储、选择](../messaging/storage/axolotl.md#zh)、轮换与失效流程。
- 一对一消息、群聊与关联设备场景的密钥分发研究。
- 消息加密、解密、签名与完整性验证的输入输出。
- 应用层[字段加密](../registration/security-fields/README.md#zh)、[请求签名](../registration/request-encryption.md#zh)和消息端到端加密的区别。
- 密钥记录的[二进制表示](../messaging/storage/axolotl.md#zh)、[地址维度](../messaging/storage/axolotl.md#sessions-zh)与[持久化行为](../messaging/storage/axolotl.md#zh)。

### 已有文章

| 文章 | 内容 |
| --- | --- |
| [axolotl.db：密钥、会话与队列](../messaging/storage/axolotl.md#zh) | [身份](../messaging/storage/axolotl.md#identities-zh)、[OPK](../messaging/storage/axolotl.md#prekeys-zh)、[signed/KEM 预密钥](../messaging/storage/axolotl.md#prekeys-zh)、[五维会话地址](../messaging/storage/axolotl.md#sessions-zh)与[群状态](../messaging/storage/axolotl.md#zh) |
| [本地存储与连接密钥](../messaging/local-storage.md#keystore-zh) | [Signal 身份密钥](../messaging/storage/axolotl.md#identities-zh)、[连接密钥密文](../messaging/local-storage.md#keystore-zh)及 [AndroidKeyStore AES 密钥](../messaging/local-storage.md#keystore-zh)的存储职责 |

### 文档组织与扩展方式

已有文章从上表进入；下方规划表仅列尚未创建的文章或子目录。各文章直接说明相应功能、字段和条件行为。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `encryption-boundaries.md` | [传输保护](../protocol/noise.md#zh-layers)、[字段加密](../registration/security-fields/README.md#zh)、[请求签名](../registration/request-encryption.md#zh)与 E2EE 的区别 |
| `identity-and-prekeys.md` | [身份密钥](../messaging/storage/axolotl.md#identities-zh)、[PreKey](../messaging/storage/axolotl.md#prekeys-zh) 与 [Signed PreKey](../messaging/storage/axolotl.md#prekeys-zh) |
| `sessions/` | 会话建立、密钥状态及状态持久化 |
| `key-lifecycle.md` | [密钥生成、存储、选择](../messaging/storage/axolotl.md#zh)、轮换与失效 |
| `group-and-device-keys.md` | 群聊及关联设备的密钥分发研究 |
| `verification.md` | 签名验证与记录一致性 |

<a id="相关主题--related-topics"></a>

### 相关主题

[登录认证与会话](../authentication/#zh) · [通信协议与数据格式](../protocol/#zh) · [消息发送与接收](../messaging/#zh) · [多设备与数据同步](../multi-device/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces message encryption and key-management concepts. It organizes [identity keys](../messaging/storage/axolotl.md#identities-en), [prekeys](../messaging/storage/axolotl.md#prekeys-en), sessions, key rotation, and verification methods, with clear distinctions between [transport protection](../protocol/noise.md#en-layers), [request signing](../registration/request-encryption.md#en), and message end-to-end encryption.

### Scope

- Responsibilities and boundaries of end-to-end encryption (E2EE) and [transport protection](../protocol/noise.md#en-layers).
- Signal Protocol concepts: [identity keys](../messaging/storage/axolotl.md#identities-en), [prekeys](../messaging/storage/axolotl.md#prekeys-en), [signed prekeys](../messaging/storage/axolotl.md#prekeys-en), and [session state](../messaging/storage/axolotl.md#sessions-en).
- [Key generation, storage, selection](../messaging/storage/axolotl.md#en), rotation, and expiration.
- Key distribution research for one-to-one conversations, groups, and companion devices.
- Inputs and outputs of message encryption, decryption, signing, and integrity verification.
- Differences between [field encryption](../registration/security-fields/README.md#en), [request signatures](../registration/request-encryption.md#en), and message E2EE.
- [Binary key-record representations](../messaging/storage/axolotl.md#en), [address dimensions](../messaging/storage/axolotl.md#sessions-en), and [persistence behavior](../messaging/storage/axolotl.md#en).

### Available Articles

| Article | Contents |
| --- | --- |
| [axolotl.db: Keys, Sessions, and Queues](../messaging/storage/axolotl.md#en) | [Identities](../messaging/storage/axolotl.md#identities-en), [OPKs](../messaging/storage/axolotl.md#prekeys-en), [signed/KEM prekeys](../messaging/storage/axolotl.md#prekeys-en), [five-dimensional sessions](../messaging/storage/axolotl.md#sessions-en), and [group state](../messaging/storage/axolotl.md#en) |
| [Local Storage and Connection Keys](../messaging/local-storage.md#keystore-en) | Storage responsibilities for [Signal identity keys](../messaging/storage/axolotl.md#identities-en), [connection-key ciphertext](../messaging/local-storage.md#keystore-en), and [AndroidKeyStore AES keys](../messaging/local-storage.md#keystore-en) |

### Documentation Organization and Expansion

Use the table above for available articles. The planning table below lists articles or subdirectories that do not yet exist. Each article explains its functions, fields, and conditional behavior directly.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `encryption-boundaries.md` | [Transport protection](../protocol/noise.md#en-layers), [field encryption](../registration/security-fields/README.md#en), [request signing](../registration/request-encryption.md#en), and E2EE |
| `identity-and-prekeys.md` | [Identity keys](../messaging/storage/axolotl.md#identities-en), [prekeys](../messaging/storage/axolotl.md#prekeys-en), and [signed prekeys](../messaging/storage/axolotl.md#prekeys-en) |
| `sessions/` | Session establishment, key state, and persistence |
| `key-lifecycle.md` | [Key generation, storage, selection](../messaging/storage/axolotl.md#en), rotation, and expiration |
| `group-and-device-keys.md` | Key-distribution research for groups and companion devices |
| `verification.md` | Signature verification and record consistency |

### Related Topics

[Login, Authentication, and Sessions](../authentication/#en) · [Communication Protocols and Data Formats](../protocol/#en) · [Sending and Receiving Messages](../messaging/#en) · [Multi-Device and Data Synchronization](../multi-device/#en)
