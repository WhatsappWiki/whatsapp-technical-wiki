# WhatsApp `/v2/register` · 提交注册验证

[注册目录](README.md#zh) · [Registration index](README.md#en) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

### 验证流程

`POST /v2/register` 在 HTTP 注册流程中提交验证材料。[exist](exist.md#zh)检查号码，[code](code.md#zh)申请验证码或启动认证路线，register 提交该路线产生的材料。验证模式为 `4` 或 `5` 时，号码验证直接走 `verify_pn` IQ 分支，由该分支处理结果。

HTTP 提交任务保留本次号码、凭据、验证模式和录入方式，异步准备完成后继续使用这些输入。密码路线先加密输入，自动配置路线先向认证组件取得证明。部分安全码验证和重置流程也复用此端点。

下表介绍常见内层字段。实际集合由验证路线、设备状态、应用配置和通道决定。文本使用普通表单编码，字节按原始字节编码。

### 号码、凭据与流程状态

| 字段 | 来源、表示与出现条件 |
| --- | --- |
| `cc` | 本次待验证号码的电话区号。文本，例如 `1`。 |
| `in` | 区号下的本地号码。保留为文本，组包时保留前导零。 |
| `code` | 当前提交的验证码或该路线定义的凭据文本。密码路线写入加密结果，准备失败时终止提交。部分安全码流程使用空串。 |
| `entered` | 调用方捕获的录入方式，十进制字节。手动提交可为 `1`，短信接收回调提交可为 `2`。 |
| `method` | HTTP 组包时筛选的验证路线文本。验证模式为 `6` 时才保留 `sms`、`voice`、`flash`、`email_otp`。识别出的 `password`、`passkey` 等路线可直接保留，筛选结果为 null 时省略。 |
| `auth_response` | 自动配置组件对 challenge 返回的证明字节，先转 Base64url 文本。该路线取得空响应时终止提交。 |
| `context` | 桥接层可先写认证上下文文本。附加字段可用合格的邀请、解禁或 Web OTP 上下文覆盖，优先级按此顺序。调用者显式提供的上下文优先于偏好推导值。 |
| `mistyped` | 号码录入时保存的纠错状态。非 null 时以字节附加，提交阶段沿用已保存状态。 |
| `client_metrics` | 紧凑 JSON 字节，包含验证尝试数 `attempts`，可带下载归因及闪呼结束、缺少呼叫 ID、呼叫 ID 无效等布尔状态。 |
| `passkey_login_status` | 已保存的 passkey 登录阶段映射成字节文本，例如 `not_attempted` 或 `no_credentials`。当前请求可使用其它验证路线。 |
| `cred_token` | 启用相应凭据流程后，用 `cc + in` 查找保存的凭据令牌，命中时以字节加入。 |
| `login` | 复用桥接层传入的两步验证登录键文本。普通验证码提交省略。 |
| `type` | 复用桥接层传入的用户类型枚举值，十进制文本。普通验证码提交省略。 |
| `security_code` | 端点被复用于验证安全码时，非密码路线可将安全码作为附加字节，并使用空 `code`。 |
| `reset` | 安全码重置路线的调用方文本，作为附加字节。 |
| `wipe_token` | 安全码重置路线的可选令牌，非 null 时作为附加字节。 |
| `vname` | 可选的已验证名称证书。调用方提供 protobuf 对象，序列化后先转 Base64url，再以所得 ASCII 字节附加。 |

### 区域、会话和恢复材料

| 字段 | 来源、表示与出现条件 |
| --- | --- |
| `lg` | 应用语言文本。 |
| `lc` | 应用地区文本，独立于语言和号码区号读取。 |
| `fdid` | 持久 phone ID 的 UUID 文本。 |
| `expid` | 持久性能设备标识。UUID 以 16 个大端字节转 Base64url。优先复用已保存值，缺失时根据 Android ID 或随机源生成。 |
| `access_session_id` | 访问会话管理器的 UUID。非 null 时以 16 个大端字节转 Base64url，会话重置可以换值。 |
| `aid` | Android 设备标识的派生表示。 |
| `fid` | 可用的 phoneyid 记录，格式为 `ID\|时间戳秒数`，以字节附加。使用独立于 `fdid` 的存储。 |
| `id` | 从本地 `rc2` 读取、结合当前号码解密的恢复材料。缺少可用记录并进入生成分支时写回新材料。表单接收原始字节。 [恢复文件](android-inputs.md#recovery-record) |
| `backup_token` | 直接解码注册偏好 `token_used_during_reg`，复用非空字节，零长时生成并保存新材料。表单接收解码后的字节。 [exist 的备份选择](exist.md#zh) |
| `hasinrc` | 组包时检查 `rc2` 文件是否存在，写入 `1` 或 `0` 字节。检查发生在读取或生成 `id` 之后。 |
| `old_phone_number` | 取得旧账号记录时，拼接旧号码的区号和本地号码，以字节附加。 |
| `entrypoint` | 账户状态可产生 `create_paa` 或 `suma` 字节标记。两次独立的状态读取可能使后一次写入覆盖前一次。 |
| `tos_version` | 相应功能启用且服务条款接受记录为正值时写固定 ASCII `5`。 |

读取 `rc2` 时，外层记录缺失可进入生成分支，部分损坏或解密异常会中断准备。备份偏好的零长回退与无效 Base64 的解码失败也分别处理。

### 公共密钥材料

下列字段取自连接与 Signal 的公用材料，编码为 Base64url 文本。经典签名预密钥的 ID、公钥和签名来自同一记录。Kotlin 路线若无法取得可用的密钥包，可继续组包，缺失的材料不填零。

| 字段 | 来源与编码前表示 |
| --- | --- |
| `authkey` | 客户端静态连接密钥的 32 字节公钥。 |
| `e_ident` | Signal 身份公钥的原始 32 字节，去掉类型前缀。 |
| `e_keytype` | 单字节 `0x05`，编码为 `BQ`。 |
| `e_regid` | Signal 注册编号的 4 字节大端表示。 |
| `e_skey_id` | 经典签名预密钥记录编号的低 24 位，3 字节大端。 |
| `e_skey_val` | 该签名预密钥公钥的原始 32 字节，去掉类型前缀。 |
| `e_skey_sig` | 该记录携带的签名原始字节。 |
| `e_pq_last_resort_id` | 可用的 PQ last-resort 预密钥记录编号的低 24 位，3 字节大端。PQ 三项齐备才一并附加。 |
| `e_pq_last_resort_val` | 同一 PQ 记录的 KEM 公钥字节，长度由 KEM 决定。 |
| `e_pq_last_resort_sig` | 同一 PQ 记录的身份签名字节。 |

### 网络、设备与归因

| 字段 | 来源、表示与出现条件 |
| --- | --- |
| `mcc` | 当前电话网络运营商代码的前三位，缺失或格式不符时用 `000`。 |
| `mnc` | 当前电话网络运营商代码的后两或三位，按三位十进制格式输出，缺失或格式不符时用 `000`。 |
| `sim_mcc` | SIM 归属运营商的 MCC，按相同格式独立读取。 |
| `sim_mnc` | SIM 归属运营商的 MNC，按相同格式独立读取。 |
| `network_operator_name` | 电话 API 返回的当前网络运营商显示名，缺失归为空串。 |
| `sim_operator_name` | 电话 API 返回的 SIM 归属运营商显示名，缺失归为空串。 |
| `has_play_store` | 查询 `com.android.vending` 能否取得包信息，以 `true` 或 `false` 字节表示。 |
| `network_radio_type` | 网络快照和蜂窝 subtype 映射出的分类编号。 [网络分类](android-inputs.md#network-type) |
| `simnum` | 能否取得至少 6 个字符的本机号码，写入 `1` 或 `0` 字节。 |
| `pid` | 当前进程编号的十进制字节。 |
| `rc` | 应用发布渠道编号的十进制字节，正式发布渠道为 `0`。 |
| `device_ram` | 系统总内存除以 2³⁰ 后按小数格式输出，可按实例缓存。 |
| `sim_type` | 启用电话环境采集时读取。无电话服务为 `2`，无 SIM 为 `0`，其它状态为 `1`。 |
| `airplane_mode_type` | 同组开关启用时读取飞行模式，归一化为 `0` 或 `1`。 |
| `cellular_strength` | 同组开关启用时读取蜂窝信号等级，通常为 `0`–`4`，缺少支持或相应对象时用 `5`。 |
| `roaming_type` | 同组开关启用且能读取网络快照时加入。非漫游 `0`，漫游 `1`，无电话服务 `2`。 |
| `sim_status` | 注册偏好中的 SIM 号码检查结果，非 null 时附加。该值可由前置流程缓存。 |
| `db` | 启用相应采集时读取系统的 `adb_enabled` 整数设置，缺键默认 `0`。 |
| `first_api_level` | 启用相应采集且设备出厂 API 级别为正数时附加十进制字节。 |
| `advertising_id` | 广告服务返回的可选文本，取决于国家和所用 provider。读取限制广告追踪标记的分支可据此省略。 [广告标识](android-inputs.md#advertising-id) |
| `preloads_app_manager_id` | 从安装归因组件或对应缓存读取的 App Manager 标识，非 null 时附加。 |
| `preloads_attribution` | 从独立归因来源或其缓存读取的 JSON 文本，非 null 时附加。 |

### 挑战材料

验证路线也可附带 `recaptcha` 校验材料。

### 表单与传输

Kotlin 路线先写号码、公共标识、恢复材料与验证凭据，尝试加入公钥包，再合并附加参数。附加参数可以覆盖同名键，覆盖保留该键的插入位置。普通文本采用 UTF-8 表单编码，字节值先按原始字节做百分号编码，随后直接拼入表单。对已编码的值再次编码会把 `%HH` 变成 `%25HH`。WAMsys 路线则由原生层合并任务参数和通道字段，最终表单顺序和应用头取决于所选通道。 [请求序列化](request-encryption.md#zh) · [发送与响应](sending-requests.md#zh)

通道合并后，内层字段经表单序列化并按路线封装。加密请求使用 `ENC` 承载加密正文。参见[请求封装](request-encryption.md#zh)与[发送流程](sending-requests.md#zh)。

`Content-Length` 按最终正文的字节数计算。

下面的合成片段展示编码前的字段值。示例 JSON 用转义符表示 `client_metrics` 字符串中的引号。

```json
{
  "cc": "1",
  "in": "2025550142",
  "code": "000000",
  "entered": "1",
  "client_metrics": "{\"attempts\":1}"
}
```

表单序列化时，`client_metrics` 的值以 UTF-8 字节做一次百分号编码。对应的内层表单片段如下。

```text
client_metrics=%7B%22attempts%22%3A1%7D
```

### 响应与后续状态

| 响应信息 | 处理含义 |
| --- | --- |
| `status=ok` | register 的业务成功分支。客户端还可读取可选的 `login`、`lid` 等身份材料，继续账户初始化。 |
| `status=fail` 与 `reason` | 客户端按原因处理，例如 `mismatch`、`stale` 或 `too_many_guesses`。HTTP 200 可以携带这些业务失败。 |
| `retry_after`、`guess_wait`、`sms_wait`、`voice_wait`、`flash_wait` | 可选的业务等待信息。HTTP 连接重试由通道另行处理。 |
| `second_factor_methods`、`security_code_set`、`reset_method`、`wipe_token` | 可选的后续认证和恢复信息，依响应状态进入相应流程。 |

HTTP 状态、JSON 解析和业务状态分别判断。应用可能因无法读取或解析响应而得到本地错误。验证路线返回第二因素或恢复要求时，流程还需处理对应状态。 [发送与响应](sending-requests.md#zh)

<a id="en"></a>

## English

### Verification Flow

`POST /v2/register` submits verification material in the HTTP registration flow. [exist](exist.md#en) checks the number, [code](code.md#en) requests a code or starts an authentication route, and register submits the resulting material. Verification modes `4` and `5` use the `verify_pn` IQ branch, which handles its own result.

The HTTP submission task captures the number, credential, verification mode, and entry method for use after asynchronous preparation. The password route first encrypts the input, and the autoconf route first obtains proof from its authentication provider. Some security-code verification and reset flows also reuse this endpoint.

The tables describe common inner fields. The actual set depends on the route, device state, app configuration, and transport. Text uses ordinary form encoding. Byte values are encoded from their raw bytes.

### Number, Credentials, and Flow State

| Field | Source, representation, and inclusion |
| --- | --- |
| `cc` | Calling code of the number being verified. Text, for example `1`. |
| `in` | National number under that calling code. It stays text, preserving leading zeros during assembly. |
| `code` | The submitted code or the credential text defined by the route. The password route writes the encrypted result and stops if preparation fails. Some security-code flows use an empty string. |
| `entered` | Entry method captured by the caller, in decimal bytes. A manual submission can use `1`, and submission from an SMS receipt callback can use `2`. |
| `method` | Verification route text selected during HTTP assembly. Only mode `6` retains `sms`, `voice`, `flash`, and `email_otp`. Recognized routes such as `password` and `passkey` can be retained directly. A null selection is omitted. |
| `auth_response` | Proof bytes returned by the autoconf provider for a challenge, encoded as Base64url text. This route stops submission if the response is empty. |
| `context` | The bridge can first set authentication context as text. Additional fields can override it with an eligible invite, unban, or web OTP context, in that order of priority. An explicit caller context takes precedence over preference-derived values. |
| `mistyped` | Number correction state saved during entry. A non-null value is appended as bytes and reused during submission. |
| `client_metrics` | Compact JSON bytes containing verification `attempts`, with optional download attribution and booleans for flash-call completion, missing call ID, or invalid call ID. |
| `passkey_login_status` | Saved passkey stage mapped to text bytes such as `not_attempted` or `no_credentials`. The current request can use another verification route. |
| `cred_token` | When the credential flow is enabled, look up the saved token by `cc + in` and add a matching value as bytes. |
| `login` | Two-step verification login key text supplied to the shared bridge. Ordinary code submission omits it. |
| `type` | User-type enum value supplied to the shared bridge, in decimal text. Ordinary code submission omits it. |
| `security_code` | When the endpoint handles security-code verification, a non-password route can add the security code as bytes and use an empty `code`. |
| `reset` | Caller-provided text for the security-code reset route, appended as bytes. |
| `wipe_token` | Optional token for the security-code reset route, appended as bytes when non-null. |
| `vname` | Optional verified-name certificate. Serialize the caller-provided protobuf, convert it to Base64url, and append the resulting ASCII bytes. |

### Locale, Session, and Recovery Material

| Field | Source, representation, and inclusion |
| --- | --- |
| `lg` | Application language text. |
| `lc` | Application region text, read separately from language and calling code. |
| `fdid` | Persistent phone ID as UUID text. |
| `expid` | Persistent performance-device ID. Encode the 16 big-endian UUID bytes as Base64url. Reuse the saved value first, or derive one from Android ID or randomness when missing. |
| `access_session_id` | UUID from the access-session manager. A non-null value becomes 16 big-endian bytes encoded as Base64url. A session reset can change it. |
| `aid` | A derived Android device identifier. |
| `fid` | Available phoneyid record, formatted as `ID\|timestamp_seconds` and appended as bytes. It uses storage separate from `fdid`. |
| `id` | Recovery material read from local `rc2` and decrypted using the current number. The generation branch saves new material when no usable record is available. The form takes the raw bytes. [Recovery file](android-inputs.md#en-recovery-record) |
| `backup_token` | Decode the registration preference `token_used_during_reg` directly, reuse nonempty bytes, or generate and save material for a zero-length value. The form takes the decoded bytes. [Backup selection in exist](exist.md#en) |
| `hasinrc` | Checks for the `rc2` file during assembly and writes `1` or `0` as bytes. This check follows reading or generating `id`. |
| `old_phone_number` | When an old account record is available, concatenate its calling code and national number and append the result as bytes. |
| `entrypoint` | Account state can produce a `create_paa` or `suma` byte marker. Two separate state reads can let the second write replace the first. |
| `tos_version` | Fixed ASCII `5` when the feature is enabled and the terms-acceptance record is positive. |

A missing outer `rc2` record can enter the generation branch. Some forms of corruption or decryption errors abort preparation. The backup preference handles zero-length fallback and invalid Base64 decoding as separate cases.

### Public Key Material

These fields come from the connection and Signal public material and use Base64url text. The classic signed-prekey ID, public key, and signature come from one record. The Kotlin route can continue assembling the request if a usable key bundle is unavailable. Missing material is left out.

| Field | Source and representation before encoding |
| --- | --- |
| `authkey` | The 32-byte public half of the client static connection key. |
| `e_ident` | The raw 32-byte Signal identity public key with the type prefix removed. |
| `e_keytype` | The byte `0x05`, encoded as `BQ`. |
| `e_regid` | The Signal registration number as four big-endian bytes. |
| `e_skey_id` | The low 24 bits of the classic signed-prekey record ID, as three big-endian bytes. |
| `e_skey_val` | The raw 32-byte public key from that record with the type prefix removed. |
| `e_skey_sig` | The raw signature bytes from that record. |
| `e_pq_last_resort_id` | The low 24 bits of an available PQ last-resort prekey record ID, as three big-endian bytes. All three PQ fields are appended together when available. |
| `e_pq_last_resort_val` | The KEM public key bytes from the same PQ record. The KEM determines their length. |
| `e_pq_last_resort_sig` | The identity signature bytes from the same PQ record. |

### Network, Device, and Attribution

| Field | Source, representation, and inclusion |
| --- | --- |
| `mcc` | The first three digits of the serving network operator code, with `000` for missing or malformed input. |
| `mnc` | The last two or three digits of the serving network operator code, formatted as three decimal digits, with `000` for missing or malformed input. |
| `sim_mcc` | The SIM home operator MCC, read independently using the same format. |
| `sim_mnc` | The SIM home operator MNC, read independently using the same format. |
| `network_operator_name` | Display name for the serving network from the telephony API. Missing names become an empty string. |
| `sim_operator_name` | Display name for the SIM home operator from the telephony API. Missing names become an empty string. |
| `has_play_store` | Whether querying `com.android.vending` returns package information, expressed as `true` or `false` bytes. |
| `network_radio_type` | Category number mapped from a network snapshot and cellular subtype. [Network classification](android-inputs.md#en-network-type) |
| `simnum` | Whether a local phone number of at least six characters is available, written as `1` or `0` bytes. |
| `pid` | Current process ID as decimal bytes. |
| `rc` | Application release-channel number as decimal bytes. The production release channel uses `0`. |
| `device_ram` | Total system memory divided by 2³⁰ and formatted as decimal text, which can be cached per instance. |
| `sim_type` | Read when telephony environment collection is enabled. No telephony service is `2`, absent SIM is `0`, and other states are `1`. |
| `airplane_mode_type` | Airplane mode under the same collection switch, normalized to `0` or `1`. |
| `cellular_strength` | Cellular signal level under the same switch, normally `0`–`4`, with `5` when the API or relevant object is unavailable. |
| `roaming_type` | Included under the same switch when a network snapshot is available. Non-roaming is `0`, roaming is `1`, and no telephony service is `2`. |
| `sim_status` | SIM-number check result from registration preferences, appended when non-null. An earlier flow can cache this value. |
| `db` | The integer system setting `adb_enabled` when its collection is enabled. An absent setting defaults to `0`. |
| `first_api_level` | Device launch API level, appended as decimal bytes when collection is enabled and the value is positive. |
| `advertising_id` | Optional text from the advertising service, subject to country and provider. A branch that reads the limit-ad-tracking flag can omit it based on that flag. [Advertising ID](android-inputs.md#en-advertising-id) |
| `preloads_app_manager_id` | App Manager identifier from the install attribution component or its cache, appended when non-null. |
| `preloads_attribution` | JSON text from a separate attribution source or its cache, appended when non-null. |

### Challenge Material

A verification route can also include `recaptcha` verification material.

### Form and Transport

The Kotlin route writes the number, shared identifiers, recovery material, and verification credentials, attempts to add the public-key bundle, and then merges additional fields. An additional field can overwrite an existing key while preserving its insertion position. Ordinary text uses UTF-8 form encoding. Byte-array values are percent-encoded from the raw bytes and inserted directly into the form. Encoding them twice changes `%HH` to `%25HH`. The WAMsys route merges task parameters and transport fields in the native layer. Final form order and application headers depend on the selected transport. [Request serialization](request-encryption.md#en) · [Sending and responses](sending-requests.md#en)

After transport-specific merging, the inner fields are serialized as a form and packaged for the selected route. Encrypted requests carry the encrypted body in `ENC`. See [request packaging](request-encryption.md#en) and [sending](sending-requests.md#en).

`Content-Length` uses the byte length of the final body.

This synthetic fragment shows field values before encoding. The example JSON escapes the quotes inside the `client_metrics` string.

```json
{
  "cc": "1",
  "in": "2025550142",
  "code": "000000",
  "entered": "1",
  "client_metrics": "{\"attempts\":1}"
}
```

Form serialization percent-encodes the UTF-8 bytes of `client_metrics` once. The resulting inner form contains this fragment.

```text
client_metrics=%7B%22attempts%22%3A1%7D
```

### Response and Follow-up State

| Response information | Handling |
| --- | --- |
| `status=ok` | The register business-success branch. The client can also read optional identity material such as `login` and `lid` and continue account initialization. |
| `status=fail` with `reason` | The client handles reasons such as `mismatch`, `stale`, or `too_many_guesses`. HTTP 200 can carry these business failures. |
| `retry_after`, `guess_wait`, `sms_wait`, `voice_wait`, `flash_wait` | Optional business wait information. The transport handles HTTP connection retries separately. |
| `second_factor_methods`, `security_code_set`, `reset_method`, `wipe_token` | Optional follow-up authentication and recovery information. The response state determines the applicable flow. |

The client evaluates HTTP status, JSON parsing, and business status separately. Failure to read or parse the response can produce a local error. When verification returns a second-factor or recovery requirement, the flow still needs to process that state. [Sending and responses](sending-requests.md#en)
