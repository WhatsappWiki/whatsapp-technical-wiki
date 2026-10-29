# _ge · 模块与路径检查

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

`_ge` 是进程可见路径的布尔JSON，不做整体CBC或Base64。它与[_gs](gs.md#zh)有部分相同探测路径，但输出结构、配置和缓存不同。

### 子字段与实际路径

| 子键 | 类型 | 规则 |
| --- | --- | --- |
| `sv` | 布尔 | `stat("/sys/module/vmw_pvscsi") == 0` |
| `sb` | 布尔 | `stat("/sys/module/vboxsf") == 0` |
| `su` | 条件布尔 | 配置低位允许时，九条路径中任一stat成功 |

探测顺序先 `sv` 后 `sb`；JSON成员排列不由探测顺序推导。`stat==0`只表示当时该路径可访问；失败不区分不存在、权限或其它errno，不执行su，也不能单凭布尔值断言系统全局环境。

### su路径

```text
/system/app/Superuser.apk
/sbin/su
/system/bin/su
/system/xbin/su
/data/local/xbin/su
/data/local/bin/su
/system/sd/xbin/su
/system/bin/failsafe/su
/data/local/su
```

该组按顺序短路：首次成功即true，全部失败才false。直接生成器只看原始参数低位 `(rawParameter & 1) != 0`，所以直接传2会关闭su。公共getter则先将配置回调的非零long规范成1（标准回调本身返回0/1），再把低位交给生成器；不能把回调返回2与直接传入2混为同一路径。关闭时不执行此组，不加入 `su`，缺键不等于false。

### 缓存和结构

公共getter在非null缓存命中时，连配置也可以不再读取。因此关闭或开启采集开关不一定立刻把已有两键对象改成三键对象。明确生成/转换/写槽与普通缓存读取是不同入口。

转换正常返回null可写空槽；在写槽前抛错则保留旧引用。安全对象整体重建后不会继承旧 `_ge`。这些行为与[_gi](gi.md#zh)的库源槽、context字典无关。

```json
{"sb":false,"sv":false}
```

示例只有两个布尔键，没有整体密文，也不补造 `su=false`。JSON作为字符串进入内层参数，之后才经过[表单与外层ENC](../request-encryption.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

`_ge` is boolean JSON describing process-visible paths, without whole-field CBC or Base64. It overlaps some [_gs](gs.md#en) probes but has different structure, configuration, and caching.

### Keys and Paths

| Key | Type | Rule |
| --- | --- | --- |
| `sv` | Boolean | `stat("/sys/module/vmw_pvscsi") == 0` |
| `sb` | Boolean | `stat("/sys/module/vboxsf") == 0` |
| `su` | Conditional boolean | When the configuration low bit enables it, any successful stat among nine paths |

Probe `sv` before `sb`; JSON order does not follow automatically from probe order. `stat==0` means the path was accessible then. Failure does not distinguish absence, permission, or other errno values. This does not execute su or alone establish the global device environment.

### su Paths

```text
/system/app/Superuser.apk
/sbin/su
/system/bin/su
/system/xbin/su
/data/local/xbin/su
/data/local/bin/su
/system/sd/xbin/su
/system/bin/failsafe/su
/data/local/su
```

Probe in order with short-circuiting: the first success yields true, all failures yield false. The direct generator tests `(rawParameter & 1) != 0`, so passing 2 directly disables su. The common getter first normalizes a nonzero configuration-callback long to 1 (the standard callback itself returns 0/1), then supplies that low bit: a callback returning 2 is not the same as passing 2 directly. A disabled gate skips the probes and omits `su`, which differs from false.

### Caching and Structure

A non-null cache hit in the common getter can skip even the configuration read. Changing the gate need not immediately turn an existing two-key object into a three-key object. Explicit generation/conversion/storage differs from ordinary cache lookup.

A conversion that normally returns null can clear the slot; an exception before storage retains the previous reference. Whole-owner rebuilding does not copy old `_ge`. These rules are separate from [_gi](gi.md#en) library source slots and context dictionaries.

```json
{"sb":false,"sv":false}
```

The example has two boolean keys, no whole-field ciphertext, and no invented `su=false`. Its JSON text becomes an inner parameter before [form and outer ENC](../request-encryption.md#en).

[Back to registration](../README.md#en)
