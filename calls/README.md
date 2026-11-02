# 语音通话能力 · Voice Calling Capabilities

[首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

已实现 WhatsApp 语音通话相关能力，涵盖发起、接听、双向语音、群组通话及结束处理。除了完成通话交互，也处理了持续语音、中继转发、参与者变化和通话后的资源衔接，可用于功能演示与集成评估。

### 已实现的能力

| 能力 | 功能表现 |
| --- | --- |
| 一对一语音 | 独立客户端与 Android 客户端之间可以相互发起、接听通话 |
| 双向语音 | 通话双方可以持续发送和接收语音 |
| 群组语音 | 支持三方同场语音，以及成员离开后的继续通话 |
| 中继通话 | 支持通过中继传递加密的语音数据 |
| 通话结束 | 支持拒接、取消、挂断及连接中断后的结束处理 |
| 消息衔接 | 语音通话结束后可以继续使用消息功能 |

### 从几个侧面理解语音能力

实时通话的工程问题集中在跨帧状态、时钟关系、队列边界、成员一致性和异步生命周期。呼叫交互只是其中一个侧面；持续声音需要这些约束同时成立。

| 侧面 | 值得关注的细节 | 对应专题 |
| --- | --- | --- |
| 编解码状态 | 状态跨帧保留，不同声音来源需要各自的解码历史 | [独立媒体状态](voice-calls.md#zh-duplex) |
| 实时队列 | 乱序处理受播放时限约束，增加缓冲会带来额外延迟 | [抖动缓冲](voice-calls.md#zh-playout) |
| 中继成本 | 网络分发与音频计算分别承担开销 | [转发与音频处理](voice-calls.md#zh-relay) |
| 多人混音 | 同一播放区间、独立来源和幅度边界需要共同满足 | [混音约束](group-calls.md#zh-voices) |
| 通话收尾 | 重复结束、在途回调和旧事件必须保持一致的资源边界 | [生命周期](voice-calls.md#zh-ending) |
| 安全边界 | 设备身份、通话材料、成员变化和服务器职责有各自的边界 | [通话安全架构](security-whitepaper.md#zh) |

### 技术问题与约束

| 主题 | 技术关注点 |
| --- | --- |
| UDP | 数据报边界、乱序、重复、分片风险与播放截止时间，见[传输约束](voice-calls.md#zh-udp) |
| PCM | 样本表示、声道布局、片段粒度及缓冲区所有权，见[声音表示](voice-calls.md#zh-pcm) |
| RTP | 包序列、媒体时间、到达时间与播放时钟，见[时钟关系](voice-calls.md#zh-rtp) |
| Opus、编码与解码 | 可变码率、跨帧状态及压缩大小与时长的区别，见[编解码约束](voice-calls.md#zh-codec) |

专题分别讨论约束、失败现象和资源取舍。设备采样、编码帧和网络包有不同粒度；包已经到达，也仍可能错过声音的有效播放时间。

### 可以观察到的结果

- 独立客户端和 Android 客户端可以交换主被叫角色，展示两个方向的呼叫与接听。
- 接通后双方持续发送和接收语音，能够分别观察两个方向的声音表现。
- 三名参与者可以同场交流；一名成员退出后，剩余两名成员继续交谈。
- 正常挂断后，原有消息功能可以继续使用。

这些结果覆盖呼叫交互、媒体持续性和成员协作几个侧面。单一的“已连接”状态不足以替代对整场交流的观察。

### 场景与范围

当前能力展示面向本地受控环境中的客户端互通，包含一对一双向语音和三方同场通话。演示内容关注实际的接听、声音收发、成员变化与结束行为；音质感受和部署环境分别评估。

### 进一步了解

- [一对一语音通话](voice-calls.md#zh)：编解码状态、PCM 粒度、RTP 时钟、UDP、实时队列和生命周期。
- [群组语音通话](group-calls.md#zh)：成员一致性、独立解码、混音幅度、有界队列和中继分发。
- [语音通话安全架构](security-whitepaper.md#zh)：结合官方白皮书，分析设备身份、通话材料生命周期、群组成员变化和 Cloud API 的端点边界。

需要功能演示、接入评估或业务适配，可通过 [Telegram: @ttjjda](https://t.me/ttjjda) 联系，说明目标平台和使用场景。

<a id="en"></a>

## English

WhatsApp voice-calling capabilities have been implemented, covering initiation, answering, two-way audio, group calls and call ending. The work also handles continuous audio, relay transport, participant changes and resources after a call, supporting demonstrations and integration evaluation.

### Implemented capabilities

| Capability | Behavior |
| --- | --- |
| One-to-one voice | An independent client and an Android client can initiate and answer calls in either direction |
| Two-way audio | Both participants can continuously send and receive speech |
| Group voice | Three participants can share a call, with conversation continuing after a member leaves |
| Relayed calling | A relay can carry encrypted voice data |
| Call ending | Rejection, cancellation, hangup and ending after a connection interruption are supported |
| Messaging continuity | Messaging remains available after the voice call ends |

### Several perspectives on voice calling

Real-time calling involves state across frames, clock relationships, queue bounds, membership consistency and asynchronous lifetimes. Calling interaction is one part; continuous audio requires those constraints to hold together.

| Perspective | Detail to consider | Topic |
| --- | --- | --- |
| Codec state | State persists across frames, and sources need independent decoding histories | [Independent media state](voice-calls.md#en-duplex) |
| Real-time queues | Reordering is bounded by playout deadlines, while added buffering increases latency | [Jitter buffering](voice-calls.md#en-playout) |
| Relay cost | Network distribution and audio computation have separate costs | [Forwarding and audio processing](voice-calls.md#en-relay) |
| Group mixing | Matching time intervals, independent sources and amplitude bounds must hold together | [Mixing constraints](group-calls.md#en-voices) |
| Call ending | Repeated ending, in-flight callbacks and stale events need consistent resource boundaries | [Lifecycle](voice-calls.md#en-ending) |
| Security boundaries | Device identity, call material, membership changes and server roles have distinct scopes | [Call security architecture](security-whitepaper.md#en) |

### Technical questions and constraints

| Topic | Engineering focus |
| --- | --- |
| UDP | Datagram boundaries, reordering, duplication, fragmentation risk and playout deadlines; see [transport constraints](voice-calls.md#en-udp) |
| PCM | Sample representation, channel layout, granularity and buffer ownership; see [sound representation](voice-calls.md#en-pcm) |
| RTP | Packet sequence, media time, arrival time and playout clocks; see [clock relationships](voice-calls.md#en-rtp) |
| Opus, encoding and decoding | Variable bitrate, state across frames and the distinction between compressed size and duration; see [coding constraints](voice-calls.md#en-codec) |

Topics address constraints, failure symptoms and resource tradeoffs separately. Device samples, codec frames and packets have different granularities; even an arrived packet can have missed its useful playout interval.

### Observable results

- The independent client and Android client can exchange caller and recipient roles to demonstrate calling and answering in both directions.
- Both sides continuously send and receive speech, allowing each direction's audio behavior to be observed separately.
- Three participants can share a conversation; after one leaves, the other two continue talking.
- Messaging remains available after a normal hangup.

These outcomes cover call interaction, media continuity and participant coordination. A single connected state does not replace observation of the whole conversation.

### Scenarios and scope

The capabilities are demonstrated through client interoperability in a controlled local environment, covering two-way one-to-one audio and three-party calls. Demonstrations focus on answering, audio exchange, membership changes and ending behavior; listening quality and deployment environments are evaluated separately.

### Learn more

- [One-to-one voice calls](voice-calls.md#en): codec state, PCM granularity, RTP clocks, UDP, real-time queues and lifecycle.
- [Group voice calls](group-calls.md#en): membership consistency, independent decoding, mix amplitudes, bounded queues and relay distribution.
- [Voice call security architecture](security-whitepaper.md#en): an analysis of the official white paper covering device identity, call-material lifetimes, membership changes and Cloud API endpoint boundaries.

For a demonstration, integration evaluation or workflow adaptation, contact [Telegram: @ttjjda](https://t.me/ttjjda) with your target platform and intended use case.
