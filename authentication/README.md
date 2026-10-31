# Login, Authentication, and Sessions · 登录认证与会话

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 账号与设备如何进入、维持和结束认证状态，整理身份材料、连接认证、关联设备登录、会话恢复及重新登录等主题。

### 介绍范围

- 首次注册、已注册账号启动、连接认证与关联设备登录的流程区别。
- 身份材料、设备标识、认证密钥和会话状态的读取与存储。
- 二维码或配对码关联流程、设备确认与授权状态研究。
- 会话恢复、断线重连、失效、退出和重新登录。
- 两步验证、PIN 与 Passkey 等相关流程。
- 账号切换、设备移除和凭据生命周期。

### 已有文章

| 文章 | 内容 |
| --- | --- |
| [本地数据库与连接密钥](../messaging/local-storage.md#keystore-zh) | keystore 偏好存储中的连接密钥密文与 AndroidKeyStore 中的 AES 密钥 |
| [Signal 身份与加密会话](../messaging/storage/axolotl.md#zh) | identities、本机注册 ID 与按账户/设备/scope/type 定位的会话 |

### 文档组织与扩展方式

已有文章从上表进入；下方规划表仅列尚未创建的文章或子目录。各文章直接说明相应功能、字段和条件行为。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `authentication-flow.md` | 首次注册、账号启动与连接认证的流程区别 |
| `credentials.md` | 身份材料、认证密钥、设备标识与存储 |
| `device-linking/` | 二维码与配对码关联、设备确认和授权 |
| `session-lifecycle.md` | 会话持久化、恢复、失效、退出与重连 |
| `two-step-verification.md` | 两步验证、PIN 与相关状态 |
| `passkeys.md` | Passkey 相关流程与平台条件 |
| `account-switching.md` | 账户切换、设备移除与凭据失效 |

<a id="相关主题--related-topics"></a>

### 相关主题

[账号注册与验证码](../registration/#zh) · [端到端加密与密钥管理](../encryption/#zh) · [多设备与数据同步](../multi-device/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces how WhatsApp accounts and devices enter, maintain, and leave authenticated states. Topics include identity material, connection authentication, companion-device login, session restoration, and reauthentication.

### Scope

- Differences between initial registration, registered-account startup, connection authentication, and companion-device login.
- Reading and storing identity material, device identifiers, authentication keys, and session state.
- QR-code and pairing-code flows, device confirmation, and authorization state.
- Session restoration, reconnection, expiration, logout, and reauthentication.
- Two-step verification, PINs, and passkey flows.
- Account switching, device removal, and credential lifecycles.

### Available Articles

| Article | Contents |
| --- | --- |
| [Local Databases and Connection Keys](../messaging/local-storage.md#keystore-en) | Connection-key ciphertext in keystore preferences and the AES key in AndroidKeyStore |
| [Signal Identities and Cryptographic Sessions](../messaging/storage/axolotl.md#en) | identities, local registration ID, and account/device/scope/type session addressing |

### Documentation Organization and Expansion

Use the table above for available articles. The planning table below lists articles or subdirectories that do not yet exist. Each article explains its functions, fields, and conditional behavior directly.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `authentication-flow.md` | Initial registration, account startup, and connection-authentication flows |
| `credentials.md` | Identity material, authentication keys, device identifiers, and storage |
| `device-linking/` | QR-code and pairing-code linking, device confirmation, and authorization |
| `session-lifecycle.md` | Session persistence, restoration, expiration, logout, and reconnection |
| `two-step-verification.md` | Two-step verification, PINs, and related states |
| `passkeys.md` | Passkey-related flows and platform conditions |
| `account-switching.md` | Account switching, device removal, and credential invalidation |

### Related Topics

[Account Registration and Verification](../registration/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Multi-Device and Data Synchronization](../multi-device/#en)
