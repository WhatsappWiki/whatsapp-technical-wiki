# 语音通话安全架构 · Voice Call Security Architecture

[通话目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

语音通话的安全边界涉及设备身份、媒体保密、成员变化和服务端协调。本文结合 WhatsApp 官方的 [《WhatsApp Encryption Overview》技术白皮书](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf)，分析这些机制的关系。各节将原文说明与技术含义分别表述。

<a id="zh-boundaries"></a>

### 身份、媒体与传输是三个不同的边界

白皮书分别介绍设备间加密会话、通话保护，以及客户端与聊天服务器之间的传输保护。设备间会话基于 Signal 协议，聊天服务器连接另有传输加密。参见 [Initiating Session Setup](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=13) 与 [Transport Security](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=35)。

理解这几个边界，可以把三个常被混淆的问题分开：

| 问题 | 关注的边界 | 技术含义 |
| --- | --- | --- |
| 对方是哪一个设备？ | 设备身份与设备授权 | 同一账户下的设备仍有各自的身份，账户名称不能替代设备验证 |
| 谁能恢复通话内容？ | 实际通信端点与通话秘密 | 数据经过某个服务器，不代表该服务器具备解密能力 |
| 网络观察者能看到什么？ | 客户端与服务器之间的连接 | 连接加密需要结合其终止位置理解，不能单独证明内容只对通话双方可见 |

技术上，身份验证、内容保密和网络可达性需要分别成立。连接成功只回答了其中一部分问题，不能替代设备身份与媒体保护状态的判断。

<a id="zh-one-to-one"></a>

### 一对一通话：设备会话与通话材料有不同生命周期

Call Setup 说明，接收方不同设备对应各自的通话秘密；实际接听设备使用对应材料保护通话。通话秘密保存在客户端内存中，仅用于通话，WhatsApp 服务器无法取得。参见 [Call Setup](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=21)。

这意味着长期的设备信任关系与一次通话的媒体状态需要分别理解：

| 对象 | 主要作用 | 生命周期分析 |
| --- | --- | --- |
| 设备身份 | 识别通信设备及其授权关系 | 不能因为一场语音挂断就被当作整套账户身份已经失效 |
| 设备间加密会话 | 保护面向具体设备的通信 | 已有会话与新呼叫之间存在复用关系，不能假定每次呼叫都重新注册设备 |
| 通话媒体上下文 | 保护当前通话中的媒体 | 应与本场通话、实际接听设备及相关资源保持一致 |

从工程角度看，“挂断后消息仍可使用”与这种生命周期划分相容；但仅凭这个功能表现，无法证明所有密钥处理都符合官方机制。同样，停止内存中的通话上下文，也不等于能够撤回对方已经听到或保存的声音。

<a id="zh-membership"></a>

### 群组通话：成员变化也会改变保密范围

Group Calling 明确说明，成员加入或离开都会触发通话密钥更新。更新期间允许旧的接收加密上下文短暂保留，以衔接仍在到达的旧媒体。参见 [Group Calling，第 21–22 页](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=21)。

技术含义在于：群组成员列表、媒体来源和当前保密范围是相关但不同的状态。界面上移除一个成员，只完成了成员展示的变化；后续媒体还需要符合新的参与范围。

新旧上下文的短暂交接体现了实时系统的取舍。成员变化通知、媒体到达和终端处理不可能天然同时完成；立即抛弃全部旧状态可能破坏声音连续性，无限保留旧状态又会模糊通话边界。交接需要有明确范围，不能简单等同于“继续使用旧密钥”。

密钥更新控制的是更新后媒体的可解密范围，不能消除成员在此前已取得的明文。这也说明，成员退出后的继续通话，既涉及声音连续性，也涉及后续内容的保密关系。

<a id="zh-devices"></a>

### 多设备：账户设备数与通话参与人数分别计算

白皮书对群组通话规定：每名活跃参与者恰有一个活跃设备。一个账户关联多个设备，不代表这些设备同时作为多个成员参加同一场群组通话。参见 [Group Calling](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=22)。

因此，分析通话时需要区分三个集合：账户已关联的设备、能够收到来电的设备，以及已经实际参与当前群组通话的活跃设备。把它们合并，会造成参与人数、声音来源和资源归属的判断错误。

设备验证解决的则是另一件事。白皮书介绍通过二维码或数字安全码核对设备身份；这种核对有助于确认通信对象，但不检查编解码器是否正确、播放是否连续或网络时延是否合适。参见 [Verifying Keys](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=33)。

<a id="zh-server"></a>

### 服务器：维护通话状态与读取声音是不同的能力

Group Calling 说明，服务器保存当前群组的参与者、发起者及部分媒体元数据，并协调成员变化和密钥更新；在这些端到端加密通话中，服务器无法取得通话秘密或实际音视频内容。参见 [Group Calling](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=22)。

由此可以理解两种同时存在的职责：服务器处理通话控制和参与关系，终端持有恢复媒体内容所需的秘密。端到端加密允许服务器承担协调工作；它也不意味着参与者列表等运行状态对服务器完全不可见。

中继与内容保密同样可以并存。Meta 的[官方通话安全说明](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/)介绍，“保护通话 IP 地址”通过服务器中继隐藏双方的直接网络地址，同时保持个人通话的端到端加密。

这份补充说明支持“服务器能够转发自己无法解读的媒体”这一架构判断。它没有给出完整的中继报文和部署方案，因此不能据此认定任意 UDP 转发服务都与 WhatsApp 官方基础设施相同。

<a id="zh-cloud"></a>

### Cloud API：安全定义取决于实际端点由谁控制

白皮书区分普通客户端端点与 Cloud API：即使仍使用 Signal 协议保护通信，涉及 Cloud API 的通话也不被 WhatsApp 视为端到端加密，因为通信不再直接送达业务方自己控制的端点。参见 [Defining End-to-End Encryption 与 Implementation with Cloud API](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=36)。

这里的判断条件是实际端点。单独看到“Business”名称，或看到网络上使用了加密，都不足以确定内容访问范围。普通 WhatsApp Business App 与使用 Meta 托管 Cloud API 的通信，需要按实际接入情况理解。

白皮书还区分“Cloud API 已关联到账户”与“Cloud API 实际参与某类通信”：关联 Cloud API 不会让它自动取得个人群组通话的媒体秘密。涉及 Cloud API 的一对一通信有单独的安全说明和用户提示。参见 [Implementation with Cloud API](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=37)。

技术上，加密算法与端点信任是两个维度。采用相同协议，可以形成不同的内容访问边界；判断时需要知道密文最终在哪里被还原，以及该端点由谁控制。

<a id="zh-reading"></a>

### 白皮书能解释什么，哪些问题还需分别分析

白皮书的核心贡献是说明通话的信任与保密关系。它的聊天服务器传输章节不能直接当作 UDP 媒体通路的完整定义；音频附件的加密介绍也不能直接替代实时语音的媒体机制。

| 技术问题 | 阅读结论 |
| --- | --- |
| 为什么同一账户仍需区分设备？ | 设备身份、接听端点和活跃参与关系各有含义 |
| 为什么群组成员变化会影响媒体安全？ | 新的参与范围需要对应新的保密范围，并处理交接 |
| 为什么服务器可以协调但无法听到声音？ | 控制状态的可见性与内容解密权限分别设计 |
| 能否确定 PCM、Opus、RTP、UDP 的完整运行方式？ | 通话章节未完整定义采样、编解码配置、媒体封装与传输调度 |
| 能否据此证明某个实现与官方机制完全一致？ | 还需逐项核对实际行为，架构描述本身不能替代实现验证 |

声音表示、媒体时钟、缓冲和混音的工程约束，可继续阅读[一对一语音](voice-calls.md#zh)与[群组语音](group-calls.md#zh)。这些问题与安全边界相互关联，也需要各自满足实时性要求。

[能力总览](README.md#zh) · [一对一语音](voice-calls.md#zh) · [群组语音](group-calls.md#zh)

<a id="en"></a>

## English

Voice-call security involves device identity, media confidentiality, membership changes and server coordination. This article analyzes their relationships using WhatsApp's official [WhatsApp Encryption Overview technical white paper](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf). Each section distinguishes the paper's statements from their engineering implications.

<a id="en-boundaries"></a>

### Identity, media and transport have different boundaries

The paper separately describes encrypted device-to-device sessions, call protection and transport protection between clients and chat servers. Device sessions use the Signal protocol; chat-server connections have additional transport encryption. See [Initiating Session Setup](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=13) and [Transport Security](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=35).

These boundaries separate three questions that are often confused:

| Question | Relevant boundary | Engineering implication |
| --- | --- | --- |
| Which device is communicating? | Device identity and authorization | Devices belonging to one account retain individual identities; an account name does not replace device verification |
| Who can recover call content? | Actual communication endpoints and call secrets | Passing through a server does not give that server decryption capability |
| What can a network observer see? | The client-to-server connection | Connection encryption must be understood together with its termination point; it alone does not establish confidentiality between call participants |

Identity verification, content confidentiality and network reachability each need to hold. A successful connection answers only some of those questions and does not replace assessment of device identity or media protection.

<a id="en-one-to-one"></a>

### One-to-one calls: device sessions and call material have different lifetimes

Call Setup describes separate call secrets for the recipient's devices; the answering device uses the corresponding material. Call secrets remain in client memory for the call, and WhatsApp servers cannot access them. See [Call Setup](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=21).

Longer-lived device trust and the media state of an individual call therefore need separate treatment:

| Object | Main role | Lifetime analysis |
| --- | --- | --- |
| Device identity | Identify a device and its authorization | Ending one call does not invalidate the entire account identity |
| Encrypted device session | Protect communication addressed to a particular device | Existing sessions can serve new calls; device registration need not be repeated for every call |
| Call media context | Protect media within the current call | Its scope should agree with the call, answering device and associated resources |

Messaging remaining available after hangup is compatible with these distinct lifetimes. That observable behavior alone, however, does not prove that all key handling matches the official mechanism. Disposing of an in-memory call context also cannot retract sound that another participant has already heard or saved.

<a id="en-membership"></a>

### Group calls: membership changes also change confidentiality scope

Group Calling states that joining or leaving triggers a call-key update. During the transition, the previous receiving cryptographic context can remain briefly available to handle older media still arriving. See [Group Calling, pages 21–22](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=21).

Membership, media sources and the current confidentiality scope are related but distinct states. Removing an avatar changes the displayed membership; subsequent media must also agree with the new participation scope.

The brief handover between contexts illustrates a real-time tradeoff. Membership notifications, media delivery and endpoint processing do not naturally complete simultaneously. Discarding all old state immediately can disrupt continuity, while retaining it indefinitely blurs call boundaries. The transition needs a defined scope and cannot simply be treated as continued use of an old key.

A key update controls the decryption scope of subsequent media. It cannot remove plaintext that a member obtained earlier. Continuing a conversation after a departure therefore concerns both audio continuity and confidentiality of later content.

<a id="en-devices"></a>

### Multiple devices: account devices and call participants are counted separately

For group calls, the paper specifies exactly one active device per active participant. Linking several devices to an account does not make them simultaneous additional members of the same group call. See [Group Calling](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=22).

Call analysis should therefore distinguish linked account devices, devices that can receive an incoming call, and active devices actually participating in the current group call. Combining these sets leads to incorrect participant counts, audio-source assignments and resource ownership.

Device verification addresses a different question. The paper describes checking identities with QR codes or numeric security codes. Such checks help confirm the communication peer; they do not assess codec correctness, audio continuity or network delay. See [Verifying Keys](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=33).

<a id="en-server"></a>

### Servers: managing call state and accessing speech are different capabilities

Group Calling describes server-held participation, initiator and selected media metadata, together with coordination of membership and key updates. For these end-to-end encrypted calls, servers cannot access call secrets or actual audio and video content. See [Group Calling](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=22).

These roles coexist: servers coordinate calls and participation, while endpoints retain the secrets needed to recover media content. End-to-end encryption permits server coordination; it does not imply that operational state such as membership is completely hidden from the server.

Relaying can also coexist with content confidentiality. Meta's [official calling-security article](https://engineering.fb.com/2023/11/08/security/whatsapp-calls-enhancing-security/) explains that Protect IP Address in Calls uses server relays to conceal participants' direct network addresses while preserving end-to-end encryption for personal calls.

That supplementary explanation supports the architectural conclusion that a server can forward media it cannot interpret. It does not provide a complete relay packet specification or deployment design, so it cannot establish that an arbitrary UDP forwarding service matches WhatsApp's infrastructure.

<a id="en-cloud"></a>

### Cloud API: the security definition depends on control of the actual endpoint

The paper distinguishes ordinary client endpoints from Cloud API. Even though communication still uses the Signal protocol, WhatsApp does not consider calls involving Cloud API end-to-end encrypted because delivery is no longer directly to an endpoint controlled by the business. See [Defining End-to-End Encryption and Implementation with Cloud API](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=36).

The condition concerns the actual endpoint. A Business label or visible use of network encryption alone does not establish who can access content. Communication with the ordinary WhatsApp Business App and communication using Meta-hosted Cloud API must be assessed according to their actual integration.

The paper also distinguishes Cloud API being linked to an account from its participation in a particular type of communication. Linking Cloud API does not automatically give it personal group-call media secrets. One-to-one communication involving Cloud API has separate security explanations and user notices. See [Implementation with Cloud API](https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf#page=37).

Encryption algorithms and endpoint trust are separate dimensions. The same protocol can produce different content-access boundaries; assessment requires knowing where ciphertext is ultimately recovered and who controls that endpoint.

<a id="en-reading"></a>

### What the paper explains and what needs separate analysis

The paper primarily explains trust and confidentiality relationships. Its chat-server transport section is not a complete definition of a UDP media path, and its audio-attachment encryption section does not substitute for real-time voice-media behavior.

| Technical question | Reading conclusion |
| --- | --- |
| Why distinguish devices belonging to one account? | Device identity, answering endpoints and active participation have different meanings |
| Why do membership changes affect media security? | Changed participation requires a corresponding confidentiality scope and a managed transition |
| How can a server coordinate calls without hearing speech? | Visibility of control state and permission to decrypt content are designed separately |
| Does the paper fully define PCM, Opus, RTP and UDP operation? | The call sections do not fully specify sampling, codec configuration, media packetization or transmission scheduling |
| Does the architecture prove that an implementation matches WhatsApp? | Actual behavior still needs individual assessment; architecture descriptions do not replace implementation verification |

For engineering constraints involving sound representation, media clocks, buffering and mixing, continue with [one-to-one voice](voice-calls.md#en) and [group voice](group-calls.md#en). These concerns interact with security boundaries while retaining their own real-time requirements.

[Capability overview](README.md#en) · [One-to-one voice](voice-calls.md#en) · [Group voice](group-calls.md#en)
