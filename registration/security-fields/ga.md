# _ga · 路径年龄与运行环境

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en)

<a id="zh"></a>

## 中文

`_ga` 外层保持JSON，不整体CBC。`ap/ai/ae`为整数秒数，`mp/mu`为布尔值，只有 `bi` 是单独加密的字符串。整份结果可由安全对象缓存，所以不是每次请求的实时设备快照。

### 子字段

| 子键 | 类型 | 输入 |
| --- | --- | --- |
| `ap` | 条件整数 | 应用sourceDir路径年龄 |
| `ai` | 条件整数 | 应用dataDir路径年龄 |
| `ae` | 条件整数 | `getExternalFilesDir(null)` 对应路径年龄 |
| `mp` | 布尔 | 当前进程可见 `/proc/*/cmdline` 的Monkey特征 |
| `mu` | 布尔 | `ActivityManager.isUserAMonkey()` 的可取得结果 |
| `bi` | 条件字符串 | boot标识流摘要文本经独立CBC |

这些年龄来自文件/目录元数据，不是PackageManager安装时间，也不是App进程运行秒数。实际生成顺序为mu、mp、ae、ai、ap、boot摘要/CBC；JSON排列不必与读取顺序相同。

### 路径年龄

路径字符串为空时省略对应键，不调用该路径年龄算法。非空路径先用lstat选择文件或目录处理。

- **文件路径**：先stat；失败返回0且不调用time。成功后独立取time，输出 `now - mtime`，未来mtime可以产生负值。
- **目录路径**：先独立取time，以now初始化最小mtime累计器，再按深度小于11遍历。目录自身mtime参与，即便之后opendir失败；对子项stat成功的普通文件，mtime也参与。符号链接目录不递归，指向普通文件的链接可参与。输出 `now - min(now, 参与的mtime)`。

ae、ai、ap各自读取路径和时间，不能把一个统一now复制给三项。空路径省键、stat失败的0和成功的0不同；文件与目录读取顺序也不相同。

### Monkey与boot标识

`mu` 是实际API结果；相应查找/调用失败分支可为false，不代表设备永远没有自动化。`mp` 遍历可见 `/proc` 条目，以C式正数解析选择候选，再读取其cmdline。按1024字节块先追加，累计超过 `0x2800` 才停止；读取失败可保留已读前缀用于匹配。UTF-8转换成功的文本做 `com.android.commands.monkey` 子串检查；不可读/不可转换条目跳过。受进程可见性限制，false不是全设备进程清单结论。

boot流来自 `/proc/sys/kernel/random/boot_id`，保留原始换行。正常读取到EOF才用完整bytes；失败使用ASCII `failed_sha`，不是部分前缀，也不是把失败当成功空流。

```text
boot_material = raw_boot_bytes if 读取正常完成 else ASCII("failed_sha")
digest_text = 标准Base64(SHA256(boot_material))
bi = 公共CBC包装(ASCII(digest_text), owner_public32_snapshot, 独立IV16)
```

