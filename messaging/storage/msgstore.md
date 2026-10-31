# msgstore.db · Messages, Receipts, and Media · 消息、回执与媒体

[数据库目录](README.md#zh) · [存储总览](../local-storage.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 六张核心表

`msgstore.db` 的消息历史通过 JID、聊天和消息行关联回执及媒体。本页展开 6 张核心表、150 列与 20 个无条件创建的显式索引；其他消息表、视图和触发器不包含在这 6 张表中。

| 表 | 列数 | 关联与职责 |
| --- | --- | --- |
| `jid` | 7 | 账户/设备 JID 的分解字段、类型及原始字符串 |
| `chat` | 53 | 聊天地址、显示/阅读游标、未读统计及聊天设置 |
| `message` | 24 | 聊天下的消息标识、方向、状态、时间及正文相关字段 |
| `receipt_device` | 5 | 消息与设备 JID 之间的回执记录 |
| `receipt_user` | 6 | 消息与用户 JID 之间的送达、已读、播放时间 |
| `message_media` | 55 | 消息对应的媒体定位、密钥材料、文件和处理元数据 |

BLOB 与 record 的定义见[存储术语](../local-storage.md#terms-zh)。下面先说明行之间的关联，再说明状态和时间。

### JID、聊天与消息的关联

`jid._id` 是库内行号。`user` 与 `server` 为非空文本，`agent/device/type` 为独立列，`raw_string` 保存字符串表示。分解字段联合唯一索引为 `(user, server, agent, device, type)`，另有 `raw_string` 唯一索引。

消息库 `jid.type` 中 PN 为 `0`、LID 为 `18`；Signal 的账户域 PN/LID 为 `0/1`。例如查询 LID 消息地址时应使用 `jid.type=18`，不能把 Signal 域值 `1` 当作消息库 LID 类型。

`chat.jid_row_id` 定位聊天 JID，`chat.account_jid_row_id` 是另一独立列，两者各有唯一性约束。`message.chat_row_id` 关联聊天，`sender_jid_row_id` 表达消息发送者，不能将发送者无条件等同于聊天对象。以下关系描述列之间的用途；本页 DDL 没有声明 `FOREIGN KEY`，不能据字段名声称 SQLite 自动执行引用约束或级联删除。

```text
jid._id <- chat.jid_row_id
chat._id <- message.chat_row_id
jid._id <- message.sender_jid_row_id
message._id <- receipt_user.message_row_id
jid._id <- receipt_user.receipt_user_jid_row_id
message._id <- receipt_device.message_row_id
jid._id <- receipt_device.receipt_device_jid_row_id
message._id <- message_media.message_row_id
```

### chat：游标、计数与设置

`display_message_row_id`、`last_message_row_id`、`last_read_message_row_id` 和 `last_read_receipt_sent_message_row_id` 分别是显示、最新消息、本机阅读及已发送阅读回执的游标。还有独立的 `*_sort_id` 列，不能把行号、排序值和时间戳当成同一个值。

`unseen_message_count`、`unseen_row_count`、`unseen_missed_calls_count`、`unseen_important_message_count`、表情回应及评论计数是不同统计。归档、隐藏、聊天锁、临时消息、限时共享、社区提示、群类型和业务聊天状态也各有列；不能将所有非零整数统一解释成一个布尔标志或从字段名补造枚举。

建表约束只对 `unseen_important_message_count`、`group_type`、`has_new_community_admin_dialog_been_acknowledged` 指定 `NOT NULL DEFAULT 0`；其他可空列不能批量假定有同样默认值。完整 53 列及类型见下方 DDL。

<a id="messages-zh"></a>

### message：标识、状态与时间

`message._id` 是本地行号，`key_id` 是非空协议消息标识。唯一索引按 `(chat_row_id, from_me, key_id, sender_jid_row_id)` 定位；`key_id` 本身不是全库唯一索引。`from_me` 区分方向，不能通过 status 推断消息必为入站或出站。

单聊出站消息具有以下状态值：

| `message.status` | 含义 | 相关用户回执列 |
| --- | --- | --- |
| `4` | 服务端 ACK | 不等于接收方已经收到 |
| `5` | 已送达 | `receipt_timestamp` |
| `13` | 已读 | `read_timestamp` |
| `8` | 已播放 | `played_timestamp` |

这些数字不是可直接比较大小的状态等级；不能用 `MAX(status)` 实现所有状态合并。上述映射用于单聊发送/回执路径，不定义全部群聊、入站和排队状态。排队状态及纯文本 SQL `message_type` 在本页不指定数值；缺乏对应业务赋值时，不应用猜测枚举填列。

`timestamp`、`received_timestamp`、`receipt_server_timestamp` 和 `server_sts` 是独立列，`sort_id` 是独立排序字段。不能从一列、字段名或网络 `t` 的单位推导全部时间列的单位；也不能把本地接收时间冒充对方实际阅读时间。正文相关列有 `text_data`、`translated_text`，媒体说明文字则另在 `message_media.media_caption`。

<a id="receipts-zh"></a>

### 用户回执与设备回执

`receipt_user` 按 `(message_row_id, receipt_user_jid_row_id)` 唯一定位用户，分别保存 `receipt_timestamp`、`read_timestamp`、`played_timestamp`。对应回执状态为 `5/13/8` 时写各自的列；这三个时间不能互相替代。

`receipt_device` 按 `(message_row_id, receipt_device_jid_row_id)` 唯一定位设备，只有 `receipt_device_timestamp` 和 `primary_device_version` 等列，没有 `read_timestamp`。设备回执时间不能冒充用户已读时间，`primary_device_version` 也不是阅读标记。

用户回执写入将其收到的时间值写入选定列；要解释为秒或毫秒，需要结合具体上游回执路径。仅凭该写入动作不能建立所有原生回执字段的统一单位。记录历史数据时，缺失的回执时间应保留为空，避免用本地处理时间替代对方的事件时间。

本机收到消息后的阅读动作和向对方发送的已读回执，与对方发送来的用户回执不同。查询“对方何时已读我发的消息”应查对应出站消息的用户回执；查询“本机是否读过入站消息”则需要本机阅读游标及对应状态。

<a id="media-zh"></a>

### message_media：媒体密钥与文件元数据

`message_media.message_row_id` 是主键，关联消息行；`chat_row_id` 另供聊天级查询。文件定位包括 `file_path`、`message_url`、`direct_path`、`metadata_url`、`dash_manifest_url` 和 `media_upload_handle` 等，不同列不能假定指向同一种资源。

媒体密钥相关列包括 BLOB `media_key`、`e2ee_media_key`，以及 `media_key_domain`、`media_key_timestamp`。`file_hash`、`enc_file_hash`、`partial_media_hash`、`partial_media_enc_hash`、`original_file_hash` 是不同摘要列。列名和 TEXT/BLOB 类型本身不定义 KDF、摘要编码、加密封装或这些列之间的替代关系。

其他字段描述文件大小/长度、MIME、宽高、时长、说明文字、转写、缩略图/扫描 sidecar、贴纸、裁剪、转码、传输和卸载状态。`motion_photo_presentation_offset_ms` 名称明确包含毫秒，但不能把该后缀推广到其他时间或时长列。`mute_video` 与 `media_transcode_quality` 的默认值为 `0`，并没有 `NOT NULL` 约束。

媒体文件实体不由这些列自动提供；仅有媒体行不等于本地文件存在、下载链接有效或所有解密材料齐全。

### 查询示例

以下参数为调用者提供的库内行号，不含真实账户或消息内容。查询聊天中的消息元数据时，分别返回协议 ID、行号、排序值和状态：

```sql
SELECT m._id, m.key_id, m.from_me, m.sender_jid_row_id,
       m.status, m.sort_id, m.timestamp,
       j.type AS chat_jid_type
FROM message AS m
JOIN chat AS c ON c._id = m.chat_row_id
JOIN jid AS j ON j._id = c.jid_row_id
WHERE m.chat_row_id = :chat_row_id
ORDER BY m.sort_id, m._id;
```

按用户区分送达、已读和播放时间，不从设备回执拼造已读值：

```sql
SELECT r.receipt_user_jid_row_id, j.type AS user_jid_type,
       r.receipt_timestamp, r.read_timestamp, r.played_timestamp
FROM receipt_user AS r
JOIN jid AS j ON j._id = r.receipt_user_jid_row_id
WHERE r.message_row_id = :message_row_id;
```

这些是结构查询，不定义缺行修复、时间单位转换、收据发送或自动更新游标的流程。

### 索引条件与完整 DDL

20 个索引中的“无条件”指创建动作不依赖配置，不代表每个索引都覆盖全部行：`chat_ephemeral_after_read_duration_index` 自身有 `WHERE ephemeral_after_read_duration > 0`，属于部分索引。

另有 `chat_sort_timestamp_index`、`chat_sort_timestamp_with_hidden` 和 `idx_message_media_file_path_v1` 依赖各自配置分支决定是否创建，未合并到下面的 20 个索引中。不能仅因名字相似就同时创建所有分支。下列 DDL 仅定义上述 6 张核心表及对应索引。

以下逐表保留全部 150 列的顺序、类型、空值约束、默认值及上述 20 个显式索引；列级 `UNIQUE` 产生的自动索引另计。

<details>
<summary>jid</summary>

```sql
CREATE TABLE jid (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    user TEXT NOT NULL,
    server TEXT NOT NULL,
    agent INTEGER,
    device INTEGER,
    type INTEGER,
    raw_string TEXT
);

CREATE UNIQUE INDEX IF NOT EXISTS jid_key_new_index ON jid ( user, server, agent, device, type );
CREATE UNIQUE INDEX IF NOT EXISTS jid_raw_string_index ON jid (raw_string);
```

</details>

<details>
<summary>chat</summary>

```sql
CREATE TABLE chat (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    jid_row_id INTEGER UNIQUE,
    hidden INTEGER,
    subject TEXT,
    created_timestamp INTEGER,
    display_message_row_id INTEGER,
    last_message_row_id INTEGER,
    last_read_message_row_id INTEGER,
    last_read_receipt_sent_message_row_id INTEGER,
    last_important_message_row_id INTEGER,
    archived INTEGER,
    sort_timestamp INTEGER,
    mod_tag INTEGER,
    gen REAL,
    spam_detection INTEGER,
    unseen_earliest_message_received_time INTEGER,
    unseen_message_count INTEGER,
    unseen_missed_calls_count INTEGER,
    unseen_row_count INTEGER,
    plaintext_disabled INTEGER,
    vcard_ui_dismissed INTEGER,
    change_number_notified_message_row_id INTEGER,
    show_group_description INTEGER,
    ephemeral_expiration INTEGER,
    ephemeral_setting_timestamp INTEGER,
    ephemeral_displayed_exemptions INTEGER,
    ephemeral_disappearing_messages_initiator INTEGER,
    unseen_important_message_count INTEGER NOT NULL DEFAULT 0,
    group_type INTEGER NOT NULL DEFAULT 0,
    last_message_reaction_row_id INTEGER,
    last_seen_message_reaction_row_id INTEGER,
    unseen_message_reaction_count INTEGER,
    unseen_comment_message_count INTEGER,
    growth_lock_level INTEGER,
    growth_lock_expiration_ts INTEGER,
    last_read_message_sort_id INTEGER,
    display_message_sort_id INTEGER,
    last_message_sort_id INTEGER,
    last_read_receipt_sent_message_sort_id INTEGER,
    has_new_community_admin_dialog_been_acknowledged INTEGER NOT NULL DEFAULT 0,
    history_sync_progress INTEGER,
    chat_lock INTEGER,
    chat_origin TEXT,
    participation_status INTEGER,
    account_jid_row_id INTEGER,
    chat_encryption_state INTEGER,
    group_member_count INTEGER,
    limited_sharing INTEGER,
    limited_sharing_setting_timestamp INTEGER,
    is_contact INTEGER,
    ephemeral_after_read_duration INTEGER,
    business_chat_state INTEGER,
    chat_props INTEGER
);

CREATE INDEX IF NOT EXISTS chat_group_type_index ON chat (group_type);
CREATE INDEX IF NOT EXISTS chat_hidden_index ON chat (hidden);
CREATE UNIQUE INDEX IF NOT EXISTS chat_account_jid_row_id_unique_index ON chat (account_jid_row_id);
CREATE INDEX IF NOT EXISTS chat_ephemeral_after_read_duration_index ON chat (ephemeral_after_read_duration) WHERE ephemeral_after_read_duration > 0;
```

</details>

<details>
<summary>message</summary>

```sql
CREATE TABLE message (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    chat_row_id INTEGER NOT NULL,
    from_me INTEGER NOT NULL,
    key_id TEXT NOT NULL,
    sender_jid_row_id INTEGER,
    status INTEGER,
    broadcast INTEGER,
    recipient_count INTEGER,
    participant_hash TEXT,
    origination_flags INTEGER,
    origin INTEGER,
    timestamp INTEGER,
    received_timestamp INTEGER,
    receipt_server_timestamp INTEGER,
    message_type INTEGER,
    text_data TEXT,
    starred INTEGER,
    lookup_tables INTEGER,
    message_add_on_flags INTEGER,
    view_mode INTEGER,
    sort_id INTEGER NOT NULL DEFAULT 0,
    translated_text TEXT,
    view_replies_thread_id INTEGER,
    server_sts INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS message_key_index ON message ( chat_row_id, from_me, key_id, sender_jid_row_id );
CREATE INDEX IF NOT EXISTS message_sort_id_index ON message (sort_id);
CREATE INDEX IF NOT EXISTS message_starred_index ON message (starred);
CREATE INDEX IF NOT EXISTS message_type_chat_index ON message ( chat_row_id, message_type );
CREATE INDEX IF NOT EXISTS message_chat_id_index ON message ( chat_row_id, _id );
CREATE INDEX IF NOT EXISTS message_type_index ON message (message_type);
CREATE INDEX IF NOT EXISTS message_chat_sort_id_index ON message ( chat_row_id, sort_id );
CREATE INDEX IF NOT EXISTS message_starred_sort_id_index ON message ( starred, sort_id );
```

</details>

<details>
<summary>receipt_device</summary>

```sql
CREATE TABLE receipt_device (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_row_id INTEGER NOT NULL,
    receipt_device_jid_row_id INTEGER NOT NULL,
    receipt_device_timestamp INTEGER,
    primary_device_version INTEGER
);

CREATE INDEX IF NOT EXISTS receipt_device_table_device_index ON receipt_device (receipt_device_jid_row_id);
CREATE UNIQUE INDEX IF NOT EXISTS receipt_device_index ON receipt_device ( message_row_id, receipt_device_jid_row_id );
```

</details>

<details>
<summary>receipt_user</summary>

```sql
CREATE TABLE receipt_user (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_row_id INTEGER NOT NULL,
    receipt_user_jid_row_id INTEGER NOT NULL,
    receipt_timestamp INTEGER,
    read_timestamp INTEGER,
    played_timestamp INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS receipt_user_index ON receipt_user( message_row_id, receipt_user_jid_row_id );
```

</details>

<details>
<summary>message_media</summary>

```sql
CREATE TABLE message_media (
    message_row_id INTEGER PRIMARY KEY,
    chat_row_id INTEGER,
    autotransfer_retry_enabled INTEGER,
    transferred INTEGER,
    face_x INTEGER,
    face_y INTEGER,
    has_streaming_sidecar INTEGER,
    page_count INTEGER,
    thumbnail_height_width_ratio REAL,
    first_scan_sidecar BLOB,
    first_scan_length INTEGER,
    message_url TEXT,
    sticker_flags INTEGER,
    raw_transcription_text TEXT,
    first_viewed_timestamp INTEGER,
    is_animated_sticker INTEGER,
    premium_message INTEGER,
    media_caption TEXT,
    metadata_url TEXT,
    motion_photo_presentation_offset_ms INTEGER,
    qr_url TEXT,
    media_key_domain INTEGER,
    e2ee_media_key BLOB,
    emoji_tags TEXT,
    is_offloaded INTEGER,
    dash_manifest_url TEXT,
    multicast_id TEXT,
    media_job_uuid TEXT,
    transcoded INTEGER,
    file_path TEXT,
    file_size INTEGER,
    suspicious_content INTEGER,
    trim_from INTEGER,
    trim_to INTEGER,
    media_key BLOB,
    media_key_timestamp INTEGER,
    width INTEGER,
    height INTEGER,
    gif_attribution INTEGER,
    direct_path TEXT,
    mime_type TEXT,
    file_length INTEGER,
    media_name TEXT,
    file_hash TEXT,
    media_duration INTEGER,
    enc_file_hash TEXT,
    partial_media_hash TEXT,
    partial_media_enc_hash TEXT,
    original_file_hash TEXT,
    mute_video INTEGER DEFAULT 0,
    doodle_id TEXT,
    media_source_type INTEGER,
    accessibility_label TEXT,
    media_transcode_quality INTEGER DEFAULT 0,
    media_upload_handle TEXT
);

CREATE INDEX IF NOT EXISTS message_media_hash_index ON message_media (file_hash);
CREATE INDEX IF NOT EXISTS message_media_chat_index ON message_media (chat_row_id);
CREATE INDEX IF NOT EXISTS message_media_original_file_hash_index ON message_media (original_file_hash);
```

</details>

### 相关主题

[存储总览与 WAL](../local-storage.md#zh) · [axolotl.db](axolotl.md#zh) · [图片、文件与语音消息](../../media/README.md#zh) · [多设备与数据同步](../../multi-device/README.md#zh)

<a id="en"></a>

## English

### Six Core Tables

`msgstore.db` links message history to receipts and media through JID, chat, and message rows. This page covers 6 core tables, 150 columns, and 20 explicitly created indexes whose creation is unconditional. Other message tables, views, and triggers are outside these 6 tables.

| Table | Columns | Relationships and responsibility |
| --- | --- | --- |
| `jid` | 7 | Decomposed account/device JID fields, type, and raw string |
| `chat` | 53 | Chat address, display/read cursors, unread counts, and chat settings |
| `message` | 24 | Message identifier within a chat, direction, status, time, and body-related fields |
| `receipt_device` | 5 | Receipt records connecting messages to device JIDs |
| `receipt_user` | 6 | Delivery, read, and playback times connecting messages to user JIDs |
| `message_media` | 55 | Media location, key material, file, and processing metadata for a message |

For BLOB and record definitions, see the [storage terminology](../local-storage.md#terms-en). The following sections explain row relationships first, then statuses and time.

### JID, Chat, and Message Relationships

`jid._id` is a database row ID. `user` and `server` are non-null text; `agent/device/type` are separate columns; `raw_string` holds the string representation. Decomposed fields have a unique index on `(user, server, agent, device, type)`, with another unique index on `raw_string`.

In message-database `jid.type`, PN is `0` and LID is `18`; Signal account-domain PN/LID values are `0/1`. For example, LID address queries use `jid.type=18`, not Signal-domain value `1`.

`chat.jid_row_id` identifies the chat JID; `chat.account_jid_row_id` is a separate column with its own uniqueness constraint. `message.chat_row_id` identifies the chat; `sender_jid_row_id` identifies the sender, who is not unconditionally the chat peer. The relationships below describe column uses. The DDL declares no `FOREIGN KEY`, so field names do not establish SQLite-enforced references or cascading deletion.

```text
jid._id <- chat.jid_row_id
chat._id <- message.chat_row_id
jid._id <- message.sender_jid_row_id
message._id <- receipt_user.message_row_id
jid._id <- receipt_user.receipt_user_jid_row_id
message._id <- receipt_device.message_row_id
jid._id <- receipt_device.receipt_device_jid_row_id
message._id <- message_media.message_row_id
```

### chat: Cursors, Counts, and Settings

`display_message_row_id`, `last_message_row_id`, `last_read_message_row_id`, and `last_read_receipt_sent_message_row_id` are distinct display, latest-message, local-read, and sent-read-receipt cursors. Separate `*_sort_id` columns also exist; row IDs, sort values, and timestamps are not interchangeable.

`unseen_message_count`, `unseen_row_count`, `unseen_missed_calls_count`, `unseen_important_message_count`, reaction counts, and comment counts are separate statistics. Archive, hidden, chat-lock, ephemeral-message, limited-sharing, community-dialog, group-type, and business-chat states also have their own columns. Nonzero integers do not all mean the same boolean, and column names do not supply missing enums.

Creation constraints specify `NOT NULL DEFAULT 0` for `unseen_important_message_count`, `group_type`, and `has_new_community_admin_dialog_been_acknowledged`. Other nullable columns must not be assigned the same default by assumption. The DDL below lists all 53 columns and their types.

<a id="messages-en"></a>

### message: Identifiers, Status, and Time

`message._id` is a local row ID; `key_id` is a non-null protocol message identifier. The unique index uses `(chat_row_id, from_me, key_id, sender_jid_row_id)`; `key_id` alone has no database-wide unique index. `from_me` indicates direction; status alone does not establish inbound versus outbound direction.

Direct outbound messages have these status values:

| `message.status` | Meaning | Related user-receipt column |
| --- | --- | --- |
| `4` | Server ACK | Does not mean the peer has received it |
| `5` | Delivered | `receipt_timestamp` |
| `13` | Read | `read_timestamp` |
| `8` | Played | `played_timestamp` |

These numbers are not an ordered rank; `MAX(status)` does not implement every status merge. This mapping covers direct outbound/receipt paths, not all group, inbound, or queued states. This page assigns no numeric queued status or plain-text SQL `message_type`; without the matching business assignment, guessed enums must not be used to fill columns.

`timestamp`, `received_timestamp`, `receipt_server_timestamp`, and `server_sts` are separate columns; `sort_id` is an independent ordering field. Neither one column, its name, nor the unit of network `t` establishes the unit of every time column. Local receipt time also does not establish the peer's actual reading time. Body-related columns include `text_data` and `translated_text`, while media captions are in `message_media.media_caption`.

<a id="receipts-en"></a>

### User and Device Receipts

`receipt_user` uniquely addresses a user by `(message_row_id, receipt_user_jid_row_id)`, with separate `receipt_timestamp`, `read_timestamp`, and `played_timestamp` columns. Receipt statuses `5/13/8` write the corresponding columns. These three times are not interchangeable.

`receipt_device` uniquely addresses a device by `(message_row_id, receipt_device_jid_row_id)`. It has `receipt_device_timestamp` and `primary_device_version`, but no `read_timestamp`. A device-receipt time is not a user's read time, and `primary_device_version` is not a reading flag.

The user-receipt writer puts its received time value into the selected column. Interpreting that value as seconds or milliseconds requires its particular upstream receipt path; this write alone does not define one unit for all native receipt fields. When recording historical data, missing receipt times should remain empty rather than be replaced with local processing times for the peer’s events.

Local reading of an incoming message and sending a read receipt to the peer differ from receiving the peer's user receipt. To ask when the peer read an outbound message, query that message's user receipts. To ask whether the local device read an incoming message, use local read cursors and the corresponding state.

<a id="media-en"></a>

### message_media: Media Keys and File Metadata

`message_media.message_row_id` is the primary key associated with a message; `chat_row_id` separately supports chat-level lookup. File-location fields include `file_path`, `message_url`, `direct_path`, `metadata_url`, `dash_manifest_url`, and `media_upload_handle`. They must not all be assumed to identify the same kind of resource.

Cryptographic-material fields include BLOBs `media_key` and `e2ee_media_key`, plus `media_key_domain` and `media_key_timestamp`. `file_hash`, `enc_file_hash`, `partial_media_hash`, `partial_media_enc_hash`, and `original_file_hash` are distinct digest columns. Names and TEXT/BLOB types alone do not define a KDF, digest encoding, encryption envelope, or substitution between columns.

Other fields cover file size/length, MIME type, dimensions, duration, captions, transcription, thumbnail/scan sidecars, stickers, trimming, transcoding, transfer, and offload state. `motion_photo_presentation_offset_ms` explicitly names milliseconds, but that suffix does not determine the units of other time or duration fields. `mute_video` and `media_transcode_quality` default to `0` without `NOT NULL` constraints.

These columns do not automatically supply the media file. A media row alone does not establish that a local file exists, a download link is valid, or all decryption material is available.

### Query Examples

Parameters below are caller-supplied database row IDs, without real accounts or message contents. This chat query returns protocol IDs, row IDs, ordering values, and status separately:

```sql
SELECT m._id, m.key_id, m.from_me, m.sender_jid_row_id,
       m.status, m.sort_id, m.timestamp,
       j.type AS chat_jid_type
FROM message AS m
JOIN chat AS c ON c._id = m.chat_row_id
JOIN jid AS j ON j._id = c.jid_row_id
WHERE m.chat_row_id = :chat_row_id
ORDER BY m.sort_id, m._id;
```

Distinguish delivery, read, and playback times by user rather than manufacturing a read value from a device receipt:

```sql
SELECT r.receipt_user_jid_row_id, j.type AS user_jid_type,
       r.receipt_timestamp, r.read_timestamp, r.played_timestamp
FROM receipt_user AS r
JOIN jid AS j ON j._id = r.receipt_user_jid_row_id
WHERE r.message_row_id = :message_row_id;
```

These structural queries do not define missing-row repair, time-unit conversion, receipt transmission, or automatic cursor updates.

### Index Conditions and Complete DDL

“Unconditional” describes creation of the 20 indexes, not coverage of every row. `chat_ephemeral_after_read_duration_index` is itself a partial index with `WHERE ephemeral_after_read_duration > 0`.

Three additional indexes—`chat_sort_timestamp_index`, `chat_sort_timestamp_with_hidden`, and `idx_message_media_file_path_v1`—are created according to their respective configuration branches and are not included in the 20 below. Similar names do not justify creating every branch together. The DDL defines only the 6 core tables and their corresponding indexes.

The following preserves the order, types, nullability constraints, and defaults of all 150 columns, plus those 20 explicit indexes by table. Automatic indexes from column-level `UNIQUE` constraints are separate.

<details>
<summary>jid</summary>

```sql
CREATE TABLE jid (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    user TEXT NOT NULL,
    server TEXT NOT NULL,
    agent INTEGER,
    device INTEGER,
    type INTEGER,
    raw_string TEXT
);

CREATE UNIQUE INDEX IF NOT EXISTS jid_key_new_index ON jid ( user, server, agent, device, type );
CREATE UNIQUE INDEX IF NOT EXISTS jid_raw_string_index ON jid (raw_string);
```

</details>

<details>
<summary>chat</summary>

```sql
CREATE TABLE chat (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    jid_row_id INTEGER UNIQUE,
    hidden INTEGER,
    subject TEXT,
    created_timestamp INTEGER,
    display_message_row_id INTEGER,
    last_message_row_id INTEGER,
    last_read_message_row_id INTEGER,
    last_read_receipt_sent_message_row_id INTEGER,
    last_important_message_row_id INTEGER,
    archived INTEGER,
    sort_timestamp INTEGER,
    mod_tag INTEGER,
    gen REAL,
    spam_detection INTEGER,
    unseen_earliest_message_received_time INTEGER,
    unseen_message_count INTEGER,
    unseen_missed_calls_count INTEGER,
    unseen_row_count INTEGER,
    plaintext_disabled INTEGER,
    vcard_ui_dismissed INTEGER,
    change_number_notified_message_row_id INTEGER,
    show_group_description INTEGER,
    ephemeral_expiration INTEGER,
    ephemeral_setting_timestamp INTEGER,
    ephemeral_displayed_exemptions INTEGER,
    ephemeral_disappearing_messages_initiator INTEGER,
    unseen_important_message_count INTEGER NOT NULL DEFAULT 0,
    group_type INTEGER NOT NULL DEFAULT 0,
    last_message_reaction_row_id INTEGER,
    last_seen_message_reaction_row_id INTEGER,
    unseen_message_reaction_count INTEGER,
    unseen_comment_message_count INTEGER,
    growth_lock_level INTEGER,
    growth_lock_expiration_ts INTEGER,
    last_read_message_sort_id INTEGER,
    display_message_sort_id INTEGER,
    last_message_sort_id INTEGER,
    last_read_receipt_sent_message_sort_id INTEGER,
    has_new_community_admin_dialog_been_acknowledged INTEGER NOT NULL DEFAULT 0,
    history_sync_progress INTEGER,
    chat_lock INTEGER,
    chat_origin TEXT,
    participation_status INTEGER,
    account_jid_row_id INTEGER,
    chat_encryption_state INTEGER,
    group_member_count INTEGER,
    limited_sharing INTEGER,
    limited_sharing_setting_timestamp INTEGER,
    is_contact INTEGER,
    ephemeral_after_read_duration INTEGER,
    business_chat_state INTEGER,
    chat_props INTEGER
);

CREATE INDEX IF NOT EXISTS chat_group_type_index ON chat (group_type);
CREATE INDEX IF NOT EXISTS chat_hidden_index ON chat (hidden);
CREATE UNIQUE INDEX IF NOT EXISTS chat_account_jid_row_id_unique_index ON chat (account_jid_row_id);
CREATE INDEX IF NOT EXISTS chat_ephemeral_after_read_duration_index ON chat (ephemeral_after_read_duration) WHERE ephemeral_after_read_duration > 0;
```

</details>

<details>
<summary>message</summary>

```sql
CREATE TABLE message (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    chat_row_id INTEGER NOT NULL,
    from_me INTEGER NOT NULL,
    key_id TEXT NOT NULL,
    sender_jid_row_id INTEGER,
    status INTEGER,
    broadcast INTEGER,
    recipient_count INTEGER,
    participant_hash TEXT,
    origination_flags INTEGER,
    origin INTEGER,
    timestamp INTEGER,
    received_timestamp INTEGER,
    receipt_server_timestamp INTEGER,
    message_type INTEGER,
    text_data TEXT,
    starred INTEGER,
    lookup_tables INTEGER,
    message_add_on_flags INTEGER,
    view_mode INTEGER,
    sort_id INTEGER NOT NULL DEFAULT 0,
    translated_text TEXT,
    view_replies_thread_id INTEGER,
    server_sts INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS message_key_index ON message ( chat_row_id, from_me, key_id, sender_jid_row_id );
CREATE INDEX IF NOT EXISTS message_sort_id_index ON message (sort_id);
CREATE INDEX IF NOT EXISTS message_starred_index ON message (starred);
CREATE INDEX IF NOT EXISTS message_type_chat_index ON message ( chat_row_id, message_type );
CREATE INDEX IF NOT EXISTS message_chat_id_index ON message ( chat_row_id, _id );
CREATE INDEX IF NOT EXISTS message_type_index ON message (message_type);
CREATE INDEX IF NOT EXISTS message_chat_sort_id_index ON message ( chat_row_id, sort_id );
CREATE INDEX IF NOT EXISTS message_starred_sort_id_index ON message ( starred, sort_id );
```

</details>

<details>
<summary>receipt_device</summary>

```sql
CREATE TABLE receipt_device (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_row_id INTEGER NOT NULL,
    receipt_device_jid_row_id INTEGER NOT NULL,
    receipt_device_timestamp INTEGER,
    primary_device_version INTEGER
);

CREATE INDEX IF NOT EXISTS receipt_device_table_device_index ON receipt_device (receipt_device_jid_row_id);
CREATE UNIQUE INDEX IF NOT EXISTS receipt_device_index ON receipt_device ( message_row_id, receipt_device_jid_row_id );
```

</details>

<details>
<summary>receipt_user</summary>

```sql
CREATE TABLE receipt_user (
    _id INTEGER PRIMARY KEY AUTOINCREMENT,
    message_row_id INTEGER NOT NULL,
    receipt_user_jid_row_id INTEGER NOT NULL,
    receipt_timestamp INTEGER,
    read_timestamp INTEGER,
    played_timestamp INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS receipt_user_index ON receipt_user( message_row_id, receipt_user_jid_row_id );
```

</details>

<details>
<summary>message_media</summary>

```sql
CREATE TABLE message_media (
    message_row_id INTEGER PRIMARY KEY,
    chat_row_id INTEGER,
    autotransfer_retry_enabled INTEGER,
    transferred INTEGER,
    face_x INTEGER,
    face_y INTEGER,
    has_streaming_sidecar INTEGER,
    page_count INTEGER,
    thumbnail_height_width_ratio REAL,
    first_scan_sidecar BLOB,
    first_scan_length INTEGER,
    message_url TEXT,
    sticker_flags INTEGER,
    raw_transcription_text TEXT,
    first_viewed_timestamp INTEGER,
    is_animated_sticker INTEGER,
    premium_message INTEGER,
    media_caption TEXT,
    metadata_url TEXT,
    motion_photo_presentation_offset_ms INTEGER,
    qr_url TEXT,
    media_key_domain INTEGER,
    e2ee_media_key BLOB,
    emoji_tags TEXT,
    is_offloaded INTEGER,
    dash_manifest_url TEXT,
    multicast_id TEXT,
    media_job_uuid TEXT,
    transcoded INTEGER,
    file_path TEXT,
    file_size INTEGER,
    suspicious_content INTEGER,
    trim_from INTEGER,
    trim_to INTEGER,
    media_key BLOB,
    media_key_timestamp INTEGER,
    width INTEGER,
    height INTEGER,
    gif_attribution INTEGER,
    direct_path TEXT,
    mime_type TEXT,
    file_length INTEGER,
    media_name TEXT,
    file_hash TEXT,
    media_duration INTEGER,
    enc_file_hash TEXT,
    partial_media_hash TEXT,
    partial_media_enc_hash TEXT,
    original_file_hash TEXT,
    mute_video INTEGER DEFAULT 0,
    doodle_id TEXT,
    media_source_type INTEGER,
    accessibility_label TEXT,
    media_transcode_quality INTEGER DEFAULT 0,
    media_upload_handle TEXT
);

CREATE INDEX IF NOT EXISTS message_media_hash_index ON message_media (file_hash);
CREATE INDEX IF NOT EXISTS message_media_chat_index ON message_media (chat_row_id);
CREATE INDEX IF NOT EXISTS message_media_original_file_hash_index ON message_media (original_file_hash);
```

</details>

### Related Topics

[Storage Overview and WAL](../local-storage.md#en) · [axolotl.db](axolotl.md#en) · [Media Transfer and Voice Messages](../../media/README.md#en) · [Multi-Device and Data Synchronization](../../multi-device/README.md#en)
