# _gs · 系统信号与临时加密包络

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en)

<a id="zh"></a>

## 中文

`_gs` 的外层为含 `em` 的JSON；`em` 不是一个root布尔值，而是经过临时X25519和AES-GCM包装的19整数CSV。路径可访问性、时间、libc随机数和prepared状态各有独立来源。

### 原始采集顺序

下面路径表对应有效的默认解码配置。运行时Configure覆盖或无效字符串状态可改变解码后的路径/格式，不能在实际解码不同的情况下仍固定使用此表。

| 顺序i（从0开始） | 路径 | stat成功基码b[i] |
| --- | --- | --- |
| 0 | `/sys/module/vmw_pvscsi` | 10 |
| 1 | `/sys/module/vboxsf` | 11 |
| 2 | `/system/build.prop` | 0 |
| 3 | `/system/app/Superuser.apk` | 100 |
| 4 | `/sbin/su` | 101 |
| 5 | `/system/bin/su` | 102 |
| 6 | `/system/xbin/su` | 103 |
| 7 | `/data/local/xbin/su` | 104 |
| 8 | `/proc/version` | 1 |
| 9 | `/data/local/bin/su` | 105 |
| 10 | `/system/sd/xbin/su` | 106 |
| 11 | `/system/bin/failsafe/su` | 107 |
| 12 | `/data/local/su` | 108 |

解码发生在stat之前，13次stat按表序执行；成功保留基码，任何非零返回或空路径用基码加1000。不读取这些文件内容，不执行su，不以errno区分失败类型。

之后按以下顺序：两次分别预清零的 `gettimeofday` → 用第二次秒/微秒低32位异或值调用 `srand` → `rand` → 读取实际prepared整数。两次timeval不能合并。系统调用返回码本身不使采集立即终止，未被覆盖的预清零内容仍参与运算；异常时间原语不能悄悄规约成正常范围。

### 19项CSV

以下为正常微秒范围的格式。`u32/u64`分别取低32/64位，`s32`按有符号32位输出，`DIV`表示整数商，`T`保留64位回绕结果；路径项的i为0..12。

```text
m = (u32(first.sec) XOR u32(first.usec)) & 0xEFFF
status[i] = b[i] + (0 if stat[i] == 0 else 1000)
T = (u64(second.sec) * 1000 + (u64(second.usec) DIV 1000)) mod 2^64
```

| CSV位置（从1开始） | 输出 |
| --- | --- |
| 1 | 格式常量 `1` |
| 2..14 | `s32(status[i] XOR u32(m + 0x1000 + 7*i))` |
| 15 | 无符号十进制 `m OR 0x1000` |
| 16 | `s32(rand XOR u32(m + 0x105B))` |
| 17 | `s32(u32(T) XOR u32(m + 0x1062))` |
| 18 | `s32(u32(T >> 32) XOR u32(m + 0x1069))` |
| 19 | `s32(prepared XOR u32(m + 0x1070))` |

以逗号拼接ASCII十进制，不加空格、换行或末尾逗号；加密长度不含C字符串末尾NUL。格式缓冲区有240字节边界，分配或格式化失败可使生成无结果。

libc `rand` 是进程状态，不能用Java/Python随机类的相同种子替代；其它线程也可能消耗同一状态。prepared在随机数之后读取，保留实际整数值，不由Google provider是否prepared推导。独立准备任务被排队不等于已经完成。

### prepared的初值与异步准备

真正新加载的该安全库映射中，prepared的初值为1337；实际执行Prepare写入4242。Configure修改解码key和字符串状态，不写prepared；Release释放单次信号对象，不把这个全局恢复初值。同一映射重复dlopen不是新实例重置。

公共流程先检查独立的准备开关，再检查GS刷新开关。准备通过once回调尝试加载并排utility任务，不等待Prepare实际写入；之后的GS采集仍可先读到旧整数。once回调即使因加载失败正常返回也算完成，后续不会因此自动重新排队；utility中的失败也不重置之前的once。只有真正观察到对应写入顺序，才能更新prepared状态。

### em加密

CSV形成后另取密码学随机32字节，来自独立DRBG，不是前面的libc `rand`。将标量按X25519规则裁剪：首字节 `&248`，末字节 `(&63)|64`。计算临时公钥E和与本组件peer公钥的共享32字节。

```text
shared32 = X25519(ephemeral_private32, GS_peer_public32)
ciphertext || tag16 = AES256-GCM(shared32, zero_nonce12, CSV_ASCII, empty_AAD)
em = Base64urlNoPadding(E_public32 || ciphertext || tag16)
_gs = 紧凑JSON对象，键em的值为上述字符串
```

没有额外SHA-256/HKDF，没有发送nonce或 `0x05` 公钥类型前缀。即使GS peer与HTTP ENC peer字节相同，它们仍是独立的输入边界；GS临时密钥、CSV与HTTP请求临时密钥、明文表单分别生成，不能复用。

### 刷新、合并与解释

本组件可用、刷新开关低位开启且生成/序列化成功时替换安全对象 `_gs`。关闭刷新或特定生成失败可保留旧槽；不能以字段非空认定刚刚重新采集。prepared的独立准备开关、Configure和刷新开关是不同状态。

