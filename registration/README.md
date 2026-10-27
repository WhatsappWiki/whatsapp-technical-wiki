# Account Registration and Verification · 账号注册与验证码

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录介绍 WhatsApp 从输入手机号码到完成账号注册的流程，重点整理号码检查、验证码申请与提交、注册请求字段及状态变化，作为注册相关详细文章的入口。

### 介绍范围

- 注册流程：国家码与号码输入、号码检查、验证码申请、验证码提交及注册结果。
- 号码规范化、输入纠错、设备号码提示与注册 UI 状态。
- `/exist` 等已观察请求的用途、参数来源、条件字段和版本差异。
- SMS 与语音验证码的申请路径、等待时间、重试和错误处理研究。
- 设备标识、持久状态、客户端配置与安全信号的生命周期。
- 请求正文、表单编码、加密封装、签名和证书链的职责。
- 成功与失败样本的对照方法，以及客户端结果与服务端决策之间的证据边界。

### 文档状态与扩展方式

本页是本主题的功能简介与导航入口。下方路径为后续文章规划，目前尚未创建；具体实现与结论以实际文章和验证记录为准。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接并标明状态。

<a id="en"></a>

## English

### Overview

This directory covers the path from phone-number input to account registration. It organizes number checks, verification-code requests and submission, registration request fields, and state transitions as an entry point to detailed registration articles.

### Scope

- Country codes, phone-number input, number checks, code requests, verification, and registration outcomes.
- Number normalization, typo correction, device-number suggestions, and registration UI state.
- Observed requests such as `/exist`: purpose, input sources, conditional fields, and version differences.
- SMS and voice verification paths, waiting periods, retries, and error handling.
- Device identifiers, persistent state, client configuration, and security-signal lifecycles.
- Request bodies, form encoding, encrypted envelopes, signatures, and certificate chains.
- Comparing successful and failed samples while separating client observations from server-side decision rules.

### Documentation Status and Expansion

This page provides the topic overview and navigation entry point. The paths below are planned articles or subdirectories and have not been created yet. Implementations and findings are defined by the actual articles and their verification records.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and record its status when the article is available.

## 后续文章规划 / Planned Articles

| Proposed path | 中文内容 | English scope |
| --- | --- | --- |
| `phone-number.md` | 国家码、号码规范化、输入纠错与设备号码提示 | Country codes, number normalization, typo correction, and device-number suggestions |
| `exist/` | 号码检查请求、参数目录、来源与响应分析 | Number-check requests, field inventories, input sources, and responses |
| `verification-code.md` | 短信与语音验证码申请、提交及等待状态 | SMS and voice verification requests, code submission, and waiting states |
| `registration-flow.md` | 注册状态机、业务回调与结果核对 | Registration state machine, business callbacks, and outcome checks |
| `identifiers.md` | 设备标识、持久状态和生命周期 | Device identifiers, persistent state, and lifecycles |
| `security-signals/` | 客户端配置、安全字段与原始输入验证 | Client configuration, security fields, and raw-input validation |
| `request-envelope.md` | 表单编码、加密封装、签名与证书链 | Form encoding, encrypted envelopes, signatures, and certificate chains |
| `errors-and-comparisons.md` | 错误分类、重试与成功失败样本对照 | Errors, retries, and comparisons of successful and failed samples |

## 相关主题 / Related Topics

[登录认证与会话 / Login, Authentication, and Sessions](../authentication/) · [通信协议与数据格式 / Communication Protocols and Data Formats](../protocol/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/)