成功空流的SHA-256与 `failed_sha` 摘要不同。`bi`明文是摘要的Base64文本，不是boot UUID原文；摘要也不能逆推出原始流。公共包装见[专题首页](README.md#cbc)。直接生成器在公钥参数为null时跳过boot/RNG/bi，或在加密正常返回null时省略bi。公共wrapper却要求先建立公共材料；该步失败不等于正常调用直接生成器的null-key分支，不能统一改成“无bi但成功”。外层仍不整体加密。

目录递归的深度检查发生在进入下一层目录时；父目录中已stat成功的普通文件即使位于深度11，其mtime仍可参与。不要把“目录不继续递归”改成“该边界子项全部忽略”。

CBC的IV16通过独立Java随机回调取得，每次调用创建SHA1PRNG并取字节，不使用GS的DRBG或libc rand。随机回调返回null与抛错不是零IV兜底，不能为了凑齐字段自动补16个零字节。

### 对象与公共材料

公共getter非null命中后跳过Application、公共材料、Monkey、路径、时钟、boot与随机源读取。生成时使用所属安全对象创建时保存的公钥快照；它不是 `gpia/_gg/_gi` 包装时查询当前连接密钥的实时别名。

因此连接换钥不自动清除旧 `_ga` 或重加密bi；整体owner替换后不继承旧结果，后续缓存miss才重新读取。普通getter、明确refresh和整体重建分开，不能把某次HTTP发送当作自动刷新点。

该字段作为JSON字符串进入[exist](../exist.md#zh)或[code](../code.md#zh)，再经过[请求封装](../request-encryption.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

The outer `_ga` remains JSON, without whole-field CBC. `ap/ai/ae` are integer seconds, `mp/mu` booleans, and only `bi` is separately encrypted text. The security owner can cache the complete result, so it is not necessarily a real-time device snapshot for each request.

### Subfields

| Key | Type | Input |
| --- | --- | --- |
| `ap` | Conditional integer | Application sourceDir path age |
| `ai` | Conditional integer | Application dataDir path age |
| `ae` | Conditional integer | Age of the path returned by `getExternalFilesDir(null)` |
| `mp` | Boolean | Monkey characteristics in process-visible `/proc/*/cmdline` |
| `mu` | Boolean | Available result from `ActivityManager.isUserAMonkey()` |
| `bi` | Conditional string | Boot-stream digest text with separate CBC |

Ages use file/directory metadata, not PackageManager installation time or process uptime. Actual generation order is mu, mp, ae, ai, ap, then boot hashing/CBC; JSON order need not match collection order.

### Path Ages

An empty path omits its key without invoking age collection. A nonempty path first uses lstat to choose file or directory handling.

- **File**: stat first. Failure returns 0 without calling time. Success takes an independent time and returns `now - mtime`; future mtime can produce a negative age.
- **Directory**: take an independent time first, initialize the minimum-mtime accumulator to now, then walk at depth less than 11. Include each directory's own mtime even if opendir later fails; successful regular-file child stat contributes too. Do not recurse into symlink directories; links to regular files can contribute. Return `now - min(now, participating mtimes)`.

ae, ai, and ap each read their own path/time; do not copy one unified now across them. Omission for an empty path, failure zero, and successful zero differ. File and directory operation order also differs.

### Monkey and Boot Identifier

`mu` uses the actual API; particular lookup/call failures can yield false, not proof of permanent absence of automation. `mp` traverses visible `/proc` entries, selects candidates through positive C-style integer parsing, and reads cmdline. Append 1024-byte blocks before stopping beyond `0x2800`; a read failure can leave a usable prefix. Successfully decoded UTF-8 text is checked for substring `com.android.commands.monkey`; skip unreadable/undecodable entries. Process visibility limits mean false is not a device-wide process inventory.

Read `/proc/sys/kernel/random/boot_id` as raw bytes, preserving its newline. Normal completion requires EOF; failure uses ASCII `failed_sha`, not a partial prefix or a successful empty stream.

```text
boot_material = raw_boot_bytes if read_completed_normally else ASCII("failed_sha")
digest_text = standard_Base64(SHA256(boot_material))
bi = shared_CBC_envelope(ASCII(digest_text), owner_public32_snapshot, independent_IV16)
```

Successful empty-stream SHA-256 differs from the `failed_sha` digest. The plaintext inside bi is Base64 digest text, not the boot UUID; its digest does not recover the original stream. See the [shared wrapper](README.md#cbc-en). The direct generator skips boot/RNG/bi for a null public input, or omits bi when encryption normally returns null. The common wrapper requires public-material construction first; failure there is not the normal direct-generator null-key branch and must not universally become a successful object without bi. The outer object is still not wholly encrypted.

The directory-depth check applies when entering the next directory. A successfully stat-ed regular child at depth 11 can still contribute mtime from its parent walk. Stopping directory recursion does not discard every child at that boundary.

CBC IV16 comes from a separate Java randomness callback that creates SHA1PRNG and obtains bytes per call, not the GS DRBG or libc rand. Null randomness and thrown errors are not a zero-IV fallback; do not insert 16 zero bytes merely to complete the field.

### Owner and Public Material

A non-null common-getter cache hit skips Application, public material, Monkey, path, clock, boot, and RNG reads. Generation uses a public-key snapshot captured at security-owner creation, not a live alias for the current-key reads used by `gpia/_gg/_gi` wrapping.

Connection-key rotation therefore does not automatically clear old `_ga` or re-encrypt bi. Whole-owner replacement does not copy the old result; a later cache miss recollects. Ordinary getter, explicit refresh, and owner rebuilding are separate; an HTTP send is not an automatic refresh event.

The field becomes JSON text in [exist](../exist.md#en) or [code](../code.md#en), followed by [request packaging](../request-encryption.md#en).

[Back to registration](../README.md#en)
