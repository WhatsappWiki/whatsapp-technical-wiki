# Security Fields · 安全字段

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

七个字段描述不同输入，不能合成一个“安全通过”布尔值。[`gpia`](gpia.md#zh)/[`_gg`](gg.md#zh) 承载 [Play 结果](gpia.md#zh)，[`_gi`](gi.md#zh)/[`_gp`](gp.md#zh) 描述应用材料，[`_ge`](ge.md#zh)/[`_gs`](gs.md#zh)/[`_ga`](ga.md#zh) 描述进程可见环境与运行状态。字段存在、字段加密成功、Google返回结果和注册业务成功是不同事件。

### 字段与表示

| 字段 | 主要内容 | 字段内部表示 | 文章 |
| --- | --- | --- | --- |
| [`gpia`](gpia.md#zh) | [Play结果](gpia.md#zh)与应用信息 | JSON → CBC → 标准Base64 | [gpia](gpia.md#zh) |
| [`_gg`](gg.md#zh) | 生成时关联的[Play结果精简包装](gg.md#zh) | JSON → 独立IV的CBC → 标准Base64 | [_gg](gg.md#zh) |
| [`_gi`](gi.md#zh) | [文件、签名、路径与目录完整性](gi.md#zh) | JSON → 独立IV的CBC → 标准Base64 | [_gi](gi.md#zh) |
| [`_gp`](gp.md#zh) | [应用声明权限名称摘要](gp.md#zh) | SHA-256 → 标准Base64 | [_gp](gp.md#zh) |
| [`_gs`](gs.md#zh) | [路径检查、时间、随机数、prepared](gs.md#zh) | JSON含em；em为X25519/GCM包络 | [_gs](gs.md#zh) |
| [`_ge`](ge.md#zh) | [模块与条件性的su路径检查](ge.md#zh) | 紧凑JSON | [_ge](ge.md#zh) |
| [`_ga`](ga.md#zh) | [路径年龄、Monkey、启动标识](ga.md#zh) | JSON；只有bi单独CBC | [_ga](ga.md#zh) |

所有结果随后作为内层参数进入[请求加密](../request-encryption.md#zh)。外层 [`ENC`](../request-encryption.md#zh) 不会替代字段内部的编码；[`H`](../request-encryption.md#zh)、[`Authorization`](../request-encryption.md#zh) 也不是上述任何字段的别名。客户端 [`token`](../exist.md#zh)、Android ID摘要 [`aid`](../android-inputs.md#aid-encoding) 和HTTP [`request_token`](../sending-requests.md#zh) 各有独立职责。

<a id="cbc"></a>

### 公共CBC包装

对 [`gpia`](gpia.md#zh)/[`_gg`](gg.md#zh)/[`_gi`](gi.md#zh) 的原生包装路径，当前连接密钥读取先从 private32 派生 `wrapping_public32`。请求 [`authkey`](../exist.md#zh) 则读取存储 public32；有效配对且读取间未换钥时可一致，不能对损坏记录或并发重建无条件等同。

```text
public_text = 标准Base64含padding(wrapping_public32)
K = SHA256(UTF8(public_text))
field = 标准Base64(IV16 || AES256-CBC-PKCS7(K, IV16, plaintext))
```

每份包装独立取16字节IV；IV作为包络前缀。AES块为16字节，PKCS7总会添加padding，包括明文恰为整块时。标准Base64保留 `+`、`/` 与必要的 `=`。进入表单时仍须按该参数的编码路径处理。

Play `requestHash` 使用之前Java读取的存储公钥、标准Base64无padding，不做上述SHA-256。[`_ga.bi`](ga.md#zh) 使用安全对象创建时的公共材料快照，不是每次重新取当前公钥。CBC密钥来自公钥材料；这层包装不应被理解为独立的持钥身份证明，也不含GCM式认证tag。

### 输入和缓存的三层生命周期

| 层 | 状态 | 关键行为 |
| --- | --- | --- |
| 库级 | `_is/_p` 源槽 | 随各自发布事件改变；重建安全对象不自动清零 |
| context级 | DEX摘要键0、完整文件摘要键2等字典缓存 | 同context可复用；只替换安全对象不等于新建字典 |
| 安全对象级 | [`_gg`](gg.md#zh)/[`_gi`](gi.md#zh)/[`_gp`](gp.md#zh)/[`_ga`](ga.md#zh)/[`_ge`](ge.md#zh)/[`_gs`](gs.md#zh)、[`aid`](../android-inputs.md#aid-encoding)结果槽及公共材料快照 | 新对象结果槽为空，不继承旧对象载荷；[gpia](gpia.md#zh)经独立callback返回，不占这些结果槽 |

异步完整文件摘要通常先在worker计算，之后由main回调发布 `_is`；`_p` 在获取文件路径后、打开/读取之前就可更新。生成器读取这些槽的先后影响字段是否出现，等待固定时长不能证明某个回调已经完成。

[`_gg`](gg.md#zh)/[`_gi`](gi.md#zh)写入对象槽，[`gpia`](gpia.md#zh)通过callback返回；三份结果独立完成，不能当作一个原子事务。单字段生成失败可能保留旧槽，整体owner重建失败则可清掉owner。[`_gp`](gp.md#zh)/[`_ge`](ge.md#zh)/[`_ga`](ga.md#zh) 的缓存命中可跳过原始读取；[`_gs`](gs.md#zh) 在允许刷新时重新采集，成功才替换。HTTP完成、UI结束、协议wrapper释放都不等于全局清缓存。

库级 `_is/_p` 在真正新库实例中从null开始；context复制不复制旧安全owner或摘要字典，而仅替换同context的owner不清库全局或原字典。已生成Map持有自己的值引用，后续槽替换不反向改写Map。普通引用release也不保证立刻销毁，须区分其它仍持有该对象的上下文或注册record。

### 在请求中使用

[exist](../exist.md#zh)与[code](../code.md#zh)的参数集合与上游流程不同。共用安全组件不意味着它们读取的是同一代对象、同一次系统状态或同一份最终Map。原生exist路径可在发送前刷新 [`_gs`](gs.md#zh) 并覆盖此前JNI输入；中间Map不一定是最终加密输入。其它认证路线必须按实际调用分支携带字段，不因已有这些文章而为缺失值填空串或旧载荷。

字段子键的JSON类型、缺键、空值与错误占位都参与实际字节。序列化的键顺序、转义与数字格式须保持所用路径的规则；对象语义相同不保证密文字节相同。共享规则的请求层说明见[加密封装](../request-encryption.md#zh)和[发送与响应](../sending-requests.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

The seven fields describe different inputs, not one “security passed” boolean. [`gpia`](gpia.md#en)/[`_gg`](gg.md#en) carry [Play outcomes](gpia.md#en), [`_gi`](gi.md#en)/[`_gp`](gp.md#en) describe application material, and [`_ge`](ge.md#en)/[`_gs`](gs.md#en)/[`_ga`](ga.md#en) describe the process-visible environment and execution state. Field presence, successful encryption, a Google outcome, and registration success are separate events.

### Fields and Representations

| Field | Main contents | Internal representation | Article |
| --- | --- | --- | --- |
| [`gpia`](gpia.md#en) | [Play outcome and application metadata](gpia.md#en) | JSON → CBC → standard Base64 | [gpia](gpia.md#en) |
| [`_gg`](gg.md#en) | [Compact form of the Play outcome associated at generation](gg.md#en) | JSON → CBC with its own IV → standard Base64 | [_gg](gg.md#en) |
| [`_gi`](gi.md#en) | [File, certificate, path, and directory integrity](gi.md#en) | JSON → CBC with its own IV → standard Base64 | [_gi](gi.md#en) |
| [`_gp`](gp.md#en) | [Declared-permission-name digest](gp.md#en) | SHA-256 → standard Base64 | [_gp](gp.md#en) |
| [`_gs`](gs.md#en) | [Path checks, time, randomness, prepared state](gs.md#en) | JSON with an X25519/GCM em envelope | [_gs](gs.md#en) |
| [`_ge`](ge.md#en) | [Module and optional su-path checks](ge.md#en) | Compact JSON | [_ge](ge.md#en) |
| [`_ga`](ga.md#en) | [Path ages, Monkey checks, boot identifier](ga.md#en) | JSON; only bi has separate CBC | [_ga](ga.md#en) |

These results become inner parameters for [request encryption](../request-encryption.md#en). Outer [`ENC`](../request-encryption.md#en) does not replace internal encoding. Neither [`H`](../request-encryption.md#en) nor [`Authorization`](../request-encryption.md#en) aliases these fields. Client [`token`](../exist.md#en), Android ID digest [`aid`](../android-inputs.md#en-aid-encoding), and HTTP [`request_token`](../sending-requests.md#en) have separate roles.

<a id="cbc-en"></a>

### Shared CBC Envelope

The native wrapping path for [`gpia`](gpia.md#en)/[`_gg`](gg.md#en)/[`_gi`](gi.md#en) derives `wrapping_public32` from current connection private32. Request [`authkey`](../exist.md#en) instead reads stored public32. They can match for a valid pair without intervening rotation; corrupted records or concurrent rebuilding do not guarantee equality.

```text
public_text = padded_standard_Base64(wrapping_public32)
K = SHA256(UTF8(public_text))
field = standard_Base64(IV16 || AES256-CBC-PKCS7(K, IV16, plaintext))
```

Each envelope uses its own 16-byte IV, prefixed to the ciphertext. AES has a 16-byte block size; PKCS7 always adds padding, including for block-aligned plaintext. Standard Base64 retains `+`, `/`, and required `=`. Form encoding still follows that parameter's path.

Play `requestHash` uses the earlier Java read of stored public32 as unpadded standard Base64, without the SHA-256 step. [`_ga.bi`](ga.md#en) uses public material captured at security-owner creation, not a fresh current-key read. The CBC key derives from public material: this wrapper is not an independent proof of private-key possession and has no GCM-style authentication tag.

### Three Input and Cache Lifetimes

| Layer | State | Behavior |
| --- | --- | --- |
| Library | `_is/_p` source slots | Change through their own publication events; rebuilding a security owner does not zero them |
| Context | Dictionaries such as DEX digest key 0 and full-file digest key 2 | Reusable within the same context; replacing an owner does not create a fresh dictionary |
| Security owner | [`_gg`](gg.md#en)/[`_gi`](gi.md#en)/[`_gp`](gp.md#en)/[`_ga`](ga.md#en)/[`_ge`](ge.md#en)/[`_gs`](gs.md#en), [`aid`](../android-inputs.md#en-aid-encoding) result slots, and public-material snapshot | A new owner starts empty; [gpia](gpia.md#en) returns separately through a callback and is not one of these slots |

A full-file digest can be computed on a worker and published to `_is` by a later main callback. `_p` can update after path acquisition but before opening/reading. Generator load order affects key presence; a fixed wait does not establish callback completion.

[`_gg`](gg.md#en)/[`_gi`](gi.md#en) publish to owner slots while [`gpia`](gpia.md#en) returns through a callback; the three results complete independently, without an atomic transaction. A field failure can retain an old slot; failed whole-owner rebuilding can clear the owner. Cache hits for [`_gp`](gp.md#en)/[`_ge`](ge.md#en)/[`_ga`](ga.md#en) can skip raw reads. Enabled [`_gs`](gs.md#en) refresh recollects and replaces only on success. HTTP completion, UI ending, or protocol-wrapper release is not a global cache-clear event.

Library `_is/_p` start null for a genuinely new library instance. Context copying does not copy the old security owner or digest dictionary; replacing only the owner within a context does not clear globals or that dictionary. An already built Map retains its own value references, unaffected by later slot replacement. Releasing one reference need not immediately destroy an object held by another context or registration record.

### Use in Requests

[exist](../exist.md#en) and [code](../code.md#en) have different parameter sets and upstream flows. Sharing security components does not establish the same object generation, system sample, or final Map. A native exist path can refresh [`_gs`](gs.md#en) before sending and overwrite earlier JNI input, so an intermediate Map need not be the final encryption input. Other authentication routes must follow their actual branch rather than filling missing values with empty strings or older payloads.

JSON types, omission, nulls, and error substitutes affect bytes. Preserve the chosen serializer's key order, escaping, and numeric format: semantic JSON equality does not establish ciphertext equality. See [encryption](../request-encryption.md#en) and [sending/response handling](../sending-requests.md#en) for request-level rules.

[Back to registration](../README.md#en)
