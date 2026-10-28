# Account Registration and Verification · 账号注册与验证码

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 功能简介

注册业务按 `exist → code → register` 衔接：exist检查号码，code申请所选验证路线，后续验证请求提交材料。code返回sent不等于已收码或完成注册。本目录围绕exist/code参数、安全字段、Android输入、请求封装与发送提供13篇文章。

### 介绍范围

1. **exist参数**：号码检查的字段目录、取值来源、条件、省略与材料选择。
2. **code参数**：短信、语音或其它验证路线的字段、方式快照与延续状态。
3. **Android输入**：系统和包信息、存储示例、证书选择、编码及其与实际缓存的衔接。
4. **安全字段**：gpia、_gg、_gi、_gp、_gs、_ge、_ga的结构、算法、输入与生命周期。
5. **加密封装**：序列化、ENC、H、证书链及不同通道/失败分支。
6. **发送与响应**：最终合并、应用头、HTTP/解析/业务分层和重试条件。

### 明文参数与加密正文

“明文参数”指外层ENC之前的字段集合；部分字段内部仍有编码或加密。exist的71项、code的70项都是已知字段目录，实际按流程选择，条件字段不要求每次出现。JSON和Java合成示意用于展示字段表示，最终HTTP正文还要经过通道合并、序列化和封装。

字段级CBC、外层ENC、签名H与Authorization分别使用自己的输入和生命周期。普通文本与预编码字节也走不同处理路径；Kotlin与WAMsys的头部、降级和重试规则按各自分支说明。

<a id="文档组织与扩展方式"></a>

### 阅读顺序

先读对应请求的字段表，再结合Android输入与安全字段专题理解来源、编码和缓存，最后阅读封装与发送。exist按号码选择的备份材料可在发送前保存；code随后直接解码偏好。两种请求的计数、纠错字段和状态快照也分别处理。

<a id="主线文章--main-articles"></a>

### 主线文章

