# Account Registration and Verification · 账号注册与验证码

[Home / 首页](../README.md) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

本目录按 `exist → code → register` 介绍注册请求，再集中讲解安全字段、明文数据加密与请求发送。阅读时先理解每个请求传什么、字段有什么含义，再理解安全字段如何构造、参数如何封装以及请求如何发出。

### 介绍范围

1. **`exist` 明文参数**：号码检查请求的用途、参数表、条件字段和字段之间的关系。
2. **`code` 明文参数**：申请短信或语音验证码时的参数、发送方式选择和请求间需要延续的状态。
3. **`register` 明文参数**：提交验证码并完成注册时的参数，以及与前两个请求的关联。
4. **安全字段详解**：集中说明 `gpia`、`_gg`、`_gi`、`_gp`、`_gs`、`_ge`、`_ga` 的内容、输入、计算、编码和生命周期。
5. **明文数据加密与请求封装**：从内层参数到序列化数据、加密内容、签名和外层正文。
6. **请求发出与响应处理**：HTTP 方法与路径、请求头、正文写入、调用顺序、响应解析和重试条件。

### 明文参数与加密正文

这里的“明文参数”指进入外层加密之前的参数集合。安全字段的值本身仍可能包含编码或加密内容；在参数表里出现，不代表它的内部内容可以直接阅读。

三个请求分别列出各自的参数，区分必选、条件出现、空值和省略。安全字段先在参数表中说明用途，再由独立文章详细展开，避免在三个请求里重复算法。`token`、`aid` 等其他派生材料也应按各自职责说明，不能与七个安全字段或外层签名混为一类。

加密、签名与发送分层介绍：`ENC` 属于加密封装，`H` 属于请求签名，`Authorization` 等属于请求头。各请求的字段和封装方式应分别说明，不预设三者具有完全相同的正文与头部。

### 文档组织与扩展方式

下方主线按阅读顺序排列。已有文章提供可点击链接，尚未建立的文章使用路径文本列出。号码输入、设备标识、验证码交互等补充内容放在对应专题中，由主线文章按需引用。

单篇内容可直接放在本目录的英文命名 Markdown 文件中；需要多篇文章的专题可新建英文子目录，并用其 `README.md` 介绍专题范围、链接具体文章。文章完成后，将下表对应路径改为可点击链接，并更新所属目录的导航。

<a id="en"></a>

## English

### Overview

This directory introduces registration requests in the order `exist → code → register`, followed by dedicated explanations of security fields, plaintext encryption, and request transmission. Start with what each request contains, then learn how security fields are constructed, how parameters are packaged, and how requests are sent.

### Scope

1. **`exist` plaintext parameters**: purpose, parameter tables, conditional fields, and relationships for the number-check request.
2. **`code` plaintext parameters**: parameters for requesting an SMS or voice verification code, delivery-method selection, and state carried between requests.
3. **`register` plaintext parameters**: parameters for submitting the verification code and completing registration, including relationships to earlier requests.
4. **Security fields**: contents, inputs, computation, encoding, and lifecycles of `gpia`, `_gg`, `_gi`, `_gp`, `_gs`, `_ge`, and `_ga`.
5. **Plaintext encryption and request packaging**: inner parameters, serialization, encryption, signatures, and outer bodies.
6. **Transmission and response handling**: HTTP methods and paths, headers, body writes, call order, response parsing, and retry conditions.

### Plaintext Parameters and Encrypted Bodies

“Plaintext parameters” means the parameter collection before outer encryption. A security field can itself contain encoded or encrypted content; appearing in a plaintext parameter table does not make its internal content directly readable.

Document each request's parameters separately, distinguishing required fields, conditional presence, null values, and omission. Introduce security fields in the request tables and explain their algorithms in dedicated articles. Describe other derived material such as `token` and `aid` according to its own role, separately from the seven security fields and outer signatures.

Explain encryption, signing, and transmission as separate layers: `ENC` is an encrypted envelope, `H` is a request signature, and values such as `Authorization` belong to the headers. Describe each request's fields and packaging individually rather than assuming identical bodies and headers across all three.

### Documentation Organization and Expansion

The main sequence below follows the reading order. Available articles have working links; articles not yet created are listed as plain paths. Supporting topics such as number input, device identifiers, and verification interactions can be linked from the relevant main article.

Place a standalone article in an English-named Markdown file in this directory. For a topic requiring several articles, create an English-named subdirectory with its own `README.md` describing the scope and linking the articles. Replace the planned path with a working link and update the parent topic navigation when the article is available.

## 主线文章 / Main Articles

| Order | Document / Path | 中文内容 | English scope |
| --- | --- | --- | --- |
| 1 | [exist.md](exist.md) | `exist` 请求的明文参数、取值来源与完整示意 | Plaintext parameters, input sources, and the full example for the `exist` request |
| 2 | `code.md` | `code` 请求的明文参数 | Plaintext parameters of the `code` request |
| 3 | `register.md` | `register` 请求的明文参数 | Plaintext parameters of the `register` request |
| 4 | `security-fields/` | 七个安全字段分别详细解释 | Dedicated explanations of the seven security fields |
| 5 | `request-encryption.md` | 参数序列化、明文加密、签名与正文封装 | Parameter serialization, encryption, signing, and body packaging |
| 6 | `sending-requests.md` | 请求头、发送顺序、HTTP 发出与响应处理 | Headers, request order, HTTP transmission, and response handling |

### 每篇参数文章的结构 / Structure of Each Parameter Article

每个请求依次介绍：请求用途与触发条件、明文参数总览、逐字段含义与类型、出现条件与空值处理、参数间关联，以及使用同格式随机值的明文示例。安全字段链接到独立专题，完整加密正文和发送代码放到后续文章。

For each request, cover its purpose and trigger conditions, plaintext parameter overview, field meanings and types, presence and null handling, parameter relationships, and a plaintext example with random values in the same representation. Link security fields to their dedicated topic; put complete encrypted bodies and transmission code in the later articles.

### 安全字段专题 / Security Field Articles

`security-fields/` 按 `gpia.md`、`gg.md`、`gi.md`、`gp.md`、`gs.md`、`ge.md`、`ga.md` 拆分。每篇解释字段职责、输入、计算步骤、明文或内部结构、编码与加密、缓存与刷新条件，以及在三个请求中的使用差异。

Organize `security-fields/` into `gpia.md`, `gg.md`, `gi.md`, `gp.md`, `gs.md`, `ge.md`, and `ga.md`. Each article explains the field's role, inputs, computation, plaintext or internal structure, encoding and encryption, caching and refresh conditions, and usage differences between the three requests.

## 补充文章 / Supporting Articles

| Document / Path | 中文内容 | English scope |
| --- | --- | --- |
| [android-inputs.md](android-inputs.md) | Android 公开 API、应用状态与字段编码示例 | Public Android APIs, application state, and field-encoding examples |
| `phone-number.md` | 国家码、号码处理、输入纠错与设备号码提示 | Calling codes, number processing, typo correction, and device-number hints |
| `identifiers.md` | 设备标识、持久状态与生命周期 | Device identifiers, persistent state, and lifecycles |
| `verification-code.md` | 验证码交互、等待与重试 | Verification interactions, waiting, and retries |

## 相关主题 / Related Topics

[登录认证与会话 / Login, Authentication, and Sessions](../authentication/) · [通信协议与数据格式 / Communication Protocols and Data Formats](../protocol/) · [端到端加密与密钥管理 / End-to-End Encryption and Key Management](../encryption/)
