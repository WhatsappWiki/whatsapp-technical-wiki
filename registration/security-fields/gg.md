# _gg · Play 结果精简包装

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en)

<a id="zh"></a>

## 中文

正常本次生成时，`_gg` 保存与[gpia](gpia.md#zh)关联的同一次Play结果，省去应用包和文件元数据。公共Map读取的已有槽也可能来自更早一次生成；`_gg` 不是第二次Google请求，也不替代 `gpia`。

### 内部结构

| 子键 | 类型 | 出现条件 |
| --- | --- | --- |
| `_ic` | 整数 | 当前结果码；含义见[结果码表](gpia.md#codes) |
| `_it` | 字符串 | 成功结果携带的真实token；没有token时省略，不填空串或null占位 |

错误分支的完整内部JSON可为：

```json
{"_ic":1004}
```

成功分支使用 `_ic=0` 和真实 `_it`。不能把其它调用的token配给本次错误码，或因没有token就把错误结果认作空字段。

### 加密与表单

内部JSON字节使用[公共CBC包装](README.md#cbc)：当前原生连接private派生public32 → 含padding的标准Base64文本 → SHA-256密钥，独立IV16，AES256-CBC/PKCS7，最后标准Base64。`_gg` 不复用 `gpia/_gi` 的IV或密文。

标准Base64的 `+ / =` 仍按表单路径处理；不擅改为Base64url或删除padding。JSON键顺序与转义参与密文，不把解析后对象相等当作字节相等。

### 发布与生命期

正常结果处理会分别计算 `_gg`、`_gi` 和 `gpia` 的包装。`_gg` 加密结果非null才进入正常发布，成功写入新槽后再管理引用；加密返回null可保留旧槽。字节转换返回null且实际到达setter的分支又可清槽，不能把两类失败合并。

本次Play获取关闭不自动清除已有 `_gg`；公共参数合并可读取仍存在的槽。整体安全对象重建后，新对象的 `_gg` 槽为空；整体重建失败也可移除旧owner。这与单字段生成失败保留旧值不同。

公共Map读取只检查现有 `_gg` 槽是否非null，不再次按号码、年龄、code或Play获取开关重验。构造完成的Map持有当时值引用，后续setter替换槽不会改变该Map；若通道随后重新合并，则以那次合并的结果为准。

三个结果发布没有统一事务，因而可以短暂出现新 `_gg` 与旧 `_gi`。不能仅凭 `gpia` callback未到就判定 `_gg` 未更新，也不根据HTTP返回自行清缓存。字段使用关系见[安全字段总览](README.md#zh)及[发送流程](../sending-requests.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

During normal fresh generation, `_gg` retains the same Play outcome associated with [gpia](gpia.md#en), omitting package/file metadata. An existing slot read by the public Map can instead come from an earlier generation. `_gg` is neither a second Google request nor a replacement for `gpia`.

### Internal Structure

| Key | Type | Presence |
| --- | --- | --- |
| `_ic` | Integer | Current outcome code; see the [code table](gpia.md#codes-en) |
| `_it` | String | Actual token from a successful outcome; omit when unavailable rather than inserting empty/null |

A complete internal error object can be:

```json
{"_ic":1004}
```

A successful outcome uses `_ic=0` with the real `_it`. Do not pair a different call's token with this outcome, or treat a tokenless error result as an empty field.

### Encryption and Form Representation

Apply the [shared CBC envelope](README.md#cbc-en) to the JSON bytes: current native connection private-derived public32 → padded standard Base64 text → SHA-256 key, independent IV16, AES256-CBC/PKCS7, then standard Base64. Do not reuse `gpia/_gi` IVs or ciphertext.

Standard Base64 `+ / =` still follow the form path. Do not switch to Base64url or strip padding. JSON order and escaping affect ciphertext; parsed-object equality is not byte equality.

### Publication and Lifetime

Normal outcome handling computes `_gg`, `_gi`, and `gpia` envelopes separately. A non-null `_gg` encryption result enters normal publication; the new slot is written before reference management. A null encryption result can retain the old slot. A byte conversion returning null and actually reaching the setter can instead clear it: these are different failures.

Disabling the current Play acquisition does not automatically clear an existing `_gg`; common parameter merging can read its retained slot. A rebuilt security owner starts with an empty `_gg`, and failed whole-owner rebuilding can remove the old owner. This differs from retaining an old value after a single-field failure.

The common Map getter checks only whether the `_gg` slot is non-null, without revalidating number, age, code, or the Play acquisition gate. A built Map retains that value reference; later setter replacement does not mutate it. A subsequent transport merge can select values again, so its result governs that merge.

Publication is not transactional, so new `_gg` and old `_gi` can temporarily coexist. A missing `gpia` callback alone does not establish that `_gg` was unchanged. An HTTP result is not a cache-clear trigger. See the [overview](README.md#en) and [sending flow](../sending-requests.md#en).

[Back to registration](../README.md#en)
