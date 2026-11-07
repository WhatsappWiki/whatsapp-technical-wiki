# Account Registration and Verification · 账号注册与验证码

[Wiki 首页](../README.md#zh) · [Wiki home](../README.md#en) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

WhatsApp 的 HTTP 注册流程通常先用 `exist` 检查号码，再用 `code` 申请验证，最后用 `register` 提交验证材料。发码阶段的 `sent` 表示发码请求的业务结果，注册是否完成由后续验证结果决定。

| 文章 | 内容 |
| --- | --- |
| [号码检查](exist.md#zh) | 号码、语言、设备和注册状态参数 |
| [申请验证](code.md#zh) | 验证路线、流程状态和发码参数 |
| [提交验证](register.md#zh) | 验证材料、提交与响应状态 |
| [Android 输入示例](android-inputs.md#zh) | 常见系统输入、应用状态与通用编码 |
| [请求封装](request-encryption.md#zh) | 表单序列化和加密正文 |
| [发送与响应](sending-requests.md#zh) | HTTP 发送、响应分层、等待与重试 |

<a id="en"></a>

## English

WhatsApp's HTTP registration flow typically checks the number with `exist`, requests verification with `code`, and submits verification material with `register`. A `sent` result belongs to the code-request stage. Registration completion depends on the subsequent verification response.

| Article | Scope |
| --- | --- |
| [Number check](exist.md#en) | Number, locale, device, and registration-state parameters |
| [Request verification](code.md#en) | Verification routes, flow state, and code-request parameters |
| [Submit verification](register.md#en) | Verification material, submission, and response state |
| [Android input examples](android-inputs.md#en) | Common system inputs, application state, and general encoding |
| [Request packaging](request-encryption.md#en) | Form serialization and encrypted bodies |
| [Sending and responses](sending-requests.md#en) | HTTP sending, response layers, waiting, and retries |
