# 多设备消息分发与状态同步 · Multi-Device Message Distribution and State Synchronization

[多设备目录](README.md#zh) · [首页](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

WhatsApp 分别管理账号、设备、加密会话和消息状态。发送时先选择目标，用各设备的独立会话加密并组包；本人其他设备的副本还要保留原聊天归属。

下文区分**当前设备目录、消息目标快照、补发差集**，以及服务端、设备、用户和群整体回执，说明设备变更、重试、去重及历史迁移后的状态变化。

### 阅读导航

| 阅读阶段 | 章节 |
| --- | --- |
| 设备与分发 | [身份](#identities-zh) · [目录](#directory-zh) · [目标筛选](#targets-zh) · [单聊加密](#pairwise-zh) · [群加密](#groups-zh) |
| 设备变化 | [增删与会话](#device-changes-zh) · [哈希与补发](#hashes-zh) |
| 发送可靠性 | [队列与重试](#queue-zh) · [连接与超时](#connection-zh) · [密文复用](#ciphertext-zh) · [去重](#deduplication-zh) |
| 状态与迁移 | [回执合并](#receipts-zh) · [计数与回退](#regression-zh) · [ChatTransfer](#history-zh) |
| 串联阅读 | [完整例子](#walkthrough-zh) · [相关主题](#related-zh) |

<a id="identities-zh"></a>

### 1. 账号、设备与会话地址

| 对象 | 作用 | 需要区分的维度 |
| --- | --- | --- |
| 用户 JID | 标识账号或聊天参与者 | PN 与 LID 属于不同账号命名空间 |
| DeviceJid | 标识用户的一台设备 | 用户身份、账号域和 device ID |
| 当前设备地址 | 判断候选目标是不是本机 | 本机 PN 地址、本机 LID 地址及实际注册设备编号 |
| Signal 会话地址 | 定位点对点加密状态 | 账号、账号类型、设备、scope、type |
| 消息身份 | 定位逻辑消息 | 聊天、方向和消息 ID，持久化另有发送者列 |

主设备通常使用 device ID `0`，关联设备使用自己的设备编号。当前发送端也可能是关联设备，此时排除本机必须使用它实际的非零设备编号。`registration_device_id` 与 Signal 的本机 registration ID 分属不同状态，不能互相替代。

PN 地址与电话号码身份相关，LID 使用另一套账号标识。账号映射可以关联两者，但数据库不自动合并不同 JID；转换设备地址时仍需保留 device ID。

点对点会话以五维键定位：

```text
(recipient_account_id, recipient_account_type,
 device_id, session_scope, session_type)
```

会话记录包括棘轮、root/chain 状态、跳过的消息密钥及待完成的预密钥状态。每台设备的状态独立推进。群 Sender Key 另按群、实际发送者设备和 bucket 等维度定位。字段与存储见 [axolotl.db 会话](../messaging/storage/axolotl.md#sessions-zh)。

<a id="directory-zh"></a>

### 2. 发送前如何取得设备目录

设备目录查询采用 `usync` IQ，逻辑结构包括：

```text
iq xmlns=usync type=get
  usync mode=... context=... sid=... index=... last=...
    query
      devices version=2
    list
      user jid=目标账号
```

响应中的 `devices/device-list/device` 提供设备编号和 `key-index`；`key-index-list` 还有字节内容、`ts`、`expected_ts`，相关分支同时处理 LID 和 hosted 设备资料。节点的二进制编码见 [Protocol Tree Node](../protocol/protocol-tree-node.md#zh)。

目录还通过设备 `add/remove` 通知更新。处理器比较通知的新旧程度，验证相关 ADV 索引资料，应用增量，再检查 `device_hash` 或 `device_lid_hash`。验证或哈希不一致时，安排重新同步；较旧通知可以被忽略。

带 `offline` 属性的设备通知可以先将账号登记到 `pending_users_to_sync_device`。这是待同步状态；任务被安排、同步已完成和发送任务已使用新目录是三个时点。正在同步的账号还会去重，避免重复安排相同工作。

目录标识设备，预密钥和现有会话提供加密条件。发现设备不等于建好会话，发送也不要求逐条查询整份目录。

<a id="targets-zh"></a>

### 3. 目标筛选与当前发送设备排除

#### 3.1 普通目标集合

用 `A0` 表示当前发送设备，`A1` 表示本人的另一台设备，`B0/B1` 表示对方的主设备和关联设备。普通完整分发可选择：

```text
对方设备：{B0, B1}
本人同步设备：{A1}
逻辑目标：{B0, B1, A1}
```

目标选择还受消息条件影响：

| 条件 | 对集合的影响 |
| --- | --- |
| 普通远端账号 | 取得目录中的对方设备，并补入对方主设备 |
| 允许本人副本 | 按账号命名空间选择有效的本人关联设备 |
| 禁止本人副本 | 跳过本人关联设备加入步骤 |
| primary-only 消息 | 采用主设备目标分支，不按普通全设备集合处理 |
| 显式目标设备 | 与计算或记录的允许集合取交集 |
| 自聊及特定消息 | 可有显式本人设备 fallback |
| 特殊账号、bot、guest、hosted | 继续经过各自的资格与地址处理条件 |

这些条件操作不同的集合，不能合成一个统一的“多设备开关”。

#### 3.2 怎样精确排除本机

本机判断分别比较当前 PN DeviceJid 和当前 LID DeviceJid：

```text
isCurrentDevice(d) =
    d == currentPnDevice
    OR d == currentLidDevice
```

本人目录刷新跳过这两个地址，读取有效关联设备时还会过滤退出记录；群目标和部分集合扩展路径再逐项排除本机。比较包含账号与实际设备编号，因此保留本人其他设备及编号相同的远端设备；`isPrimary` 或同账号判断不能替代它。

排除发生在目录维护和具体集合路径，加密入口再与允许集合相交，并非所有消息统一在最后 `remove(current)`。

#### 3.3 当前目录与消息目标快照

已保存消息通常读取自己的目标快照，显式目标和批量加密输入仍与允许集合相交；身份变化使目标失效时可丢弃它。旧消息不自动采用整份最新目录，但[同步补发任务](#hashes-zh)可验证差集、登记新目标并定向补发。

<a id="pairwise-zh"></a>

### 4. 单聊怎样为每台设备加密

#### 4.1 会话准备与密文类型

发送任务检查会话 requirement，缺少资料时可批量获取预密钥。已有会话使用当前棘轮；附带预密钥建链资料时输出 `pkmsg`，普通会话密文使用 `msg`。待完成预密钥状态可跨多条发送保留，因此 `pkmsg` 不限于第一条业务消息。

每个目标先形成对应的业务 Message protobuf，再序列化、添加业务 padding，并用该目标的会话加密：

```text
C_B0 = SignalEncrypt(Session[A0,B0], Message)
C_B1 = SignalEncrypt(Session[A0,B1], Message)
C_A1 = SignalEncrypt(Session[A0,A1], DeviceSent(destination=B, Message))
```

三次操作分别推进对应发送链。`C_B0` 不能直接作为 `B1` 或 `A1` 的密文使用。

#### 4.2 本人设备如何知道原聊天

给本人关联设备的副本可使用 `Message.deviceSentMessage`，其字段为：

| 字段 | 编号 | 内容 |
| --- | --- | --- |
| `deviceSentMessage` | 外层 Message 的 31 | 本人已发送消息包装 |
| `destinationJid` | 包装内 1 | 原聊天目标 |
| `message` | 包装内 2 | 原业务 Message |
| `phash` | 包装内 3 | 条件存在的参与集合摘要 |

包装位于端到端加密载荷内，并将 `messageContextInfo` 移到外层。`destinationJid` 保留原聊天，外层 DeviceJid 选择接收设备，所以本人副本与对方副本的 protobuf 可以不同。接收方校验本人 PN/LID 身份、非本机源设备、字段 presence 和 destination 对应关系后解包，沿用原 ID。

#### 4.3 多份密文如何一次组包

逐设备加密结果可以组织成 `Map<DeviceJid, ciphertext>`，再组成同一个逻辑消息节点：

```text
message to=B id=M
  participants
    to jid=B0
      enc v=2 type=msg或pkmsg : C_B0
    to jid=B1
      enc v=2 type=msg或pkmsg : C_B1
    to jid=A1
      enc v=2 type=msg或pkmsg : C_A1
```

此示意树按[二进制节点编码](../protocol/protocol-tree-node.md#zh)，`enc` 为字节载荷；其他分支可用顶层 `enc` 或附加节点。设备加密、message、socket write 和 TCP 分段是不同计数，`enc v` 也与 Signal 内部版本分开。

~~~mermaid
sequenceDiagram
    participant A as A0 发送端
    participant S as 服务端
    participant B0 as B0 对方主设备
    participant B1 as B1 对方关联设备
    participant A1 as A1 本人其他设备
    A->>A: 选择目标并分别加密
    A->>S: message M，携带多个目标 enc
    S->>B0: B0 对应密文
    S->>B1: B1 对应密文
    S->>A1: A1 对应的 DeviceSent 密文
    B0->>B0: 用对应会话解密
    B1->>B1: 用对应会话解密
    A1->>A1: 解密并恢复原聊天归属
~~~

服务端根据目标设备转交对应密文，消息端到端加密与[连接层 Noise 保护](../protocol/noise.md#zh-layers)分别承担不同职责。

<a id="groups-zh"></a>

### 5. 群正文与 Sender Key 分发

普通群发送把正文加密和密钥分发分开：

| 部分 | 加密与寻址 |
| --- | --- |
| 群正文 | 使用群 namespace 下实际发送者设备的 Sender Key 链，形成一份 `skmsg` |
| Sender Key Distribution Message（SKDM） | 放入业务 Message，按需通过目标设备的点对点会话加密为 `msg/pkmsg` |

SKDM 包括 key ID、iteration、chain key 和签名公钥；业务 `senderKeyDistributionMessage` 还携带 groupId 与原始 SKDM 字节。发送链派生本条正文的加密材料；正文签名使用对应 Sender Key 记录的签名私钥，SKDM 传递的是签名公钥。

```text
message to=Group id=M
  enc type=skmsg : 群正文密文
  participants
    to jid=需要分发密钥的设备1
      enc type=msg或pkmsg : 该设备的 SKDM 密文
    to jid=需要分发密钥的设备2
      enc type=msg或pkmsg : 该设备的 SKDM 密文
```

SKDM 按群设备状态选目标并与消息快照相交；有相应状态的设备不必每条正文重复分发。新设备、状态缺失或分发标志重置时，后续发送再补分发。

`A0` 与 `A1` 分别作为发送设备时，具有各自的群发送链、iteration 和签名状态。不能让同一账号不同设备共享一个固定群 AES key 或直接共用发送计数器。

群发送前还有条件性的 `force-one-one`：满足群目标、无特定 participant/显式目标、启用相应 hash-mismatch 策略等条件时，当前参与集合哈希与任务保存值不同，可改为逐设备加密正文。这发生在发出前，不需要等待服务端 ACK 或解密 retry receipt。其它一次性、primary/companion 和专用 ratchet 分支按各自条件处理。

<a id="device-changes-zh"></a>

### 6. 设备增删与会话更新

#### 6.1 key-index 变化也是一次替换

目录差集比较“设备地址 → key-index”：

```text
added   = 新地址 + 同地址但 key-index 改变
removed = 已删除地址 + 同地址但 key-index 改变
```

#### 6.2 立即删除与暂存会话

移除按待处理工作和配置门控分流：

| 情况 | 处理 |
| --- | --- |
| 需要立即清理 | 锁定相应地址，删除 session 和 identity |
| 纯移除且仍有相关工作等条件成立 | 登记 `dead_devices`，暂存 session，同时删除 identity |
| 同地址 key-index 改变 | 不属于纯移除集合，进入 session/identity 重置 |

相应 session 删除按归一化后的账号、账号类型和 device ID 执行，不附加 scope/type 条件，因此可清理这个地址的多个 scope/type 行及对应缓存；它不会自动扩大为所有无关 PN/LID 地址。

`dead_devices` 等待工作与 in-flight 门控允许后处理：设备已返回则移除标记并保留状态，否则删除会话、identity 和该发送设备的 Sender Key。仍有工作时可每 `2000 ms` 重查，索引达到 `15` 后仍可保留登记，并非 30 秒后强制删除。

#### 6.3 群密钥分发状态

新增群设备会建立 `sent_sender_key=false` 的记录，后续发送才按需分发 SKDM。移除路径根据设备是否已取得 Sender Key 等条件删除相应发送 key，并重置整组或 bucket 的分发标志，例如 `sent_sender_key=0`、`sent_sender_key_bucket=NULL`。

设备目录更新、会话清理和 Sender Key 重置可能由不同异步任务完成；通知接收成功不等于这些动作在同一时刻全部完成。具体存储见 [axolotl.db](../messaging/storage/axolotl.md#zh)。

<a id="hashes-zh"></a>

### 7. 目录哈希、消息 phash 与差集补发

| 摘要 | 所描述的集合 |
| --- | --- |
| `device_hash` / `device_lid_hash` | 对应账号、地址表示下的设备目录 |
| 消息级 `phash` | 该调用路径选定的消息参与/设备集合 |
| DeviceSent 中的 `phash` | 本人副本包装条件携带的相关集合摘要 |

哈希原语按分支格式化 JID，经 Java 字符串排序、拼接、SHA-256 后取前 6 字节，用标准 Base64 表示并加 `2:` / `3:` 前缀。去重和输入集合由调用者决定，仍需 ADV 和 Signal 身份验证。

普通服务端 ACK 携带非空 `phash` 时，相应处理分支会与该消息的本地摘要比较。该本地输入可包含消息目标、本机地址及条件性的 guest；**摘要里包含本机不等于把消息再发给本机**。

失配可安排 `SyncDeviceAndResendMessageJob`：

1. 将原消息目标及相关本机地址投影成账号集合，发起需要的目录/密钥同步。
2. 等相应结果与任务条件满足后，重读原消息和原目标。
3. 重新计算有效候选，求 `newTargets = candidates − originalTargets`。
4. 对差集执行 PN/LID、hosted、bot、版本及消息类型等资格过滤。
5. 登记新增目标，把原消息 ID 与显式差集交给发送流程。

它不是向所有原设备重新群发，也不是每次设备通知都扫描全部旧消息。普通补发任务的有效期为构造时当前时间加 `3600000 ms`；编辑分支有独立设置，缺失消息、业务终止状态、到期或退出登录等条件可结束该任务。

<a id="queue-zh"></a>

### 8. 发送任务、重试预算与有效期

#### 8.1 本地交接与实际执行

普通内部发送先准备业务 protobuf，再创建 `SendE2EMessageJob` 入队。本地交接回调也可出现在丢弃分支，不能表示任务已发出或收到服务端 ACK。

队列由线程池执行，同 groupId 互斥、不同 group 可并发；业务 retry 可使用 `_retry` 后缀。Job 去重、目标 in-flight 和 group 互斥分别维护。

任务的就绪检查未通过时留在 pending，不执行、不消耗次数；条件通知、入队和 group 释放等事件唤醒扫描，没有统一的固定扫描周期。

#### 8.2 异常后的额外等待

对于允许重试的 checked Exception：任务仍被判定为 ready 时，worker 可直接再执行，队列层额外 backoff 为 `0`；否则退回 pending 等事件。这个 `0` 不包含加密、ACK 等待、线程调度和重连耗时。

发送还有可配置的人工延迟：以概率 `p` 等待固定 `d` 毫秒。它不随失败次数递增，也不等于连接层退避。`RuntimeException` 经过独立的向外抛出路径，`Error` 也不按普通 checked Exception 重试逻辑处理。

#### 8.3 四类计数不能合并

| 机制 | 条件与上限 |
| --- | --- |
| 普通 Send / Sync Job 执行 | 默认最多 `100` 次执行，含首次；等待 requirements 不计一次执行 |
| 单目标加密失败 | 同一任务内按目标累计；计数 `1/2/3` 允许再试，`>3`（达到 4）停止该异常的重试 |
| 接收端解密 retry receipt | 普通单聊 `count>4` 拒绝；发送任务另检查业务 `retryCount>4` |
| 独立未确认 stanza 重放 | 默认登记计数从 1 开始、`count<3` 才继续；通常最多 3 次登记发送，部分通话配置可覆盖 |

100 次执行计数是 transient，进程恢复后从 0 开始；失败不逐次保存 attempt/nextRun，其他可序列化计数也只恢复入库快照。持久化受消息类别限制，因此 100 不是跨重启的永久重试上限。

#### 8.4 解密失败后的定向修复

retry receipt 用原消息 ID 定位已发消息，并指定请求设备。普通单聊分支检查 `registration` 恰为 4 字节、retry version（缺省 1）、count（缺省 1）、消息存在性/类型/时间和原目标资格。消息早于调整时钟减 60 天、普通请求设备不在原目标中或该设备已送达等情况有拒绝分支。

携带密钥时需要 identity 和 signed-prekey，one-time key 可省；`offline=true` 还要求已有相应会话且 remote registration ID 匹配。之后仍执行身份、签名与信任处理。registration 或 Alice base key 变化可触发会话重建要求；反复请求却继续使用相同 base key 时，可要求不同 base key 的新会话。

允许修复后，重建原业务内容、保留逻辑消息 ID、面向请求设备创建发送任务，重新进入加密。群/广播另有当前与历史设备、成员、送达及主/关联设备门控，不应套用普通单聊的单一集合规则。

#### 8.5 有效期是另一条边界

普通准备路径的默认过期时间为锚点加 `86400000 ms`（24 小时）；显式过期时间戳为正值时，与默认值取最小值。重发在相应配置条件下可用原消息时间作为锚点，而非总从重发时再延长一天。

Send 在 `now>=expireTimeMs` 时处理到期；Sync 的正有效期检查使用 `now>expirationMs`，`0` 在该 Sync 检查中关闭到期限制。到期不自带所有路径通用的唤醒定时器，被其他条件阻塞的任务可能等事件后才被取出结束。

<a id="connection-zh"></a>

### 9. 连接恢复、ACK 超时与队列唤醒

#### 9.1 连接退避与消息重试分开

重连调度器使用 Fibonacci 序列，初始 seed 为 `1`、序列值上限为 `17280`：

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ... , 17280, 17280, ...
B = 序列值 × 10000 ms
```

`B=0` 时立即提交连接请求；`B>0` 时请求等待为：

```text
delay = floor(B/2) + ((Random.nextLong() & Long.MAX_VALUE) % B)
```

| 名义 B | 加抖动后的请求等待 |
| --- | --- |
| 10 秒 | 5000..14999 ms |
| 封顶 48 小时 | 24 小时..72 小时减 1 ms |

48 小时只限制名义值。调度被去重时不推进序列，调度数也不等于 TCP 失败数；成功连接重置序列。`force` 清 pending deadline、`reset` 重置序列，网络、push 和前台入口可组合使用。

登录失败类型 6 的相应限幅分支在结果超过 `900000 ms` 时改为 `899970..900029 ms`；这里的随机偏移是毫秒，且不适用于所有断线。重连调度器没有与消息任务共享“100 次后停止”的预算。

#### 9.2 普通消息的 45 秒 watchdog

ACK Future 本身使用无数值参数的等待。ConnectionThread 处理 Writer 正常返回通知后，安排 `45000 ms` watchdog：到点重读消息（可命中缓存），若仍存在、排名低于服务端接收门槛 `4` 且不是排除状态 `20`，就要求强制断线；清理令未完成 Future 失败，再由 Job 判断重试。

起点是处理写出通知，Writer 返回不表示服务端接收。新通知追加计时、不延长旧 deadline；已达状态门槛则跳过。失败可清多项计时，因此这是连接级监控，不是每个 Future 独立的 45 秒 TimeoutException。

启用早期诊断配置（38734）时，在 `22500 ms` 检查同一门槛，仅记录诊断，不直接断线。其它超时分别为：

| 机制 | 阈值/起点 | 行为 |
| --- | --- | --- |
| active-mode 监控 | 32 秒，active 请求提交或相应 Writer 消息入队后 | 满足条件时请求断线 |
| ping 健康监控 | 配置值限制在 8..32 秒 | connected/ready 等门控满足后通知断线 |
| 明确的 IQ Future 入口 | 32 秒，登记 callback 起 | 使对应 Future 失败，不由该 timer 主动断线 |
| 一般 IQ / tracked stanza | 调用方正 timeout | 调用相应超时 callback；传 0 的分支没有此 timer |
| 单个 socket connect | 30 秒 | 限制该连接建立，不覆盖 DNS/握手/登录总耗时 |
| 登录后的长阻塞 read | `companion_registration_state` 为 2..6 或 10..17 时 3 分钟，其余相应登录路径为 17 分钟 | 限制一次阻塞读取，不是某条消息的 ACK 时限 |

#### 9.3 断线后怎样再次执行

```text
断线 → 连接条件变假、ACK Future 失败 → Job 按异常/ready 决定再执行或回 pending
恢复 → provider 事件 → 队列重新检查 group/任务就绪 → 分派可执行任务
```

ready 不等于 `socket.isConnected()`；状态变更、通知、扫描和执行各是一步，连接恢复不表示任务已发出。

同步补发任务还可要求 `OFFLINE_RESUME_COMPLETED`：收到 offline complete 之后，离线处理任务计数归零才进入该状态。仅收到 complete 不够；这个 requirement 也不应强加给全部普通发送任务。

<a id="ciphertext-zh"></a>

### 10. 什么时候复用密文，什么时候重新加密

| 再发送所在层 | 业务 ID | 端到端密文 |
| --- | --- | --- |
| 普通 Job 重执行、解密修复或设备差集补发，且到达加密阶段 | 保留原 ID | 按各自目标和当前会话/群状态重新加密 |
| Writer 保留已有 Message/Node 并重放 | 使用节点内 ID | 可沿用节点中已有 `enc` 字节 |
| 传输层重新写节点 | 使用同一逻辑节点 | 仍用当前连接的计数重新做传输层 AEAD |

任务可复用保存的 protobuf、padding 输入等材料，但到达加密阶段仍用当前状态生成密文并推进发送链。部分目标成功后其他目标失败，不保证全体状态回滚，所以下次同 ID 的密文未必相同。

Writer 的离线保留有明确边界：普通聊天发送使用的 Android message type `8` 在 disconnected 排除表内，不加入通用离线队列。写入异常可暂存原 Message 引用，但再次处理仍断线时会被该规则过滤。

另一种接受预构造 Node 的 type `486`、`dropIfOffline=false` 路径，可在 ready/callback 等条件满足时离线保留并重放原节点。独立 tracked stanza 管理也可保存原 Message；这些路径不是普通聊天 Job 的默认重试机制，不能把其默认登记次数套给全部消息。

复用 E2E `enc` 不等于复用整个网络帧：每次节点写出仍用当前连接密钥和递增计数做传输 AEAD。[Noise](../protocol/noise.md#zh)负责连接保护，[TCP 帧](../protocol/tcp-framing.md#zh)负责定界。

<a id="deduplication-zh"></a>

### 11. ACK 丢失、同消息 ID 与去重

#### 11.1 不同层使用不同键

| 层 | 身份或约束 |
| --- | --- |
| 业务 MessageKey | `chat + from_me + id` |
| 发送 Job 并发去重 | `jid + id + editVersion + participant`，不含显式设备集合或密文 |
| 目标 in-flight | MessageKey 下的原消息和目标集合 |
| ACK callback | class、remote、id、participant、recipient、type、editVersion、附加属性、optional child、category |
| 消息表唯一索引 | `chat_row_id + from_me + key_id + sender_jid_row_id` |
| Signal 重放检测 | 对应会话/ratchet chain 的 counter 和可用 skipped keys |

同一个裸 ID 在不同聊天或方向下不是同一个业务 Key。同 ID 的编辑、participant 或 receipt 类型也可能使用不同任务/ACK 键。发送 Job 的并发去重不含目标集合，另建相同键但不同 targets 的任务仍可能被抑制。

Job 去重只登记进程内活动任务，完成/取消后移除、恢复时重建，并非永久完成历史。缓存常用三列业务键查询，物理唯一索引另有 sender 列，需结合 sender 规范化和冲突处理。

#### 11.2 晚到 ACK 可以匹配哪次等待

ACK callback 按归一化的逻辑 stanza 键登记。相同键再次登记可覆盖当前 callback；收到 ACK 时先从 map 移除匹配项，再完成 callback。

| 时序 | 结果 |
| --- | --- |
| 已消费 ACK，之后没有新登记 | 重复 ACK 找不到该项 |
| 断线清表后，旧 ACK 先于新登记 | 没有当前 callback 可匹配 |
| 重试已用相同键登记，旧 ACK 后到 | 可以消费当前 callback |
| ID 相同但 participant/recipient/edit/type 等不同 | 不能只按 ID 匹配 |

ACK 不携带“这是第几份密文”的本地加密 attempt 身份，所以匹配成功不证明它对应最新密文字节，也不等于全部设备送达。

编辑 NACK 另有受限回退：精确匹配失败、`class=message`、ACK 缺 edit 且含 error 等条件成立时，可匹配去掉 edit 后唯一对应的候选；不是按裸 ID 任意消费。

普通服务端 ACK 的主状态更新另有守卫：重读状态排名已经达到 `4` 时返回，不再执行该普通状态和服务端时间写入。callback 消费与状态持久化是两条异步链。

普通 Send 的“已被服务端确认则退出”也有条件：业务 retryCount 为 0、没有非空显式目标、消息状态已到门槛，并满足相应配置/聊天分支。带显式目标或正 retryCount 的设备补发不能一概被 server ACK 阻止。

#### 11.3 接收端为什么仍要业务去重

接收端从源 DeviceJid 投影到用户身份，再按聊天、方向和 ID 等定位消息；本人的 peer 副本还按原 destination 归属到相应聊天。PN/LID 映射按对应目录与迁移状态执行。

同 ID 再到达至少有三种情况：

1. **原 Signal 密文重复到达**：counter 已消费，可能进入 old-counter 处理；是否仍有 skipped key 影响解密。
2. **同 ID 的新 Signal 密文到达**：新 counter 可以成功解密，之后仍需业务消息去重。
3. **另一台设备首次收到**：对该设备仍是首次处理，不能因别的设备收过就跨设备丢弃。

Signal 不读取业务 ID。确认旧 counter 对应已缓存/已存消息时可再发 stanza ACK，未确认或 placeholder 待修复时可走修复/NACK；解密后的业务重复按门控发 receipt。唯一冲突阻止重复四元组插入，但仍可有 placeholder 替换、已有行更新；普通重复跳过新插入通知，却可触发 duplicate observer。这些响应不统称已读。

客户端这些机制分别约束本地处理。它们本身不定义服务端的幂等历史保留时间或跨设备 exactly-once 保证。

<a id="receipts-zh"></a>

### 12. 设备、用户与群回执怎样合并

#### 12.1 三种确认与两张明细表

| 事件 | 所说明的状态 |
| --- | --- |
| 服务端 ACK | 服务端接收该逻辑发送请求 |
| 设备相关 receipt | 对应目标设备的处理结果，经资格过滤后记入设备/用户层 |
| read / played | 对应用户阅读/播放阶段，经隐私等门控后合并 |

两张明细表的唯一键分别是：

```text
receipt_device UNIQUE(message_row_id, receipt_device_jid_row_id)
receipt_user   UNIQUE(message_row_id, receipt_user_jid_row_id)
```

设备表保留具体设备的一份最早有效回执时间和 `primary_device_version`，并非每台设备三个独立的 delivery/read/played 时间槽。三个阶段时间位于用户表：`receipt_timestamp`、`read_timestamp`、`played_timestamp`。同一个有效用户键下的多台设备不会变成多个已读人。

#### 12.2 阶段比较与时间合并

| 回执 | 送出消息相关内部值 | 处理要点 |
| --- | --- | --- |
| delivery / sender | `5` | 送达阶段 |
| read | `13` | 已读阶段 |
| played | `8` | 已播放阶段 |
| inactive | `15` | 独立分流 |
| read-self | `17` | 本人设备间阅读同步 |
| played-self | `18` | 本人设备间播放同步 |

正常阶段排名为 `4 → 5 → 13 → 8`，不是数值递增。普通用户记录的最高状态由 played 正时间、read 正时间、delivery 正时间依次决定，不能用 `Math.max(status)` 代替排名。

普通实时回执通过前置过滤后，三槽内存合并只接受正时间、保留较早值，更新一个槽不清其他槽。同级或较低事件可先转为 device-only：单聊比较主状态，多参与者比较该用户最高阶段。因此，迟到但更早的 read 时间不保证改写用户表，设备时间仍可单独更新。[历史导入的缓存/SQL 写入](#history-zh)有另外的边界。

普通 `t` 以秒传入再转毫秒，缺失可得到 0 并被有效性检查拒绝；多目标 participants 有单独的缺失时间 fallback。时间查询还可用更高阶段支持较低阶段：查 delivery 可依次取 delivery/read/played，查 read 可取 read/played；这不是补写缺失的数据库槽。

本人账号的某些 delivery 和 inactive 事件只更新设备，不应算作对方用户已送达。隐私和资格门控还可把 read/played 按对应条件作为 delivery 合并。

#### 12.3 单聊与群整体门槛

普通单聊的一次有效对方回执可以提升状态，不要求对方每台设备都回来。多接收者聚合读取该消息保存的 `recipient_count=N`，定义：

```text
P = played 用户数
R = read 或 played 用户数
D = delivery、read 或 played 用户数

F = 8   if P >= N
    13  else if R >= N
    5   else if D >= N
    4   otherwise
```

例如 `N=2`，A 有手机和电脑、B 有手机。A 两台设备都回 read，用户层仍只增加 A 一人；B 的有效 read 到达后才满足两人已读。更高阶段可满足较低门槛，但不会凭空生成所有较低阶段时间。

#### 12.4 本人阅读同步与 receipt 批量

`read-self` 更新本人接收消息的阅读范围，不等于对方 read13；接收消息的内部状态还可能使用 `16/10` 等独立映射。发送端结合 force、隐私、聊天资格和有效关联设备记录选择 read 或 read-self，有设备记录也不代表设备在线。

read receipt 可以把首个消息 ID 放在外层 `receipt.id`，其余 ID 放入 `list/item`。这是多条逻辑消息批量，不是设备计数。`played-self` 收到本人其他设备的同步后，有防回声分支阻止再次沿同一路发送；本地状态更新仍可能已经发生。

<a id="regression-zh"></a>

### 13. recipient_count、乱序与界面回退

#### 13.1 N 如何形成

| 阶段 | N 的来源或更新 |
| --- | --- |
| 普通群消息插入前 | 参与者目录快照计数 − 1 + 特殊 recipient 修正；此群分支不先检查本人是否在目录 |
| 复制对象 | 可拷贝已有 N |
| 首次允许的 server ACK | 根据 ACK count 和对应分支修正 N |
| 数据库读回 | 还原保存的 `recipient_count` 列 |
| ChatTransfer 普通群文本导入 | 按目标端插入时目录计算，详见下一节 |

普通 server ACK 路径在现有状态排名已经达到 `4` 时，先于 N 更新返回；读回也不重新枚举群成员。当前群成员或关联设备变化不能直接当作已送达/已读旧消息 N 被更新的依据。Status 受众、广播 clone 和专用业务有各自计数路径，不应套用普通群公式。

插入前处理也有例外：已发对象满足特定的保留现有计数标志时保留 N，部分消息类型跳过该计数处理。普通 ChatTransfer 群文本新对象不满足这一保留分支，见[下一节](#history-zh)。

#### 13.2 哪些前提下乱序不会降低整体状态

在固定正 N、有效用户键稳定、三槽用户记录/容器未被删除或替换、初始主状态与明细一致且顺序处理的条件下，各用户最高阶段、P/R/D 都不会因迟到低级回执减少；聚合结果也不会下降。

- 用户已 read 后到 delivery：转 device-only，不清 read。
- 同批 read、delivery 都被旧快照放行：用户 upsert 在聚合前完成，delivery 仍不清 read。
- 同一用户另一设备重复：仍用同一有效用户键，不多算一人。
- N 单独减少：放宽门槛，不会使上述公式下降。

这些前提不构成任意并发调度的串行保证。N 增大、明细缺失/替换、身份映射改变有效容器，或主状态被独立恢复后，需重新检查主状态和门槛是否一致。

#### 13.3 允许写低的条件

聚合器计算结果低于旧状态时，会记录 `statusDowngrade`，但该分支仍继续写入；普通消息 status setter 也不以历史最高值阻止赋值。

例如旧主状态 `13`、`N=2`，当前明细只有 A 已读；B 的有效 delivery 进入重算后，`P=0,R=1,D=2`，结果为 `5`，允许 `13→5`。必要条件是旧主状态已高于当前 N/明细支持的阶段；如果 A、B 明细本来都 read，A 迟到的 delivery 不会造出这个例子。

该聚合分支没有额外的 N>0 守卫，N<=0 时到达计算会先满足 P>=N 而选择 8。解释具体消息仍需看其实际 N 生产条件，不能把非正边界当作正常已读语义。

#### 13.4 PN/LID 与设备清理

SQL 不自动合并 PN/LID。单聊有 canonical/alternate 处理，普通群不能直接套用；用户数变化需检查映射和实际保留的记录。

身份清理与用户已读清除也应分开：

| 清理分支 | 实际影响 |
| --- | --- |
| 指定 companion 身份清理 | 删除匹配非主设备、且设备回执时间为 NULL 的行，清相关设备缓存 |
| 指定 primary 身份清理 | 以 NULL 主设备时间消息与已有用户明细消息求交，删匹配主设备行、清设备缓存并增加 version |
| 撤回/存储重置 | 可另行删除用户回执或失效用户缓存，属于其他入口 |

前两种回执清理不删用户 read/played，也不改 N 或立即重算群状态。

#### 13.5 较低状态怎样显示

普通气泡按当前主状态绑定：4 为单勾，5 为普通色双勾，13/8 为已读色双勾；相同 drawable 仍重设 tint，缓存不保留最高 status。因此 `13→5` 可变色，`5→4` 可变形。

主状态先写内存再写 SQL，影响行数非 1 的诊断不回滚该内存值。详情页另读用户时间，主状态下降不表示 read 时间已清空；其他 UI 应按各自消费者处理。

<a id="history-zh"></a>

### 14. ChatTransfer、历史明细与孤立回执

#### 14.1 READ 主状态与空回执可以一起导出

ChatTransfer protobuf 分支分别导出主 status 和 `userReceipt`，回执窗口由配置控制；整库备份不套此字段保留逻辑。

配置 `22701` 开启时，每个 chunk 建立参数时计算：

```text
receiptCutoffTimeMs = System.currentTimeMillis() - 7,776,000,000
7,776,000,000 ms = 90 天

message.timestamp <= receiptCutoffTimeMs:
    不导出该消息的 userReceipt
```

比较的是**消息时间**，不是回执时间。主状态仍将内部 `13/8` 独立映射为 proto `READ/PLAYED`（4/5）；fresh builder 默认空 userReceipt，所以源明细完整也可导出高状态和空明细。外层不统一排除 90 天旧消息，仍可用、未到期或被保留且满足聊天条件的消息可进入此分支；配置关闭时截止值默认 0。

通过窗口的三个时间槽来自缓存或 message ID 批量读取。预取失败可回退逐消息读取，但单条读取错误与整条消息跳过另有分支，并非任意异常都清空回执。

#### 14.2 导入恢复主状态，却按目标目录赋 N

普通群文本新对象在主 status 为 0 时恢复 proto READ/PLAYED 为内部 13/8。proto 不带历史 `recipient_count`，该对象也不满足保留 N 的 outgoing 特例，因此插入前按目标活动目录算 N（无特殊 recipient 时为人数减 1），不按 userReceipt 条数，也不校正高主状态。

聊天不存在时才补导有效 participant，已有聊天沿用目录；成员逐项 update、不先清组。因此 N 不保证等于导出列表长度减 1，也不保证等于消息原发送人数。

例如目标目录有 6 人含本人，恢复 status13，导入 3 名已读用户：可以保存为 `status=13,N=5,R=3`。非空明细也可以不足以支持原主状态。

#### 14.3 消息入库与明细导入的顺序

主消息 INSERT 的单消息事务标记成功并关闭返回后，才导入 from_me 的 userReceipt；本方法不将全部主消息和明细包成批事务。

用户明细导入先在缓存中逐槽合并较早正时间；只要任一槽变化，SQL 就写入本次传入的全部正时间，而非逐槽读取合并后的缓存值。因此数据库的某个时间可能变晚、与缓存不同；非正输入不清既有行的旧槽。此步骤不重算主 status，空列表执行 0 次，人数不足 N 也不在此拒绝高主状态。

此处直接 INSERT，失败可记录异常后继续明细块；执行明细循环既不证明插入成功，也不表示临时对象的 N 已合入既有行。

#### 14.4 孤立回执的回放条件

消息尚未找到的回执可保存在 `receipt_orphaned`，其唯一键包含：

```text
(chat_row_id, from_me, key_id,
 receipt_device_jid_row_id, receipt_recipient_jid_row_id, status)
```

timestamp 不在该键中。回放按消息 Key 查询、按 `_id ASC` 处理，不按阶段排序，也没有统一的 90 天消息年龄 WHERE。进入普通群聚合前，recipient 须为空或匹配消息指定的 recipient，并通过身份、正时间和消息资格检查；隐私设置还可把 read/played 转为 delivery，特殊回执另行分流。正常完成后按 Key 删除 orphan 行，不自动留到明细导入完再回放。

唯一索引的效果仍受各列 NULL/default 约束影响，不能仅凭这个字段列表推断缺失 recipient 的所有输入都只保留一条。

直接导入与登记回放的插入/更新入口不同。**插入完成不能代替回放任务的实际排队及资格条件**。

排队和提交也不同：事务 `setSuccessful` 仅标记成功，`close/endTransaction` 才结束事务；独立 Handler 投递不是 after-commit callback。已排队任务何时看到数据，仍受外层事务、数据库锁和线程调度影响。

#### 14.5 回放先后与回退结果

| 条件 | 结果 |
| --- | --- |
| 明细先完整导入，固定 N 且状态一致 | 同级/较低用户事件转 device-only，顺序本身不使状态下降 |
| 明细循环结束，但窗口或其他条件使明细仍不足 N | 后续新的有效用户事件仍可能触发较低聚合结果 |
| 独立回放已排队，且在明细补齐前实际处理 | 可能按不足明细重算；之后明细导入自身不恢复主 status |
| 没有有效事件进入群重算 | 高主状态与不足明细可以继续共存，不会仅因空列表立即下降 |

例：旧消息导出时省略全部回执；导入恢复 13，目录含本人共 3 人使 N=2，用户明细为空。随后 A 的有效回执仍按 read 阶段被接受并进入重算，`R=1,D=1`，结果为 4，可发生 `13→4`。若 N=1，这条 read 已满足已读门槛；若明细被补充或事件阶段/资格改变，应按实际数据重算，不能直接套用本例。

无明细导出的回退条件不依赖抢跑；讨论完整明细被回放抢先处理，则须同时有独立任务已排队、主行可见、用户记录未齐和合格事件。

<a id="walkthrough-zh"></a>

### 15. 一次多设备发送的完整例子

| 步骤 | 状态变化 |
| --- | --- |
| 1. A0 向 B 发送 M | 保存 B0/B1/A1 为目标，排除 A0；对方使用各自 Message 密文，本人副本使用 DeviceSent 密文 |
| 2. ACK 丢失后按条件重执行 | 服务端可能已接收；断线清理使 Future 失败（可由 watchdog 触发），Job 再判断重试。恢复 ready 且通过业务门控、到达加密阶段才重加密，仍用 ID M |
| 3. B0 再次收到 M | 新 counter 解密成功后，仍按业务消息身份去重并按门控确认 |
| 4. 后续 ACK 的非空 phash 失配 | 独立同步发现新增 B2，验证并登记有效差集，只向新增目标补发 M |
| 5. B0/B1 回 read | 合入同一有效 B 用户记录，不算两人已读 |

Job 重执行、差集补发和回执分别维护键、计数与条件；同 ID 不要求密文、网络帧或目标集合相同。

<a id="related-zh"></a>

### 16. 相关主题

- [Protocol Tree Node](../protocol/protocol-tree-node.md#zh)：message、participants、receipt 的二进制树编码。
- [Noise 握手与传输](../protocol/noise.md#zh)：连接认证、传输密钥与加密层边界。
- [axolotl.db](../messaging/storage/axolotl.md#zh)：会话地址、预密钥、Sender Key 与处理队列。
- [msgstore.db](../messaging/storage/msgstore.md#receipts-zh)：消息身份、主状态和用户/设备回执表。
- [本地数据库与存储](../messaging/local-storage.md#zh)：缓存、事务、WAL 与一致性。
- [语音与视频通话目录](../calls/README.md#zh)：通话信令、设备接听与媒体链路，与消息分发分别处理。

[返回多设备目录](README.md#zh) · [English](#en)

<a id="en"></a>

## English

WhatsApp separates account, device, session, and message state. Sending selects targets, encrypts with their independent sessions, and assembles routable nodes; own-device copies also retain the original conversation.

The article distinguishes **the current directory, a message's target snapshot, and a resend difference set**, plus server, device, user, and aggregate group receipts, to explain device changes, retries, deduplication, and history-transfer state.

### Reading Guide

| Reading stage | Sections |
| --- | --- |
| Devices and distribution | [Identities](#identities-en) · [Directory](#directory-en) · [Targets](#targets-en) · [Pairwise encryption](#pairwise-en) · [Group encryption](#groups-en) |
| Device changes | [Changes and sessions](#device-changes-en) · [Hashes and resends](#hashes-en) |
| Send reliability | [Jobs/retries](#queue-en) · [Connections/timeouts](#connection-en) · [Ciphertext reuse](#ciphertext-en) · [Deduplication](#deduplication-en) |
| State and transfer | [Receipts](#receipts-en) · [Counts/regression](#regression-en) · [ChatTransfer](#history-en) |
| Putting it together | [Complete example](#walkthrough-en) · [Related topics](#related-en) |

<a id="identities-en"></a>

### 1. Accounts, Devices, and Session Addresses

| Object | Role | Dimensions to distinguish |
| --- | --- | --- |
| User JID | Identifies an account or conversation participant | PN and LID are different account namespaces |
| DeviceJid | Identifies one device of a user | User identity, account domain, and device ID |
| Current-device address | Determines whether a candidate is this device | Its PN address, LID address, and actual registered device number |
| Signal session address | Locates pairwise cryptographic state | Account, account type, device, scope, and type |
| Message identity | Locates a logical message | Conversation, direction, and ID; persistence also has a sender column |

The primary device normally uses device ID `0`; companions have their own device numbers. A companion can itself be the sender, so excluding it requires its actual nonzero device number. `registration_device_id` and the local Signal registration ID belong to separate state and cannot substitute for each other.

PN addresses relate to phone-number identity; LID uses another account identifier. Mapping can associate them, but a database does not automatically merge distinct JIDs. Device-address conversion must retain the device ID.

Pairwise sessions use a five-dimensional lookup key:

```text
(recipient_account_id, recipient_account_type,
 device_id, session_scope, session_type)
```

Session records contain ratchet, root/chain state, skipped message keys, and pending prekey state. Each device's state advances independently. Group Sender Keys additionally use dimensions such as the group, actual sender device, and bucket. See [axolotl.db sessions](../messaging/storage/axolotl.md#sessions-en) for fields and storage.

<a id="directory-en"></a>

### 2. Obtaining the Device Directory

A directory query uses a `usync` IQ with the following logical structure:

```text
iq xmlns=usync type=get
  usync mode=... context=... sid=... index=... last=...
    query
      devices version=2
    list
      user jid=targetAccount
```

The response's `devices/device-list/device` supplies device numbers and `key-index`. The `key-index-list` also has byte content, `ts`, and `expected_ts`; related branches handle LID and hosted-device information. See [Protocol Tree Node](../protocol/protocol-tree-node.md#en) for binary encoding.

Device `add/remove` notifications also update the directory. Processing compares notification freshness, validates relevant ADV index material, applies the delta, and checks `device_hash` or `device_lid_hash`. Validation or hash failures schedule another synchronization; older notifications can be ignored.

A device notification carrying `offline` may first register its account in `pending_users_to_sync_device`. This is pending state: scheduling a task, completing synchronization, and a send job using the refreshed directory are three different moments. Accounts already being synchronized are deduplicated to avoid scheduling the same work repeatedly.

The directory identifies devices; prekeys and existing sessions enable encryption. Discovery does not create a session, and sends need not query the full directory each time.

<a id="targets-en"></a>

### 3. Target Selection and Excluding the Current Sender

#### 3.1 Ordinary Target Sets

Let `A0` be the current sender, `A1` another device of the same account, and `B0/B1` the recipient's primary and companion devices. Ordinary full distribution can select:

```text
Recipient devices: {B0, B1}
Own synchronization devices: {A1}
Logical targets: {B0, B1, A1}
```

Message conditions further affect selection:

| Condition | Effect on the set |
| --- | --- |
| Ordinary remote account | Obtain its directory devices and include its primary device |
| Own copy permitted | Select eligible own companions in the appropriate account namespace |
| Own copy suppressed | Skip adding own companion devices |
| Primary-only message | Use the primary-device branch rather than the ordinary full-device set |
| Explicit device targets | Intersect with the computed or recorded allowed set |
| Self-chat and certain messages | May have an explicit own-device fallback |
| Special accounts, bots, guests, or hosted devices | Apply their eligibility and address conditions |

These conditions operate on different sets; they do not form a single universal multi-device switch.

#### 3.2 Identifying This Device Precisely

The current-device test compares both the current PN DeviceJid and LID DeviceJid:

```text
isCurrentDevice(d) =
    d == currentPnDevice
    OR d == currentLidDevice
```

Own-directory refresh skips both addresses, and eligible-companion reads also filter logged-out records. Group targets and some expansion paths then remove matching devices. Account and actual device number are compared, retaining other own devices and same-numbered remote devices; neither `isPrimary` nor account ownership replaces this test.

Exclusion occurs in directory/set processing, followed by allowed-set intersection at encryption. Not every message uses one final `remove(current)`.

#### 3.3 Current Directory Versus Message Target Snapshot

Stored messages normally read their own target snapshot. Explicit targets and batch-encryption inputs still intersect with allowed sets, and an identity-invalidated target can be dropped. An older message does not automatically adopt the whole latest directory, but [synchronization/resend jobs](#hashes-en) can validate differences, register new targets, and resend specifically to them.

<a id="pairwise-en"></a>

### 4. Encrypting Separately for Each Device

#### 4.1 Session Preparation and Ciphertext Types

Send jobs check session requirements and can fetch missing prekeys in bulk. Existing sessions use current ratchet state; ciphertext carrying prekey establishment material uses `pkmsg`, while ordinary session ciphertext uses `msg`. Pending prekey state can span multiple sends, so `pkmsg` is not limited to the first business message.

Each target first receives its corresponding business Message protobuf. It is serialized, business padding is applied, and the target's session encrypts it:

```text
C_B0 = SignalEncrypt(Session[A0,B0], Message)
C_B1 = SignalEncrypt(Session[A0,B1], Message)
C_A1 = SignalEncrypt(Session[A0,A1], DeviceSent(destination=B, Message))
```

These operations advance three corresponding send chains. `C_B0` cannot be reused as the ciphertext for `B1` or `A1`.

#### 4.2 Preserving the Original Conversation on Own Devices

An own-companion copy can use `Message.deviceSentMessage`:

| Field | Number | Contents |
| --- | --- | --- |
| `deviceSentMessage` | 31 in outer Message | Own-sent message wrapper |
| `destinationJid` | 1 inside wrapper | Original conversation destination |
| `message` | 2 inside wrapper | Original business Message |
| `phash` | 3 inside wrapper | Conditional digest of the related participant set |

The wrapper is end-to-end encrypted and moves `messageContextInfo` outward. `destinationJid` retains the conversation; outer DeviceJid selects the device, so own and remote copies can have different protobufs. Reception checks own PN/LID identity, a noncurrent source device, field presence, and destination consistency before unwrapping and retaining the original ID.

#### 4.3 Assembling Multiple Ciphertexts

Per-device results can form a `Map<DeviceJid, ciphertext>` and then one logical message node:

```text
message to=B id=M
  participants
    to jid=B0
      enc v=2 type=msg or pkmsg : C_B0
    to jid=B1
      enc v=2 type=msg or pkmsg : C_B1
    to jid=A1
      enc v=2 type=msg or pkmsg : C_A1
```

This schematic tree uses [binary node encoding](../protocol/protocol-tree-node.md#en), with `enc` as bytes; other branches can use top-level `enc` or additional nodes. Device encryptions, messages, socket writes, and TCP segments have different counts. `enc v` also differs from Signal's internal version.

~~~mermaid
sequenceDiagram
    participant A as A0 sender
    participant S as Server
    participant B0 as B0 recipient primary
    participant B1 as B1 recipient companion
    participant A1 as A1 own other device
    A->>A: Select targets and encrypt separately
    A->>S: Message M with multiple target enc nodes
    S->>B0: Ciphertext for B0
    S->>B1: Ciphertext for B1
    S->>A1: DeviceSent ciphertext for A1
    B0->>B0: Decrypt with its corresponding session
    B1->>B1: Decrypt with its corresponding session
    A1->>A1: Decrypt and restore original conversation
~~~

The server forwards each ciphertext to its target device. Message end-to-end encryption and [connection-level Noise protection](../protocol/noise.md#en-layers) have separate responsibilities.

<a id="groups-en"></a>

### 5. Group Bodies and Sender Key Distribution

Ordinary group sending separates body encryption from key distribution:

| Part | Encryption and addressing |
| --- | --- |
| Group body | The actual sender device's Sender Key chain in the group namespace produces one `skmsg` |
| Sender Key Distribution Message (SKDM) | A business Message carries it; target-device pairwise sessions encrypt it as `msg/pkmsg` when needed |

SKDM contains the key ID, iteration, chain key, and signing public key. The business `senderKeyDistributionMessage` additionally carries groupId and raw SKDM bytes. The send chain derives per-message encryption material. The body is signed with the corresponding Sender Key record's signing private key; SKDM carries the signing public key.

```text
message to=Group id=M
  enc type=skmsg : group body ciphertext
  participants
    to jid=deviceNeedingKey1
      enc type=msg or pkmsg : SKDM ciphertext for that device
    to jid=deviceNeedingKey2
      enc type=msg or pkmsg : SKDM ciphertext for that device
```

SKDM targets use group-device state intersected with the message snapshot. Devices holding the corresponding state need no SKDM with every body. Later sends can distribute it to new devices or after missing state or reset flags.

When `A0` and `A1` send separately, each has its own group send chain, iteration, and signing state. They cannot share one fixed account-wide group AES key or directly share a send counter.

There is also a conditional pre-send `force-one-one` branch. With conditions such as a group destination, no specific participant/explicit targets, and the relevant hash-mismatch policy enabled, a current participant-set hash differing from the job's saved hash can switch body encryption to per-device sessions. This occurs before transmission and requires neither a server ACK nor a decryption retry receipt. One-time, primary/companion, and specialized ratchet branches retain their own conditions.

<a id="device-changes-en"></a>

### 6. Device Changes and Session Updates

#### 6.1 A Key-Index Change Is Also a Replacement

Directory differences compare a device-address-to-key-index mapping:

```text
added   = new addresses + same addresses with changed key-index
removed = deleted addresses + same addresses with changed key-index
```

#### 6.2 Immediate Deletion and Retained Sessions

Removal branches on pending work and configuration gates:

| Case | Action |
| --- | --- |
| Immediate cleanup required | Lock the address and delete its session and identity |
| Pure removal with relevant work and other conditions | Register `dead_devices`, retain the session temporarily, and delete the identity |
| Same address with a changed key-index | Outside the pure-removal set; reset session/identity |

The relevant session deletion uses normalized account, account type, and device ID without scope/type predicates. It can therefore clear multiple scope/type rows and associated caches for that address, without automatically covering unrelated PN/LID addresses.

`dead_devices` waits for work/in-flight gates. A returning device loses its marker and retains state; an absent device loses its session, identity, and sender-device Sender Key. Pending work can trigger checks every `2000 ms`; poll index `15` can still retain registration, rather than forcing deletion after 30 seconds.

#### 6.3 Group-Key Distribution State

A new group device receives a record with `sent_sender_key=false`; a later send distributes SKDM as needed. Removal can delete the relevant sender key and reset whole-group or bucket distribution flags depending on conditions such as whether the device had received a Sender Key. Examples include `sent_sender_key=0` and `sent_sender_key_bucket=NULL`.

Directory updates, session cleanup, and Sender Key resets can run as separate asynchronous tasks. Receiving a notification does not mean they all complete simultaneously. See [axolotl.db](../messaging/storage/axolotl.md#en) for storage.

<a id="hashes-en"></a>

### 7. Directory Hashes, Message phash, and Difference Resends

| Digest | Set described |
| --- | --- |
| `device_hash` / `device_lid_hash` | An account's device directory in the corresponding address representation |
| Message-level `phash` | The message participant/device set chosen by that calling path |
| `phash` inside DeviceSent | A conditional digest of the related set for an own-device wrapper |

The primitive formats JIDs for its branch, sorts by Java string order, concatenates, hashes with SHA-256, takes the first 6 bytes, and uses standard Base64 with a `2:` / `3:` prefix. Callers select input sets and handle deduplication; ADV/Signal identity validation remains necessary.

When an ordinary server ACK carries a nonempty `phash`, the relevant branch compares it with a local digest for the message. Local inputs can include message targets, the current device, and conditional guests. **Including the current device in a hash does not make it a send target.**

A mismatch can schedule `SyncDeviceAndResendMessageJob`:

1. Project original targets and the relevant own address into accounts, then initiate required directory/key synchronization.
2. Once results and requirements permit, reread the original message and its target set.
3. Recompute eligible candidates and calculate `newTargets = candidates − originalTargets`.
4. Filter the difference by PN/LID, hosted, bot, version, message-type, and other eligibility rules.
5. Register new targets and submit the original message ID with the explicit difference set.

This does not resend to every original device or scan every old message on each directory notification. Ordinary resend jobs expire at construction time plus `3600000 ms`; edited-message branches have separate settings. Missing messages, terminating business states, expiration, or logout can end the task.

<a id="queue-en"></a>

### 8. Send Jobs, Retry Budgets, and Expiration

#### 8.1 Local Handoff and Actual Execution

Ordinary internal sending prepares the business protobuf, then creates and enqueues a `SendE2EMessageJob`. Local handoff callbacks also occur in dropping branches, so they prove neither transmission nor server acknowledgment.

The queue uses a worker pool with per-groupId exclusion; different groups can run concurrently. Business retries can use `_retry` groups. Job deduplication, target in-flight state, and group exclusion remain separate.

Jobs failing their readiness check remain pending without executing or consuming attempts. Provider notifications, enqueueing, and group release wake scanning; there is no universal fixed scan interval.

#### 8.2 Additional Waiting After Exceptions

After a retryable checked Exception, a worker can execute the job again if it is still judged ready: extra queue backoff is `0`. Otherwise it returns to pending for an event. This `0` excludes encryption, ACK waiting, thread scheduling, and reconnection time.

Sending also has a configurable artificial delay: sleep a fixed `d` milliseconds with probability `p`. It does not grow with the failure count or define connection backoff. `RuntimeException` follows a separate outward-throwing path, and `Error` is not handled by the ordinary checked-exception retry loop.

#### 8.3 Four Different Counters

| Mechanism | Conditions and limit |
| --- | --- |
| Ordinary Send / Sync Job execution | Default maximum `100` executions, including the first; requirement waiting does not count as execution |
| Per-target encryption failure | Accumulated per target within the job; counts `1/2/3` permit another attempt; `>3` (reaching 4) stops retrying this exception |
| Receiver decryption retry receipt | Ordinary direct messages reject `count>4`; the send job separately checks business `retryCount>4` |
| Independent unacknowledged-stanza replay | Default registration starts at 1 and continues only while `count<3`, normally at most 3 registrations; some call settings override it |

The transient 100-execution counter restarts from 0 after process restoration. Failures are not individually checkpointed as attempt/nextRun, and other serializable counters restore their stored snapshot. Persistence also depends on message category, so 100 is not a permanent across-restarts limit.

#### 8.4 Targeted Repair After Decryption Failure

A retry receipt locates a sent message by its original ID and identifies the requesting device. The ordinary direct-message branch checks a `registration` of exactly 4 bytes, retry version (default 1), count (default 1), message existence/type/time, and original-target eligibility. Messages older than adjusted time minus 60 days, ordinary requesters outside the original targets, and devices already marked delivered have rejection branches.

When keys are attached, identity and signed-prekey are required; a one-time key can be absent. `offline=true` additionally requires an existing session with matching remote registration ID. Identity, signature, and trust checks still apply. Registration or Alice base-key changes can require session reconstruction; repeated requests retaining the same base key can require a new session with a different base key.

Permitted repair reconstructs the business content, retains the logical ID, targets the requester, and enters encryption through a send job. Groups/broadcasts have their own current/historical-device, membership, delivery, and primary/companion gates; they do not share one ordinary direct-message set rule.

#### 8.5 Expiration Is a Separate Boundary

The ordinary preparation path defaults to an expiration timestamp of anchor plus `86400000 ms` (24 hours). A positive explicit expiration timestamp is clamped to the minimum of itself and that default. A resend can use the original message timestamp as its anchor under the relevant configuration, rather than always receiving another day from the resend time.

Send handles expiry at `now>=expireTimeMs`. Sync's positive-expiration check uses `now>expirationMs`, with `0` disabling that Sync check. Expiration does not provide a universal wakeup timer; a job blocked on other requirements may wait for an event before being retrieved and ended.

<a id="connection-en"></a>

### 9. Connection Recovery, ACK Timeouts, and Queue Wakeup

#### 9.1 Connection Backoff and Message Retry Are Separate

The reconnect scheduler uses a Fibonacci sequence, initially seeded with `1` and capped at `17280`:

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ... , 17280, 17280, ...
B = sequenceValue × 10000 ms
```

`B=0` submits a connection request immediately. For `B>0`, the requested wait is:

```text
delay = floor(B/2) + ((Random.nextLong() & Long.MAX_VALUE) % B)
```

| Nominal B | Requested wait with jitter |
| --- | --- |
| 10 seconds | 5000..14999 ms |
| Capped at 48 hours | 24 hours..72 hours minus 1 ms |

48 hours caps only the nominal value. Deduplicated scheduling does not advance the sequence, and scheduling count differs from TCP failure count. Successful connection resets it. `force` clears the pending deadline, while `reset` resets the sequence; network, push, and foreground paths can combine them.

The relevant login-failure-type-6 limiting branch replaces results above `900000 ms` with `899970..900029 ms`. The offset is in milliseconds, and the rule does not apply to every disconnect. Reconnection does not share the message job's 100-execution budget.

#### 9.2 The Ordinary Message's 45-Second Watchdog

The ACK Future waits without a numeric timeout argument. When ConnectionThread handles a normal Writer-return notification, it schedules a `45000 ms` watchdog. At the deadline it rereads the message, possibly from cache. If still present, below server-received rank `4`, and not excluded state `20`, it requests forced disconnection. Cleanup fails unfinished Futures, after which the Job evaluates retry.

Timing starts at notification handling; Writer return does not prove server reception. New notifications add timers without extending an earlier deadline, and messages already at the state threshold are skipped. Failure can clear multiple timers: this is connection-level monitoring, not an independent 45-second TimeoutException per Future.

When early-diagnostic configuration 38734 is enabled, the same threshold is checked at `22500 ms` for diagnostics without directly disconnecting. Other timeouts are:

| Mechanism | Threshold/start | Action |
| --- | --- | --- |
| Active-mode monitoring | 32 seconds after submitting the active request or enqueueing the corresponding Writer message | Request disconnect under its conditions |
| Ping health monitoring | Configured threshold clamped to 8..32 seconds | Notify disconnection when connected/ready and other gates hold |
| A dedicated IQ Future entry | 32 seconds from callback registration | Fail its Future; this timer does not itself disconnect |
| General IQ / tracked stanza | Positive caller-supplied timeout | Invoke its timeout callback; branches passing 0 have no such timer |
| One socket connect | 30 seconds | Limit that connection attempt, not total DNS/handshake/login time |
| Post-login long blocking read | 3 minutes when `companion_registration_state` is 2..6 or 10..17; 17 minutes in the other corresponding login path | Limit one blocking read, not one message's ACK |

#### 9.3 How Jobs Become Executable Again

```text
Disconnect → connection requirement false, ACK Future fails → Job checks exception/readiness to retry or return to pending
Recovery → provider event → queue rechecks group/job readiness → dispatch eligible jobs
```

Ready differs from `socket.isConnected()`. State changes, notifications, scanning, and execution are separate steps; recovery does not mean jobs have sent.

A synchronization/resend job can additionally require `OFFLINE_RESUME_COMPLETED`: after offline complete arrives, outstanding offline-processing work must reach zero before that state is entered. Merely receiving complete is insufficient, and this requirement should not be imposed on every ordinary send job.

<a id="ciphertext-en"></a>

### 10. Reusing Ciphertext Versus Encrypting Again

| Layer performing the resend | Business ID | End-to-end ciphertext |
| --- | --- | --- |
| Ordinary Job reexecution, decryption repair, or device-difference resend reaches encryption | Original ID retained | Encrypt again for its targets using current session/group state |
| Writer replays a retained Message/Node | ID inside that node | Can retain existing `enc` bytes |
| Transport writes a node again | Same logical node | Transport AEAD runs again with the current connection counter |

Jobs can reuse saved protobufs, padding inputs, and related material, but reaching encryption still generates ciphertext and advances chains using current state. Partial target success followed by another failure does not guarantee rollback of all state; the next same-ID ciphertext may differ.

Writer retention has explicit limits. Ordinary chat Android message type `8` is excluded from the generic offline queue while disconnected. A write exception can temporarily retain the original Message reference, but it is filtered if still disconnected when processed again.

A separate type `486` path accepting a preconstructed Node with `dropIfOffline=false` can retain it offline and replay it when readiness/callback conditions permit. Independent tracked-stanza management can also retain an original Message. These are not the ordinary chat Job's default retry mechanism, and their registration limits do not apply to every message.

E2E `enc` reuse does not reuse a complete network frame: each node write uses current connection keys and an incrementing counter for transport AEAD. [Noise](../protocol/noise.md#en) protects the connection; [TCP framing](../protocol/tcp-framing.md#en) defines boundaries.

<a id="deduplication-en"></a>

### 11. Lost ACKs, Same Message IDs, and Deduplication

#### 11.1 Different Layers Use Different Keys

| Layer | Identity or constraint |
| --- | --- |
| Business MessageKey | `chat + from_me + id` |
| Send Job concurrency deduplication | `jid + id + editVersion + participant`, excluding explicit device sets and ciphertext |
| Target in-flight state | Original message and target set under MessageKey |
| ACK callback | class, remote, id, participant, recipient, type, editVersion, extra attributes, optional child, category |
| Message-table unique index | `chat_row_id + from_me + key_id + sender_jid_row_id` |
| Signal replay checks | Counter and available skipped keys of the relevant session/ratchet chain |

A bare ID in another conversation or direction is a different business key. Edits, participants, and receipt types can also produce different job/ACK keys. Send Job concurrency deduplication excludes targets, so another same-key job with different targets can still be suppressed.

Job deduplication registers active process-local tasks, removes them on completion/cancellation, and rebuilds on restoration, rather than retaining completion history. Cache lookup commonly uses three business-key columns; the physical index adds sender. Both require sender normalization and collision handling.

#### 11.2 Which Wait Can a Late ACK Complete?

ACK callbacks register under a normalized logical stanza key. Registering the same key again can replace the current callback. An incoming ACK removes its matching entry before completing the callback.

| Ordering | Result |
| --- | --- |
| ACK consumed, with no later registration | A duplicate ACK finds no entry |
| Disconnect cleared the table; old ACK precedes new registration | No current callback matches |
| Retry registered the same key; old ACK arrives afterward | It can consume the current callback |
| Same ID but different participant/recipient/edit/type, etc. | ID alone cannot match it |

The key does not identify which locally encrypted attempt produced the ciphertext. A match therefore does not prove acknowledgment of the newest ciphertext bytes or delivery to all devices.

Edited NACKs have a limited fallback: when exact matching fails, `class=message`, and the ACK lacks edit but contains error, along with the other gates, it can consume the unique candidate matching after edit is removed. This is not arbitrary bare-ID matching.

Ordinary server-ACK state persistence has a separate guard: rereading a state already at rank `4` or higher returns before that ordinary state/server-time write. Callback consumption and state persistence are separate asynchronous paths.

The ordinary Send guard that exits for an already-server-acknowledged message is conditional too: business retryCount must be 0, explicit targets must be empty, state must have reached the threshold, and the relevant configuration/chat branch must hold. Server ACK does not unconditionally stop device resends with explicit targets or positive retryCount.

#### 11.3 Why Reception Still Needs Business Deduplication

Reception projects the source DeviceJid to user identity, then locates the message by conversation, direction, ID, and related fields. Own peer copies return to the original destination conversation. PN/LID mapping depends on directory and migration state.

At least three same-ID arrivals are possible:

1. **The original Signal ciphertext arrives again**: its counter may already be consumed, leading to old-counter processing; remaining skipped keys affect decryption.
2. **Fresh Signal ciphertext carries the same business ID**: a new counter may decrypt successfully, after which business-message deduplication is still necessary.
3. **Another device receives it for the first time**: this remains a first reception on that device, regardless of another device's history.

Signal does not read the business ID. A confirmed old-counter cached/stored message can receive another stanza ACK; unconfirmed duplicates or placeholders needing repair can take repair/NACK paths. Decrypted business duplicates can send receipts under their gates. Unique conflicts prevent duplicate-tuple insertion, while placeholder replacement or existing-row updates remain possible. Ordinary duplicates skip new-insert notification but can emit a duplicate observer event. These responses are not all read receipts.

These client mechanisms constrain local handling. They do not themselves define server idempotency-history retention or an across-device exactly-once guarantee.

<a id="receipts-en"></a>

### 12. Aggregating Device, User, and Group Receipts

#### 12.1 Three Confirmations and Two Detail Tables

| Event | Meaning |
| --- | --- |
| Server ACK | Server reception of the logical send request |
| Device-related receipt | A target device's result, admitted into device/user processing by eligibility checks |
| read / played | A user's reading/playback stage, merged after privacy and other gates |

The detail tables use separate unique keys:

```text
receipt_device UNIQUE(message_row_id, receipt_device_jid_row_id)
receipt_user   UNIQUE(message_row_id, receipt_user_jid_row_id)
```

The device table retains one earliest valid receipt timestamp and `primary_device_version` for a device, rather than three independent delivery/read/played timestamps per device. The user table has the three stage slots: `receipt_timestamp`, `read_timestamp`, and `played_timestamp`. Multiple devices under the same effective user key do not become multiple readers.

#### 12.2 Stage Comparison and Timestamp Merging

| Receipt | Related outbound internal value | Handling |
| --- | --- | --- |
| delivery / sender | `5` | Delivery stage |
| read | `13` | Read stage |
| played | `8` | Played stage |
| inactive | `15` | Separate branch |
| read-self | `17` | Reading synchronization between own devices |
| played-self | `18` | Playback synchronization between own devices |

Ordinary stage rank is `4 → 5 → 13 → 8`, not increasing numeric order. An ordinary user's highest stage is determined by a positive played timestamp, then read, then delivery. `Math.max(status)` cannot replace this ranking.

For ordinary live receipts admitted by the state filter, in-memory three-slot merging accepts only positive times and keeps earlier values without clearing other slots. Equal/lower events can first take device-only handling: direct chats compare main state, and multi-recipient messages compare the user's highest stage. An earlier late-read timestamp therefore need not rewrite the user table, while device time can update independently. [Historical import's cache/SQL writes](#history-en) have a separate boundary.

Ordinary `t` values arrive in seconds and convert to milliseconds; missing values can become 0 and fail validity checks. Multi-target participants have a separate missing-time fallback. Timestamp queries can also use a higher stage for a lower one: delivery can fall back through delivery/read/played, and read through read/played. This does not populate missing database slots.

Some own-account delivery and inactive events update only device state and should not count as delivery to the remote user. Privacy and eligibility gates can also merge read/played as delivery under their conditions.

#### 12.3 Direct Chats and Group Thresholds

One valid remote receipt can advance an ordinary direct chat; it need not wait for every recipient device. Multi-recipient aggregation reads the message's saved `recipient_count=N` and defines:

```text
P = users with played
R = users with read or played
D = users with delivery, read, or played

F = 8   if P >= N
    13  else if R >= N
    5   else if D >= N
    4   otherwise
```

For `N=2`, suppose A has a phone and computer and B has a phone. Both of A's devices reading still count as one user, A. A valid read from B is needed to satisfy two readers. Higher stages support lower thresholds without generating all lower-stage timestamps.

#### 12.4 Self-Read Synchronization and Receipt Batches

`read-self` updates reading ranges for the account's incoming messages rather than remote read13. Incoming state also has separate mappings such as `16/10`. The sender selects read versus read-self using force, privacy, chat eligibility, and valid companion records; a device record does not establish that it is online.

A read receipt can place the first ID in outer `receipt.id` and additional IDs in `list/item`. This batches logical messages, not devices. A received `played-self` synchronization has an anti-echo branch preventing transmission back along the same path, while local state may already have changed.

<a id="regression-en"></a>

### 13. recipient_count, Reordering, and UI Regression

#### 13.1 How N Is Produced

| Stage | Source or update of N |
| --- | --- |
| Before ordinary group insertion | Participant-directory snapshot count − 1 + special-recipient correction; this group branch does not first test self membership |
| Object copying | Can copy an existing N |
| First permitted server ACK | ACK count with branch-specific corrections |
| Database hydration | Restore the saved `recipient_count` column |
| ChatTransfer ordinary group-text import | Compute from the target directory at insertion, as described next |

The ordinary server-ACK path returns before updating N once state has reached rank `4`. Hydration does not enumerate current group membership again. Current member/device changes alone do not establish that an already-delivered/read old message's N has changed. Status audiences, broadcast clones, and specialized products have their own count paths.

Preinsert processing has exceptions too: an outgoing object satisfying the specific preserve-existing-count flag retains N, and some message types skip this count handling. A fresh ordinary ChatTransfer group-text object does not meet that preservation branch; see the [next section](#history-en).

#### 13.2 When Reordering Does Not Lower State

Assume fixed positive N, stable effective user keys, no removal/replacement of the ordinary three-slot records/container, initially consistent main state and details, and sequential processing. Late lower-stage receipts cannot reduce users' highest stages or P/R/D, so the aggregate cannot fall.

- Delivery after a user's read takes device-only handling without clearing read.
- Read and delivery admitted against an older batch snapshot are upserted before aggregation; delivery still preserves read.
- Another device of the same user uses the same effective user key, without adding another reader.
- Decreasing N alone relaxes the thresholds and cannot lower this formula.

These premises do not establish serialization under arbitrary concurrency. A larger N, missing/replaced details, changed identity mapping, or an independently restored main state requires checking consistency again.

#### 13.3 The Condition Allowing a Lower Write

When recomputation ranks below the old state, the aggregator records `statusDowngrade` but continues writing. The ordinary status setter does not block the assignment using a historical maximum either.

For example, let old main state be `13`, `N=2`, and only A currently have read details. A valid delivery from B enters recomputation: `P=0,R=1,D=2`, yielding `5`, so `13→5` is permitted. The necessary condition is an old main state above what current N/details support. If A and B both already have read records, A's late delivery cannot create this example.

The aggregate branch has no extra N>0 guard; N<=0 satisfies P>=N first and selects 8 when computation is reached. A specific message still requires examining its count producer. This edge case should not be treated as ordinary read semantics.

#### 13.4 PN/LID and Device Cleanup

SQL does not automatically merge PN/LID. Direct chats have canonical/alternate handling that ordinary groups cannot simply inherit. User-count changes require checking mappings and retained records.

Identity cleanup and clearing user-read state are different:

| Cleanup branch | Effect |
| --- | --- |
| Designated companion identity cleanup | Delete matching nonprimary device rows whose device-receipt timestamp is NULL; clear related device cache |
| Designated primary identity cleanup | Intersect messages with NULL primary-device time and existing user details; delete matching primary rows, clear device cache, increment version |
| Revocation/storage reset | Other entries can delete user receipts or invalidate user caches |

The first two receipt cleanups neither delete user read/played slots nor change N or immediately recompute group state.

#### 13.5 Displaying a Lower State

Ordinary bubbles bind current main state: 4 gives one check, 5 neutral double checks, and 13/8 read-colored double checks. Even the same drawable receives its tint again; cache does not preserve a maximum status. Thus `13→5` can change color and `5→4` shape.

Main state is written to memory before SQL; a diagnostic for an update affecting other than 1 row does not roll memory back. Details read user timestamps separately, so lower main state does not mean read time was cleared. Other UI must be assessed through its respective consumers.

<a id="history-en"></a>

### 14. ChatTransfer, Historical Details, and Orphan Receipts

#### 14.1 READ and Empty Receipts Can Be Exported Together

ChatTransfer's protobuf branch exports main status and `userReceipt` separately, with a configured receipt window. Whole-database backup does not inherit this field-retention rule.

With configuration `22701` enabled, each chunk constructs:

```text
receiptCutoffTimeMs = System.currentTimeMillis() - 7,776,000,000
7,776,000,000 ms = 90 days

message.timestamp <= receiptCutoffTimeMs:
    omit this message's userReceipt
```

This compares **message time**, not receipt time. Main state still independently maps internal `13/8` to proto `READ/PLAYED` (4/5). A fresh builder defaults userReceipt to empty, so complete source details can still yield high state and empty details. Outer selection does not universally exclude 90-day-old messages: available, unexpired or retained messages meeting conversation eligibility can reach this branch. Disabled configuration defaults the cutoff to 0.

The three timestamp slots passing the window load from cache or message-ID batches. Prefetch failures can fall back to per-message reads, but individual errors and whole-message skipping have separate paths; not every exception clears receipts.

#### 14.2 Import Restores Main State but Computes Target-Side N

A fresh ordinary group-text object restores proto READ/PLAYED to internal 13/8 when main state is 0. The proto lacks historical `recipient_count`, and this object does not meet the preserve-N outgoing exception. Preinsert therefore computes N from the target's active directory (size minus 1 without a special recipient), without counting userReceipt entries or reconciling high main state.

Valid participants are imported only for a missing conversation; existing conversations retain their directory. Import updates entries without clearing the group. N therefore need not equal the exported list length minus 1 or the original send-time count.

For example, a target directory of 6 including self, restored status13, and 3 imported readers can leave `status=13,N=5,R=3`. Nonempty details can still be insufficient for the old main state.

#### 14.3 Main Insertion Before Detail Import

The main INSERT's single-message transaction is marked successful and closed before returning to import from_me userReceipt entries. This method does not batch all main/detail operations in one transaction.

Detail import first merges earlier positive times into each cache slot. If any slot changes, SQL writes all positive timestamps supplied by this call, rather than reading each merged cache value. A database timestamp can therefore become later and differ from cache. Nonpositive inputs leave existing row slots untouched. This step does not recompute main status; an empty list performs 0 iterations, and a count below N does not reject high main state here.

This is a direct INSERT; failure can be logged before continuing into details. Running the detail loop proves neither insertion success nor merging the temporary object's N into an existing row.

#### 14.4 Conditions for Orphan Replay

Receipts whose message is not found can enter `receipt_orphaned`, whose unique key includes:

```text
(chat_row_id, from_me, key_id,
 receipt_device_jid_row_id, receipt_recipient_jid_row_id, status)
```

Timestamp is outside that key. Replay queries by message key and processes `_id ASC`, rather than stage order, without a universal 90-day message-age WHERE. Ordinary group aggregation requires recipient to be absent or match the message-specific recipient, plus identity, positive-time, and message-eligibility checks. Privacy settings can convert read/played to delivery, and special receipts take separate branches. Normal completion deletes that key's orphan rows without automatically retaining them for replay after detail import.

The unique index's effect also depends on each column's NULL/default constraints. Its field list alone does not establish that every input with a missing recipient retains exactly one row.

Direct import and replay-registering insertion/update entries differ. **Insertion completion cannot substitute for actual replay scheduling and eligibility conditions.**

Scheduling and commit also differ. Transaction `setSuccessful` marks success; `close/endTransaction` ends it. An independent Handler post is not an after-commit callback. Visibility to an already-queued task still depends on outer transactions, database locks, and scheduling.

#### 14.5 Ordering and Regression Outcomes

| Condition | Result |
| --- | --- |
| Details imported fully first, fixed N, consistent state | Equal/lower user events take device-only handling; order alone does not lower state |
| Detail loop finished but the window/other conditions leave fewer details than N | A later newly admitted user event can still compute a lower aggregate |
| Independent replay is queued and actually runs before details are complete | It may aggregate insufficient details; later detail import does not itself restore main status |
| No valid event enters group recomputation | High main state and insufficient details can coexist without immediate regression from an empty list alone |

For example, export omits an old message's receipts. Import restores 13, the directory has 3 members including self so N=2, and user details are empty. A valid receipt from A is still admitted as read and enters aggregation. With `R=1,D=1`, the result is 4, permitting `13→4`. If N=1, this read already satisfies the read threshold. Added details or changed event stage/eligibility require recalculation from the actual data; the example cannot be applied unchanged.

Regression with omitted export details does not require a race. Replay overtaking complete input details instead requires an independently queued task, a visible main row, incomplete user records, and an eligible event together.

<a id="walkthrough-en"></a>

### 15. A Complete Multi-Device Send Example

| Step | State change |
| --- | --- |
| 1. A0 sends M to B | Record B0/B1/A1 as targets, excluding A0; recipients get separate Message ciphertexts and own copies use DeviceSent ciphertext |
| 2. Conditional reexecution after ACK loss | The server may have accepted it; disconnect cleanup fails the Future, possibly triggered by the watchdog, before the Job evaluates retry. Encrypt again only when ready, business gates pass, and encryption is reached, retaining ID M |
| 3. B0 receives M again | After a fresh counter decrypts, business identity still deduplicates and acknowledgment follows its gates |
| 4. A later ACK's nonempty phash mismatches | Independent synchronization discovers new B2, validates/registers the eligible difference, and sends M only to added targets |
| 5. B0/B1 return read | Merge under one effective B user record, without counting two readers |

Job reexecution, difference resends, and receipts maintain separate keys, counters, and conditions. Same ID does not require identical ciphertext, network frames, or targets.

<a id="related-en"></a>

### 16. Related Topics

- [Protocol Tree Node](../protocol/protocol-tree-node.md#en): binary tree encoding for message, participants, and receipt nodes.
- [Noise Handshakes and Transport](../protocol/noise.md#en): connection authentication, transport keys, and encryption-layer boundaries.
- [axolotl.db](../messaging/storage/axolotl.md#en): session addresses, prekeys, Sender Keys, and processing queues.
- [msgstore.db](../messaging/storage/msgstore.md#receipts-en): message identity, main state, and user/device receipt tables.
- [Local Databases and Storage](../messaging/local-storage.md#en): caches, transactions, WAL, and consistency.
- [Voice and Video Calls](../calls/README.md#en): call signaling, answering-device selection, and media paths, handled separately from message distribution.

[Back to Multi-Device Topics](README.md#en) · [中文](#zh)
