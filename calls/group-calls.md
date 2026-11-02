# 群组语音通话 · Group Voice Calls

[通话目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

已实现三方同场语音通话。多个参与者可以加入同一场通话、进行双向语音交流，并在成员离开后继续交谈。

### 已实现的交互

| 能力 | 功能表现 |
| --- | --- |
| 发起群组通话 | 创建同一场多人语音交流 |
| 接听与加入 | 受邀成员接听后加入通话 |
| 同场语音 | 多个参与者能够发送并接收其他成员的语音 |
| 成员离开 | 一名成员离开后，剩余成员继续通话 |
| 正常结束 | 参与者可以离开或结束当前通话 |

<a id="zh-membership"></a>

### 成员集合与媒体对象的一致性

群组成员有各自的参与状态，整场通话也有自己的生命周期。邀请人数、已经接听的人数和当前可播放的声音来源数量，可以暂时不同。

成员退出时，本地可能仍有属于该成员的排队音频和在途任务。成员集合与媒体对象需要保持一致：旧声音不能因为晚到就重新进入当前混音，已经结束的成员也不能被迟到事件恢复。

当前三方场景支持一名成员退出后，剩余两名成员继续通话。这里的连续性依赖成员状态与媒体生命周期的共同收拢，不是简单减少界面上的一个头像。

<a id="zh-voices"></a>

### 多人混音：独立解码、时间对齐与幅度边界

每个远端声音来源保留独立的解码状态和待播放内容。压缩音频包含跨帧状态，把不同成员的压缩字节交替送进同一份解码状态，会混淆各自的历史。

混音面向具有一致样本含义的 PCM。采样率、声道布局和播放时间需要可比较；仅仅把多个网络包的内容拼接起来，不能形成正常的多人声音。

| 混音约束 | 技术含义 |
| --- | --- |
| 时间对齐 | 同一播放区间的声音才参与相应输出，迟到内容不塞进后续区间 |
| 幅度边界 | 多路样本相加可能超出 PCM 数值范围，需要处理削波和溢出 |
| 独立来源 | 一个成员的解码状态或暂缺片段不应污染其他成员的声音 |
| 有界队列 | 单个慢来源或慢消费者不能让整场声音无限积压 |

当前终端混音对累加后的样本执行幅度限制。限制可以避免数值溢出，但发生削波仍可能产生失真，因此“输出没有溢出”和“听感没有变化”不能等同。

RTP 来源信息用于区分媒体流，成员身份则属于通话参与关系。一个人的媒体对象变化，与新增一名参与者不是同一件事。

<a id="zh-relay"></a>

### 中继分发与终端混音的分工

群组中继在当前参与范围内分发加密语音，终端负责自己的声音处理。转发侧无需把各成员的声音全部解码后再合成，因此网络分发成本与音频计算成本分别存在。

参与人数增加，会扩大需要管理的连接和声音来源，但不意味着本地同一段声音必须按接收人数重复编码。媒体处理与多目标分发需要分别理解，不能仅由人数推算编码器数量或计算开销。

通话隔离同样是运行约束：不同群组的数据不能串入同一场交流；成员离开后的旧转发与旧媒体活动，需要与当前参与范围保持一致。

<a id="zh-observation"></a>

### 判断多人语音是否真正持续

多人语音的持续性需要结合成员、媒体和输出三个侧面判断。单一的“群组已连接”状态，不能描述哪一名成员有声音，或输出中的声音来自正常解码还是补偿。

| 观察面 | 需要区分的结果 |
| --- | --- |
| 成员 | 已邀请、已接听、仍参与和已经离开 |
| 声音来源 | 当前来源、过期来源与暂时没有可用片段的来源 |
| 解码与混音 | 可用 PCM、时间区间对应关系和混音幅度限制 |
| 输出队列 | 正常消费、积压丢弃与缺口补偿 |
| 生命周期 | 单个成员离开与整场通话结束 |

三方描述的是同时参与交流的人数。窗口数量、网络包数或媒体来源数量，都不能直接代替参与人数。

### 当前展示范围

当前展示为本地受控环境中的三方客户端互通，包含同场语音、成员退出、剩余成员继续交谈和正常结束，可用于多人语音场景的功能展示与集成评估。

[能力总览](README.md#zh) · [一对一语音](voice-calls.md#zh) · [通话安全架构](security-whitepaper.md#zh)

<a id="en"></a>

## English

Three-party voice calling has been implemented. Participants can join the same call, exchange speech in both directions and continue their conversation after a member leaves.

### Implemented interactions

| Capability | Behavior |
| --- | --- |
| Starting a group call | Create a shared voice conversation |
| Answering and joining | Invited participants join by answering |
| Shared voice | Participants can send speech and receive other members' audio |
| A member leaving | The remaining participants continue talking after one member leaves |
| Normal ending | Participants can leave or end the current call |

<a id="en-membership"></a>

### Consistency between membership and media objects

Group members have individual participation state, while the whole call has its own lifecycle. Invitation count, answered participants and currently playable audio sources can temporarily differ.

When a member leaves, queued audio and in-flight tasks may still belong to that member. Membership and media objects must remain consistent: late audio must not re-enter the current mix, and delayed events must not revive a departed member.

The current three-party scenario supports continued conversation between the remaining two participants after one leaves. That continuity depends on coordinated membership and media lifetimes, rather than merely removing an avatar.

<a id="en-voices"></a>

### Mixing: independent decoders, timing and amplitude limits

Each remote audio source retains independent decoder state and pending playout content. Compressed audio carries state across frames; alternating unrelated members' compressed bytes through one decoder confuses their histories.

Mixing operates on PCM with comparable sample meaning. Sample rates, channel layouts and playout positions need to agree; concatenating network payloads does not create a usable group conversation.

| Mixing constraint | Technical meaning |
| --- | --- |
| Time alignment | Audio contributes to its matching playout interval; late content is not spliced into a later interval |
| Amplitude bounds | Summed samples can exceed the PCM range, requiring clipping and overflow handling |
| Independent sources | One member's decoder state or temporary gaps must not contaminate the others |
| Bounded queues | A slow source or consumer must not create unlimited backlog for the conversation |

The current endpoint mix limits sample amplitudes after accumulation. This prevents numeric overflow, but clipping can still distort audio; bounded numeric output does not mean unchanged listening quality.

RTP source information distinguishes media streams, while participant identity belongs to call membership. A change in one person's media objects is not the same event as adding a participant.

<a id="en-relay"></a>

### Relay distribution and endpoint mixing

Group relays distribute encrypted speech within the current participation scope, while endpoints process their own audio. Forwarding does not require decoding and mixing every member's speech at the relay, so network distribution and audio computation have separate costs.

More participants expand the connections and sources to manage, but do not imply that identical local audio must be encoded again for every recipient. Media processing and delivery to multiple destinations are distinct; participant count alone does not determine encoder count or computation cost.

Call isolation is another runtime constraint: data from different groups must not enter the same conversation, and old forwarding or media activity after departure must agree with current participation scope.

<a id="en-observation"></a>

### Assessing whether group audio remains continuous

Group continuity needs membership, media and output considered together. One connected-group state cannot describe which member has audio or whether output comes from normal decoding or compensation.

| Perspective | Results to distinguish |
| --- | --- |
| Membership | Invited, answered, still participating and departed |
| Audio sources | Current sources, expired sources and sources temporarily lacking playable content |
| Decoding and mixing | Usable PCM, matching time intervals and mix amplitude limits |
| Output queue | Normal consumption, backlog discards and gap compensation |
| Lifecycle | One member leaving versus the entire call ending |

Three-party means three people participating simultaneously. Window count, packet count and media-source count do not directly substitute for participant count.

### Current demonstration scope

The demonstration covers three-party client interoperability in a controlled local environment, including shared audio, a member leaving, the remaining members continuing and normal ending. It supports feature demonstrations and integration evaluation for group-voice scenarios.

[Capability overview](README.md#en) · [One-to-one voice](voice-calls.md#en) · [Call security architecture](security-whitepaper.md#en)