| 顺序 | 文章 | 内容 |
| --- | --- | --- |
| 1 | [exist 参数](exist.md#zh) | 71项已知内层字段、条件省略、号码备份选择与缓存，附JSON/Java合成示意。 |
| 2 | [code 参数](code.md#zh) | 70项已知内层字段、验证路线、已保存备份与交互状态，附JSON/Java合成示意。 |
| 3 | [安全字段总览](security-fields/README.md#zh) | 七字段的职责与表示，gpia回调、对象结果槽、context字典及库级状态的生命周期。 |
| 4 | [参数序列化与加密封装](request-encryption.md#zh) | 普通文本/预编码字节、Kotlin/原生序列化、ENC、H、证书链及失败分支。 |
| 5 | [发送流程与响应处理](sending-requests.md#zh) | 最终合并、Kotlin/WAMsys通道、应用头、响应分层、等待与重试分支。 |

<a id="每篇参数文章的结构--structure-of-each-parameter-article"></a>

### 每篇参数文章的结构

参数文章按主题列出含义、取值来源、表示与出现条件、示例值，随后介绍组装、缓存和通道差异，并提供中英文一致的JSON/Java合成展示对象。七个安全字段各有同语言详解入口；最终加密与响应行为由封装和发送文章展开。

<a id="安全字段专题--security-field-articles"></a>

### 安全字段专题

gpia/_gg承载Play结果，_gi/_gp描述应用材料，_gs/_ge/_ga描述环境或执行状态。gpia独立回调与其它结果槽分别发布；字段存在、加密完成、Google结果及注册业务成功是不同事件。

| 文章 | 内容 |
| --- | --- |
| [gpia · Play结果与应用信息](security-fields/gpia.md#zh) | Play结果码、requestHash、provider准备与限流、超时范围和CBC包装。 |
| [_gg · Play结果精简包装](security-fields/gg.md#zh) | `_ic/_it`内部结构、独立IV与发布，以及公共Map保留旧槽的条件。 |
| [_gi · 应用完整性信息](security-fields/gi.md#zh) | 主安装文件、DEX、签名与目录摘要，CBC包装及源槽/context/结果缓存。 |
| [_gp · 声明权限摘要](security-fields/gp.md#zh) | 声明权限名称的modified UTF-8、排序与摘要，空值/失败材料及缓存。 |
| [_gs · 系统信号与加密包络](security-fields/gs.md#zh) | 13次路径检查、19整数CSV、prepared状态、独立随机源与X25519/GCM包络。 |
| [_ge · 模块与路径检查](security-fields/ge.md#zh) | 模块与条件su路径、布尔JSON、配置转换及缓存命中行为。 |
| [_ga · 路径年龄与运行环境](security-fields/ga.md#zh) | 路径年龄、Monkey与boot输入，仅bi单独CBC，公共材料快照及整份结果缓存。 |

<a id="补充文章--supporting-articles"></a>

### 补充文章

| 文章 | 内容 |
| --- | --- |
| [Android 字段获取代码](android-inputs.md#zh) | 四组Java辅助类：系统输入、示例存储、包与证书读取、字节编码；区分原始API与字段生命周期。 |

<a id="相关主题--related-topics"></a>

### 相关主题

[登录认证与会话](../authentication/README.md#zh) · [通信协议与数据格式](../protocol/README.md#zh) · [加密与密钥管理](../encryption/README.md#zh)

<a id="en"></a>

## English

### Overview

The business flow is `exist → code → register`: exist checks a number, code requests the selected verification route, and later verification submits material. A code sent result does not establish receipt or completed registration. This directory contains 13 articles on exist/code parameters, security fields, Android inputs, packaging, and sending.

### Scope

1. **exist parameters**: number-check field catalog, sources, conditions, omission, and material selection.
2. **code parameters**: SMS, voice, or other verification routes, captured method, and retained state.
3. **Android inputs**: system/package information, example storage, certificate selection, encoding, and actual cache boundaries.
4. **Security fields**: structure, algorithms, inputs, and lifetimes of gpia, _gg, _gi, _gp, _gs, _ge, and _ga.
5. **Packaging**: serialization, ENC, H, certificate chains, and transport/failure branches.
6. **Sending and responses**: final merging, application headers, HTTP/parsing/business layers, and retry conditions.

### Plaintext Parameters and Encrypted Bodies

“Plaintext parameters” means the fields before outer ENC; individual fields can still contain encoding or encryption. The 71 exist fields and 70 code fields are known catalogs. Select fields for the actual flow rather than requiring every conditional field on each request. Synthetic JSON/Java objects illustrate representations; the final HTTP body also requires transport merging, serialization, and packaging.

Field CBC, outer ENC, signature H, and Authorization have separate inputs and lifetimes. Ordinary text and preencoded bytes also follow different handling. Kotlin and WAMsys headers, fallbacks, and retry rules follow their respective branches.

<a id="documentation-organization-and-expansion"></a>

### Reading Order

Start with the relevant request’s field table, use Android inputs and security topics for sources/encoding/caches, then read packaging and sending. exist can save number-selected backup material before sending; code later directly decodes the preference. Their counters, correction fields, and captured state remain request-specific.

### Main Articles

| Order | Article | Scope |
| --- | --- | --- |
| 1 | [exist parameters](exist.md#en) | 71 known inner fields, conditional omission, number-bound backup selection, and caches, with synthetic JSON/Java illustrations. |
| 2 | [code parameters](code.md#en) | 70 known inner fields, verification routes, retained backup and interaction state, with synthetic JSON/Java illustrations. |
| 3 | [Security-field overview](security-fields/README.md#en) | Roles and representations of seven fields; lifetimes of the gpia callback, owner slots, context dictionaries, and library state. |
| 4 | [Serialization and request encryption](request-encryption.md#en) | Ordinary text/preencoded bytes, Kotlin/native serialization, ENC, H, certificate chains, and failure branches. |
| 5 | [Sending and response handling](sending-requests.md#en) | Final merging, Kotlin/WAMsys paths, application headers, response layers, waits, and retry branches. |

### Structure of Each Parameter Article

Parameter articles group meanings, sources, representation/presence conditions, and examples, followed by assembly, cache, and transport differences. Both languages share synthetic JSON/Java display objects. Each security field links to a same-language explanation; packaging and sending articles explain final encryption and response behavior.

### Security Field Articles

gpia/_gg carry Play outcomes, _gi/_gp describe application material, and _gs/_ge/_ga describe environment or execution state. The separate gpia callback and other result slots publish independently. Field presence, completed encryption, Google outcomes, and registration-business success are different events.

| Article | Scope |
| --- | --- |
| [gpia · Play outcome and application metadata](security-fields/gpia.md#en) | Play outcome codes, requestHash, provider preparation/rate limits, timeout scope, and CBC wrapping. |
| [_gg · Compact Play outcome](security-fields/gg.md#en) | Internal `_ic/_it` structure, independent IV/publication, and retained older slots in the public Map. |
| [_gi · Application integrity](security-fields/gi.md#en) | Primary installation file, DEX, certificate and directory digests; CBC and source-slot/context/result caches. |
| [_gp · Declared-permission digest](security-fields/gp.md#en) | Modified UTF-8, sorting, and hashing of declared permission names; empty/failure material and caching. |
| [_gs · System signals and encrypted envelope](security-fields/gs.md#en) | 13 path checks, a 19-integer CSV, prepared state, separate randomness, and an X25519/GCM envelope. |
| [_ge · Module and path checks](security-fields/ge.md#en) | Module and conditional su-path checks, boolean JSON, configuration conversion, and cache-hit behavior. |
| [_ga · Path ages and runtime environment](security-fields/ga.md#en) | Path ages, Monkey and boot inputs; CBC only for bi, a public-material snapshot, and whole-result caching. |

### Supporting Articles

| Article | Scope |
| --- | --- |
| [Android input examples](android-inputs.md#en) | Four Java helpers for system inputs, example storage, package/certificate reads, and byte encoding; distinguish raw APIs from field lifetimes. |

### Related Topics

[Login, Authentication, and Sessions](../authentication/README.md#en) · [Communication Protocols and Data Formats](../protocol/README.md#en) · [Encryption and Key Management](../encryption/README.md#en)
