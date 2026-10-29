# _gp · 声明权限摘要

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

`_gp` 是应用声明权限名称的摘要，不是当前已授予权限列表，也不是Manifest文件整体摘要。输入来自传入Application的PackageManager，按包名读取 `GET_PERMISSIONS` 对应的 `requestedPermissions` 数组。

### 变换

1. 每个名称按JNI modified UTF-8转字节。常规ASCII权限名与标准UTF-8一致；NUL和补充字符的编码不能一般化为普通UTF-8。
2. 按无符号字节字典序排序，公共前缀相同则短串在前；保留重复，不加分隔符。
3. 拼接后没有任何字节时，改用ASCII文本 `default_permission`。
4. SHA-256后做标准Base64，保留padding。正常摘要原始32字节，Base64文本44字符。

```text
names = sort_unsigned_bytes(modified_UTF8(each_permission))
input = concat(names) if concat(names) is nonempty else ASCII("default_permission")
_gp = standard_Base64(SHA256(input))
```

这不是对“排序后权限数”或每个名称分别摘要。两个权限名拼接的边界不写入摘要；不得为了可读性加逗号、换行或JSON数组。

### 空值与失败

有Application但权限API/数组不可用的相应分支可走空集合材料。Application本身缺失时可直接返回null，不等于发送 `SHA256(default_permission)`。数组内null成员也不能自动补空名称。

`requestedPermissions` 描述声明而非运行时授权，所以授予或撤销READ_PHONE_STATE不必改变 `_gp`；声明集合本身变化才改变相应输入。电话权限状态另见[exist参数](../exist.md#zh)。

### 缓存与请求

安全对象的权限结果槽非null时复用，跳过PackageManager。首次准备入口与公共Map读取不是一回事；空槽不表示Map读取会自动查询权限。此槽也不是[_gi](gi.md#zh)的context摘要字典。

`default_permission`对应的非null错误/空拼接摘要也会缓存，不因PackageManager稍后恢复就自动重试；没有按权限变化或时间失效的重验。相反，cache miss且Application缺失返回null时，没有建立这个默认摘要缓存。安全owner缺失时即使临时算出结果，也不能假定已经保存进公共结果槽。

整体安全对象重建不继承旧 `_gp`，但仍须实际执行准备过程才生成新值。客户端 `token`、Play token和外层 `H` 都不能替代它。最终表示继续经过[表单与外层封装](../request-encryption.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

`_gp` digests declared permission names, not granted runtime permissions or the entire Manifest file. The supplied Application's PackageManager reads `requestedPermissions` for its package using `GET_PERMISSIONS`.

### Transformation

1. Encode each name using JNI modified UTF-8. Ordinary ASCII permission names match standard UTF-8, but NUL and supplementary characters do not generally do so.
2. Sort unsigned byte sequences lexicographically, with shorter equal-prefix sequences first. Retain duplicates and add no separators.
3. If concatenation contains no bytes, substitute ASCII `default_permission`.
4. Apply SHA-256 and padded standard Base64. A normal digest is 32 raw bytes and 44 Base64 characters.

```text
names = sort_unsigned_bytes(modified_UTF8(each_permission))
input = concat(names) if concat(names) is nonempty else ASCII("default_permission")
_gp = standard_Base64(SHA256(input))
```

This is not a digest of a permission count or of separately hashed names. Name boundaries are not included. Do not add commas, newlines, or a JSON array for readability.

### Missing Inputs and Failure

With an Application, particular unavailable permission-API/array branches can use empty-collection material. A missing Application can instead return null, not `SHA256(default_permission)`. Null array members are not automatically empty names.

`requestedPermissions` describes declarations, so granting/revoking READ_PHONE_STATE need not change `_gp`; changing the declaration input does. See [exist parameters](../exist.md#en) for runtime phone-permission state.

### Caching and Requests

A non-null permission-result slot in the security owner is reused without querying PackageManager. Preparation and reading the common Map are distinct; an empty slot does not mean the Map getter queries permissions. This is separate from the [_gi](gi.md#en) context digest dictionary.

The non-null `default_permission` digest is cached too; later PackageManager recovery does not automatically retry it, nor is there a time/permission-change revalidation here. A cache miss with no Application returns null without creating that default-digest cache. If the security owner is absent, a computed temporary result does not establish storage in the common result slot.

Rebuilding the security owner does not copy old `_gp`, but actual preparation is still required to generate it. Client `token`, Play tokens, and outer `H` cannot replace it. The resulting text proceeds through [form and outer packaging](../request-encryption.md#en).

[Back to registration](../README.md#en)
