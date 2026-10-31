# Login, Authentication, and Sessions · 登录认证与会话

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 账号与设备如何进入、维持和结束认证状态，整理[身份材料](../messaging/local-storage.md#zh)、[连接认证](../protocol/noise.md#zh)、关联设备登录、会话恢复及重新登录等主题。

### 介绍范围

- [首次注册](../registration/README.md#zh)、已注册账号启动、[连接认证](../protocol/noise.md#zh)与关联设备登录的流程区别。
- [身份材料](../messaging/local-storage.md#zh)、[设备标识](../registration/android-inputs.md#persistent-identifiers)、[认证密钥](../messaging/local-storage.md#keystore-zh)和[会话状态](../messaging/local-storage.md#zh)的读取与存储。
- 二维码或配对码关联流程、设备确认与授权状态研究。
- 会话恢复、[断线重连](../protocol/noise.md#zh-fallback)、失效、退出和重新登录。
- 两步验证、PIN 与 Passkey 等相关流程。
- 账号切换、设备移除和凭据生命周期。

### 已有文章

| 文章 | 内容 |
| --- | --- |
| [本地数据库与连接密钥](../messaging/local-storage.md#keystore-zh) | [keystore 偏好存储中的连接密钥密文](../messaging/local-storage.md#keystore-zh)与 [AndroidKeyStore 中的 AES 密钥](../messaging/local-storage.md#keystore-zh) |
| [Signal 身份与加密会话](../messaging/storage/axolotl.md#zh) | [identities](../messaging/storage/axolotl.md#identities-zh)、[本机注册 ID](../messaging/storage/axolotl.md#identities-zh) 与[按账户/设备/scope/type 定位的会话](../messaging/storage/axolotl.md#sessions-zh) |

### 文档组织与扩展方式

已有文章从上表进入；下方规划表仅列尚未创建的文章或子目录。各文章直接说明相应功能、字段和条件行为。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="后续文章规划--planned-articles"></a>

### 后续文章规划

| 规划路径 | 内容 |
| --- | --- |
| `authentication-flow.md` | [首次注册](../registration/README.md#zh)、账号启动与[连接认证](../protocol/noise.md#zh)的流程区别 |
| `credentials.md` | [身份材料](../messaging/local-storage.md#zh)、[认证密钥](../messaging/local-storage.md#keystore-zh)、[设备标识](../registration/android-inputs.md#persistent-identifiers)与存储 |
| `device-linking/` | 二维码与配对码关联、设备确认和授权 |
| `session-lifecycle.md` | [会话持久化](../messaging/storage/axolotl.md#sessions-zh)、恢复、失效、退出与[重连](../protocol/noise.md#zh-fallback) |
| `two-step-verification.md` | 两步验证、PIN 与相关状态 |
| `passkeys.md` | Passkey 相关流程与平台条件 |
| `account-switching.md` | 账户切换、设备移除与凭据失效 |

<a id="相关主题--related-topics"></a>

### 相关主题

[账号注册与验证码](../registration/#zh) · [端到端加密与密钥管理](../encryption/#zh) · [多设备与数据同步](../multi-device/#zh)

<a id="en"></a>

## English

### Overview

This directory introduces how WhatsApp accounts and devices enter, maintain, and leave authenticated states. Topics include [identity material](../messaging/local-storage.md#en), [connection authentication](../protocol/noise.md#en), companion-device login, session restoration, and reauthentication.

### Scope

- Differences between [initial registration](../registration/README.md#en), registered-account startup, [connection authentication](../protocol/noise.md#en), and companion-device login.
- Reading and storing [identity material](../messaging/local-storage.md#en), [device identifiers](../registration/android-inputs.md#en-persistent-identifiers), [authentication keys](../messaging/local-storage.md#keystore-en), and [session state](../messaging/local-storage.md#en).
- QR-code and pairing-code flows, device confirmation, and authorization state.
- Session restoration, [reconnection](../protocol/noise.md#en-fallback), expiration, logout, and reauthentication.
- Two-step verification, PINs, and passkey flows.
- Account switching, device removal, and credential lifecycles.

### Available Articles

| Article | Contents |
| --- | --- |
| [Local Databases and Connection Keys](../messaging/local-storage.md#keystore-en) | [Connection-key ciphertext](../messaging/local-storage.md#keystore-en) in [keystore preferences](../messaging/local-storage.md#keystore-en) and the [AES key in AndroidKeyStore](../messaging/local-storage.md#keystore-en) |
| [Signal Identities and Cryptographic Sessions](../messaging/storage/axolotl.md#en) | [identities](../messaging/storage/axolotl.md#identities-en), [local registration ID](../messaging/storage/axolotl.md#identities-en), and [account/device/scope/type session addressing](../messaging/storage/axolotl.md#sessions-en) |

### Documentation Organization and Expansion

Use the table above for available articles. The planning table below lists articles or subdirectories that do not yet exist. Each article explains its functions, fields, and conditional behavior directly.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

### Planned Articles

| Proposed path | Scope |
| --- | --- |
| `authentication-flow.md` | [Initial registration](../registration/README.md#en), account startup, and [connection-authentication](../protocol/noise.md#en) flows |
| `credentials.md` | [Identity material](../messaging/local-storage.md#en), [authentication keys](../messaging/local-storage.md#keystore-en), [device identifiers](../registration/android-inputs.md#en-persistent-identifiers), and storage |
| `device-linking/` | QR-code and pairing-code linking, device confirmation, and authorization |
| `session-lifecycle.md` | [Session persistence](../messaging/storage/axolotl.md#sessions-en), restoration, expiration, logout, and [reconnection](../protocol/noise.md#en-fallback) |
| `two-step-verification.md` | Two-step verification, PINs, and related states |
| `passkeys.md` | Passkey-related flows and platform conditions |
| `account-switching.md` | Account switching, device removal, and credential invalidation |

### Related Topics

[Account Registration and Verification](../registration/#en) · [End-to-End Encryption and Key Management](../encryption/#en) · [Multi-Device and Data Synchronization](../multi-device/#en)