原生exist发送路径可在Java获取后再次刷新GS，合并时覆盖早先输入，因此两次em不同不自动说明路径环境改变。需要比较的是最终[表单加密输入](../request-encryption.md#zh)所使用的值。单个stat成功、masked CSV或外层加密成功都不是服务端接受或完整注册成功的充分条件。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

`_gs` is outer JSON containing `em`. Rather than one root boolean, `em` envelopes a 19-integer CSV using ephemeral X25519 and AES-GCM. Path accessibility, time, libc randomness, and prepared state are distinct inputs.

### Raw Collection Order

The table describes the valid default decoder configuration. Runtime Configure overrides or invalid-string state can change decoded paths/formats; do not force this table when actual decoding differs.

| Index i, starting at 0 | Path | Successful-stat base b[i] |
| --- | --- | --- |
| 0 | `/sys/module/vmw_pvscsi` | 10 |
| 1 | `/sys/module/vboxsf` | 11 |
| 2 | `/system/build.prop` | 0 |
| 3 | `/system/app/Superuser.apk` | 100 |
| 4 | `/sbin/su` | 101 |
| 5 | `/system/bin/su` | 102 |
| 6 | `/system/xbin/su` | 103 |
| 7 | `/data/local/xbin/su` | 104 |
| 8 | `/proc/version` | 1 |
| 9 | `/data/local/bin/su` | 105 |
| 10 | `/system/sd/xbin/su` | 106 |
| 11 | `/system/bin/failsafe/su` | 107 |
| 12 | `/data/local/su` | 108 |

Decode before stat, then perform 13 probes in table order. Success keeps the base; any nonzero return or null path uses base+1000. This reads no file contents, executes no su, and does not distinguish errno categories.

Next: two separately pre-zeroed `gettimeofday` calls → `srand` with the XOR of the second seconds/microseconds low 32 bits → `rand` → actual prepared integer. Do not merge the timeval reads. Return codes alone do not terminate collection; untouched pre-zeroed fields still participate. Do not silently normalize unusual time outputs into the normal range.

<a id="the19-value-csv"></a>

### The 19-value CSV

This format covers normal microsecond inputs. `u32/u64` take low 32/64 bits, `s32` formats signed 32-bit integers, `DIV` is integer quotient, and `T` wraps to 64 bits. Path index i is 0..12.

```text
m = (u32(first.sec) XOR u32(first.usec)) & 0xEFFF
status[i] = b[i] + (0 if stat[i] == 0 else 1000)
T = (u64(second.sec) * 1000 + (u64(second.usec) DIV 1000)) mod 2^64
```

| CSV position, starting at 1 | Output |
| --- | --- |
| 1 | Format constant `1` |
| 2..14 | `s32(status[i] XOR u32(m + 0x1000 + 7*i))` |
| 15 | Unsigned decimal `m OR 0x1000` |
| 16 | `s32(rand XOR u32(m + 0x105B))` |
| 17 | `s32(u32(T) XOR u32(m + 0x1062))` |
| 18 | `s32(u32(T >> 32) XOR u32(m + 0x1069))` |
| 19 | `s32(prepared XOR u32(m + 0x1070))` |

Join ASCII decimal values with commas, without spaces, newline, or a trailing comma. Encryption excludes the C-string NUL. Formatting has a 240-byte buffer bound; allocation/formatting failure can yield no result.

libc `rand` is process state, not interchangeable with similarly seeded Java/Python RNGs. Other threads can consume it. Read prepared after randomness and retain its actual integer value; do not derive it from Google provider readiness. Queued independent preparation is not completed preparation.

### prepared Initialization and Asynchronous Preparation

A genuinely new mapping of this security library initializes prepared to 1337; actual Prepare execution writes 4242. Configure changes decoder key/string state, not prepared. Release frees an individual signal object without restoring the global initial value. Repeated dlopen of the same mapping is not a fresh instance.

The common flow checks the separate preparation gate before the GS refresh gate. Preparation uses a once callback to attempt loading and queue a utility task, without waiting for Prepare’s write; collection can still read the older integer. A once callback returning normally after loader failure is also complete, so it does not automatically queue again. Utility failure does not reset that earlier once. Model prepared changes only at their actual write events.

### em Encryption

After CSV construction, obtain a separate cryptographic 32-byte DRBG output, not libc `rand`. Clamp the scalar for X25519: first byte `&248`, last byte `(&63)|64`. Derive ephemeral public E and shared32 with this component's peer key.

```text
shared32 = X25519(ephemeral_private32, GS_peer_public32)
ciphertext || tag16 = AES256-GCM(shared32, zero_nonce12, CSV_ASCII, empty_AAD)
em = Base64urlNoPadding(E_public32 || ciphertext || tag16)
_gs = compact JSON with em holding the string above
```

There is no extra SHA-256/HKDF, transmitted nonce, or `0x05` public-key prefix. Even if GS and HTTP ENC peer bytes match, their input boundaries are distinct. GS ephemeral keys/CSV and HTTP ephemeral keys/plaintext forms are independently generated, not reused.

### Refresh, Merge, and Interpretation

When the component is available, the refresh gate's low bit is enabled, and generation/serialization succeeds, replace owner `_gs`. Disabled refresh or particular failures can retain an old slot; nonempty does not mean freshly collected. Independent preparation, Configure, and refresh gates are different states.

Native exist sending can refresh after the Java getter and overwrite earlier input during merging. Different em values do not alone establish an environmental change. Compare the value actually used by the final [form-encryption input](../request-encryption.md#en). A successful stat, masked CSV, or successful encryption is not sufficient proof of server acceptance or complete registration.

[Back to registration](../README.md#en)
