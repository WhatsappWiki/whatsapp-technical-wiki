# Local Databases and Storage · 本地数据库与存储

[消息目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 各类存储保存什么

WhatsApp 将消息历史、Signal 加密状态和连接认证密钥分别保存。查消息时主要读取 `msgstore.db`；处理消息加解密时主要读取 `axolotl.db`；连接认证密钥的加密副本保存在名为 `keystore` 的偏好存储中。

| 名称 | 存储形式 | 保存内容 |
| --- | --- | --- |
| `axolotl.db` | SQLite 数据库 | Signal 身份密钥、预密钥、单聊会话、群聊 Sender Key 和相关处理队列 |
| `msgstore.db` | SQLite 数据库 | JID 地址、聊天、消息、用户/设备回执和媒体元数据 |
| `keystore` | SharedPreferences 偏好存储 | 连接静态密钥对的加密副本，以及副本选择、校验相关状态 |
| AndroidKeyStore | Android 系统密钥存储 | 用于加密其中一份连接密钥副本的 AES 密钥 |

数据库文件由当前账户使用的 Context 和数据库组件定位。`keystore` 是偏好存储名称，其文件形式和位置由存储后端决定。

### 阅读入口

| 文章 | 内容 |
| --- | --- |
| [数据库专题目录](storage/README.md#zh) | 两类数据库的阅读顺序、结构与相关主题 |
| [axolotl.db：密钥、会话与队列](storage/axolotl.md#zh) | 15 张应用表、20 个显式索引、密钥编码、地址维度与生命周期 |
| [msgstore.db：消息、回执与媒体](storage/msgstore.md#zh) | 6 张核心表、150 列、20 个无条件创建的索引、状态与查询示例 |

<a id="terms-zh"></a>

### 常用术语

| 术语 | 本专题中的含义 |
| --- | --- |
| BLOB | SQLite 保存二进制字节的字段类型；字节内容由具体字段定义 |
| `record` | 协议记录；预密钥、会话和 Sender Key 表主要将其序列化为 protobuf，再存入 BLOB |
| raw32 / typed33 | raw32 是 32 字节原始密钥；typed33 公钥为类型字节 `0x05` 加 32 字节公钥 |
| `scope` / `type` | 单聊会话的用途范围和类别，对应 `session_scope` / `session_type` 两个独立字段 |
| `bucket` | 同一群、发送者和设备下进一步区分 Sender Key 记录的字符串维度，对应 `bucket_id`，默认空串 |
| 密钥包装 | 使用另一把密钥加密密钥材料；此处指连接密钥对的存储加密 |

### 地址、会话与回执

单聊会话按对方账户、账户域、设备、scope 和 type 定位，完整协议记录保存加解密所需的棘轮状态。群聊 Sender Key 另按群、发送者、设备和 bucket 定位。消息历史通过 JID、聊天和消息行关联各项业务数据。

PN 表示电话号码身份，LID 表示另一类账户身份。两类存储对它们使用不同编号：

| 字段 | PN | LID |
| --- | --- | --- |
| Signal `recipient_account_type` | `0` | `1` |
| 消息库 `jid.type` | `0` | `18` |

地址转换需要按身份类别转换编号。`recipient_account_id` 表示对方账户；容纳多个本机账户的存储还需各自的本机账户隔离方式。

用户回执记录对方的送达、已读或播放状态，设备回执记录对应设备的信息。本机阅读游标和已经发出的已读回执也分别维护；这些状态应按对应字段解释。

### WAL、一致快照与事务

Android 数据库打开路径启用 WAL。活动库可能同时存在主 `.db`、`-wal` 和 `-shm` 文件；已经提交的数据仍可能留在 WAL 中。读取或备份应使用一致快照，不能仅复制活动主文件并假定其中包含最新全部提交。WAL 的提交与 checkpoint 是不同操作，见 [SQLite WAL 说明](https://www.sqlite.org/wal.html)。

即使多个数据库通过同一连接 `ATTACH`，WAL 下也只保证单个数据库文件内的事务原子性；主机在多文件提交中崩溃时，不能据此承诺所有文件共同提交或共同回滚。跨库一致性需结合实际日志模式和提交条件判断，见 [SQLite ATTACH 事务条件](https://www.sqlite.org/lang_attach.html)。

数据库事务也不会自动回滚已经发布的内存缓存、外部通知或应用回调。会话写入、预密钥消费、历史入库与回执排队之间的顺序必须分别处理；不能从表结构推导出所有这些步骤组成一个跨库原子事务。

### Signal 记录如何保存

本机 Signal 身份密钥在 `identities` 中使用独立的公钥和私钥列；预密钥、会话和群聊 Sender Key 主要保存为协议 protobuf BLOB。对应存取流程直接保存和解析这些字节。

BLOB 只说明字段存放二进制数据。字段是否包含密文、文件和系统是否提供其他保护，需要分别判断。

<a id="keystore-zh"></a>

### keystore 与 AndroidKeyStore 的关系

`keystore` 偏好存储保存连接认证静态 Curve25519 密钥对的加密副本。连接密钥对与 `axolotl.db` 中的 Signal 身份密钥对独立，用途也不同。

连接密钥对的明文排列为 `private32 || public32`：32 字节私钥在前，32 字节公钥在后，共 64 字节。两种副本采用不同的加密方式：

| 偏好键 | 保存内容与加密方式 |
| --- | --- |
| `client_static_keypair_pwd_enc` | type 2：以密码材料派生 AES 密钥的副本；PBKDF2-HMAC-SHA1 使用原始 8-bit 密码语义，16 轮、派生 16 字节密钥，再用 AES/OFB 加密 |
| `client_static_keypair_enc` | type 0 Android 副本；使用 AndroidKeyStore 中别名为 `aes_auth_key` 的 AES 密钥，通过 AES/GCM/NoPadding 加密 |
| `can_user_android_key_store` | 副本选择相关状态 |
| `client_static_keypair_enc_success`、`client_static_keypair_enc_failed`、`android_key_store_verified_failures` | 校验与读回相关计数 |

`keystore` 保存加密后的连接密钥对，AndroidKeyStore 保存其中一种加密方式所用的 AES 密钥。应用通过相应方式解密后使用连接密钥对。当前选择哪个副本还取决于校验和读回条件，单个偏好键的存在并不表示该副本正在使用。

### 结构与存取规则

`axolotl.db` 文章列出 15 张应用表；`msgstore.db` 文章只展开 6 张核心表，其他消息表、视图和触发器不在这 6 张表的范围内。

存取密钥和会话时，应保留完整二进制 record，并按完整联合地址定位。预密钥游标、last-resort 标记和上传状态分别维护；同一会话的读取、修改和写回还需与对应锁和缓存保持一致。

### 相关主题

[端到端加密与密钥管理](../encryption/README.md#zh) · [登录认证与会话](../authentication/README.md#zh) · [图片、文件与语音消息](../media/README.md#zh) · [多设备与数据同步](../multi-device/README.md#zh)

<a id="en"></a>

## English

### What Each Store Contains

WhatsApp stores message history, Signal cryptographic state, and connection-authentication keys separately. Message queries mainly read `msgstore.db`; message encryption and decryption mainly read `axolotl.db`. Encrypted copies of the connection-authentication key pair reside in a preference store named `keystore`.

| Name | Storage form | Contents |
| --- | --- | --- |
| `axolotl.db` | SQLite database | Signal identity keys, prekeys, direct-message sessions, group Sender Keys, and related processing queues |
| `msgstore.db` | SQLite database | JID addresses, chats, messages, user/device receipts, and media metadata |
| `keystore` | SharedPreferences preference store | Encrypted copies of the connection static key pair and copy-selection/verification state |
| AndroidKeyStore | Android system key storage | The AES key used to encrypt one of the connection-key copies |

The current account's Context and database components locate database files. `keystore` is a preference-store name; its storage backend determines the file form and location.

### Reading Entries

| Article | Contents |
| --- | --- |
| [Database topic index](storage/README.md#en) | Reading order, structure, and related topics for both databases |
| [axolotl.db: Keys, Sessions, and Queues](storage/axolotl.md#en) | 15 application tables, 20 explicit indexes, key encodings, address dimensions, and lifecycles |
| [msgstore.db: Messages, Receipts, and Media](storage/msgstore.md#en) | 6 core tables, 150 columns, 20 unconditionally created indexes, statuses, and query examples |

<a id="terms-en"></a>

### Common Terms

| Term | Meaning in this topic |
| --- | --- |
| BLOB | A SQLite field type for binary bytes; each field defines the byte contents |
| `record` | A protocol record; prekey, session, and Sender Key tables mainly serialize it as protobuf and store it in a BLOB |
| raw32 / typed33 | raw32 is 32 raw key bytes; a typed33 public key is type byte `0x05` followed by 32 public-key bytes |
| `scope` / `type` | A direct-message session's usage scope and category, represented by separate `session_scope` / `session_type` fields |
| `bucket` | A string dimension further distinguishing Sender Key records for a group, sender, and device; stored in `bucket_id`, defaulting to empty text |
| Key wrapping | Encrypting key material with another key; here, encryption of the stored connection key pair |

### Addresses, Sessions, and Receipts

Direct-message sessions are addressed by peer account, account domain, device, scope, and type. Their complete protocol records hold the ratchet state required for encryption and decryption. Group Sender Keys additionally use group, sender, device, and bucket addressing. Message history links business data through JID, chat, and message rows.

PN identifies a phone-number identity; LID identifies another account-identity category. The two stores use different numbers for these categories:

| Field | PN | LID |
| --- | --- | --- |
| Signal `recipient_account_type` | `0` | `1` |
| Message-database `jid.type` | `0` | `18` |

Address conversion must map numbers according to identity category. `recipient_account_id` identifies the peer account; storage containing several local accounts also needs its own local-account isolation.

User receipts record the peer's delivery, read, or playback state; device receipts hold information for the corresponding device. Local read cursors and sent read receipts are maintained separately as well. Each state requires its corresponding fields.

### WAL, Consistent Snapshots, and Transactions

The Android database opening path enables WAL. An active database may have a main `.db`, a `-wal`, and a `-shm` file; committed data can still reside in the WAL. Reads and backups need a consistent snapshot rather than a copy of only the active main file. WAL commit and checkpoint are separate operations; see the [SQLite WAL documentation](https://www.sqlite.org/wal.html).

Even when several databases are `ATTACH`ed to one connection, WAL provides transaction atomicity within each file rather than across the entire set if the host crashes during a multi-file commit. Cross-database consistency depends on the actual journal mode and commit conditions; see the [SQLite ATTACH transaction conditions](https://www.sqlite.org/lang_attach.html).

A database transaction also does not automatically undo published memory caches, external notifications, or application callbacks. Session writes, prekey consumption, history insertion, and receipt queuing need explicit ordering; table definitions do not establish a cross-database atomic transaction covering them all.

### How Signal Records Are Stored

The local Signal identity uses separate public/private columns in `identities`. Prekeys, sessions, and group Sender Keys mainly use protocol protobuf BLOBs. Their storage paths directly save and parse those bytes.

BLOB only specifies binary data. Whether a field contains ciphertext and whether file or system layers provide other protection are separate questions.

<a id="keystore-en"></a>

### How keystore Relates to AndroidKeyStore

The `keystore` preference store holds encrypted copies of the connection-authentication static Curve25519 key pair. This pair is independent of the Signal identity pair in `axolotl.db` and serves a different purpose.

Its plaintext layout is `private32 || public32`: a 32-byte private key followed by a 32-byte public key, totaling 64 bytes. The two copies use different encryption methods:

| Preference key | Contents and encryption method |
| --- | --- |
| `client_static_keypair_pwd_enc` | Type 2: a copy encrypted with a password-derived AES key; PBKDF2-HMAC-SHA1 uses raw 8-bit password semantics, 16 iterations, and a 16-byte derived key, followed by AES/OFB encryption |
| `client_static_keypair_enc` | Type 0 Android copy; AES/GCM/NoPadding with the AES key under AndroidKeyStore alias `aes_auth_key` |
| `can_user_android_key_store` | State involved in copy selection |
| `client_static_keypair_enc_success`, `client_static_keypair_enc_failed`, `android_key_store_verified_failures` | Counters involved in verification and readback |

`keystore` holds the encrypted connection pair; AndroidKeyStore holds the AES key used by one encryption method. The application decrypts the pair through the corresponding method before use. Copy selection also depends on validation and readback conditions; a preference key's presence alone does not establish that its copy is selected.

### Schema and Storage Rules

The `axolotl.db` article lists 15 application tables; the `msgstore.db` article covers only 6 core tables. Other message tables, views, and triggers are outside those 6 tables.

Key and session storage must preserve complete binary records and use complete compound addresses. Prekey cursors, last-resort flags, and upload state are maintained separately. Reading, modifying, and saving the same session also requires coordination with its locks and cache.

### Related Topics

[End-to-End Encryption and Key Management](../encryption/README.md#en) · [Login, Authentication, and Sessions](../authentication/README.md#en) · [Media Transfer and Voice Messages](../media/README.md#en) · [Multi-Device and Data Synchronization](../multi-device/README.md#en)
