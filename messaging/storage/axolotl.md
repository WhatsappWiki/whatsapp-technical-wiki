# axolotl.db · Keys, Sessions, and Queues · 密钥、会话与队列

[数据库目录](README.md#zh) · [存储总览](../local-storage.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 数据库职责与表目录

`axolotl.db` 保存 Signal 身份、预密钥、单聊加密会话、群聊发送者状态及相关处理队列。本页包含 15 张应用表及 20 个显式创建的索引；列级 `UNIQUE` 形成的 SQLite 自动索引另计。表和索引名称中的 `v33/v36` 是标识符的一部分。

| 表 | 职责 |
| --- | --- |
| `identities` | 本机身份密钥对、注册 ID、next IDs，以及远端身份公钥和验证元数据 |
| `prekeys` | Curve25519 一次性预密钥（OPK）及分发/上传状态 |
| `signed_prekeys` | 签名预密钥及历史记录 |
| `kyber_prekeys` | ML-KEM 预密钥，包括一次性与 last-resort |
| `sessions` | 按完整地址定位的双棘轮会话记录 |
| `sender_keys` | 普通群聊 Sender Key 状态 |
| `fast_ratchet_sender_keys` | 独立的 fast-ratchet 群状态 |
| `prekey_uploads` | 预密钥上传批次时间与类型元数据 |
| `message_base_key` | 按消息和对方设备定位的 `last_alice_base_key` |
| `pq_session_upgrade_device_state` | 设备级 PQ 会话升级状态 |
| `preacks` | `ptn` BLOB 队列 |
| `unordered_stanza_queue` | 未排序 stanza 的标识、载荷、解密元数据和处理信息 |
| `e2ee_stanza_queue` | E2EE stanza 的离线/重试计数、pkmsg/skmsg 标记与排序信息 |
| `chat_stanza_queue` | 聊天 stanza 的载荷、生成标记与排序/处理信息 |
| `stanza_sts` | 按 stanza 类型保存的 `sts` 值 |

BLOB、record、raw32/typed33、scope 和 bucket 的定义见[存储术语](../local-storage.md#terms-zh)。

### identities：本机身份与远端身份

本机行固定按 `(recipient_id, recipient_type, device_id) = (-1, 0, 0)` 选择。正常生成的 `public_key` 是 `0x05 || raw32`，即 typed33；`private_key` 是 raw32。`registration_id` 独立持久化，初始化范围为 `1..2147483646`，不会在每次发送或读取时重新随机，也不能额外压到 14 bit。

`next_prekey_id` 与 `next_kyber_prekey_id` 是两个独立游标，正常初值各为 `1..16777214`。它们不是 signed-prekey ID 游标。`mark_as_verified`、`mark_as_verified_action_seq` 与 `account_encryption_attestation_type` 是独立的身份元数据，不属于公私钥编码。

远端身份写入保存对方地址、公钥、时间和 attestation 等信息，不保存对方私钥，也不写本机的注册 ID 或 next IDs。本机与远端共表不代表每行都有私钥。

本机公私钥原始字节与注册 ID 使用独立缓存；注册 ID 的 `0` 是未缓存哨兵。身份缓存命中后仍需解码 typed 公钥，严格公钥检查继续生效。缓存可以在数据库资源关闭前发布，关闭失败不保证撤销缓存。不能把两个 getter 当成一个原子身份快照。

以下只查询结构元数据，不输出密钥字节：

```sql
SELECT recipient_id, recipient_type, device_id, registration_id,
       length(public_key) AS public_bytes,
       length(private_key) AS private_bytes,
       next_prekey_id, next_kyber_prekey_id
FROM identities
WHERE recipient_id = -1 AND recipient_type = 0 AND device_id = 0;
```

### 预密钥 record 的编码

`record` 是协议 protobuf BLOB；列类型没有把它简化成单独的 AES key/IV。字段编号与 wire 类型如下。

| protobuf 字段 | `prekeys` | `signed_prekeys` | `kyber_prekeys` |
| --- | --- | --- | --- |
| 1 | `uint32 id` | `uint32 id` | `uint32 id` |
| 2 | `bytes` typed33 公钥 | `bytes` typed33 公钥 | `bytes` KEM raw 公钥 |
| 3 | `bytes` raw32 私钥 | `bytes` raw32 私钥 | `bytes` KEM 展开私钥 |
| 4 | 无此字段 | `bytes` 签名，正常生成 64 字节 | `bytes` 本机 Signal identity 对 KEM 公钥的签名 |
| 5 | 无此字段 | `fixed64`，独立的毫秒时间值 | `fixed64`，毫秒时间值 |

signed 与 KEM 使用相同的五字段外层结构，但公私钥内容不同。KEM 公钥不加 Curve 的 `0x05` 类型前缀。ML-KEM-512/768/1024 的公钥长度分别为 `800/1184/1568` 字节，展开私钥分别为 `1632/2400/3168` 字节；存储层不应统一截成 32 字节，也不能只凭表名固定选择一种长度。

`signed_prekeys.timestamp` 是写入时应用调整时钟除以 1000 的秒值；record 字段 5 是独立的毫秒时间值，二者不是同一个字段或同一次取时。

### 预密钥选择、上传与消费

| 对象 | 选择与写入 | 消费和保留 |
| --- | --- | --- |
| 普通 OPK | 按 `prekey_id` 读取；待上传为 `sent_to_server=0 AND direct_distribution=0`，有限额但无排序保证；批量保存同时推进本机 `next_prekey_id`，空批次不推进 | pkmsg 明文回调正常返回且会话保存后消费；Curve last-resort `0xFFFFFF` 由上层跳过删除，底层按 ID 删除本身没有保护 |
| Signed prekey | 活跃记录先按 `_id DESC LIMIT 1`；仅当该行 SQL `prekey_id=0xFFFFFF`，再按 `prekey_id<8388607 ORDER BY prekey_id DESC LIMIT 1` 回退，回退无行保留首个 BLOB | 不在每条消息解密后立即删除；旧记录仍可按 ID 处理迟到的 pkmsg |
| KEM prekey | 按 `prekey_id` 读取；普通待上传另要求 `last_resort_key=0`；last-resort 可按标记和上传状态选择最新 `_id` | 删除谓词是 `prekey_id=? AND last_resort_key=0`；保留依据 SQL 标记，不是固定编号 |

Signed prekey 对外协议 ID 来自 record 的 `id`，SQL ID 用于选择，不能强行用 SQL 列替换。Signed ID 独立随机生成，不使用 `identities.next_prekey_id` 作为游标。

上传成功会写入 `sent_to_server` 和秒级 `upload_timestamp`，并记录 `prekey_uploads`。该历史表的经典上传路径使用默认 `key_type=0`，KEM 路径显式写 `1`；这个局部区分不能直接变成所有表的同名 `key_type` 的完整枚举。KEM 的 `last_resort_key` 位于 SQL 列，不在五字段 record 内。

### sessions：五维地址

| 维度 | 含义 |
| --- | --- |
| `recipient_account_id` | 对方账户字符串；Phone 地址可在 SQL 前映射到 LID |
| `recipient_account_type` | Signal 账户域；PN=`0`，LID=`1` |
| `device_id` | 对方设备编号 |
| `session_scope` | `DEFAULT=0`、`STATUS=1` |
| `session_type` | `REGULAR=0`、`PQ=1`、`STATELESS=2` |

唯一索引保留上述五维。地址转换不能丢掉 device、scope 或 type。`session_type=PQ` 与 record 内的 `sessionVersion` 是两个命名空间，不能据前者直接断言后者为某个固定值。消息库 LID 的 `jid.type=18` 也不能写入这里的 LID 域列。

`record` 外层 protobuf 的字段 1 为当前会话，字段 2 为重复的历史会话。当前和历史会话使用以下字段：

| 字段号 | 名称 | 类型或内容 |
| --- | --- | --- |
| 1 | `sessionVersion` | `uint32` |
| 2 | `localIdentityPublic` | `bytes` |
| 3 | `remoteIdentityPublic` | `bytes` |
| 4 | `rootKey` | `bytes` |
| 5 | `previousCounter` | `uint32` |
| 6 | `senderChain` | 发送链 |
| 7 | `receiverChains` | 重复的接收链 |
| 9 | `pendingPreKey` | 待完成的预密钥建链状态 |
| 10 | `remoteRegistrationId` | `uint32` |
| 11 | `localRegistrationId` | `uint32` |
| 12 | `needsRefresh` | `bool` |
| 13 | `aliceBaseKey` | `bytes` |
| 14 | `pendingKyberPreKey` | 待完成的 KEM 建链状态 |

此结构没有声明 field 8 `pendingKeyExchange`。链结构的字段 1/2 为 ratchet 公钥/发送方私钥，字段 3 为 `chainKey{1:index, 2:key}`，字段 4 为重复的乱序消息密钥 `messageKeys{1:index, 2:cipherKey, 3:macKey, 4:iv}`。完整 BLOB 同时包含根密钥、双向链、跳过的消息密钥、历史会话和 pending 状态；只保存当前 AES key/IV 会丢失重启与乱序处理能力。

### 会话的读取、更新、删除与缓存

读取先规范化地址，再查询缓存或 SQL。数据库缺行的结果也可以缓存为空会话，后续读取可能直接使用该缓存；返回给加密引擎的记录经过独立解析，不应共享协调层可变对象。过大或损坏的 record 有精确清理及空会话替换路径，不能把所有缺失都解释成应该重新生成本机身份。

保存按五维先 `UPDATE record`，未命中才 `INSERT`；只有插入写秒级 `timestamp`，后续 record 更新不刷新它。因此 `sessions.timestamp` 不是最后收发或最后解密时间。

精确删除使用五维；普通设备会话移除使用账户、域、设备三维，覆盖该设备的所有 scope/type。实现接口时必须保留两者的差别。

缓存清理使用 map、候选集合及其他集合协作；超过 1000 时只清候选项，某些账户类型不缓存。这不是严格的 1000 条 LRU 上限。锁路径可采用缓存 monitor 或地址相关条带锁，写入另配数据库事务。

保存顺序可以是外层事务开始 → 内层事务写入并关闭 → 替换缓存 → 外层事务完成。缓存不必在最终提交之后才发布；数据库失败、回调失败与内存回滚不能合并为一项保证。

### 群聊状态与 fast-ratchet

`sender_keys` 按 `(group_id, device_id, sender_account_id, sender_account_type, bucket_id)` 唯一定位。普通 SenderKeyRecord 可含多个状态：key ID、chain iteration/seed、签名公钥及可选私钥、跳过的消息密钥 seed。本机发送状态有签名私钥，从其他设备分发消息导入的接收状态只有签名公钥。它不是每群全体成员共同持有的一条相同秘密；不同 bucket/namespace 也需隔离。

普通读取从 SQL BLOB 新建记录对象。保存先更新 record，未命中才插入并写秒级 timestamp；后续更新不刷新 timestamp。写后通知失败不保证撤销数据库写入。

`fast_ratchet_sender_keys` 使用独立格式，联合键为 `(group_id, sender_id, sender_type, device_id)`，不能用普通 SenderKeyRecord 解析。此表没有 timestamp 列。`sender_id` 虽声明 INTEGER affinity，写入流程使用字符串；不能仅凭声明断言行值一定是整数。

### 辅助记录与时间

`message_base_key` 以消息 remote/from_me/id 加 recipient/type/device 定位 `last_alice_base_key`，辅助重试与 base-key 判断，不代替 SessionRecord。`prekey_uploads` 不保存密钥材料；PQ 升级表和 stanza 队列也不是身份私钥表。

队列显式区分 `time_sec` 与 `create_time_ms`；其他 timestamp 列必须按各自写入逻辑解释，不能统一套一个时间单位。

### 完整建表与显式索引

以下按表列出全部字段、默认值、约束和显式索引。示例创建空结构，不包含任何账户材料；`sqlite_sequence` 等 SQLite 管理表不计入 15 张应用表。

<details>
<summary>identities</summary>

```sql
CREATE TABLE identities (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    recipient_id INTEGER,
    recipient_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER,
    registration_id INTEGER,
    public_key BLOB,
    private_key BLOB,
    next_prekey_id INTEGER,
    next_kyber_prekey_id INTEGER,
    timestamp INTEGER,
    account_encryption_attestation_type INTEGER NOT NULL DEFAULT 0,
    mark_as_verified INTEGER,
    mark_as_verified_action_seq INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS identities_idx ON identities (recipient_id, recipient_type, device_id);
CREATE INDEX IF NOT EXISTS identities_attestation_type_idx ON identities (account_encryption_attestation_type);
```

</details>

<details>
<summary>prekeys</summary>

```sql
CREATE TABLE prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER UNIQUE,
    sent_to_server INTEGER,
    record BLOB,
    direct_distribution INTEGER,
    upload_timestamp INTEGER,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>prekey_uploads</summary>

```sql
CREATE TABLE prekey_uploads (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    upload_timestamp INTEGER,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>kyber_prekeys</summary>

```sql
CREATE TABLE kyber_prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER NOT NULL UNIQUE,
    sent_to_server INTEGER,
    record BLOB NOT NULL,
    direct_distribution INTEGER,
    upload_timestamp INTEGER,
    last_resort_key INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS kyber_prekeys_last_resort_key_idx ON kyber_prekeys (last_resort_key);
```

</details>

<details>
<summary>sessions</summary>

```sql
CREATE TABLE sessions (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id INTEGER,
    record BLOB,
    timestamp INTEGER,
    recipient_account_id TEXT,
    recipient_account_type INTEGER,
    session_type INTEGER NOT NULL DEFAULT 0,
    session_scope INTEGER NOT NULL DEFAULT 0
);

CREATE UNIQUE INDEX IF NOT EXISTS sessions_idx_v33 ON sessions (device_id, recipient_account_id, recipient_account_type, session_type, session_scope );
CREATE INDEX IF NOT EXISTS sessions_account_idx ON sessions (recipient_account_id, recipient_account_type, device_id );
```

</details>

<details>
<summary>signed_prekeys</summary>

```sql
CREATE TABLE signed_prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER UNIQUE,
    timestamp INTEGER,
    record BLOB,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>message_base_key</summary>

```sql
CREATE TABLE message_base_key (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    msg_key_remote_jid TEXT NOT NULL,
    msg_key_from_me INTEGER NOT NULL DEFAULT 0,
    msg_key_id TEXT NOT NULL,
    recipient_id INTEGER,
    recipient_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER NOT NULL DEFAULT 0,
    last_alice_base_key BLOB NOT NULL,
    timestamp INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS message_base_key_idx ON message_base_key (msg_key_remote_jid, msg_key_from_me, msg_key_id, recipient_id, recipient_type, device_id);
```

</details>

<details>
<summary>sender_keys</summary>

```sql
CREATE TABLE sender_keys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id TEXT NOT NULL,
    device_id INTEGER NOT NULL DEFAULT 0,
    record BLOB NOT NULL,
    timestamp INTEGER,
    sender_account_id TEXT,
    sender_account_type INTEGER,
    bucket_id TEXT NOT NULL DEFAULT ''
);

CREATE UNIQUE INDEX IF NOT EXISTS sender_keys_idx_v36 ON sender_keys (group_id, device_id, sender_account_id, sender_account_type, bucket_id);
CREATE INDEX IF NOT EXISTS sender_keys_account_idx ON sender_keys (group_id, sender_account_id, sender_account_type, device_id);
```

</details>

<details>
<summary>fast_ratchet_sender_keys</summary>

```sql
CREATE TABLE fast_ratchet_sender_keys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id TEXT NOT NULL,
    sender_id INTEGER NOT NULL,
    sender_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER NOT NULL DEFAULT 0,
    record BLOB NOT NULL
);

CREATE UNIQUE INDEX IF NOT EXISTS fast_ratchet_sender_keys_idx ON fast_ratchet_sender_keys (group_id, sender_id, sender_type, device_id);
```

</details>

<details>
<summary>preacks</summary>

```sql
CREATE TABLE preacks (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    ptn BLOB NOT NULL
);

```

</details>

<details>
<summary>unordered_stanza_queue</summary>

```sql
CREATE TABLE unordered_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    stanza_type INTEGER NOT NULL,
    stanza_payload BLOB NOT NULL,
    protobuf BLOB,
    decrypt_metadata BLOB,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    process_count INTEGER
);

CREATE INDEX IF NOT EXISTS unordered_stanza_class_index ON unordered_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS unordered_chat_jid_index ON unordered_stanza_queue (chat_jid);
```

</details>

<details>
<summary>e2ee_stanza_queue</summary>

```sql
CREATE TABLE e2ee_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    stanza_payload BLOB NOT NULL,
    offline_count INTEGER,
    e2ee_retry_count INTEGER NOT NULL,
    has_pkmsg INTEGER NOT NULL DEFAULT 0,
    has_skmsg INTEGER NOT NULL DEFAULT 0,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    sort_id INTEGER NOT NULL,
    process_count INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS e2ee_stanza_class_index ON e2ee_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS e2ee_chat_jid_index ON e2ee_stanza_queue (chat_jid);
CREATE INDEX IF NOT EXISTS e2ee_sender_jid_index ON e2ee_stanza_queue (sender_jid);
CREATE INDEX IF NOT EXISTS e2ee_sort_id_index ON e2ee_stanza_queue (sort_id);
```

</details>

<details>
<summary>chat_stanza_queue</summary>

```sql
CREATE TABLE chat_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    stanza_payload BLOB NOT NULL,
    stanza_type INTEGER NOT NULL,
    protobuf BLOB,
    decrypt_metadata BLOB,
    generated INTEGER NOT NULL DEFAULT 0,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    sort_id INTEGER NOT NULL,
    process_count INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS chat_stanza_class_index ON chat_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS chat_queue_chat_jid_index ON chat_stanza_queue (chat_jid);
CREATE INDEX IF NOT EXISTS chat_sender_jid_index ON chat_stanza_queue (sender_jid);
CREATE INDEX IF NOT EXISTS chat_queue_sort_id_index ON chat_stanza_queue (sort_id);
```

</details>

<details>
<summary>pq_session_upgrade_device_state</summary>

```sql
CREATE TABLE pq_session_upgrade_device_state (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_jid TEXT NOT NULL UNIQUE,
    status INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS pq_session_upgrade_device_state_status_idx ON pq_session_upgrade_device_state (status);
```

</details>

<details>
<summary>stanza_sts</summary>

```sql
CREATE TABLE stanza_sts (
    stanza_type INTEGER PRIMARY KEY,
    sts INTEGER NOT NULL DEFAULT 0
);

```

</details>

### 相关主题

[存储总览与连接密钥](../local-storage.md#zh) · [msgstore.db](msgstore.md#zh) · [端到端加密与密钥管理](../../encryption/README.md#zh) · [多设备与数据同步](../../multi-device/README.md#zh)

<a id="en"></a>

## English

### Responsibilities and Table Catalog

`axolotl.db` stores Signal identities, prekeys, direct-message cryptographic sessions, group sender state, and related processing queues. This page covers 15 application tables and 20 explicitly created indexes; SQLite automatic indexes from column-level `UNIQUE` constraints are separate. The `v33/v36` strings in table/index identifiers are part of their names.

| Table | Responsibility |
| --- | --- |
| `identities` | Local identity key pair, registration ID, next IDs; remote identity public keys and verification metadata |
| `prekeys` | Curve25519 one-time prekeys (OPKs) and distribution/upload state |
| `signed_prekeys` | Signed prekeys and historical records |
| `kyber_prekeys` | ML-KEM prekeys, including one-time and last-resort keys |
| `sessions` | Double-ratchet session records addressed by the complete address |
| `sender_keys` | Ordinary group Sender Key state |
| `fast_ratchet_sender_keys` | Separate fast-ratchet group state |
| `prekey_uploads` | Prekey upload-batch time and type metadata |
| `message_base_key` | `last_alice_base_key` addressed by message and peer device |
| `pq_session_upgrade_device_state` | Device-level PQ session-upgrade state |
| `preacks` | Queue of `ptn` BLOBs |
| `unordered_stanza_queue` | Unordered stanza identifiers, payloads, decryption metadata, and processing information |
| `e2ee_stanza_queue` | E2EE stanza offline/retry counts, pkmsg/skmsg flags, and ordering information |
| `chat_stanza_queue` | Chat stanza payloads, generation flags, ordering, and processing information |
| `stanza_sts` | `sts` values keyed by stanza type |

For BLOB, record, raw32/typed33, scope, and bucket definitions, see the [storage terminology](../local-storage.md#terms-en).

### identities: Local and Remote Identities

The local row is selected by `(recipient_id, recipient_type, device_id) = (-1, 0, 0)`. Normally generated `public_key` is `0x05 || raw32`, or typed33; `private_key` is raw32. `registration_id` is persisted independently and initialized in `1..2147483646`. It is neither regenerated for each send/read nor additionally clamped to 14 bits.

`next_prekey_id` and `next_kyber_prekey_id` are independent cursors, each normally initialized in `1..16777214`. Neither is a signed-prekey ID cursor. `mark_as_verified`, `mark_as_verified_action_seq`, and `account_encryption_attestation_type` are separate identity metadata, not part of the key encoding.

Remote identity writes store the peer address, public key, time, and attestation information, not the peer's private key or the local registration ID/next IDs. Sharing a table does not mean every row has a private key.

The local raw public/private key bytes and registration ID use separate caches; registration ID `0` is an uncached sentinel. A cached identity still requires typed-public-key decoding with strict checks when enabled. A cache may be published before database resources close, and close failure need not undo it. The two getters do not form an atomic identity snapshot.

The following query returns structural metadata without key bytes:

```sql
SELECT recipient_id, recipient_type, device_id, registration_id,
       length(public_key) AS public_bytes,
       length(private_key) AS private_bytes,
       next_prekey_id, next_kyber_prekey_id
FROM identities
WHERE recipient_id = -1 AND recipient_type = 0 AND device_id = 0;
```

### Prekey Record Encoding

`record` is a protocol protobuf BLOB, not a single AES key/IV. Field numbers and wire types are:

| Protobuf field | `prekeys` | `signed_prekeys` | `kyber_prekeys` |
| --- | --- | --- | --- |
| 1 | `uint32 id` | `uint32 id` | `uint32 id` |
| 2 | `bytes` typed33 public key | `bytes` typed33 public key | `bytes` raw KEM public key |
| 3 | `bytes` raw32 private key | `bytes` raw32 private key | `bytes` expanded KEM private key |
| 4 | Absent | `bytes` signature, normally 64 bytes | `bytes` signature over the KEM public key by the local Signal identity |
| 5 | Absent | `fixed64`, independent millisecond time value | `fixed64`, millisecond time value |

Signed and KEM records use the same five-field outer structure with different key contents. KEM public keys do not have the Curve `0x05` type prefix. ML-KEM-512/768/1024 public keys are `800/1184/1568` bytes; expanded private keys are `1632/2400/3168` bytes. Storage must not truncate them to 32 bytes or select one size solely from the table name.

The SQL `signed_prekeys.timestamp` is the write-time application-adjusted clock divided by 1000, in seconds. Record field 5 is a separate millisecond value; these are neither the same field nor the same clock read.

### Prekey Selection, Upload, and Consumption

| Object | Selection and writes | Consumption and retention |
| --- | --- | --- |
| Ordinary OPK | Read by `prekey_id`; pending uploads use `sent_to_server=0 AND direct_distribution=0`, with a limit but no ordering guarantee; batch saves also advance local `next_prekey_id`, except for an empty batch | Consumed after the pkmsg plaintext callback returns normally and the session is saved; the caller skips deletion of Curve last-resort `0xFFFFFF`, while the underlying ID-based delete has no such protection |
| Signed prekey | Select active record by `_id DESC LIMIT 1`; only when its SQL `prekey_id=0xFFFFFF`, fall back to `prekey_id<8388607 ORDER BY prekey_id DESC LIMIT 1`; retain the first BLOB if fallback is empty | Not deleted immediately after each decrypted message; historical records remain accessible by ID for delayed pkmsg |
| KEM prekey | Read by `prekey_id`; ordinary pending uploads additionally require `last_resort_key=0`; last-resort lookup can select newest `_id` by flag and upload state | Delete uses `prekey_id=? AND last_resort_key=0`; retention depends on the SQL flag rather than a fixed ID |

The signed prekey's protocol ID comes from record `id`; the SQL ID selects the row and must not replace it. Signed IDs are generated independently rather than using `identities.next_prekey_id` as a cursor.

Successful upload writes `sent_to_server`, a second-based `upload_timestamp`, and `prekey_uploads` history. The classical upload path uses that history table's default `key_type=0`; the KEM path explicitly writes `1`. This local distinction is not a complete enum for every column named `key_type`. The KEM `last_resort_key` flag is a SQL column, outside the five-field record.

### sessions: Five-Dimensional Addressing

| Dimension | Meaning |
| --- | --- |
| `recipient_account_id` | Peer account string; a Phone address can be mapped to LID before SQL |
| `recipient_account_type` | Signal account domain; PN=`0`, LID=`1` |
| `device_id` | Peer device number |
| `session_scope` | `DEFAULT=0`, `STATUS=1` |
| `session_type` | `REGULAR=0`, `PQ=1`, `STATELESS=2` |

The unique index preserves all five dimensions. Address conversion must retain device, scope, and type. `session_type=PQ` and record `sessionVersion` occupy different namespaces; the former does not fix the latter. Message-database LID `jid.type=18` also must not be copied into the Signal LID-domain column.

Outer record protobuf field 1 is the current session, and field 2 holds repeated previous sessions. Current and historical sessions use these fields:

| Field number | Name | Type or contents |
| --- | --- | --- |
| 1 | `sessionVersion` | `uint32` |
| 2 | `localIdentityPublic` | `bytes` |
| 3 | `remoteIdentityPublic` | `bytes` |
| 4 | `rootKey` | `bytes` |
| 5 | `previousCounter` | `uint32` |
| 6 | `senderChain` | Sender chain |
| 7 | `receiverChains` | Repeated receiver chains |
| 9 | `pendingPreKey` | Pending prekey session-establishment state |
| 10 | `remoteRegistrationId` | `uint32` |
| 11 | `localRegistrationId` | `uint32` |
| 12 | `needsRefresh` | `bool` |
| 13 | `aliceBaseKey` | `bytes` |
| 14 | `pendingKyberPreKey` | Pending KEM session-establishment state |

This structure does not declare field 8 `pendingKeyExchange`. Chain fields 1/2 contain ratchet public/sender private keys; field 3 is `chainKey{1:index, 2:key}`, and field 4 is repeated skipped-message material `messageKeys{1:index, 2:cipherKey, 3:macKey, 4:iv}`. The complete BLOB includes root keys, both directions of chains, skipped-message keys, historical sessions, and pending state. Keeping only a current AES key/IV loses restart and out-of-order handling.

### Session Reads, Updates, Deletion, and Caching

Reads normalize the address before consulting the cache or SQL. A missing database row can also be cached as an empty session, allowing later reads to use that cached result directly. The engine receives a separately parsed record rather than the coordinator's mutable object. Oversized or corrupt records have exact cleanup and empty-session replacement paths; absence does not always mean the local identity should be regenerated.

Saving first uses the five-dimensional address to `UPDATE record`, then `INSERT`s on a miss. Only insertion writes a second-based `timestamp`; later record updates do not refresh it. `sessions.timestamp` is therefore not the last send, receive, or decryption time.

Exact deletion uses five dimensions. Ordinary device-session removal uses account, domain, and device, covering every scope/type for that device. Storage interfaces must preserve the distinction.

Cache eviction combines a map, a candidate set, and other sets. When the map exceeds 1000, only candidates are evicted; some account types are not cached. This is not a strict 1000-entry LRU bound. Locking may use a cache monitor or address-related striped locks, with database transactions for writes.

A save can run in this order: outer transaction begins → inner transaction writes and closes → cache replacement → outer transaction completes. Cache publication need not follow final commit; database failure, callback failure, and memory rollback are separate concerns.

### Group State and Fast-Ratchet

`sender_keys` is uniquely addressed by `(group_id, device_id, sender_account_id, sender_account_type, bucket_id)`. Ordinary SenderKeyRecord can contain several states: key ID, chain iteration/seed, signing public/optional private keys, and skipped-message key seeds. Local sending state has a signing private key; receiving state imported from another device's distribution message has only a signing public key. This is not one identical secret shared by every group member; different buckets/namespaces also require isolation.

Ordinary reads construct a new record from SQL bytes. Saving updates record first, inserting with a second-based timestamp on a miss; later updates do not refresh the timestamp. A failed post-write notification need not undo the database write.

`fast_ratchet_sender_keys` uses a separate format and compound key `(group_id, sender_id, sender_type, device_id)`; it must not be parsed as ordinary SenderKeyRecord. This table has no timestamp column. Although `sender_id` declares INTEGER affinity, its writer supplies a string, so the declaration alone does not establish integer values in every row.

### Auxiliary Records and Time

`message_base_key` addresses `last_alice_base_key` by message remote/from_me/id plus recipient/type/device. It supports retry and base-key decisions rather than replacing SessionRecord. `prekey_uploads` contains no key material; PQ upgrade state and stanza queues are not identity-private-key tables either.

Queues explicitly distinguish `time_sec` from `create_time_ms`; other timestamp columns require their own writer semantics rather than one universal unit.

### Complete Table DDL and Explicit Indexes

The following lists every field, default, constraint, and explicit index by table. It creates empty structures without account material. SQLite-managed tables such as `sqlite_sequence` are outside the 15 application tables.

<details>
<summary>identities</summary>

```sql
CREATE TABLE identities (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    recipient_id INTEGER,
    recipient_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER,
    registration_id INTEGER,
    public_key BLOB,
    private_key BLOB,
    next_prekey_id INTEGER,
    next_kyber_prekey_id INTEGER,
    timestamp INTEGER,
    account_encryption_attestation_type INTEGER NOT NULL DEFAULT 0,
    mark_as_verified INTEGER,
    mark_as_verified_action_seq INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS identities_idx ON identities (recipient_id, recipient_type, device_id);
CREATE INDEX IF NOT EXISTS identities_attestation_type_idx ON identities (account_encryption_attestation_type);
```

</details>

<details>
<summary>prekeys</summary>

```sql
CREATE TABLE prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER UNIQUE,
    sent_to_server INTEGER,
    record BLOB,
    direct_distribution INTEGER,
    upload_timestamp INTEGER,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>prekey_uploads</summary>

```sql
CREATE TABLE prekey_uploads (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    upload_timestamp INTEGER,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>kyber_prekeys</summary>

```sql
CREATE TABLE kyber_prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER NOT NULL UNIQUE,
    sent_to_server INTEGER,
    record BLOB NOT NULL,
    direct_distribution INTEGER,
    upload_timestamp INTEGER,
    last_resort_key INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS kyber_prekeys_last_resort_key_idx ON kyber_prekeys (last_resort_key);
```

</details>

<details>
<summary>sessions</summary>

```sql
CREATE TABLE sessions (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id INTEGER,
    record BLOB,
    timestamp INTEGER,
    recipient_account_id TEXT,
    recipient_account_type INTEGER,
    session_type INTEGER NOT NULL DEFAULT 0,
    session_scope INTEGER NOT NULL DEFAULT 0
);

CREATE UNIQUE INDEX IF NOT EXISTS sessions_idx_v33 ON sessions (device_id, recipient_account_id, recipient_account_type, session_type, session_scope );
CREATE INDEX IF NOT EXISTS sessions_account_idx ON sessions (recipient_account_id, recipient_account_type, device_id );
```

</details>

<details>
<summary>signed_prekeys</summary>

```sql
CREATE TABLE signed_prekeys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    prekey_id INTEGER UNIQUE,
    timestamp INTEGER,
    record BLOB,
    key_type INTEGER NOT NULL DEFAULT 0
);

```

</details>

<details>
<summary>message_base_key</summary>

```sql
CREATE TABLE message_base_key (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    msg_key_remote_jid TEXT NOT NULL,
    msg_key_from_me INTEGER NOT NULL DEFAULT 0,
    msg_key_id TEXT NOT NULL,
    recipient_id INTEGER,
    recipient_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER NOT NULL DEFAULT 0,
    last_alice_base_key BLOB NOT NULL,
    timestamp INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS message_base_key_idx ON message_base_key (msg_key_remote_jid, msg_key_from_me, msg_key_id, recipient_id, recipient_type, device_id);
```

</details>

<details>
<summary>sender_keys</summary>

```sql
CREATE TABLE sender_keys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id TEXT NOT NULL,
    device_id INTEGER NOT NULL DEFAULT 0,
    record BLOB NOT NULL,
    timestamp INTEGER,
    sender_account_id TEXT,
    sender_account_type INTEGER,
    bucket_id TEXT NOT NULL DEFAULT ''
);

CREATE UNIQUE INDEX IF NOT EXISTS sender_keys_idx_v36 ON sender_keys (group_id, device_id, sender_account_id, sender_account_type, bucket_id);
CREATE INDEX IF NOT EXISTS sender_keys_account_idx ON sender_keys (group_id, sender_account_id, sender_account_type, device_id);
```

</details>

<details>
<summary>fast_ratchet_sender_keys</summary>

```sql
CREATE TABLE fast_ratchet_sender_keys (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    group_id TEXT NOT NULL,
    sender_id INTEGER NOT NULL,
    sender_type INTEGER NOT NULL DEFAULT 0,
    device_id INTEGER NOT NULL DEFAULT 0,
    record BLOB NOT NULL
);

CREATE UNIQUE INDEX IF NOT EXISTS fast_ratchet_sender_keys_idx ON fast_ratchet_sender_keys (group_id, sender_id, sender_type, device_id);
```

</details>

<details>
<summary>preacks</summary>

```sql
CREATE TABLE preacks (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    ptn BLOB NOT NULL
);

```

</details>

<details>
<summary>unordered_stanza_queue</summary>

```sql
CREATE TABLE unordered_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    stanza_type INTEGER NOT NULL,
    stanza_payload BLOB NOT NULL,
    protobuf BLOB,
    decrypt_metadata BLOB,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    process_count INTEGER
);

CREATE INDEX IF NOT EXISTS unordered_stanza_class_index ON unordered_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS unordered_chat_jid_index ON unordered_stanza_queue (chat_jid);
```

</details>

<details>
<summary>e2ee_stanza_queue</summary>

```sql
CREATE TABLE e2ee_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    stanza_payload BLOB NOT NULL,
    offline_count INTEGER,
    e2ee_retry_count INTEGER NOT NULL,
    has_pkmsg INTEGER NOT NULL DEFAULT 0,
    has_skmsg INTEGER NOT NULL DEFAULT 0,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    sort_id INTEGER NOT NULL,
    process_count INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS e2ee_stanza_class_index ON e2ee_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS e2ee_chat_jid_index ON e2ee_stanza_queue (chat_jid);
CREATE INDEX IF NOT EXISTS e2ee_sender_jid_index ON e2ee_stanza_queue (sender_jid);
CREATE INDEX IF NOT EXISTS e2ee_sort_id_index ON e2ee_stanza_queue (sort_id);
```

</details>

<details>
<summary>chat_stanza_queue</summary>

```sql
CREATE TABLE chat_stanza_queue (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    stanza_id TEXT NOT NULL,
    stanza_key BLOB NOT NULL UNIQUE,
    stanza_class INTEGER NOT NULL,
    chat_type INTEGER,
    chat_jid TEXT,
    sender_jid TEXT,
    stanza_payload BLOB NOT NULL,
    stanza_type INTEGER NOT NULL,
    protobuf BLOB,
    decrypt_metadata BLOB,
    generated INTEGER NOT NULL DEFAULT 0,
    time_sec INTEGER NOT NULL,
    create_time_ms INTEGER NOT NULL,
    sort_id INTEGER NOT NULL,
    process_count INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS chat_stanza_class_index ON chat_stanza_queue (stanza_class);
CREATE INDEX IF NOT EXISTS chat_queue_chat_jid_index ON chat_stanza_queue (chat_jid);
CREATE INDEX IF NOT EXISTS chat_sender_jid_index ON chat_stanza_queue (sender_jid);
CREATE INDEX IF NOT EXISTS chat_queue_sort_id_index ON chat_stanza_queue (sort_id);
```

</details>

<details>
<summary>pq_session_upgrade_device_state</summary>

```sql
CREATE TABLE pq_session_upgrade_device_state (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    device_jid TEXT NOT NULL UNIQUE,
    status INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS pq_session_upgrade_device_state_status_idx ON pq_session_upgrade_device_state (status);
```

</details>

<details>
<summary>stanza_sts</summary>

```sql
CREATE TABLE stanza_sts (
    stanza_type INTEGER PRIMARY KEY,
    sts INTEGER NOT NULL DEFAULT 0
);

```

</details>

### Related Topics

[Storage Overview and Connection Keys](../local-storage.md#en) · [msgstore.db](msgstore.md#en) · [End-to-End Encryption and Key Management](../../encryption/README.md#en) · [Multi-Device and Data Synchronization](../../multi-device/README.md#en)
