# Request Encryption · 参数序列化与加密封装

[注册目录](README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

本页说明内层参数如何序列化为表单，以及加密请求如何使用 `ENC` 承载正文。

### 1. 内层值与编码身份

| 值类别 | 例子 | 处理 |
| --- | --- | --- |
| 普通文本 | cc、in、method | UTF-8与所选表单路径的转义 |
| JSON 文本 | client_metrics | 先序列化 JSON，再将它作为字符串字段 |
| Base64url文本 | authkey、e_ident、expid | 保持无padding表示；不是原始二进制的通用替代 |
| 原始二进制 | id、backup_token、foa_backup_token | 按字节百分号编码；Kotlin 构造器标记预编码身份，避免再次转义 |

缺键、null、空串、0和false不同。[exist明文示意](exist.md#plaintext-example)和[code明文示意](code.md#plaintext-example)均采用展示用字符串，不是可以整份发送的字典。恢复字段显示的 `%HH` 已代表原始字节；不能先对该显示文本再做一次百分号编码。

### 2. 序列化路径

Kotlin HTTP构造器以插入有序Map和“已预编码键”集合区分两类值。按插入顺序用 `&` 连接 `key=value`，不按名字排序；覆盖已有键保留原位置。key始终按Java `URLEncoder(...,"UTF-8")` 编码。

| 输入 | 普通URLEncoder值 | 原始字节预编码值 |
| --- | --- | --- |
| 空格 | `+` | `%20` |
| `*` | `*` | `%2A` |
| `~` | `%7E` | `~` |
| `+` | `%2B` | `%2B` |
| `0x00`字节 | 按文本编码路径处理 | `%00` |

原始字节预编码仅保留字母、数字和 `-._~`，其余用大写 `%HH`。构造器拼接已标记的值时不再URLEncoder。Base64url字段虽然通常没有需转义的字符，仍属于普通文本路径，不因此加入预编码集合。

原生 WAMsys 使用自己的字典和枚举规则。原始字节编码使用 `-._~` 作为不转义的标点字符，已预编码的恢复字段直接拼接。不同通道的枚举和转义可能产生不同的表单字节。

仅展示编码差异的例子：普通字段值 `a b*~` 成为 `note=a+b*%7E`；同样ASCII字节走预编码路径成为 `blob=a%20b%2A~`。预编码片段再转义会错误产生 `%2520`。

### 3. ENC

取新临时X25519密钥对，使用本通道服务器公钥求共享密钥。服务器公钥不是账户 `authkey`、Signal身份或任意别的通道输入。

```text
P = UTF8(serialized_form)
E_public32 = X25519_public(ephemeral_private32)
shared32 = X25519(ephemeral_private32, server_public32)
C || T16 = AES256-GCM(shared32, nonce=zero12, plaintext=P, AAD=empty)
ENC = Base64urlNoPadding(E_public32 || C || T16)
```

共享32字节直接作为AES256 key，没有额外SHA-256或HKDF。GCM tag为16字节，nonce12为零且不放入包络，AAD为空。内部若公钥格式为 `0x05 || public32`，输出只保留raw32；没有公钥长度前缀。整体原始包络长度为 `32 + len(P) + 16`。

每次构造新的加密包络都使用新的临时密钥。

### 4. 外层正文

`Content-Length` 按最终发送正文的字节数计算。

内层参数对象、最终序列化字节、加密包络、外层body与HTTP写入边界须分开。头部和响应处理见[发送流程](sending-requests.md#zh)，请求字段见[exist](exist.md#zh)与[code](code.md#zh)。

[返回注册目录](README.md#zh)

<a id="en"></a>

## English

This page describes how inner parameters are serialized as a form and how encrypted requests use `ENC` to carry the body.

### 1. Values and Encoding Identity

| Category | Examples | Handling |
| --- | --- | --- |
| Ordinary text | cc, in, method | UTF-8 and the chosen form escaping |
| JSON text | client_metrics | Serialize JSON and include it as a string field |
| Base64url text | authkey, e_ident, expid | Preserve unpadded text; not a universal replacement for raw bytes |
| Raw binary | id, backup_token, foa_backup_token | Byte-wise percent encoding; the Kotlin builder tracks preencoded identity to avoid a second escape |

Omission, null, empty text, 0, and false differ. Both the [exist field illustration](exist.md#plaintext-example-en) and [code field illustration](code.md#plaintext-example-en) use display strings, not one sendable dictionary for all flows. `%HH` in recovery examples already represents raw bytes; do not percent-encode that display text again.

### 2. Serialization Paths

The Kotlin HTTP builder uses an insertion-ordered Map and a set of preencoded keys. Join `key=value` pairs with `&` in insertion order, without name sorting; overwriting a key retains its position. Keys always use Java `URLEncoder(...,"UTF-8")`.

| Input | Ordinary URLEncoder value | Preencoded raw-byte value |
| --- | --- | --- |
| Space | `+` | `%20` |
| `*` | `*` | `%2A` |
| `~` | `%7E` | `~` |
| `+` | `%2B` | `%2B` |
| Byte `0x00` | Follow the text-encoding path | `%00` |

Raw-byte preencoding preserves only letters, digits, and `-._~`, using uppercase `%HH` otherwise. Marked values are appended without another URLEncoder pass. Base64url fields remain ordinary text, despite usually requiring no escaping; they are not thereby marked preencoded.

Native WAMsys uses its own dictionary and enumeration rules. Byte encoding leaves the punctuation characters `-._~` unescaped and inserts preencoded recovery fields directly. Enumeration and escaping can produce different form bytes across transports.

For illustration, ordinary `a b*~` yields `note=a+b*%7E`; those same ASCII bytes on the preencoded path yield `blob=a%20b%2A~`. Re-escaping that fragment incorrectly produces `%2520`.

### 3. ENC

Generate an ephemeral X25519 pair and agree with the selected transport's server public key. This key is not account `authkey`, Signal identity, or an arbitrary other channel input.

```text
P = UTF8(serialized_form)
E_public32 = X25519_public(ephemeral_private32)
shared32 = X25519(ephemeral_private32, server_public32)
C || T16 = AES256-GCM(shared32, nonce=zero12, plaintext=P, AAD=empty)
ENC = Base64urlNoPadding(E_public32 || C || T16)
```

Use shared32 directly as the AES256 key, with no extra SHA-256/HKDF. GCM tag16 follows ciphertext; the implicit zero12 nonce is not transmitted, and AAD is empty. If an internal key representation is `0x05 || public32`, retain only raw32 in the output, without a length prefix. Raw envelope length is `32 + len(P) + 16`.

Each new encrypted envelope uses a fresh ephemeral key.

### 4. The Outer Body

`Content-Length` is the byte length of the final body sent over HTTP.

Keep the parameter object, final serialized bytes, encrypted envelope, outer body, and HTTP write distinct. See [sending](sending-requests.md#en) for headers/responses and [exist](exist.md#en)/[code](code.md#en) for field sets.

[Back to registration](README.md#en)
