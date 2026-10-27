# WhatsApp Wiki · WhatsApp 技术知识库

<a id="contact"></a>

## Telegram Contact: [@ttjjda](https://t.me/ttjjda)

**WhatsApp 注册、登录、通信协议、消息收发与语音通话技术文档。**

**Technical documentation on WhatsApp registration, authentication, protocols, messaging, and voice calls.**

[中文](#zh) · [English](#en) · [技术目录 / Technical Index](#technical-index) · [参与贡献 / Contributing](#contributing) · [Contact](#contact)

<a id="zh"></a>

## 中文

### 项目介绍

WhatsApp Wiki 是一个面向开发者、协议研究者和移动端工程师的非官方技术知识库，围绕 WhatsApp 账号注册、登录认证、通信协议、消息发送与接收、媒体传输、语音及视频通话、多设备同步等主题组织文档。

项目希望把分散的分析笔记整理为可检索、可追溯、可复现的技术资料：既解释业务流程，也记录字段来源、编码方式、状态变化、实现思路与验证边界。内容以技术文档、流程说明、字段表、研究记录和参考代码为主要形式。

当前仓库首先发布这份项目介绍与技术目录，后续将逐步整理注册与请求字段分析等文章。下方目录是知识库的内容规划；各主题的完成情况以实际文章及其验证记录为准。列入目录不表示相关功能已经实现、协议已经完整恢复，或所有平台及版本均已验证。

### 适合谁阅读

- 希望理解 WhatsApp 注册、验证码和账号会话流程的开发者。
- 研究即时通信、端到端加密、消息可靠性和多设备架构的工程师。
- 分析 Android、iOS、Web 或 Desktop 客户端的移动端与协议研究者。
- 希望提交文档、修正文义、补充实验或提供中英文翻译的贡献者。

<a id="technical-index"></a>

### 技术目录

| 章节 | 内容范围 | English keywords |
| --- | --- | --- |
| [01 · 账号注册与验证码](#zh-registration) | 号码检查、验证码申请与提交、注册状态及请求字段 | Registration, Verification, OTP |
| [02 · 登录认证与会话](#zh-authentication) | 身份材料、会话持久化、设备关联与重新登录 | Login, Authentication, Session, Device Linking |
| [03 · 通信协议与数据格式](#zh-protocol) | 连接建立、握手、帧结构、编码和请求封装 | Protocol, Handshake, Serialization, Protobuf |
| [04 · 加密与密钥管理](#zh-encryption) | 身份密钥、会话密钥、预密钥与加密边界 | E2EE, Signal Protocol, Key Management |
| [05 · 消息发送与接收](#zh-messaging) | 消息构建、接收、回执、重试、去重与群聊 | Send / Receive Messages, ACK, Receipts |
| [06 · 图片、文件与语音消息](#zh-media) | 媒体上传下载、加解密、文件元数据与语音消息 | Media, Upload, Download, Voice Messages |
| [07 · 语音与视频通话](#zh-calls) | 呼入呼出、通话信令、媒体传输与网络质量 | Voice Calls, Video Calls, VoIP, Signaling |
| [08 · 多设备与数据同步](#zh-multi-device) | 设备关联、消息分发、历史同步与离线恢复 | Multi-Device, Companion Devices, Sync |
| [09 · 客户端分析与验证](#zh-analysis) | 静态分析、运行时观测、字段追踪与参考实现 | Reverse Engineering, JNI, Native, Validation |
| [10 · 错误排查与术语](#zh-troubleshooting) | 错误分类、诊断流程、术语与常见问题 | Troubleshooting, Error Codes, FAQ |

<a id="zh-registration"></a>

### 01 · 账号注册与验证码

- 注册流程：国家码与号码输入、号码检查、验证码申请、验证码提交及注册结果。
- 号码规范化、输入纠错、设备号码提示与注册 UI 状态。
- `/exist` 等已观察请求的用途、参数来源、条件字段和版本差异。
- SMS 与语音验证码的申请路径、等待时间、重试和错误处理研究。
- 设备标识、持久状态、客户端配置与安全信号的生命周期。
- 请求正文、表单编码、加密封装、签名和证书链的职责。
- 成功与失败样本的对照方法，以及客户端结果与服务端决策之间的证据边界。

<a id="zh-authentication"></a>

### 02 · 登录认证与会话

- 首次注册、已注册账号启动、连接认证与关联设备登录的流程区别。
- 身份材料、设备标识、认证密钥和会话状态的读取与存储。
- 二维码或配对码关联流程、设备确认与授权状态研究。
- 会话恢复、断线重连、失效、退出和重新登录。
- 两步验证、PIN 与 Passkey 等相关流程的版本化记录。
- 账号切换、设备移除和凭据生命周期。

<a id="zh-protocol"></a>

### 03 · 通信协议与数据格式

- 注册请求、长连接消息协议和通话信令的边界。
- 连接建立、握手、身份认证、心跳及断线重连。
- 二进制帧、节点与属性、消息封装和序列化格式研究。
- Protobuf 等结构化数据的 schema、字段类型和版本变化。
- 请求标识、序列号、时间戳、回执和错误表示。
- UTF-8、字节序、URL 编码、Base64 与 Base64url 的具体使用位置。
- 原始字节、解码结果、运行时观测和协议推断之间的对应关系。

<a id="zh-encryption"></a>

### 04 · 端到端加密与密钥管理

- 端到端加密（E2EE）与传输层保护的职责和边界。
- Signal Protocol 相关概念：Identity Key、PreKey、Signed PreKey 和会话状态。
- 密钥生成、存储、选择、轮换与失效流程。
- 一对一消息、群聊与关联设备场景的密钥分发研究。
- 消息加密、解密、签名与完整性验证的输入输出。
- 应用层字段加密、请求签名和消息端到端加密的区别。
- 原始输入复算、字节比较和参考实现的验证范围。

<a id="zh-messaging"></a>

### 05 · 消息发送与接收

- 发送路径：创建消息、生成标识、序列化、加密、发送和状态更新。
- 接收路径：收到数据、解析、解密、存储、通知与回执。
- 文本、引用、回复、转发、表情回应及消息编辑或撤回的结构研究。
- 服务端确认、送达回执、已读回执与它们各自代表的状态。
- 超时、重试、重复消息、顺序处理和离线消息恢复。
- 群聊消息的发送、接收、成员变化与相关事件。
- 本地数据库、发送队列和 UI 显示之间的一致性。

<a id="zh-media"></a>

### 06 · 图片、文件与语音消息

- 图片、视频、音频、文档、贴纸与语音消息的数据结构。
- 媒体预处理、缩略图、类型识别和文件元数据。
- 上传、下载、媒体引用与缓存生命周期。
- 媒体密钥、加解密、摘要和完整性校验研究。
- 语音消息录制、编码、时长、波形和播放状态。
- 下载失败、链接失效、缓存缺失和重新获取。

语音消息是可发送和下载的音频消息；实时语音通话在下一章单独整理。

<a id="zh-calls"></a>

### 07 · 语音与视频通话

- 呼出与呼入流程：发起、来电、响铃、接受、拒绝、取消及挂断。
- 通话标识、参与者、设备选择和信令状态机。
- 呼叫建立、协商、超时、重连与通话结束原因。
- 媒体链路、直连或中继、NAT 穿透及候选地址研究。
- 音频采集、编码、传输、接收、解码与播放。
- 视频采集、编解码、分辨率变化和音视频同步。
- 静音、扬声器、蓝牙、音频路由与设备切换。
- 丢包、延迟、抖动、弱网与通话质量指标。
- 群组通话、多设备来电和通话记录。
- 与 RTP / RTCP、SRTP、ICE / STUN / TURN、WebRTC 等技术的关系核验。

上述标准与技术名称是待核验的研究主题；具体使用情况以目标客户端、版本和实际证据为准。

<a id="zh-multi-device"></a>

### 08 · 多设备与数据同步

- 主设备、关联设备和设备列表的关系。
- 设备关联、身份确认、会话建立与解除关联。
- 消息在多个设备之间的分发、确认和状态同步。
- 历史消息、联系人、群组和应用状态同步。
- 离线队列、冲突、重复事件与恢复流程。
- Web / Desktop 与移动端之间的行为差异。

<a id="zh-analysis"></a>

### 09 · 客户端分析与验证

- APK / DEX、资源文件、Native 库和符号的静态分析。
- 运行时观测、调用链追踪、JNI 边界与网络发送边界定位。
- 字段从系统 API、偏好、数据库及业务状态到请求正文的来源追踪。
- Python、Android Java 与 Native 参考实现。
- 原始输入、期望输出和生成结果分离的验证方法。
- 字节级比较、负向对照、合成测试与真实样本验证的区别。
- 客户端版本、构建哈希、实验时间、平台与配置记录。

<a id="zh-troubleshooting"></a>

### 10 · 错误排查、术语与 FAQ

- 注册失败、验证码未到达、认证失效与连接中断。
- 消息发送失败、接收延迟、解密错误和媒体下载失败。
- 来电未响铃、呼叫无法接通、单向音频和通话中断。
- 错误码、异常、客户端回调与实际业务结果之间的关系。
- 平台、版本、网络和配置差异导致的行为变化。
- 缩写、字段名、术语索引和常见问题。

### 阅读方式与文档约定

建议按“注册与登录 → 通信协议与加密 → 消息与媒体 → 通话与多设备”的顺序阅读，也可以通过目录直接定位主题。

每篇技术文章应尽量注明：目标平台与版本、问题背景、流程或调用链、字段及编码、证据来源、验证方法、结果和仍未确认的部分。静态分析、运行时观测、离线复算和服务端实际接受应分别说明。

示例统一使用合成或脱敏数据。公开文档不包含真实验证码、访问令牌、私钥、完整认证材料或可识别个人身份的原始记录。

<a id="en"></a>

## English

### About

WhatsApp Wiki is an unofficial technical knowledge base for developers, protocol researchers, and mobile engineers. It organizes research on WhatsApp account registration, login and authentication, communication protocols, sending and receiving messages, media transfer, voice and video calls, and multi-device synchronization.

The project aims to turn scattered research notes into searchable, traceable, and reproducible documentation. Articles explain workflows, field origins, encodings, state transitions, implementation approaches, and verification limits through technical notes, diagrams, field references, experiments, and reference code.

This repository starts with this project overview and technical index. Articles on registration, request-field analysis, and other topics will be added incrementally. The index below defines the planned documentation scope. Actual coverage is determined by the published articles and their verification records. A listed topic does not imply an implemented feature, a fully recovered protocol, or validation across every platform and version.

### Intended Audience

- Developers studying WhatsApp registration, verification codes, authentication, and account sessions.
- Engineers researching instant messaging, end-to-end encryption, delivery reliability, and multi-device systems.
- Researchers analyzing Android, iOS, Web, and Desktop clients.
- Contributors improving documentation, experiments, reference implementations, or translations.

### Technical Index

| Chapter | Scope | Navigation |
| --- | --- | --- |
| 01 · Registration and Verification | Phone-number checks, OTP requests, verification, and request fields | [Read scope](#en-registration) |
| 02 · Login and Authentication | Credentials, sessions, device linking, logout, and recovery | [Read scope](#en-authentication) |
| 03 · Protocols and Data Formats | Connections, handshakes, framing, serialization, and encoding | [Read scope](#en-protocol) |
| 04 · Encryption and Keys | E2EE, Signal Protocol concepts, key storage, and rotation | [Read scope](#en-encryption) |
| 05 · Sending and Receiving Messages | Message pipelines, acknowledgments, receipts, and retries | [Read scope](#en-messaging) |
| 06 · Media and Voice Messages | Uploads, downloads, media encryption, and audio messages | [Read scope](#en-media) |
| 07 · Voice and Video Calls | Incoming / outgoing calls, signaling, media, and call quality | [Read scope](#en-calls) |
| 08 · Multi-Device and Synchronization | Companion devices, message distribution, and history sync | [Read scope](#en-multi-device) |
| 09 · Client Analysis and Validation | Reverse engineering, runtime observation, and reference code | [Read scope](#en-analysis) |
| 10 · Troubleshooting and Glossary | Diagnostics, errors, terminology, and FAQs | [Read scope](#en-troubleshooting) |

<a id="en-registration"></a>

### 01 · Account Registration and Verification

- Country codes, phone-number input, number checks, code requests, verification, and registration outcomes.
- Number normalization, typo correction, device-number suggestions, and registration UI state.
- Observed requests such as `/exist`: purpose, input sources, conditional fields, and version differences.
- SMS and voice verification paths, waiting periods, retries, and error handling.
- Device identifiers, persistent state, client configuration, and security-signal lifecycles.
- Request bodies, form encoding, encrypted envelopes, signatures, and certificate chains.
- Comparing successful and failed samples while separating client observations from server-side decision rules.

<a id="en-authentication"></a>

### 02 · Login, Authentication, and Sessions

- Differences between initial registration, registered-account startup, connection authentication, and companion-device login.
- Reading and storing identity material, device identifiers, authentication keys, and session state.
- QR-code and pairing-code flows, device confirmation, and authorization state.
- Session restoration, reconnection, expiration, logout, and reauthentication.
- Version-specific research on two-step verification, PINs, and passkeys.
- Account switching, device removal, and credential lifecycles.

<a id="en-protocol"></a>

### 03 · Communication Protocols and Data Formats

- Boundaries between registration requests, persistent messaging connections, and call signaling.
- Connection establishment, handshakes, authentication, heartbeats, and reconnection.
- Binary frames, nodes and attributes, message envelopes, and serialization formats.
- Structured formats such as Protobuf: schemas, field types, and version changes.
- Request identifiers, sequence numbers, timestamps, acknowledgments, and errors.
- Specific uses of UTF-8, byte order, URL encoding, Base64, and Base64url.
- Mapping raw bytes to decoded data, runtime observations, and protocol hypotheses.

<a id="en-encryption"></a>

### 04 · End-to-End Encryption and Key Management

- Responsibilities and boundaries of end-to-end encryption (E2EE) and transport protection.
- Signal Protocol concepts: identity keys, prekeys, signed prekeys, and session state.
- Key generation, storage, selection, rotation, and expiration.
- Key distribution research for one-to-one conversations, groups, and companion devices.
- Inputs and outputs of message encryption, decryption, signing, and integrity verification.
- Differences between field encryption, request signatures, and message E2EE.
- Verification limits of raw-input recomputation, byte comparisons, and reference implementations.

<a id="en-messaging"></a>

### 05 · Sending and Receiving Messages

- Outbound pipeline: creation, identifiers, serialization, encryption, transmission, and state updates.
- Inbound pipeline: reception, parsing, decryption, storage, notifications, and receipts.
- Research on text messages, quotes, replies, forwarding, reactions, edits, and deletion events.
- Server acknowledgments, delivery receipts, read receipts, and their distinct meanings.
- Timeouts, retries, duplicate handling, ordering, and offline-message recovery.
- Group messaging, membership changes, and related events.
- Consistency between local databases, outbound queues, and UI state.

<a id="en-media"></a>

### 06 · Media Transfer and Voice Messages

- Data structures for images, videos, audio, documents, stickers, and voice messages.
- Media preparation, thumbnails, type detection, and file metadata.
- Uploads, downloads, media references, and cache lifecycles.
- Media keys, encryption, decryption, hashes, and integrity verification.
- Voice-message recording, encoding, duration, waveforms, and playback state.
- Download failures, expired references, missing cached files, and retrieval.

Voice messages are audio messages that can be sent and downloaded. Real-time voice calls are covered separately below.

<a id="en-calls"></a>

### 07 · Voice and Video Calls

- Outgoing and incoming flows: initiation, incoming notification, ringing, acceptance, rejection, cancellation, and hangup.
- Call identifiers, participants, device selection, and signaling state machines.
- Call setup, negotiation, timeouts, reconnection, and termination reasons.
- Media paths, direct or relayed connections, NAT traversal, and address candidates.
- Audio capture, encoding, transmission, reception, decoding, and playback.
- Video capture, codecs, resolution changes, and audio/video synchronization.
- Mute, speakerphone, Bluetooth, audio routing, and device switching.
- Packet loss, latency, jitter, weak networks, and call-quality metrics.
- Group calls, multi-device ringing, and call history.
- Investigation of relationships to RTP / RTCP, SRTP, ICE / STUN / TURN, and WebRTC.

These standards and technology names identify research topics to verify. Their actual use must be established for the target client and version through evidence.

<a id="en-multi-device"></a>

### 08 · Multi-Device and Data Synchronization

- Relationships between primary devices, companion devices, and device lists.
- Device linking, identity confirmation, session establishment, and unlinking.
- Message distribution, acknowledgments, and state synchronization across devices.
- History, contacts, groups, and application-state synchronization.
- Offline queues, conflicts, duplicate events, and recovery.
- Behavioral differences between mobile, Web, and Desktop clients.

<a id="en-analysis"></a>

### 09 · Client Analysis and Validation

- Static analysis of APK / DEX files, resources, Native libraries, and symbols.
- Runtime observation, call-chain tracing, JNI boundaries, and application send boundaries.
- Tracking field origins from system APIs, preferences, databases, and business state into request bodies.
- Python, Android Java, and Native reference implementations.
- Verification with separated raw inputs, expected outputs, and generated results.
- Distinctions between byte comparisons, negative controls, synthetic tests, and live-sample validation.
- Recording client versions, build hashes, experiment times, platforms, and configurations.

<a id="en-troubleshooting"></a>

### 10 · Troubleshooting, Glossary, and FAQ

- Registration failures, missing verification codes, expired authentication, and disconnections.
- Failed sends, delayed reception, decryption errors, and media download failures.
- Missing incoming-call notifications, setup failures, one-way audio, and dropped calls.
- Relationships between error codes, exceptions, client callbacks, and business outcomes.
- Behavior changes caused by platform, version, network, and configuration differences.
- Acronyms, field names, terminology, and frequently asked questions.

### Reading Guide and Documentation Conventions

A suggested reading order is registration and authentication → protocols and encryption → messaging and media → calls and multi-device systems. The index also provides direct access to individual topics.

Technical articles should identify the target platform and version, problem, workflow or call chain, fields and encodings, evidence, verification method, results, and unresolved questions. Static analysis, runtime observation, offline recomputation, and actual server acceptance should be described separately.

Examples use synthetic or redacted data. Public documentation excludes real verification codes, access tokens, private keys, complete authentication material, and identifiable personal records.

<a id="contributing"></a>

## 参与贡献 / Contributing

欢迎通过 Issue 提出问题、报告文档错误或建议研究主题，通过 Pull Request 补充文章、实验、参考代码和翻译。提交时请注明目标平台及版本、复现步骤、证据和验证范围；推测或未验证结论请明确标记。

Use Issues for questions, documentation corrections, and topic suggestions. Pull requests are welcome for articles, experiments, reference code, and translations. Include the target platform and version, reproduction steps, evidence, and verification scope. Clearly label hypotheses and unverified conclusions.

中英文内容应保持含义一致；专有字段名、类名、接口名和协议标识保留原文。添加新章节时，请同步更新两种语言的目录。

Keep Chinese and English content aligned. Preserve original field names, class names, interface names, and protocol identifiers. Update both indexes when adding chapters.

## 项目声明 / Project Notice

本项目为独立、非官方技术文档项目，与 WhatsApp 或 Meta 不存在官方隶属、合作或背书关系。WhatsApp 及相关名称和标识归各自权利人所有。研究结论适用于文中明确记录的版本与条件。

This is an independent, unofficial technical documentation project. It is not affiliated with, sponsored by, or endorsed by WhatsApp or Meta. Product names and trademarks belong to their respective owners. Research findings apply to the versions and conditions explicitly documented.

## 搜索主题 / Search Topics

WhatsApp 技术文档、WhatsApp 注册、验证码、登录认证、通信协议、消息发送、消息接收、语音消息、语音通话、视频通话、端到端加密、多设备同步、Android 逆向、协议分析。

WhatsApp wiki, WhatsApp registration, phone verification, OTP, login, authentication, session management, WhatsApp protocol, message sending and receiving, messaging, delivery receipts, voice messages, voice calls, video calls, VoIP, end-to-end encryption, E2EE, Signal Protocol, multi-device synchronization, Android reverse engineering, protocol analysis.
