# gpia · Play 结果与应用信息

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en)

<a id="zh"></a>

## 中文

`gpia` 将一次Play Integrity结果与应用完整性信息组合成JSON，再进行[字段级CBC包装](README.md#cbc)。它不是Google token本身；一个携带错误码而没有token的对象仍可形成字段载荷。

### 内容与类型

| 子键 | 类型 | 含义与来源 |
| --- | --- | --- |
| `code` | 整数 | 应用映射后的结果码；0表示正常返回token |
| `token` | 字符串，条件出现 | 真实StandardIntegrityToken结果；错误路径不填假token |
| `sizeInBytes` | 字符串 | 安装主文件独立stat的大小文本；失败分支可为 `-1` |
| `packageName` | 字符串 | 应用实际包名 |
| `shatr` | 字符串 | 前最多10MiB的SHA-256/标准Base64；前缀摘要失败用 `-` |
| `p` | 字符串，条件出现 | 已发布的sourceDir路径 |
| `cert` | 字符串，取得时出现 | 选定首张签名证书DER的SHA-1/标准Base64 |
| `sha256` | 字符串，条件出现 | 已发布的完整安装文件SHA-256/标准Base64 |

`p/sha256`受源槽发布时机影响；不能把缺键补成空串，或固定认为每次都包含同一组键。应用元数据与[_gi](gi.md#zh)共享部分输入，但结果对象的子键名不同。

<a id="codes"></a>

### 结果码

| 值 | 本地意义 |
| --- | --- |
| `0` | 正常得到token |
| `1000` | 落入通用Exception映射 |
| `1001` | provider尚未准备且网络前置检查失败 |
| `1002` | 本地prepare调用限流 |
| `1003` | request入口没有prepared provider |
| `1004` | 限定Google阶段的超时 |
| `1005` | 进入Google阶段前缺少连接公钥 |
| 其它API错误整数 | ApiException的状态码原样保留，不统一改成1000 |

这些是应用侧结果分类，不能把1002或1004称作Google设备完整性判决。`code`不是注册响应 `status`，也不单独决定号码路由或注册成败。

### 获取顺序与限流

1. 获取开关关闭时省略本次 `gpia` 获取，不构造一个“关闭错误码”。
2. 读取连接存储公钥；非null时用标准Base64无padding作为 `requestHash`，不先做SHA-256，也不改用Base64url。
3. 在指定dispatcher进入超时范围，先检查provider槽；非null时无需再次prepare。
4. 未prepared时先检查网络，再按调整后的wallclock和light偏好决定是否允许prepare；获准才更新计数与最后调用时间，并调用真实 `prepareIntegrityToken`。
5. prepare成功发布provider后继续 `provider.request`，取得token或异常并映射结果；不是prepare成功后直接当作已有token。
6. Google阶段结束后再完成应用元数据组合与字段包装；后续包装/发布不属于该Google超时范围。

本地限流以最后一次获准prepare为参照：间隔不足60000ms则累计，计数达到5时拒绝；达到60000ms才重置为1。每次获准调用更新时间戳，拒绝不更新时间戳，所以不是固定整分钟窗口。计数与时间分别apply，两次wallclock/偏移读取也独立；已有provider绕过这组限流。

### 缓存、超时与回调

prepared只表示provider引用非null，本地没有由此推出Google内部有效期。这一应用层不做in-flight合并，多个调用可能在先前prepare完成前再次prepare；成功回调各自发布provider，失败回调不自动清掉另一次成功结果。

超时时长来自配置，不固定为某个示例值。开始时间在指定dispatcher实际进入后，不能把调用排队时间自动计入同一计时器。协程取消或超时不等于取消Google Task、移除listener或阻止迟到prepare成功写入provider。

超时配置小于等于0时，provider检查、prepare与request block都不执行；不是只把最后的await改成零等待。结果映射只捕获指定Exception链，Error不统一改成1000，获取开关读取自身失败也不自动生成“关闭结果”。provider缓存与token返回分开，request成功回调本身不在这一层建立token持久缓存。

request成功/失败listener默认走主线程；prepare成功listener也用默认主线程，prepare失败使用显式executor。回调线程与后续协程恢复dispatcher不同。取消后的首次迟到resume不恢复已取消consumer，但再一次resume可抛重复完成异常，不是无限静默忽略。native包装在Google计时范围之后独立等待callback，没有由该Google timer覆盖。

正常本次生成时，[_gg](gg.md#zh)与gpia共用结果码和可选token，但分别序列化、各取IV、独立发布。最终Map也可读取已有的旧_gg槽，不能仅由此推定同次。原生CBC使用当前连接密钥读取派生的公钥，可能晚于Java构造 `requestHash` 的公钥读取；配对损坏或期间换钥时不保证相同。详见[公共材料与缓存](README.md#zh)。

### 结构示意

下面是错误分支的JSON节选，不是完整键集；其它字段仍按实际生成规则构造。这里不是Google verdict JSON，也不是最终字段密文。

```json
{"packageName":"com.example.client","sizeInBytes":"8192","code":1002}
```

字段作为字符串进入[exist](../exist.md#zh)或[code](../code.md#zh)内层，再由[请求封装](../request-encryption.md#zh)处理。不要以客户端 `token`、推送token或历史 `gpia` 密文代替新的Play结果。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

`gpia` combines one Play Integrity outcome with application-integrity metadata, then applies the [field-level CBC envelope](README.md#cbc-en). It is not just the Google token. An outcome with an error code and no token can still form a payload.

### Contents and Types

| Key | Type | Meaning and input |
| --- | --- | --- |
| `code` | Integer | Application-mapped outcome; 0 means a normally returned token |
| `token` | Conditional string | Real StandardIntegrityToken result; no invented token on error |
| `sizeInBytes` | String | Independent stat size of the primary installation file; failure can yield `-1` |
| `packageName` | String | Actual application package name |
| `shatr` | String | SHA-256/standard Base64 of at most the first 10 MiB; prefix-hash failure uses `-` |
| `p` | Conditional string | Published sourceDir path |
| `cert` | String when available | SHA-1/standard Base64 of the selected first signing-certificate DER |
| `sha256` | Conditional string | Published full installation-file SHA-256/standard Base64 |

Source-slot publication affects `p/sha256`; do not replace absent keys with empty strings or assume a fixed key set. Some inputs are shared with [_gi](gi.md#en), but the output key names differ.

<a id="codes-en"></a>

### Outcome Codes

| Value | Local meaning |
| --- | --- |
| `0` | Token returned normally |
| `1000` | Generic Exception mapping |
| `1001` | Provider unprepared and network prerequisite failed |
| `1002` | Local prepare-call rate limit |
| `1003` | No prepared provider at the request entry |
| `1004` | Timeout of the bounded Google stage |
| `1005` | Missing connection public key before Google processing |
| Other API integers | ApiException status preserved rather than replaced by 1000 |

These are application outcome categories. Neither 1002 nor 1004 is a Google device-integrity verdict. `code` is not the registration response `status` and does not alone determine number routing or registration success.

### Acquisition and Rate Limiting

1. A disabled acquisition gate omits this `gpia` acquisition rather than creating a disabled-error code.
2. Read stored connection public32 and use unpadded standard Base64 as `requestHash`, without SHA-256 or the Base64url alphabet.
3. Enter the timeout on the designated dispatcher and check the provider slot. A non-null provider avoids another prepare.
4. If unprepared, check the network, then adjusted wallclock and light preferences. An allowed prepare updates count/timestamp and calls real `prepareIntegrityToken`.
5. After prepare publishes a provider, continue to `provider.request`, obtaining a token or mapping an exception. Prepare success alone is not a token result.
6. Compose metadata and wrap the field afterward; this later processing is outside the Google timeout.

The local limiter measures from the last allowed prepare: under 60000 ms it accumulates, refusing at count 5; at or beyond 60000 ms it resets to 1. Allowed calls update the timestamp, refusals do not. It is not a fixed wallclock-minute window. Count/timestamp use separate apply operations and separate clock/offset reads. An existing provider bypasses this limiter.

### Caching, Timeout, and Callbacks

Prepared means a non-null provider reference; it does not establish Google's internal expiry. This application layer does not coalesce in-flight prepares, so more can start before an earlier one completes. Success callbacks publish independently; failure does not clear another call's successful provider.

The timeout comes from configuration, not a fixed illustrative value. It begins after entering the designated dispatcher, so queued time is not automatically part of the same timer. Coroutine cancellation or timeout does not itself cancel the Google Task, remove listeners, or prevent late prepare success from publishing.

A nonpositive timeout skips the entire provider-check/prepare/request block, not merely its last await. Outcome mapping catches the specified Exception chain, not all Errors as 1000; failure to read the acquisition gate is not automatically a disabled outcome. Provider caching and token delivery differ: request success does not itself persist a token in this layer.

Request success/failure listeners default to the main thread. Prepare success also uses the default main thread, while prepare failure has an explicit executor. Callback thread and coroutine-resume dispatcher differ. The first late resume after cancellation does not revive the consumer; another resume can throw a duplicate-completion exception, not be ignored indefinitely. Native packaging waits for its own callback after the Google timer’s scope.

During normal fresh generation, [_gg](gg.md#en) and gpia share an outcome code and optional token, while serialization, IVs, and publication remain separate. A final Map can also read a retained older _gg slot, so presence alone does not establish the same generation. Native CBC uses the current connection-key-derived public key, read potentially later than the Java public key for `requestHash`; corrupted pairs or intervening rotation need not match. See [shared material and caches](README.md#en).

### Structure Example

This is an excerpt of error-branch JSON, not a complete key set; other fields follow their generation rules. It is neither Google verdict JSON nor the final encrypted field.

```json
{"packageName":"com.example.client","sizeInBytes":"8192","code":1002}
```

The field enters [exist](../exist.md#en) or [code](../code.md#en) as a string before [request packaging](../request-encryption.md#en). Client `token`, push tokens, and older `gpia` ciphertext are not substitutes for a new Play outcome.

[Back to registration](../README.md#en)
