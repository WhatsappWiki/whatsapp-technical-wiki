# Login, Authentication, and Sessions · 登录认证与会话

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 账号与设备如何进入、维持和结束认证状态，整理身份材料、连接认证、关联设备登录、会话恢复及重新登录等主题。

### 介绍范围

- 首次注册、已注册账号启动、连接认证与关联设备登录的流程区别。
- 身份材料、设备标识、认证密钥和会话状态的读取与存储。
- 二维码或配对码关联流程、设备确认与授权状态研究。
- 会话恢复、断线重连、失效、退出和重新登录。
- 两步验证、PIN 与 Passkey 等相关流程的版本化记录。
- 账号切换、设备移除和凭据生命周期。

### 文档组织与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="en"></a>

## English

### Overview

This directory introduces how WhatsApp accounts and devices enter, maintain, and leave authenticated states. Topics include identity material, connection authentication, companion-device login, session restoration, and reauthentication.

### Scope

- Differences between initial registration, registered-account startup, connection authentication, and companion-device login.
- Reading and storing identity material, device identifiers, authentication keys, and session state.
- QR-code and pairing-code flows, device confirmation, and authorization state.
- Session restoration, reconnection, expiration, logout, and reauthentication.
- Version-specific research on two-step verification, PINs, and passkeys.
- Account switching, device removal, and credential lifecycles.

### Documentation Organization and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `authentication-flow.md` | 首次注册、账号启动与连接认证的流程区别 | Initial registration, account startup, and connection-authentication flows |
| `credentials.md` | 身份材料、认证密钥、设备标识与存储 | Identity material, authentication keys, device identifiers, and storage |
| `device-linking/` | 二维码与配对码关联、设备确认和授权 | QR-code and pairing-code linking, device confirmation, and authorization |
| `session-lifecycle.md` | 会话持久化、恢复、失效、退出与重连 | Session persistence, restoration, expiration, logout, and reconnection |
| `two-step-verification.md` | 两步验证、PIN 与相关状态 | Two-step verification, PINs, and related states |
| `passkeys.md` | Passkey 相关流程、平台与版本差异 | Passkey-related flows and platform/version differences |
| `account-switching.md` | 账户切换、设备移除与凭据失效 | Account switching, device removal, and credential invalidation |

## 相关主题 / Related Topics

[账号注册与验证码 / Account Registration and Verification](../registration/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/) · [多设备与数据同步 / Multi-Device and Data Synchronization](../multi-device/)
