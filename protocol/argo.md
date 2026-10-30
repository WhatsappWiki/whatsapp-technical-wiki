# Argo · 二进制格式与编解码 · Binary Format and Encoding

[首页](../README.md#zh) · [协议目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

Argo 是用于表示结构化业务值的二进制格式。读取时由 Header 模式与 Wire Schema 决定类型、字段顺序和块布局；写入时从对象生成标量块与 Core。本文覆盖 Argo 1.2 的载荷边界、模式、schema、类型、Python 读写、精确回写及资源限制。

阅读顺序：[载荷边界](#zh-boundaries) → [Header、块与 Core](#zh-header) → [schema 选择](#zh-schema) → [类型与错误](#zh-types) → [Python API](#zh-python) → [业务接入](#zh-business) → [逐字节回写](#zh-rewrite) → [资源限制](#zh-limits)。

<a id="zh-boundaries"></a>

### 1. 载荷边界与节点关系

接收流程先完成传输层与节点层，再处理节点中的业务载荷：

```text
TCP 帧 → Noise 认证解密 → 明文标志与可选解压
       → Protocol Tree Node → 业务节点的 data: bytes → Argo 值
```

帧长度与流式取包见 [TCP 帧协议](tcp-framing.md#zh)，密钥和认证解密见 [Noise](noise.md#zh)，节点标签、属性和叶子字节见 [Protocol Tree Node](protocol-tree-node.md#zh)。只有业务声明或处理器选择 Argo 时才执行最后一步；其他节点的 data 可以采用不同格式。

对声明 `format="argo"` 的结果，输入是该业务节点内完整的原始字节。节点层继续保存 `data: bytes`，不添加 `xmlns:view`、`view:encoding` 或 `view:bytes`。日志中的 XML 和 Base64 只是展示或存档表示；Base64 输入须先还原为 bytes，不把 XML、Base64 文本或 TCP/Noise 外层头传给 Argo 解码器。

<a id="zh-header"></a>

### 2. Header、标量块与 Core

非 InlineEverything 模式将消息分为 Header、可选 UserFlags、标量块和最后的 Core；每个块都带自己的长度 Label：

```text
message := Header [UserFlags]
           (Label(标量块字节数) + 标量块) × N
           Label(Core 字节数) + Core
```

Core 保存值结构、长度、存在性与引用等控制信息，标量块保存按类型或自定义 key 分组的值。块本身不携带 key 文本，解码时按 schema 遍历中的首次访问顺序分配。InlineEverything 模式在 Header/UserFlags 后直接写 Core，标量一并内联，不带上述块长度封装或回引用。

Label 是 signed int64 的 zig-zag 值，再使用最短 varint 编码；例如 Label(0) 为 `00`，Label(1) 为 `02`，Label(7) 为 `0e`，Label(-1) 为 `01`，Label(-2) 为 `03`。块长度不能为负。

下列合成例将整数 7 按 `VARINT` schema 和 Header `0x04` 编码：

```text
04 | 02 | 0e | 00
Header | Int 块长度 1 | 整数 7 | Core 长度 0
```

```python
from whatsapp_argo import Header, read, write

header = Header(out_of_band_field_errors=True, self_describing_errors=False)
raw = write(7, "VARINT", header=header)
assert raw == bytes.fromhex("04 02 0e 00")
assert read(raw, "VARINT").value == 7
```

Header 的模式位如下：

| 位 | Header 参数 | 行为 |
| --- | --- | --- |
| 0 | inline_everything | header 后直接为 Core，值内联，无块长度封装和回引用 |
| 1 | self_describing | 记录/数组/标量包含类型 marker，记录包含字段名 |
| 2 | out_of_band_field_errors | nullable ERROR marker 不携带带内错误列表 |
| 3 | self_describing_errors | ERROR 使用带字段名的自描述对象 |
| 4 | null_terminated_strings | 字符串仍有字节长度，数据后另加零终止字节 |
| 5 | no_deduplication | 禁用标量回引用 |
| 6 | user_flags 非 None | HasUserFlags，后随 Varbit 用户位 |

最高位保留，置位即拒绝。用户位以七个数据位加最低位 continuation 编码，按高位到低位排列，最后右补零；API 将其归一化为七位整数倍。`None` 表示无用户位，空 tuple 表示带一组七个零位。

裸标量使用标准块：String/Bytes 默认去重、Int/Float 不去重、Boolean 始终 Core；裸 FIXED 直接放 Core。自定义 BLOCK 按其 key 和 dedupe 工作，只有 String/Bytes 支持实际去重。DESC 和 BLOCK<DESC> 自身带类型。NTS 不改变 UTF-8 字节长度，也不把内部零字节当作结束符。

无 schema 的 `read` 仅对自描述输入按 DESC/标准块解释。自定义块 key、FIXED 长度和应用类型仍需外部 schema；SelfDescribing 不会在 wire 中补入这些信息。不能据此猜测未知应用 schema。

<a id="zh-schema"></a>

### 3. Wire Schema 与 operation 选择

Wire Schema 描述字段顺序、类型与标量块选择。operation 名称与 doc ID 用于从 store 中选择对应 schema；它们属于业务关联，不能由响应字段名猜测。

下面使用合成 operation 与输入，展示 mapping 直接注入：

```python
from whatsapp_argo import Codec, OperationRegistry, load_schema, load_schema_store

schema = {"type": "RECORD", "fields": [{"name": "count", "of": "VARINT"}]}
registry = OperationRegistry({"ExampleQuery": schema}, {"123": "ExampleQuery"})
codec = registry.codec_for("123")
payload = codec.write({"count": 7})
assert codec.read(payload).value == {"count": 7}
```

schema 支持 `STRING / BOOLEAN / VARINT / FLOAT64 / BYTES / FIXED / BLOCK / NULLABLE / ARRAY / RECORD / DESC / ERROR / PATH / EXTENSIONS`。`FIXED` 需要 `length`；`BLOCK` 需要标量 `of`、`key` 和布尔 `dedupe`；`RECORD.fields` 按顺序排列，每项含 `name/of/omittable`，`omittable` 默认 false。标量名可以使用字符串简写。`read_schema_store` / `write_schema_store` 可直接读写 bytes；未知类型或 schema 属性、重复字段和无效引用均报 `ArgoError`。

未知 doc ID 不回退、不猜 operation。schema 描述 wire 布局，不是 GraphQL 查询文本；本库不自动从 GraphQL SDL/查询推导 schema。保留的 `UNION` marker 未定义为可用 1.2 wire 类型，直接拒绝。

`load_schema` 接受 schema mapping、标量名简写或 JSON 文件路径；`load_schema_store` 接受 operation→schema mapping、JSON store 文件、wire-type store 文件或 bytes。JSON store 可表示为 name→schema mapping，或 `{"type":"STORE","types":[{"name":"ExampleQuery","type":...}]}`。`read_schema_store` / `write_schema_store` 处理二进制 store；JSON 形状与二进制 wire 形状分别解析。

<a id="zh-types"></a>

### 4. 类型、缺失、错误与扩展

| Wire 类型 | Python 值 |
| --- | --- |
| STRING / BOOLEAN / VARINT / FLOAT64 | `str / bool / int64 / float`，严格区分 bool 和 int |
| BYTES / FIXED | bytes；写入也接受 bytearray、memoryview，FIXED 严格检查长度 |
| RECORD / ARRAY | dict / list；写入 ARRAY 也接受 tuple |
| NULLABLE | `None`、普通非空值，或 `FieldErrors` |
| DESC | dict、list、str、bytes、bool、int64、float、None，可嵌套 |
| ERROR | message 必需，可选 locations、path、extensions 的 dict |
| PATH | 字段名 str 与非负数组下标 int 组成的 list |
| EXTENSIONS | 字符串键的 dict，值按 DESC 编码 |

```python
from whatsapp_argo import FieldErrors, Header, NonNull, read, write

error = {
    "message": "temporarily unavailable",
    "locations": [{"line": 2, "column": 3}],
    "path": ["items", 0, "name"],
    "extensions": {"retry": True, "opaque": b"\x00\xff"},
}
# 将 errors 放进真实 schema 的 ARRAY<ERROR> 字段即可；字段名由 schema 定义。
# 带内 nullable 错误由独立对象表达，不能与 None 混淆。
schema = {"type": "NULLABLE", "of": {"type": "STRING"}}
raw = write(FieldErrors(({"message": "failed"},)), schema,
            header=Header(out_of_band_field_errors=False))
assert read(raw, schema).value == FieldErrors(({"message": "failed"},))

# NULLABLE<DESC> 的外层非空、内层 null，使用显式包装。
desc_nullable = {"type": "NULLABLE", "of": {"type": "DESC"}}
assert read(write(NonNull(None), desc_nullable), desc_nullable).value == NonNull(None)
```

omittable 字段不在 dict 中表示 absent；存在且为 `None` 表示 nullable null；空数组、空对象、空字节串均保留。`FieldErrors(())` 表示错误 marker；带外模式只允许空 errors 元组，具体错误应放在响应的 errors 字段。自描述模式不能表示 FieldErrors 与 null 的区别，写入 FieldErrors 会报错；调用者需明确选择 null 或响应错误，库不静默丢弃错误。无法区分的嵌套 nullable 非空内层 null 同样拒绝。

typed PATH 的整数是 schema 中的字段序号或数组下标，API 返回可读字段名。默认以根 RECORD 的 `data` 字段类型为路径根；没有 data 时使用根 RECORD。独立 ERROR/PATH 可传 `path_schema=...`。SelfDescribingErrors 的 path 使用自描述字符串/整数，不需要字段序号。

带内 FieldErrors 的路径默认相对于错误传播到的 nullable 字段；`field_errors.full_paths(prefix)` 可生成完整路径。若对接以响应根解释这些路径的实现，显式选择 `inline_error_paths="root"`。这两个选项都不会从载荷中自动推断。

默认 `error_encoding="omittable"` 使用可省略字段的 typed ERROR：locations/path/extensions 可以 absent。需要 nullable 字段表示时，可显式选 `error_encoding="nullable"`：typed ERROR 必须提供四个键，locations/path/extensions 用 `None` 表示 null，extensions 直接为 DESC Object/Null。两种格式没有模式位区别，不能混读。自描述 ERROR 允许额外字符串键，并按 DESC 完整保留；普通 typed ERROR 拒绝额外键。格式选项可传给 read/write/Codec，Document 会记录并复用。

<a id="zh-python"></a>

### 5. Python read/write API

```python
from whatsapp_argo import Codec, Header, read, write

schema = {
    "type": "RECORD",
    "fields": [
        {"name": "title", "of": {"type": "STRING"}},
        {"name": "count", "of": {"type": "VARINT"}},
        {"name": "extra", "of": {"type": "NULLABLE", "of": {"type": "BYTES"}},
         "omittable": True},
    ],
}
codec = Codec(schema)
raw = codec.write({"title": "hello", "count": 7, "extra": None})
document = codec.read(raw)
assert document.value == {"title": "hello", "count": 7, "extra": None}
assert codec.write(document) == raw

# 顶层接口与 Codec 共用同一实现。
document = read(raw, schema)
document.value["count"] = 8
changed = write(document)  # 每次由当前对象、schema 和 header 重新生成
assert changed != raw

# Header() 默认 0x0c。
# 0x04 只启用带外字段错误，SelfDescribing 关闭。
whatsapp_header = Header(out_of_band_field_errors=True, self_describing_errors=False)
whatsapp_raw = write(document, header=whatsapp_header)
assert whatsapp_raw[0] == 0x04
assert read(whatsapp_raw, schema).value == document.value
```

`read` 返回包含 value、header、schema、layout、path_schema 及格式选项的 `Document`；`write` 可接受普通值加 schema，也可直接接受 Document。`Codec` 在调用间不共享消息块或去重表，可并发复用；不要在调用中修改其配置。解码需要完整消息，所有 scalar blocks 和 Core 必须消费到 EOF，多余块或后缀直接报错。

<a id="zh-business"></a>

### 6. 接入业务结果

响应处理器沿原请求关联 operation 与 schema：先以响应标识找到待处理请求，再根据该请求的 doc ID 选择 Wire Schema，最后解析业务结果字节。响应字段名、数据长度或解码异常都不用于自动切换 operation。

调用者提供 wire schema store、persist ID 映射和业务结果文件。将 `doc_id` 替换为当前请求在映射中存在的值：

```python
from pathlib import Path
from whatsapp_argo import OperationRegistry

registry = OperationRegistry.from_files("wire-types.argo", "persist-ids.json")
doc_id = "123"
codec = registry.codec_for(doc_id)
payload = Path("response.argo").read_bytes()
document = codec.read(payload)
rebuilt = codec.write(document)
```

persist ID 映射支持 `{"123":"ExampleQuery"}`，也支持 `{"ExampleQuery":{"doc_id":"123"}}` 形状；后一种条目可用 `operation_name` 显式指定 operation。重复 doc ID、缺失 schema 或未知 doc ID 都报错。输入文件和业务节点的 bytes 必须对应同一 operation。

Argo 的 BYTES/FIXED 保持为 bytes。JSON 展示属于上层转换：仅在业务规定时对具体字节字段做 Base64 等编码，不把所有 BYTES 自动改成字符串，也不把展示编码混入 wire schema。

结构解析成功与业务状态分开处理。业务错误、请求超时、重试和响应验签由对应处理器维护；单纯读取或重新编码对象不执行网络发送。

<a id="zh-rewrite"></a>

### 7. 默认写入与逐字节回写

默认写入为确定的实现策略：按 schema 顺序写记录，String/Bytes 优先复用首个相等值。Argo 同一语义可有不同的合法字节，例如重复字面量与回引用、字段顺序。

```python
from whatsapp_argo import read, write

schema = {"type": "RECORD", "fields": [{"name": "count", "of": {"type": "VARINT"}}]}
payload = write({"count": 7}, schema)
document = read(payload, schema, preserve_layout=True)
assert write(document, preserve_layout=True) == payload
document.value["count"] = 9
changed = write(document, preserve_layout=True)
```

可选布局只保留字符串/字节的字面量或引用选择、自描述对象/记录顺序和 schema/header 约束；不缓存或直接返回原始报文。写入仍从当前对象生成所有标量块和 Core。修改后的值与旧回引用不相等、更换 schema/header 或改变不兼容的字段结构时会报错，可显式改用默认写入。浮点始终从当前 float 的 IEEE754 位产生，保留负零及 NaN 位模式；不会用相等判断把改值吞掉。schema store 使用 fresh 默认编码，不提供布局保留。

<a id="zh-limits"></a>

### 8. 错误处理与资源限制

`Limits` 可限制消息字节、字符串/FIXED 字节、容器项、深度、块数、用户位数和累计遍历项数。默认消息 16 MiB、单标量 8 MiB、容器 100,000 项、深度 128。读入和构建均受限；标签只接受最短 signed int64 zig-zag varint，拒绝截断、越界和非最短表示。FLOAT64 支持 IEEE754 的无穷、NaN 和负零。配置文件的 I/O 故障保留为标准 `OSError`。

| Limits 参数 | 默认值 | 约束 |
| --- | --- | --- |
| `max_message_bytes` | 16 MiB | 整条消息与配置文件字节数 |
| `max_string_bytes` | 8 MiB | 单个字符串、字节值或 FIXED 的字节数 |
| `max_container_items` | 100,000 | 单个容器项数 |
| `max_depth` | 128 | 递归深度 |
| `max_blocks` | 4096 | 标量块数 |
| `max_user_flag_bits` | 4096 | UserFlags 位数 |
| `max_total_values` | 1,000,000 | 累计遍历项预算 |

非法 schema、类型不匹配、长度或引用越界、重复字段、无效 UTF-8、截断、未消费块以及多余后缀抛出 `ArgoError`。`Limits` 参数须为非负整数。`Codec` 每次调用使用独立的消息状态，可复用；并发调用时不修改其配置。

### 相关主题

[TCP 帧协议](tcp-framing.md#zh) · [Noise 握手与加密传输](noise.md#zh) · [Protocol Tree Node](protocol-tree-node.md#zh) · [消息发送与接收](../messaging/README.md#zh)

[English](#en)

<a id="en"></a>

## English

Argo is a binary format for structured business values. Header modes and the Wire Schema determine types, field order and block layout when reading; writing generates scalar blocks and Core from objects. This article covers Argo 1.2 payload boundaries, modes, schemas, types, Python read/write, exact rewriting and resource limits.

Read in order: [payload boundaries](#en-boundaries) → [header, blocks and Core](#en-header) → [schema selection](#en-schema) → [types and errors](#en-types) → [Python API](#en-python) → [business integration](#en-business) → [exact rewriting](#en-rewrite) → [resource limits](#en-limits).

<a id="en-boundaries"></a>

### 1. Payload Boundaries and Nodes

Receive processing completes the transport and node layers before interpreting the business payload:

```text
TCP frame → Noise authenticated decryption → plaintext flags and optional decompression
          → Protocol Tree Node → business-node data: bytes → Argo value
```

See [TCP Framing](tcp-framing.md#en) for lengths and stream handling, [Noise](noise.md#en) for keys and authenticated decryption, and [Protocol Tree Node](protocol-tree-node.md#en) for tags, attributes and leaf bytes. The last step applies only when the business declaration or handler selects Argo; other nodes can use different data formats.

For a result declaring `format="argo"`, input is the complete raw payload inside that business node. The node layer retains `data: bytes` without adding `xmlns:view`, `view:encoding` or `view:bytes`. Logged XML and Base64 are display or archival representations. Decode Base64 to bytes first; do not pass XML, Base64 text or outer TCP/Noise headers to the Argo decoder.

<a id="en-header"></a>

### 2. Header, Scalar Blocks and Core

Outside InlineEverything mode, a message contains the Header, optional UserFlags, scalar blocks and a final Core. Each block has its own length Label:

```text
message := Header [UserFlags]
           (Label(scalar-block byte length) + scalar block) × N
           Label(Core byte length) + Core
```

Core holds value structure, lengths, presence and reference controls. Scalar blocks group values by type or a custom key. Blocks do not carry key text; schema traversal assigns them on first access. InlineEverything writes Core directly after Header/UserFlags, placing scalars inline without the block-length framing or references above.

A Label is a zig-zag signed int64 encoded as a minimal varint. Label(0) is `00`, Label(1) is `02`, Label(7) is `0e`, Label(-1) is `01`, and Label(-2) is `03`. Block lengths cannot be negative.

This synthetic example encodes integer 7 with a `VARINT` schema and header `0x04`:

```text
04 | 02 | 0e | 00
Header | Int-block length 1 | integer 7 | Core length 0
```

```python
from whatsapp_argo import Header, read, write

header = Header(out_of_band_field_errors=True, self_describing_errors=False)
raw = write(7, "VARINT", header=header)
assert raw == bytes.fromhex("04 02 0e 00")
assert read(raw, "VARINT").value == 7
```

Header mode bits are:

| Bit | Header parameter | Behavior |
| --- | --- | --- |
| 0 | inline_everything | Core directly after header; inline scalars, no block lengths/references |
| 1 | self_describing | Type markers, record names and container labels |
| 2 | out_of_band_field_errors | Nullable ERROR carries no in-band error list |
| 3 | self_describing_errors | ERROR is a self-describing named object |
| 4 | null_terminated_strings | Byte length remains; a zero byte follows string data |
| 5 | no_deduplication | Disable scalar backreferences |
| 6 | user_flags is not None | HasUserFlags followed by Varbit user bits |

The reserved high bit must be zero. User flags use seven data bits plus the low continuation bit, ordered most-significant first and right-padded with zeros. The API normalizes to whole seven-bit groups. None means no user flags; an empty tuple encodes seven zero bits.

Bare scalars use standard blocks: String/Bytes default to deduplication, Int/Float do not, Boolean is always in Core. Bare FIXED is directly in Core. Custom BLOCK uses its key/dedupe; only String/Bytes support actual deduplication. DESC and BLOCK<DESC> carry their own type labels. NTS neither changes UTF-8 lengths nor treats embedded zero bytes as terminators.

Schema-free `read` interprets self-describing input as DESC with standard scalar blocks. Custom block keys, FIXED lengths and application types still need an external schema; SelfDescribing does not add this information to the wire. Do not use it to guess an unknown application schema.

<a id="en-schema"></a>

### 3. Wire Schemas and Operation Selection

A Wire Schema describes field order, types and scalar-block selection. Operation names and document IDs select the appropriate schema from a store; they belong to business correlation and cannot be guessed from response field names.

This synthetic operation and value illustrate direct mapping injection:

```python
from whatsapp_argo import Codec, OperationRegistry, load_schema, load_schema_store

schema = {"type": "RECORD", "fields": [{"name": "count", "of": "VARINT"}]}
registry = OperationRegistry({"ExampleQuery": schema}, {"123": "ExampleQuery"})
codec = registry.codec_for("123")
payload = codec.write({"count": 7})
assert codec.read(payload).value == {"count": 7}
```

Supported wire types: `STRING / BOOLEAN / VARINT / FLOAT64 / BYTES / FIXED / BLOCK / NULLABLE / ARRAY / RECORD / DESC / ERROR / PATH / EXTENSIONS`. FIXED requires `length`. BLOCK requires scalar `of`, `key`, and boolean `dedupe`. Ordered RECORD fields contain `name/of/omittable`, with omittable defaulting to false. Scalar names also accept string shorthand. `read_schema_store` / `write_schema_store` operate on bytes. Unknown types/properties, duplicate fields and invalid references raise `ArgoError`.

Unknown doc IDs never select a fallback or guessed operation. A wire schema is not GraphQL query text; this library does not derive schemas from GraphQL SDL/queries. The reserved UNION marker is not an available 1.2 wire type and is rejected.

`load_schema` accepts a schema mapping, scalar shorthand or JSON file path. `load_schema_store` accepts an operation→schema mapping, JSON store file, wire-type store file or bytes. JSON stores can use a name→schema mapping or `{"type":"STORE","types":[{"name":"ExampleQuery","type":...}]}`. `read_schema_store` / `write_schema_store` handle binary stores; JSON and binary wire shapes are parsed separately.

<a id="en-types"></a>

### 4. Types, Absence, Errors and Extensions

| Wire type | Python value |
| --- | --- |
| STRING / BOOLEAN / VARINT / FLOAT64 | `str / bool / int64 / float`, with bool distinct from int |
| BYTES / FIXED | bytes; writing also accepts bytearray and memoryview; FIXED enforces length |
| RECORD / ARRAY | dict / list; writing ARRAY also accepts tuple |
| NULLABLE | `None`, an ordinary non-null value, or `FieldErrors` |
| DESC | Nested dict, list, str, bytes, bool, int64, float, None |
| ERROR | dict with required message and optional locations, path, extensions |
| PATH | list of str field names and nonnegative int array indices |
| EXTENSIONS | dict with string keys and DESC values |

```python
from whatsapp_argo import FieldErrors, Header, NonNull, read, write

error = {
    "message": "temporarily unavailable",
    "locations": [{"line": 2, "column": 3}],
    "path": ["items", 0, "name"],
    "extensions": {"retry": True, "opaque": b"\x00\xff"},
}
# Put errors into the actual schema's ARRAY<ERROR> field; schema defines its name.
schema = {"type": "NULLABLE", "of": {"type": "STRING"}}
raw = write(FieldErrors(({"message": "failed"},)), schema,
            header=Header(out_of_band_field_errors=False))
assert read(raw, schema).value == FieldErrors(({"message": "failed"},))

desc_nullable = {"type": "NULLABLE", "of": {"type": "DESC"}}
assert read(write(NonNull(None), desc_nullable), desc_nullable).value == NonNull(None)
```

An absent omittable field is missing from the dict; a present `None` is nullable null. Empty arrays, objects and byte strings remain present. `FieldErrors(())` is the error marker. Out-of-band mode permits only an empty error tuple; put detailed errors in the response errors field. Self-describing mode cannot distinguish FieldErrors from null, so writing FieldErrors raises: callers must explicitly choose null or response errors. The library does not silently discard errors. An unrepresentable non-null nested nullable whose inner value is null is also rejected.

Typed PATH integers are schema field ordinals or array indices; the API exposes readable names. The default path root is the root RECORD's data field type, or that RECORD if no data field exists. Standalone ERROR/PATH can supply `path_schema=...`. SelfDescribingErrors encodes paths as self-describing string/integer lists, without field ordinals.

In-band FieldErrors paths default to being relative to the nullable field where propagation stops. `field_errors.full_paths(prefix)` produces complete paths. For an implementation that interprets these paths from the response root, explicitly select `inline_error_paths="root"`. Neither behavior is inferred from a payload.

The default `error_encoding="omittable"` uses omittable typed ERROR fields: locations/path/extensions can be absent. For nullable fields, explicit `error_encoding="nullable"` requires all four typed ERROR keys, uses None for null locations/path/extensions, and encodes extensions directly as DESC Object/Null. No mode bit distinguishes these formats, so they cannot be mixed implicitly. Self-describing ERROR permits additional string keys and preserves them as DESC; ordinary typed ERROR rejects extra keys. Pass format options to read/write/Codec; Document records and reuses them.

<a id="en-python"></a>

### 5. Python read/write API

```python
from whatsapp_argo import Codec, Header, read, write

schema = {
    "type": "RECORD",
    "fields": [
        {"name": "title", "of": {"type": "STRING"}},
        {"name": "count", "of": {"type": "VARINT"}},
        {"name": "extra", "of": {"type": "NULLABLE", "of": {"type": "BYTES"}},
         "omittable": True},
    ],
}
codec = Codec(schema)
raw = codec.write({"title": "hello", "count": 7, "extra": None})
document = codec.read(raw)
assert document.value == {"title": "hello", "count": 7, "extra": None}
assert codec.write(document) == raw

document = read(raw, schema)
document.value["count"] = 8
changed = write(document)  # Rebuild from current values, schema and header.
assert changed != raw

# Header() defaults to 0x0c.
# 0x04 enables only out-of-band field errors; SelfDescribing is off.
whatsapp_header = Header(out_of_band_field_errors=True, self_describing_errors=False)
whatsapp_raw = write(document, header=whatsapp_header)
assert whatsapp_raw[0] == 0x04
assert read(whatsapp_raw, schema).value == document.value
```

`read` returns a `Document` containing value, header, schema, layout, path_schema and format options. `write` accepts either a value plus schema or a Document. `Codec` shares no message blocks or reference tables between calls and can be reused concurrently; do not mutate its configuration during a call. Decoding requires one complete message. Every scalar block and Core must reach EOF; unused blocks and suffixes are errors.

<a id="en-business"></a>

### 6. Business-Result Integration

The response handler carries forward the original request's operation and schema: find the pending request by response identifier, select its Wire Schema from the request's doc ID, then parse the business-result bytes. Field names, payload length and decoding exceptions do not automatically select another operation.

The caller supplies a wire schema store, persist-ID mapping and business-result file. Replace `doc_id` with a value present in the mapping for the current request:

```python
from pathlib import Path
from whatsapp_argo import OperationRegistry

registry = OperationRegistry.from_files("wire-types.argo", "persist-ids.json")
doc_id = "123"
codec = registry.codec_for(doc_id)
payload = Path("response.argo").read_bytes()
document = codec.read(payload)
rebuilt = codec.write(document)
```

Persist-ID mappings accept `{"123":"ExampleQuery"}` and `{"ExampleQuery":{"doc_id":"123"}}`. Entries in the latter shape can explicitly select an operation with `operation_name`. Duplicate doc IDs, missing schemas and unknown doc IDs raise errors. Input files and business-node bytes must correspond to the same operation.

Argo BYTES/FIXED remain bytes. JSON display is an upper-layer conversion: apply Base64 or another encoding to a specific byte field only when its business format requires it. Do not automatically convert every BYTES value to text or mix display encoding into the wire schema.

Keep structural decoding separate from business state. Business errors, request timeouts, retries and response-signature checks belong to their respective handlers; reading or re-encoding an object does not send a network request.

<a id="en-rewrite"></a>

### 7. Default Encoding and Exact Rewriting

The default writer follows a deterministic policy: schema record order and the first matching String/Bytes reference. Equivalent Argo values can have different valid bytes, such as repeated literals versus references or record order.

```python
from whatsapp_argo import read, write

schema = {"type": "RECORD", "fields": [{"name": "count", "of": {"type": "VARINT"}}]}
payload = write({"count": 7}, schema)
document = read(payload, schema, preserve_layout=True)
assert write(document, preserve_layout=True) == payload
document.value["count"] = 9
changed = write(document, preserve_layout=True)
```

The optional layout records literal/reference choices, self-describing object/record order, and schema/header constraints. It never caches or returns an original message. Every scalar block and Core is generated from current values. A changed reference value, changed schema/header or incompatible field structure raises; callers can explicitly use the default writer. Floats always come from the current float's IEEE754 bits, preserving signed zero and NaN payloads without using equality to suppress edits. Schema-store writes are fresh default encoding and have no layout-preserving option.

<a id="en-limits"></a>

### 8. Errors and Resource Limits

`Limits` bounds message/scalar bytes, container items, depth, blocks, user bits and cumulative traversal items. Defaults: 16 MiB message, 8 MiB scalar, 100,000 items per container, depth 128. Reads and construction are bounded. Labels require minimal signed int64 zig-zag varints; truncation, overflow and nonminimal encodings fail. FLOAT64 supports IEEE754 infinity, NaN and negative zero. Configuration file I/O failures remain standard `OSError` exceptions.

| Limits parameter | Default | Bound |
| --- | --- | --- |
| `max_message_bytes` | 16 MiB | Complete message and configuration-file bytes |
| `max_string_bytes` | 8 MiB | Bytes per string, byte value or FIXED |
| `max_container_items` | 100,000 | Items per container |
| `max_depth` | 128 | Recursion depth |
| `max_blocks` | 4096 | Scalar blocks |
| `max_user_flag_bits` | 4096 | UserFlags bits |
| `max_total_values` | 1,000,000 | Cumulative traversal-item budget |

Invalid schemas, type mismatches, out-of-range lengths/references, duplicate fields, invalid UTF-8, truncation, unused blocks and trailing bytes raise `ArgoError`. `Limits` parameters must be nonnegative integers. Each `Codec` call uses independent message state and the codec can be reused; do not modify its configuration during concurrent calls.

### Related Topics

[TCP Framing](tcp-framing.md#en) · [Noise Handshakes and Encrypted Transport](noise.md#en) · [Protocol Tree Node](protocol-tree-node.md#en) · [Sending and Receiving Messages](../messaging/README.md#en)

[中文](#zh)
