# _gi · 应用完整性信息

[注册目录](../README.md#zh) · [中文](#zh) · [English](#en)

<a id="zh"></a>

## 中文

`_gi` 将安装文件、主DEX、签名和实际目录信息组成JSON，再做[公共CBC包装](README.md#cbc)。子键来源和缓存层不同；它不是一个对整个安装目录统一计算的摘要。

### 子字段

| 子键 | 类型 | 生成规则 |
| --- | --- | --- |
| `_dh` | 字符串 | 主 `classes.dex` 解压流的SHA-256/标准Base64；context字典键0可缓存 |
| `_iln` | 字符串 | dataDir树中符合条件的 `.so` basename字节集合摘要 |
| `_isb` | 字符串 | 独立stat取得主安装文件大小；失败可为 `-1` |
| `_ip` | 字符串 | 传入应用的包名 |
| `_p` | 条件字符串 | 本次生成开始时读取的sourceDir发布槽 |
| `_ln` | 字符串 | nativeLibraryDir中的条目名称集合摘要，不仅是 `.so` |
| `_ist` | 字符串 | 主安装文件前最多 `0xA00000` 字节的SHA-256/标准Base64；该摘要失败可使用 `-` |
| `_icr` | 取得时的字符串 | 所选首签名DER的SHA-1/标准Base64 |
| `_is` | 条件字符串 | 已由完整文件摘要任务发布的SHA-256/标准Base64 |
| `did` | 条件字符串 | 配置允许时读取 `ro.build.display.id`，不是设备唯一ID |
| `_acj` | 条件数组 | 另一签名采集开关允许时的多证书摘要投影；不固定补空数组 |

数字大小采用字符串，摘要通常为标准Base64含padding。键集合受配置、返回值与异步状态控制，不固定为十键。

### 文件与签名读取

完整/前缀摘要读取sourceDir，成功取得路径后先发布 `_p`，再打开文件；后续打开或读取失败不撤销路径发布。摘要流按1MiB分块，完整模式要求EOF，前缀模式达到限制也可完成；正常read返回0不等于EOF。`_isb` 独立stat，不简单用某次已读文件数组长度代替。

`_dh` 独立读取主安装路径中的ZIP条目 `classes.dex`，计算SHA-256；不是客户端 `token` 使用的DEX MD5。失败占位先按下表文本取摘要，发送的是摘要而非原始错误词。

| 条件 | 摘要输入文本 |
| --- | --- |
| 包路径API失败或null | `package_path_failed` |
| 路径stat失败 | `path_not_found` |
| ZIP/条目/流/读取失败 | `corrupted_apk` |
| 必要类/方法或相关分配失败分支 | `signal_error` |

`_icr` 从初始Application的PackageManager读取签名数组，保留数组首项选择，对DER做SHA-1。该路径使用的签名API选择不同于客户端 `token` 的SigningInfo/history分支，不互换证书选择规则。条件性的 `_acj` 会另读签名数组；两者不保证是同一次API快照。摘要不等于证书链信任验证。

### 目录名称摘要

`_ln` 使用nativeLibraryDir：根必须是目录，跳过 `.` 与 `..`，取其余条目的原始basename字节，不读取文件内容，也不只筛 `.so`。

`_iln` 从dataDir深度0开始，只有深度小于11且lstat为目录才向下遍历。对子项stat成功、为普通文件且名称末尾为 `.so` 时收集basename；递归不进入符号链接目录，指向普通文件的链接可贡献其名称。

两组名称都按无符号字节字典序排序、保留重复、无分隔拼接，做SHA-256/标准Base64；没有名称时摘要 `signal_error`。不要先按Unicode规范化或去重，或以文件内容摘要替代名称摘要。

### 发布顺序与缓存

生成先读取 `_is`、`_p`，再取得包装公钥和后续元数据；之后计算 `_ist` 才可能刷新 `_p`。因此一次生成可以没有 `_p`，即便该次后段已经发布了新路径。

完整摘要在context键2缓存，worker计算后由main回调尝试发布 `_is`； `_dh` 的键0独立缓存，包括特定错误摘要。库级源槽、context字典与最终 `_gi` 密文槽不是同一状态。重建安全对象不自动清除另外两层；新密文不证明全部文件重新读取，固定等待也不保证发布完成。

完整摘要lookup命中context键2时，不重读路径，也不会通过文件路径读取再次发布 `_p`；失败null不写键2，下次可重算。字典get/set各有自己的锁，不将整个文件读取包成一个锁内事务，多个worker可同时miss。

main回调仅在库全局 `_is` 仍为null时写入：null结果让槽继续为空，首个实际发布的非null结果留下；已有非null值不被其它context的后续回调覆盖。这个检查/写入不提供任意并发下的CAS保证，不能从worker排队顺序推导最终发布者。

`_acj`对原始签名数组null/空的返回与“非空数组但全部元素投影失败”的空数组不同，不强制改成同一结果。`did`的属性读取长度为0时可保留空字符串；超出该读取器的长度范围可返回null，不等于拿Build.DISPLAY补缺。额外HP分支的键结构不能由这里列出的普通字段反推。

CBC公钥来自当前原生连接密钥读取，独立IV；不把请求 `authkey` 的存储公钥、Google requestHash读取和此时派生公钥合成一次原子采样。详见[总览](README.md#zh)与[请求加密](../request-encryption.md#zh)。

[返回注册目录](../README.md#zh)

<a id="en"></a>

## English

`_gi` combines installation-file, primary-DEX, signing, and directory information into JSON, then uses the [shared CBC envelope](README.md#cbc-en). Its keys have different inputs and cache layers; it is not one digest of an entire installation directory.

### Subfields

| Key | Type | Generation |
| --- | --- | --- |
| `_dh` | String | SHA-256/standard Base64 of decompressed primary `classes.dex`; cacheable under context key 0 |
| `_iln` | String | Digest of qualifying `.so` basename bytes in the dataDir tree |
| `_isb` | String | Independent stat size of the primary installation file; failure can yield `-1` |
| `_ip` | String | Package name of the supplied application |
| `_p` | Conditional string | sourceDir publication slot read at generation start |
| `_ln` | String | Digest of nativeLibraryDir entry names, not just `.so` files |
| `_ist` | String | SHA-256/standard Base64 of at most the first `0xA00000` bytes; failure can use `-` |
| `_icr` | String when available | SHA-1/standard Base64 of the selected first signing DER |
| `_is` | Conditional string | Published full-file SHA-256/standard Base64 |
| `did` | Conditional string | Configuration-gated `ro.build.display.id`, not a unique device ID |
| `_acj` | Conditional array | Multi-certificate digest projection under another collection gate; not a mandatory empty array |

Sizes are strings and digests normally use padded standard Base64. Configuration, return values, and asynchronous state determine presence; there is no fixed ten-key object.

### File and Certificate Reads

Full/prefix hashing obtains sourceDir and publishes `_p` before opening the file. Later failure does not undo publication. The stream uses 1 MiB chunks: full mode requires EOF, while reaching the limit also completes prefix mode. A normal zero-byte read is not EOF. `_isb` uses an independent stat, not simply the length of an earlier byte array.

`_dh` separately reads ZIP entry `classes.dex` from the package path and computes SHA-256, unlike the DEX MD5 used by client `token`. Failure substitutes are hashed first; the field contains a digest, not the error word itself.

| Condition | Digest-input text |
| --- | --- |
| Package-path API failed or returned null | `package_path_failed` |
| Path stat failed | `path_not_found` |
| ZIP/entry/stream/read failure | `corrupted_apk` |
| Required metadata lookup or related allocation failure branches | `signal_error` |

`_icr` reads the initial Application's PackageManager signature array, keeps first-element selection, and hashes its DER with SHA-1. This signature API differs from client `token` SigningInfo/history selection; do not swap certificate-selection rules. Conditional `_acj` reads the array again, not necessarily from the same API snapshot. A digest is not certificate-chain trust validation.

### Directory-name Digests

`_ln` uses nativeLibraryDir: require a directory root, skip `.` and `..`, and collect other raw basename bytes. It neither hashes file contents nor filters only `.so` names.

`_iln` starts at dataDir depth 0, descending only into lstat directories at depth less than 11. Collect a child's basename when stat succeeds, it is a regular file, and its name ends in `.so`. Do not descend into symlink directories; links to regular files can contribute names.

For both collections, sort by unsigned-byte lexicographic order, retain duplicates, concatenate without separators, then SHA-256/standard Base64. No names uses `signal_error` as input. Do not Unicode-normalize, deduplicate, or substitute content hashes.

### Publication Order and Caching

Generation reads `_is` and `_p` first, then the wrapping public key and later metadata. Subsequent `_ist` computation can publish `_p`. A generated object can therefore omit `_p` even if its later work publishes a new path.

The full digest uses context key 2; worker computation is followed by a main callback attempting `_is` publication. `_dh` independently caches under key 0, including particular error digests. Library source slots, context dictionaries, and the final `_gi` ciphertext slot are distinct states. Rebuilding a security owner does not clear the other two layers. New ciphertext does not prove all files were reread, and a fixed wait does not establish publication.

A context-key 2 cache hit does not reread the path or republish `_p` through that read. A null full-hash failure is not stored and can be retried. Dictionary get/set have separate locks, not one lock spanning file hashing; multiple workers can miss concurrently.

The main callback writes library-global `_is` only while it is null. A null result leaves it empty; the first actually published non-null result remains, without replacement by later callbacks from other contexts. This check/write is not a CAS guarantee for arbitrary concurrency; worker submission order does not determine the publisher.

For `_acj`, a null/empty original signature array differs from a nonempty array whose elements all fail projection and yield an empty array. A zero-length property read can preserve empty `did`; out-of-range lengths can return null rather than falling back to Build.DISPLAY. The extra HP branch’s key structure cannot be inferred from the ordinary fields listed here.

CBC uses the current native connection-key read and an independent IV. Request `authkey`, the Google requestHash read, and this derived public key are not one atomic sample. See the [overview](README.md#en) and [request encryption](../request-encryption.md#en).

[Back to registration](../README.md#en)
