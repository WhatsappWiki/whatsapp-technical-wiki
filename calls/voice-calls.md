# 一对一语音通话 · One-to-One Voice Calls

[通话目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

已实现独立客户端与 Android 客户端之间的一对一双向语音通话，支持从任一侧发起呼叫，并由另一侧正常接听。

### 通话体验的几个观察面

| 阶段 | 功能表现 |
| --- | --- |
| 发起 | 主叫发出呼叫，进入等待接听的过程 |
| 来电 | 被叫接收来电并选择接听或拒绝 |
| 接通 | 双方进行持续的语音交互 |
| 结束 | 任一侧可以挂断，双方结束本场通话 |
| 后续 | 通话结束后可以继续消息交流 |

<a id="zh-duplex"></a>

### 双向通话中的独立媒体状态

发送方向需要保留编码器的跨帧状态，接收方向需要保留解码器与播放状态。这些状态与具体的声音来源绑定，不能把每个网络包都当成一份彼此独立的音频文件。

压缩语音通常利用前后帧的连续性。频繁重建编解码器，或让不同声音来源共用同一份解码状态，会破坏这种连续性，影响后续声音恢复。

单向无声可以按输入、发送活动、有效解码和设备输出分别判断。两个方向的计数与状态应独立解释，避免用一个总流量或“已连接”标志覆盖整场通话。

<a id="zh-codec"></a>

### Opus：压缩数据与媒体时长分别理解

Opus 是实时音频中的编解码名称。实际通话的编解码分支取决于协商与运行配置；文本标签或某个编码接口的名称，不足以单独确定压缩负载的解释方式。

| 维度 | 工程影响 |
| --- | --- |
| 编码帧长 | 影响一次处理的声音时长、调度节奏和等待开销 |
| 码率与可变码率 | 影响压缩数据量；相同时长的声音不一定产生相同字节数 |
| 编码复杂度 | 影响计算开销，与声音内容和实时预算一起考虑 |
| 静音处理 | 可以改变数据发送密度，播放时间仍要保持连续 |

压缩包大小不能直接代替样本数或媒体时长。解码也会更新内部状态：重复把同一段内容当作新语音处理，可能重复推进声音时间线；错误数据被丢弃后，后续播放还需要保持一致的状态边界。

<a id="zh-playout"></a>

### 抖动缓冲：受播放时限约束的队列

网络包可能乱序或成批到达，音频设备却按自己的时钟持续取样。抖动缓冲需要把到达波动与播放节奏隔开，并根据可用内容和播放时限处理暂缺片段。

缓冲增大会提高等待迟到内容的机会，同时增加交谈延迟。一个包即使最终到达，也可能已经错过对应的播放区间；把旧内容插入后面的语音会改变时间关系，而不是恢复原来的交谈。

接收队列与输出队列都需要容量边界。设备或上层消费者变慢时，持续积压会把短暂卡顿变成长时间延迟，因此队列长度、过期丢弃与补偿输出应分别观察。补偿维持的是播放连续性，不代表缺失语音被完整还原。

<a id="zh-rtp"></a>

### RTP：包序列、媒体时间与本机时钟

RTP 为实时媒体提供序列和时间语义，音频压缩方式由对应媒体格式决定。包的先后、声音在媒体中的位置以及本机何时收到数据，是不同的信息。

| 信息 | 表达什么 |
| --- | --- |
| 包序列 | 网络媒体单元的先后关系，用于理解缺失、重复与重排 |
| RTP 媒体时间 | 声音在媒体时钟中的位置，不是服务器墙钟 |
| 到达时间 | 本机收到数据的时刻，包含网络波动的影响 |
| 播放时钟 | 设备或播放调度实际需要输出声音的节奏 |

一个网络包不必恰好对应一个设备回调或一个编码帧。RTP 时间步进按媒体时钟解释，不能直接使用压缩字节数换算。序列或时间值的自然回绕也与新会话开始不同，需要结合连续的会话状态理解。

<a id="zh-device"></a>

### 音频设备：实时回调与声学条件

设备采样率、编码器输入格式和 RTP 时钟频率分别有自己的含义。它们不一致时，需要在对应音频边界处理格式或节奏，不能直接按设备回调次数推算网络发包次数。

实时音频回调对阻塞和调度抖动敏感。网络等待、控制任务或慢消费者占用音频执行时间，会让设备来不及取得下一段声音，即使网络仍在收包，也可能出现断续。

回声处理需要与实际播放的声音相关联。听筒、扬声器和耳机形成不同的声学路径；输出设备改变后，原来的回声条件不一定继续成立。参与者静音、采集静音和本地播放静音也应分别理解。

<a id="zh-pcm"></a>

### PCM：样本布局决定声音含义

PCM 用采样值序列表达声音。解释一段 PCM，需要同时知道采样率、声道数、样本表示、字节序及声道排列。把双声道数据当单声道，或混用整数与浮点样本的数值范围，都会改变声音的解释。

在固定样本宽度下，PCM 字节量等于每声道样本数、声道数与每样本字节数的乘积。这一关系描述原始音频数据量，不决定压缩包大小或 UDP 数据报长度。

| 粒度 | 主要约束 |
| --- | --- |
| 设备回调块 | 由本机设备和音频调度提供，可能需要累积或拆分 |
| 编码帧 | 需要满足当前编解码格式的完整输入要求 |
| 网络数据报 | 由传输和媒体组织方式决定，保留自己的消息边界 |

三种粒度不必一一对应。片段交接还涉及缓冲区所有权：异步使用的声音数据不能随着前一次设备回调结束而失效。

<a id="zh-relay"></a>

### 中继：转发成本与音频处理成本

已实现的中继承担加密语音数据的转发职责，不需要对每份数据执行 Opus 解码与重新编码。转发流量与编解码计算是不同的资源开销，中继通路有流量也不意味着终端已经产生可播放声音。

语音媒体使用独立的 UDP 数据通路，通话交互与连续媒体的运行状态分别管理。控制交互仍可用时，媒体方向也可能发生中断，因此两类状态不能仅用一个连接结果合并判断。

不同通话的参与范围需要隔离。中继的连接与转发状态、终端的媒体状态以及设备声音状态各自保留边界，定位问题时应分别核对。

<a id="zh-udp"></a>

### UDP：数据报边界与有效到达时间

UDP 保留数据报边界，不提供应用所需的有序、可靠送达承诺。把多个数据报拼成普通字节流，或仅按收包次数推断完整语音，会丢失媒体组织所需的信息。

| 网络现象 | 对实时语音的影响 |
| --- | --- |
| 乱序 | 内容可能仍完整，但需要在播放时限内处理顺序 |
| 重复 | 重复数据不应被当作新的语音再播放 |
| 迟到 | 包已到达，但对应声音可能已经没有播放价值 |
| 丢失 | 需要区分等待、补偿和利用其他可用内容的效果 |

实时语音关注有效到达时间。等待重传可能改善数据完整性，也可能错过播放时限；补偿可以缓和听感，却不等于原始内容已到达。过大的数据报还会增加网络分片的风险，因此包数、包大小与播放连续性不能只优化其中一项。

<a id="zh-ending"></a>

### 生命周期：结束动作需要幂等

拒接、接听前取消、通话中挂断和连接中断分别产生结束结果。结束请求可能与网络回调、音频回调和界面操作交错到达，单独关闭界面不能代表所有后台活动已经停止。

| 生命周期约束 | 要避免的问题 |
| --- | --- |
| 重复结束保持一致结果 | 多次释放、重复结束事件与状态回退 |
| 在途音频访问与释放协调 | 释放后仍读取旧缓冲或使用已关闭的编解码状态 |
| 旧通话事件与新通话隔离 | 迟到事件误结束新呼叫，或恢复已经结束的声音 |
| 通话资源与消息连接分开管理 | 挂断语音时误伤后续消息交流 |

当前展示包含正常挂断后的消息交流。通话收尾既是用户状态处理，也是对线程、队列和媒体对象生命周期的协调。

### 当前可展示的侧面

独立客户端可以主动呼叫 Android 客户端，也可以接受 Android 客户端发起的呼叫。展示内容包括两个方向的接听、持续双向语音、拒接、接听前取消、正常挂断和通话后的消息交流。

这些能力面向本地受控环境中的客户端互通，可用于语音交互原型、功能展示和既有业务的接入评估。

[能力总览](README.md#zh) · [群组语音](group-calls.md#zh) · [通话安全架构](security-whitepaper.md#zh)

<a id="en"></a>

## English

Two-way one-to-one voice calling has been implemented between an independent client and an Android client. Either side can initiate a call and the other can answer normally.

### Several views of the call experience

| Stage | Behavior |
| --- | --- |
| Initiation | The caller starts a call and waits for an answer |
| Incoming call | The recipient receives the call and chooses to answer or reject it |
| Connected | Both sides participate in continuous voice interaction |
| Ending | Either side can hang up, ending the call for both participants |
| Afterwards | Participants can continue exchanging messages |

<a id="en-duplex"></a>

### Independent media state in a two-way call

The sending direction retains codec state across frames, while the receiving direction retains decoder and playout state. These states belong to particular audio sources; each network packet is not an independent audio file.

Compressed speech commonly relies on continuity between frames. Recreating codecs frequently or sharing one decoder state across unrelated sources breaks that continuity and affects subsequent reconstruction.

One-way silence can be examined through input, sending activity, valid decoding and device output separately. Interpret each direction's counters and state independently instead of reducing the call to total traffic or a connected flag.

<a id="en-codec"></a>

### Opus: compressed size and media duration

Opus is a codec name used in real-time audio. The actual coding branch depends on negotiation and runtime configuration; a text label or encoder-interface name alone does not determine how to interpret a compressed payload.

| Dimension | Engineering effect |
| --- | --- |
| Codec frame duration | Affects the amount of sound processed, scheduling cadence and waiting overhead |
| Bitrate and variable bitrate | Affect compressed size; equal durations need not produce equal byte counts |
| Encoder complexity | Affects computation and must fit the content and real-time budget |
| Silence handling | Can change transmission density while playout time remains continuous |

Compressed packet size does not substitute for sample count or media duration. Decoding also updates internal state: treating the same content as new speech again can advance the audio timeline twice. After invalid data is discarded, subsequent playout still needs consistent state boundaries.

<a id="en-playout"></a>

### Jitter buffering: a queue bounded by playout deadlines

Packets may arrive out of order or in bursts, while audio devices consume samples on their own clocks. Jitter buffering separates arrival variation from playout cadence and handles temporary gaps according to available content and deadlines.

A larger buffer gives late content more time to arrive but increases conversational delay. A packet can eventually arrive after its playout interval has passed; inserting that old content into later speech changes the timing rather than restoring the original conversation.

Receive and output queues both need capacity limits. A slow device or consumer can turn a brief stall into sustained delay through accumulated backlog. Queue length, expired-content discards and compensation output should therefore be observed separately. Compensation maintains continuity rather than completely restoring missing speech.

<a id="en-rtp"></a>

### RTP: packet sequence, media time and local clocks

RTP supplies sequence and timing semantics for real-time media; the corresponding media format defines audio compression. Packet order, the sound's media position and local arrival time convey different information.

| Information | Meaning |
| --- | --- |
| Packet sequence | Ordering of network media units, helping distinguish gaps, duplicates and reordering |
| RTP media time | Position in the media clock, rather than a server wall clock |
| Arrival time | Local receipt time, including the effects of network variation |
| Playout clock | The cadence at which the device or playback scheduler needs audio output |

A network packet need not correspond to one device callback or codec frame. RTP time advances in the media clock, not in compressed bytes. Natural sequence or timestamp wraparound also differs from a new session and must be interpreted with continuous session state.

<a id="en-device"></a>

### Audio devices: real-time callbacks and acoustics

Device sample rate, encoder input format and RTP clock frequency have distinct meanings. Differences require handling at their respective audio boundaries; device callback counts do not directly determine network packet counts.

Real-time audio callbacks are sensitive to blocking and scheduling jitter. Network waits, control work or slow consumers occupying audio execution time can prevent a device from receiving its next sound block in time, even while network packets continue arriving.

Echo processing relates to the sound actually played. Earpieces, speakers and headphones create different acoustic paths; an output change can invalidate earlier echo conditions. Participant mute, capture mute and local playback mute are also separate states.

<a id="en-pcm"></a>

### PCM: sample layout determines meaning

PCM represents sound as sample values. Interpreting a block requires its sample rate, channel count, sample representation, byte order and channel layout. Treating stereo as mono or mixing integer and floating-point amplitude ranges changes the meaning of the sound.

For a fixed sample width, PCM byte count is the product of samples per channel, channel count and bytes per sample. This describes raw audio volume rather than compressed packet size or UDP datagram length.

| Granularity | Main constraint |
| --- | --- |
| Device callback block | Supplied by device scheduling and may need accumulation or splitting |
| Codec frame | Must satisfy the current codec format's complete input requirements |
| Network datagram | Follows transport and media organization, retaining its message boundary |

These granularities need not match one-to-one. Buffer ownership also matters at handoff: audio used asynchronously cannot become invalid when the earlier device callback finishes.

<a id="en-relay"></a>

### Relaying: forwarding cost versus audio-processing cost

The implemented relay forwards encrypted voice data without decoding and re-encoding each item as Opus. Traffic forwarding and codec computation consume different resources; relay traffic does not establish that endpoints produced playable audio.

Voice media uses a separate UDP data path, while call interaction and continuous media retain their own runtime state. Control interaction can remain available during a media-direction interruption, so one connection result cannot stand for both.

Participation scopes of different calls need isolation. Relay connectivity and forwarding, endpoint media and device audio retain distinct state boundaries that should be checked separately during diagnosis.

<a id="en-udp"></a>

### UDP: datagram boundaries and useful arrival time

UDP preserves datagram boundaries without providing the ordered, reliable delivery an application may require. Concatenating datagrams into an ordinary byte stream or inferring complete speech from receipt counts alone loses information needed for media organization.

| Network condition | Effect on real-time audio |
| --- | --- |
| Reordering | Content may remain complete but must be reordered within playout deadlines |
| Duplication | Repeated data must not be played again as new speech |
| Late arrival | The packet arrived, but its audio may no longer be useful for playout |
| Loss | Waiting, compensation and using other available content have different effects |

Real-time speech depends on useful arrival time. Retransmission can improve completeness while missing the playout deadline; compensation can soften a gap without delivering the original content. Oversized datagrams also increase fragmentation risk, so packet count, size and playout continuity cannot be optimized in isolation.

<a id="en-ending"></a>

### Lifecycle: ending must be idempotent

Rejection, cancellation before answering, hangup and connection interruption produce their respective outcomes. End requests can interleave with network callbacks, audio callbacks and interface actions; closing the screen alone does not establish that background activity stopped.

| Lifecycle constraint | Failure to avoid |
| --- | --- |
| Repeated end requests preserve a consistent result | Double release, duplicate ending events and state regression |
| Coordinate in-flight audio access with release | Reading freed buffers or using closed codec state |
| Isolate old-call events from new calls | Late events ending a new call or reviving ended audio |
| Manage call resources separately from messaging connectivity | A voice hangup disrupting later messages |

Messaging after normal hangup is included in the current demonstration. Teardown coordinates user state alongside the lifetimes of threads, queues and media objects.

### What can currently be demonstrated

The independent client can call the Android client and answer calls initiated from Android. Demonstrations include answering in both directions, continuous two-way speech, rejection, cancellation before answering, normal hangup and messaging afterwards.

These capabilities cover client interoperability in a controlled local environment, supporting voice-interaction prototypes, feature demonstrations and integration evaluation for existing workflows.

[Capability overview](README.md#en) · [Group voice](group-calls.md#en) · [Call security architecture](security-whitepaper.md#en)
