# code · 明文参数与取值来源

[注册目录](README.md#zh) · [中文](#zh) · [English](#en) · [明文示意](#plaintext-example) · [Full Example](#plaintext-example-en) · [Telegram: @ttjjda](https://t.me/ttjjda)

<a id="zh"></a>

## 中文

`/v2/code` 用于申请短信、语音或其它验证路线的验证码/认证流程，携带号码、验证方式、客户端身份与设备状态。它处于 `exist → code → register` 的发码阶段；提交用户收到的验证码由后续验证请求处理。

本文按功能分组说明 70 个内层参数，其中包含受采集配置或认证类型控制的字段；实际请求并非固定发送 70 项。选择 SMS、voice、flash 或 autoconf 时，应使用对应流程捕获的参数与状态。[Android 原始输入示例](android-inputs.md#zh)介绍系统读取与部分编码入口；下表中的 code 专用读取和表示规则以本文为准。

表格与明文示意均将字段值表示为字符串。`client_metrics`、`_ge`、`_ga` 等字段的字符串内容仍是 JSON；“明文”指进入外层加密之前，不表示字段内部没有编码或加密。`id` 与 `backup_token` 为原始二进制材料，本文以百分号编码文本展示。

号码采用示例号码；标识、摘要与密码学载荷均为合成展示值。固定宽度字段保持表示长度，变长字段示例不定义协议长度；这些值不能组成有效凭据、签名或安全证明。

### 1. 号码、语言与验证方式

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `cc` | 待验证号码的电话国家码。 | 与本地号码一起捕获的已确认号码状态。 | 数字字符串，不带 +。一般号码流程携带；指定认证类型可同时省略 cc/in。 | `"1"` |
| `in` | 国家码之外的本地号码。 | 同一份已确认号码状态。 | 保留字符串；与 token、id 使用的号码一致。不是待提交的验证码。 | `"2025550147"` |
| `lg` | 应用当前语言。 | 应用 Resources.Configuration 对应的 Locale；[语言读取](android-inputs.md#app-locale)。 | 语言文本缓存非null命中时可直接复用；未命中时读取语言管理器解析的Locale，格式为小写2–3字母，否则zz。不由电话国家码推导。 | `"en"` |
| `lc` | 应用语言环境的地区。 | 同一应用 Locale 的 country。 | 两位大写字母或三位数字；不合格式时为 ZZ。与 cc 分开。 | `"US"` |
| `method` | 本次选择的验证路线。 | 发码页面或业务调用捕获的方式参数。 | 文本，如 sms、voice、flash、autoconf。发送前保存最近尝试方式，不等待 sent；挂起后不按新按钮状态改写。 | `"sms"` |
| `rc` | 发布渠道编号。 | 应用发布渠道配置。 | 十进制文本，例如正式发布渠道 0；与 Android API 级别无关。 | `"0"` |

### 2. 设备与访问会话标识

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `fdid` | 持久 phone-id。 | 应用 light 偏好的 phoneid_id 与配套 timestamp。 | 带连字符的 UUID 文本；缺失时生成保存，导入或同步可更新。不是每次请求新建。 | `"ce9ecf75-f380-4676-aa7f-a91c421c4466"` |
| `expid` | 持久性能设备标识。 | 优先复用 perf_device_id；缺失时以 Android ID 形成 name UUID，正常 null/空白时改用随机 UUID。 | UUID 的 16 字节大端表示 → 无 padding 的 Base64url。已有非 null 偏好不重新查 Android ID。 | `"YucHBVW5kh37FAN49P82LQ"` |
| `access_session_id` | 关联一段访问会话中的请求。 | 会话管理器内存状态与 reg_prefs.access_session_id。 | UUID16 → 无 padding 的 Base64url。已有内存 wrapper 时直接复用；缺 wrapper 才读偏好并按需生成。显式 reset 保存/发布新值后条件同步原生状态。 | `"hzw8ZMqxkqN-07b-V3NJdg"` |
| `aid` | Android ID 的摘要。 | 应用 ContentResolver 读取 Android ID；[原始输入](android-inputs.md#android-id)。 | 按 JNI modified UTF-8 取字节，SHA-256 → 标准 Base64 含 padding；明确的读取失败分支摘要 signal_error。所属安全对象缓存结果。 | `"IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY="` |

### 3. 恢复与注册材料

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `id` | 本地恢复材料。 | 读取并解密应用 files/rc2；号码参与口令。 | 原始字节按字节百分号编码。读取返回 null 才新建并尝试保存；格式/解密错误各有处理，不能一律当缺失。 | `"%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF"` |
| `backup_token` | 注册阶段保存的备份辅助材料。 | reg_prefs.token_used_during_reg 文本先用 Base64 解码。 | 非空数组直接复用，零长才生成保存；原始字节百分号编码。自建 Light 后端的缺值/错类型可先变成默认空串，非法非空 Base64 仍会失败；不重新走 exist 备份选择。 | `"%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C"` |
| `hasinrc` | 本地恢复文件是否存在。 | 应用 files 目录中的 rc2 文件；[文件检查](android-inputs.md#recovery-record)。 | 存在为 1，否则 0；不说明内容可解密或有效。 | `"1"` |

### 4. 连接与端到端加密公开身份

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `authkey` | 独立连接认证密钥的公钥。 | 连接 AuthKeyStore 的当前公开半部。 | 存储 public32 → 无 padding 的 Base64url。原生 CBC 包装另从连接 private32 派生公钥；有效配对且读取状态一致时才可视为相同材料。 | `"Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8"` |
| `e_ident` | Signal 本机身份公钥。 | 已有 Signal 本机身份记录。 | 去类型前缀后的 raw32 → 无 padding 的 Base64url；不为一次 code 随机重建。 | `"oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8"` |
| `e_keytype` | Signal 公钥类型标签。 | 公钥类型常量。 | 单字节 0x05 → Base64url 文本 BQ；不是十进制文本 5。 | `"BQ"` |
| `e_regid` | Signal 本机注册编号。 | 当前 Signal store 及其缓存。 | 整数的 4 字节大端表示 → 无 padding 的 Base64url。 | `"AAAwOQ"` |
| `e_skey_id` | 当前签名预密钥的编号。 | 选中的同一条 signed-prekey 记录。 | 记录 id 低 24 位的 3 字节大端表示 → 无 padding 的 Base64url；不是直接拿 SQL 行号。 | `"AAFB"` |
| `e_skey_val` | 该签名预密钥公钥。 | 与 e_skey_id 同一记录。 | 去类型前缀后的 raw32 → 无 padding 的 Base64url。 | `"HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ"` |
| `e_skey_sig` | 该预密钥记录保存的签名。 | 与编号、公钥同一次记录快照。 | 签名字节 → 无 padding 的 Base64url；不能用 HTTP 的 H 代替，也不能跨记录拼接。 | `"EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUV…qjHygUlWToo4Lg"` |

### 5. 客户端校验与安全字段

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `token` | 绑定应用内容与本地号码的客户端校验值。 | 包名、实际 logo 资源流、所选签名证书、主 classes.dex 和本地号码。 | PBKDF2-HMAC-SHA1 迭代 128 次、派生 64 字节密钥后形成 HMAC-SHA1，输出标准 Base64 含 padding。不是 Google token 或验证码。 | `"4s+/5KpaPw6zhEmWAm+hHBGzUqQ="` |
| `gpia` | Play Integrity 结果与应用完整性信息的组合。 | 同一次 Play 异步结果，加上应用元数据。 | JSON → AES-CBC 包装 → 标准 Base64。包装公钥取当前原生连接密钥读取结果，不直接复制请求 authkey；Play 获取受配置控制，可返回 token 或错误码。 [详解](security-fields/gpia.md#zh) | `"YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgj…BrYvpEIBOarw=="` |
| `_gg` | Play 结果的精简包装。 | 正常本次生成时，与 gpia 共用结果码和可选 token；公共 Map 读取旧槽不保证同次。 | 内部为 _ic 与可选 _it；取当前原生连接公钥、独立 IV 做 CBC/标准 Base64。各字段独立发布，单字段失败或门控不一定清旧值。 [详解](security-fields/gg.md#zh) | `"s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X"` |
| `_gi` | 安装内容、签名、路径与库目录完整性信息。 | 应用/DEX 字节流、证书和实际目录；[包材料](android-inputs.md#package-material)。 | 复合 JSON 经 CBC/标准 Base64 包装，包装公钥来自当前原生连接密钥读取。键受配置、局部缓存和异步发布控制；重建结果对象不保证清掉全部源缓存。 [详解](security-fields/gi.md#zh) | `"F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnn…c3kiQTvjmAMQ=="` |
| `_gp` | 应用声明权限名称集合摘要。 | PackageInfo.requestedPermissions；不是运行时已授予权限集合。 | JNI modified UTF-8 名称按无符号字节排序、保留重复、无分隔拼接 → SHA-256 → 标准 Base64 含 padding。空拼接（含全空名称）或指定 API 失败用 default_permission；cache miss 且 Application 缺失可返回 null。非 null 错误摘要也可缓存。 [详解](security-fields/gp.md#zh) | `"kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE="` |
| `_gs` | 路径检查、时间、随机数与执行准备状态组成的安全信号。 | 13 个 stat、时间原语、libc 随机数、prepared 状态及独立加密随机源。 | 外层 JSON 含 em；em 为临时公钥32与 AES-GCM 密文/tag16 的无 padding Base64url。按允许的刷新流程更新。 [详解](security-fields/gs.md#zh) | `"{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_Nh…JzSb2wjKFZTI\"}"` |
| `_ge` | 模块检测与可选 su 路径检查。 | 实际可见路径的 stat 结果与对应配置。 | 紧凑 JSON：sv 对应 /sys/module/vmw_pvscsi，sb 对应 /sys/module/vboxsf，均为 stat==0 布尔值；su 条件出现，缺键不等于 false。非 null 缓存命中可跳过配置与路径检查，整体不做 CBC/Base64。 [详解](security-fields/ge.md#zh) | `"{\"sb\":false,\"sv\":false}"` |
| `_ga` | 路径年龄、Monkey 与启动标识信息。 | 路径 mtime/独立时间调用、进程与系统 API、boot_id 原始流。 | 外层 JSON：ap/ai/ae 为秒数，mp/mu 为布尔；仅 bi 用所属对象创建时的公共材料快照单独 CBC。整份可缓存，不随每次请求或换钥自动重采样/重加密。 [详解](security-fields/ga.md#zh) | `"{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2A…30,\"mu\":false}"` |

### 6. 运营商、设备与网络状态

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `mcc` | 当前电话网络运营商的移动国家代码。 | TelephonyManager.getNetworkOperator() 的完整字符串。 | 须全串匹配 3 位 MCC 加 2–3 位 MNC；取首三位。缺失或不匹配默认 000。 | `"310"` |
| `mnc` | 当前电话网络运营商的移动网络代码。 | 与 mcc 同次解析。 | MNC 先转整数，再按 Locale.US %03d 输出三位；例如 46001 → 001。 | `"260"` |
| `sim_mcc` | SIM 归属运营商的移动国家代码。 | TelephonyManager.getSimOperator()。 | 同样要求完整 5–6 位格式，默认 000；不能从待验证号码推算。 | `"310"` |
| `sim_mnc` | SIM 归属运营商的移动网络代码。 | 与 sim_mcc 同次解析。 | MNC 整数按三位格式输出，默认 000；与当前接入网络代码分开。 | `"260"` |
| `network_radio_type` | 应用归类后的网络类别。 | 异步网络状态与蜂窝 subtype；[网络输入](android-inputs.md#network-type)。 | 十进制枚举：旧路径只有type1归Wi-Fi类1；SDK≥31且管理器已初始化的新路径将transport1/3归1。其它类型按所选mobile/wifi状态映射，非mobile且非wifi为-1；LTE111、NR113、未知mobile0。直接读取当前快照，不等待500ms通知。 | `"1"` |
| `simnum` | 能否取得达到长度门槛的本机号码。 | 权限允许时优先订阅列表首个非 null 号码，再取电话服务号码。 | 非 null 且 Java length ≥ 6 为 1，否则 0。不是 SIM 数量，不检查是否等于当前输入。 | `"0"` |
| `pid` | 采集进程编号。 | android.os.Process.myPid()。 | 十进制文本，进程重启可变。 | `"23456"` |
| `device_ram` | 系统总内存的 GiB 文本。 | ActivityManager.MemoryInfo.totalMem ÷ 2³⁰。 | totalMem除以1024³，以实际格式化Locale的#.##输出GiB文本，非null结果可按实例缓存。服务缺失/指定NullPointerException分支先取-1字节再除法和格式化，不直接发送"-1"；不是availMem。 | `"6"` |
| `db` | ADB 开启设置的读取结果。 | Settings.Global.adb_enabled，默认 0。 | 配置允许才加入十进制文本；关闭采集时省略，不补 0。 | `"0"` |

### 7. 可选电话环境字段

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `sim_type` | 应用归类后的 SIM 可用性。 | 电话服务是否存在与 getSimState()。 | 同组采集开关开启时加入：无服务 2、SIM absent 0、其它 1。 | `"0"` |
| `airplane_mode_type` | 飞行模式开关。 | Settings.Global.airplane_mode_on。 | 同组开关控制；读取值归一化为 0/1。 | `"0"` |
| `cellular_strength` | 蜂窝信号等级或不可用兜底。 | API ≥ 28 且电话服务与信号对象可用时取 getLevel()。 | 同组开关控制；通常为 0..4，缺支持/服务/首次对象时为 5。两次信号读取不原子，异常不能一律当 5。 | `"5"` |
| `roaming_type` | 电话网络漫游分类。 | 网络快照与 isNetworkRoaming()。 | 同组开关且重新读取到网络信息才加入：非漫游 0、漫游 1、无电话服务 2；无网络信息省略。不与 network_radio_type 构成原子快照。 | `"0"` |

### 8. 自动收码、流程统计与诊断

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `mistyped` | 号码纠错和电话权限综合状态。 | 号码输入流程产生并保存到 light 的 mistyped_state。 | 非 null 才加入，空串保留；自建 Light 对缺值/错类型返回 null 时省略。code 不重算纠错或以 -1 补缺，7 可表示电话信息权限不满足。 | `"7"` |
| `reason` | 客户端共享注册诊断。 | 内存诊断状态；相关页面恢复/保存 registration_failure_reason。 | 异步准备前捕获；空串也携带。读取不清空，可包含先前请求、收码或本地校验标签，不是当前响应 reason 的复制。 | `""` |
| `hasav` | 本次自动收码方式/状态。 | 页面的 SMS Retriever 标记与 RECEIVE_SMS 授权，或特定 fallback 的直接参数。 | 标准页面：Retriever true → 2，否则 RECEIVE_SMS 授予 → 1，其它正常结果 → 0；特殊 fallback 可直接传 0。已有但错类型的 Retriever 偏好可按 false 覆盖 Intent；null 省略。 | `"1"` |
| `client_metrics` | 本次发码业务统计和安装归因。 | reg_attempts_generate_code 的尝试快照及归因组件。 | 紧凑 JSON，attempts 为整数；业务 producer 加一并保存，序列化不再加。自建 Light 对错类型计数默认 0，可形成 attempts=1；不复制 exist 计数或 late-stub 子字段。 | `"{\"attempts\":2,\"app_campaign_download_source\":\"unknown\|unknown\"}"` |
| `education_screen_displayed` | 闪呼教育页面的本地展示标记。 | light.pref_flash_call_education_screen_displayed，默认 false。 | true/false 文本；教育页创建到达写入点时置 true，相关号码页创建置 false，清理可删除。不是已看完页面。 | `"false"` |
| `prefer_sms_over_flash` | 相对闪呼优先短信的本地偏好。 | light.pref_prefer_sms_over_flash，默认 false。 | true/false 文本；满足特定 SMS 选择条件才写 true，号码页初始化可置 false；读取不消费。 | `"false"` |
| `feo2_query_status` | 关联服务能力查询的已保存诊断。 | light.pref_autoconf_feo2_query_status；缺键默认 did_not_query。 | code 只读，不 query。自建 Light 对缺值、Map null 或错类型返回 did_not_query；空串/未知文本保留。只有实际返回 null 到上层才触发非空检查，诊断重置不清能力缓存。 | `"did_not_query"` |

### 9. 特定流程的附加参数

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `advertising_id` | 广告服务提供的可重置标识。 | 广告 ID provider；[广告接入](android-inputs.md#advertising-id)。 | 非null且国家/流程条件允许时加入；新provider分支LAT=true省略，旧直接读取分支不读取LAT。没有失败后自动改走另一分支，不用随机UUID或空串补缺。 | `"98ea2934-49f4-4245-ae15-339d0d3c6145"` |
| `push_code` | 推送渠道提供的注册验证码材料。 | 偏好 registration_push_notif_code。 | 非 null 且非空才加入；与 push_token 不同。 | `"example-push-code"` |
| `context` | 特定认证上下文。 | 上游认证流程。 | 非 null 文本；例如 invite_registration，具体值取决于调用流程。 | `"invite_registration"` |
| `client_start_message` | Autoconf 的客户端起始消息。 | Autoconf 准备结果中的有效 byte[]。 | 无 padding 的 Base64url；该路线消息为空会在发送前失败，不是 SMS 必填字段。 | `"SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA"` |
| `clicked_education_link` | 教育链接点击状态。 | 当前流程的 UI 整数参数。 | 0 → false，1 → true，其它整数省略；受认证类型分支控制。 | `"false"` |
| `manage_call_permission` | 流程中的通话管理权限状态。 | 当前流程提供的整数参数。 | 只将 0/1 映射为 false/true，其它省略；不把整数直接发出。 | `"true"` |
| `call_log_permission` | 流程中的通话记录权限状态。 | 当前流程提供的整数参数。 | 只将 0/1 映射为 false/true，其它省略；与实际权限读取流程配套。 | `"false"` |
| `sim_status` | 流程缓存的 SIM 辅助状态。 | reg_prefs.pref_cached_sim_status。 | 非 null 文本才加入；不是 sim_type 或系统 sim_state。 | `"1"` |
| `old_phone_number` | 已有账户的旧号码。 | 已有账户对象中的旧 cc 与 national。 | 按旧 cc + national 直接拼接，不带 +；不为此初始化缺失账户对象。 | `"12025550186"` |
| `first_api_level` | 设备出厂 API 级别。 | ro.product.first_api_level。 | 配置允许且属性为正值时加入；不等于当前 SDK_INT。 | `"28"` |
| `recaptcha` | reCAPTCHA 结果或诊断包装。 | 校验 token、长度、年龄、错误与阶段状态。 | 可以是诊断 JSON 文本；之后非空上游字符串可覆盖同名键，不固定为裸 token。 | `"{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}"` |
| `fid` | 关联标识记录的组合文本。 | 标识组件提供的非 null 记录。 | 按记录成员以竖线拼接；不能仅凭缩写当作 Firebase Installation ID。 | `"9b55089e-f213-4d94-985d-3759a62fe082\|7318042560"` |
| `preloads_app_manager_id` | 预装渠道应用管理标识。 | 对应预装 provider。 | 非 null 时加入；不是包名或系统序列号。 | `"example-manager-id"` |
| `preloads_attribution` | 预装渠道归因。 | 对应归因 provider。 | 非 null 时加入，不与 client_metrics 简单互换。 | `"example-attribution"` |
| `tos_version` | 特定条款流程编号。 | 条款分支条件。 | 符合相应条件时写十进制文本 5。 | `"5"` |
| `entrypoint` | 添加账号或特定账户流程入口。 | PAA/suma 入口与账户状态。 | 条件文本，例如 suma；不是广告获取日志的 code_entrypoint 标签。 | `"suma"` |
| `cred_token` | 绑定号码的凭据。 | 凭据功能配置允许时按当前号码查询映射。 | 有非 null 凭据才加入；不能用 Google token 代替。 | `"example-number-credential"` |
| `e_pq_last_resort_id` | 后量子 last-resort 记录标识。 | 同一条 PQ 记录。 | 与 val/sig 三项均非 null 才成组加入；原始字节 → 无 padding Base64url。 | `"t1gawg"` |
| `e_pq_last_resort_val` | 该 PQ 记录的公钥材料。 | 与 id/sig 同一记录。 | 无 padding Base64url；实际长度由算法/记录决定，不按展示示例补齐或截断。 | `"_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di"` |
| `e_pq_last_resort_sig` | 该 PQ 记录的签名材料。 | 与 id/val 配套的签名。 | 无 padding Base64url；三项完整才加入，示例不定义长度。 | `"oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMn…Iw3mVuAhhN-ddw"` |
| `login` | 特定用户认证分支的登录材料。 | 该分支传入的有效材料。 | 条件文本；不是无条件复制之前响应的同名值。 | `"example-login-material"` |
| `type` | 特定用户/认证类型。 | 调用该请求的业务类型参数。 | 十进制文本；类型分支也可能影响 cc/in 与 UI 参数是否出现。 | `"0"` |

### 组装要点

- **号码与方式快照**：`cc/in/method` 在异步准备前捕获；等待期间修改 UI 或偏好不会反向改变已捕获参数。`token` 使用 national，`id` 的号码口令使用相应规约后的号码，不能串用另一号码材料。
- **备份材料延续**：`backup_token` 先解码注册偏好；坏 Base64 不等于正常空材料。先前 exist 的选择可在响应之前写入此偏好，清理或换号任务也可写空。code 随后遇零长材料会再次生成。各读写的锁不把整个“读取—生成—写回”变成原子事务。
- **交互和诊断**：`reason` 捕获后不随新 writer 改变，空串保留；`mistyped` 读已有偏好而不重算；`hasav=0` 可能来自直接 fallback，不足以判定短信权限被拒。`client_metrics.attempts` 在业务快照产生时递增，重用或序列化该快照不再计数。
- **身份材料成组**：连接公钥与 Signal 身份各自读取；三个 `e_skey_*` 必须来自同一记录，三个 PQ 字段仅在材料齐全时一并加入。读取现有材料不等于触发密钥初始化、上传或轮换。
- **采集与缓存**：`sim_type/airplane_mode_type/cellular_strength/roaming_type` 受同组开关控制，最后一项还要求网络信息；`db` 有自己的开关。系统调用与网络回调不构成原子快照，不能将读取异常统一换成 0、false 或空串。
- **安全对象生命周期**：正常本次生成的 `gpia/_gg` 共用一次 Play 结果，最终 Map 却可读取保留的旧槽；`_gi` 是应用材料而非 Play 结果。对象、context 与库级缓存须按下述独立生命周期处理。
- **诊断与查询分开**：`feo2_query_status` 读取不会发起 provider 查询；准备任务可先写 `did_not_query` 而保留既有 capabilities 缓存。错误字符串不能唯一定位哪一次 IPC 或权限操作。

#### 偏好读取、迁移与清理

偏好返回值取决于实际后端。自建 Light 后端对缺键、内存 Map 中的 null 和错误类型返回调用方默认值，不将字符串或整数强行转成布尔。因而 `backup_token` 可进入默认空串后的生成分支，`mistyped` 可因默认 null 省略，`feo2_query_status` 可回到 `did_not_query`。这不覆盖初始化失败、其它后端行为或非空坏 Base64 的解码异常；同名存储还可能复用早先已建立的系统后端。

活 Editor 的 `clear/apply` 先改变该实例内存 Map，后续落盘失败不自动回滚；单纯删除或替换磁盘文件不使已加载实例每次 get 都重读。存储清理也不直接清除页面 Retriever 标记、会话 wrapper、连接密钥缓存或已捕获的请求值。

注册偏好迁移只处理登记键，例如 `reason` 与最近尝试 `method` 的持久键。源键存在时，目标已有键则保留目标但仍删除源；目标缺键才复制。目标提交、源删除提交和迁移版本推进分步执行，提交返回 false 不会自动回滚。phone-id、`perf_device_id`、`access_session_id`、`token_used_during_reg`、发码尝试与归因键、纠错/教育/Retriever/FEO2 状态不属于这份迁移清单，也不在 localSettings 设置备份白名单；该设置恢复只写允许键，不整库清空。文件级恢复、明确业务 writer 与整库清理须分别处理。

#### 会话、密钥与安全对象

访问会话 reset 先生成并保存/发布 Java UUID，再按 bootstrap 状态同步；原生注册 record 不存在时，同步入口直接无动作，不排队补发。已开始的请求可继续使用先前取得的 record。原生账户 session、协议 store wrapper 和安全对象的重建是另一条链，不自行生成这个 UUID，也不等于重新创建 Java 身份或连接密钥。

协议 store wrapper 的释放与 signed-prekey 按 ID 删除不同。按 ID 删除使用完整 32 位 ID，不用字段 `e_skey_id` 发送时的低 24 位；桥接成功返回不保证实际删除行数大于零。销毁 wrapper 不自动执行该删除回调，不能假定每次 code 请求或原生 reset 都删除预密钥；该回调的具体业务调度条件尚未确定。

安全对象的结果槽包括 `_gg/_gi/_gp/_ga/_ge/_gs` 与 `aid`；`gpia` 经独立 callback 返回，不占这些槽。正常本次生成的 `gpia/_gg` 共用一次 Play 结果，`_gi` 则描述应用材料，不承载 Play 结果码或 token。三者分别包装和发布，单字段失败可能保留旧槽，公共 Map 也可能读到旧 `_gg`；不能保证最终字段都来自同一次生成。整体 owner 重建不继承旧结果槽，同 context 的摘要字典与库全局源槽另有生命周期。详见[安全字段生命周期](security-fields/README.md#zh)。

`_gi` 的完整摘要在 context 键2仅缓存非 null 成功结果，DEX 摘要键0也可缓存规定的错误摘要；它们不等于字段密文槽。main 回调只在库级 `_is` 仍为 null 时尝试发布，首个实际发布的非 null 值会保留；`_p` 在取得路径后、打开文件前即可改变。重建 owner 不清这两层状态，已构造 Map 保留当时值引用，不被后续换槽反向改写。HTTP 完成、页面结束或一次引用释放也不等于全局清理。

`_gp/_ge/_ga` 的非 null 缓存命中可跳过原始读取：权限错误摘要也可缓存，模块配置变化不自动重采，换钥不自动重加密 `_ga.bi`。`_gs` 由独立刷新流程在成功后替换；真正新加载的安全库映射中 prepared 初值为1337，实际 Prepare 写为4242，排队不等于完成。该全局与 Google provider 的 prepared 槽不同。

#### 网络通知与字段取样

网络对象的惰性创建与系统 callback 注册分开。首次 callback 注册失败不自动安装旧广播；已有广播模式升级失败则可继续保留旧广播。500ms 定时任务只合并通知，发送通知时再读当前状态，不是注册重试、HTTP 重试或字段有效期。

`network_radio_type` 与 `roaming_type` 各自在 getter 执行时读取网络状态，后者再独立查电话服务。它们不等待合并通知，也不共享一个原子快照；即使原始网络字段各自为 volatile，跨字段、权限和系统 API 的多次读取仍可能交错。

#### 编码与加密层次

普通文本按所选通道进入表单，不把所有值统一套用一种安全字符集。Kotlin 构造器的 key 与普通文本值使用 Java `URLEncoder(...,"UTF-8")`：空格为 `+`、`*` 保留、`~` 为 `%7E`。原始恢复字节走预编码路径，仅字母、数字和 `-._~` 保留，其它字节为大写 `%HH`；构造器记录预编码身份，拼接时不再转义。保留字段自身 Base64 字母表与 padding，示例中已百分号编码的恢复字节不再编码。

原生 WAMsys 使用自己的 typed dictionary 与枚举布局，不套用 Kotlin 的 Map 顺序或推定所有文本字符的转义完全相同。JSON 型值先序列化内部对象，再作为一个字符串字段进入所选表单。具体差异见[序列化路径](request-encryption.md#zh)。

`gpia`、`_gg`、`_gi` 的字段级包装为：

```text
wrapping_public32 = 当前原生连接密钥读取所派生的公钥
K = SHA256(UTF8(标准Base64含padding(wrapping_public32)))
field = 标准Base64(IV16 || AES256-CBC-PKCS7(K, IV16, JSON_UTF8))
```

三份字段各用自己的 IV。这里的 `wrapping_public32` 由原生连接密钥读取路径从 private32 派生；请求 `authkey` 和 Play `requestHash` 则读取存储 public32。有效公私钥配对且期间没有换钥时可一致，但读取来源、时间和表示不同，不能对损坏记录或并发换钥无条件合并。Play 的 `requestHash` 是该存储公钥的标准 Base64 无 padding，不套上述 SHA-256。

`_ge` 与 `_ga` 外层保留 JSON；`_ga.bi` 对正常读完且保留原始换行的 boot 标识流做 SHA-256/标准 Base64，再单独 CBC。读取失败先改用 ASCII `failed_sha` 求摘要，不使用部分前缀，也不等同成功空流。其 CBC 公共材料来自所属安全对象创建时的快照，不是每次重新读取当前连接公钥；整份 `_ga` 已缓存时，也不会仅因换钥自动更新。

`_gs.em` 使用独立临时 X25519 密钥，共享32字节直接作为 AES256-GCM key，nonce 为零12字节，AAD 为空，tag16；封装为无 padding 的 Base64url(`临时公钥32 || 密文 || tag16`)。这里的 prepared 是执行安全组件状态，不是 Google provider 的 prepared 状态。

最终传输层再序列化内层参数并执行对应的外层封装。`ENC`、`H` 与 HTTP 头不计入本文的 70 个内层参数；签名算法及实际头部由所选传输路径和配置决定，不能把一个通道的头部集合套到所有 code 请求。

#### 外层封装与发送分支

内层参数先完成通道合并与所需刷新，再序列化实际 form 字节；字段目录和展示 JSON 都不是最终 HTTP 正文。正常加密分支的 `ENC` 使用新临时 X25519 共享32字节直接作 AES256-GCM key，零 nonce12、空 AAD、tag16，无额外 KDF；输出 raw 公钥32与密文/tag 的无 padding Base64url。启用证明时，`H` 签 `ENC` 文本；`Authorization` 是匹配签名者的逆序 DER 链拼接后标准 Base64 含 padding，无 Bearer 前缀。

Kotlin builder 的加密开关、endpoint 证明开关与签名者内部开关分别控制分支：成功加密可为仅 `ENC` 或 `ENC/H`，后者签名不可用时仍可有空 `H=`。未加密或加密失败回退时使用原始 query，若附 H 则签该 query。不能把这些回退或 WAMsys 的六项应用头套给所有请求；签名与证书读取也不是原子快照。[封装分支](request-encryption.md#zh)与[发送流程](sending-requests.md#zh)分别说明其条件。

Kotlin 网络重试最多选择4个 attempt 条目，含首次 ORIGINAL；解析成功标记为 true 时立即停止，HTTP200 的业务 fail 也如此。它复用参数 builder 与预编码集合，可重新生成 ENC/H，不重跑上游 UI、Google 或业务计数；这不定义 WAMsys 的重试策略。传输成功、JSON 解析和业务结果须分层判断。

<a id="plaintext-example"></a>

### 明文示意

下例按表格顺序展示全部 70 个名字，便于检索。条件分支集中展示，包括普通请求中不会同时使用的 SMS 与 Autoconf 材料；应按实际流程选择，不整份发送。所有值均为字符串，这个对象不是最终 HTTP 正文。

```json
{
  "cc": "1",
  "in": "2025550147",
  "lg": "en",
  "lc": "US",
  "method": "sms",
  "rc": "0",
  "fdid": "ce9ecf75-f380-4676-aa7f-a91c421c4466",
  "expid": "YucHBVW5kh37FAN49P82LQ",
  "access_session_id": "hzw8ZMqxkqN-07b-V3NJdg",
  "aid": "IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY=",
  "id": "%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF",
  "backup_token": "%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C",
  "hasinrc": "1",
  "authkey": "Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8",
  "e_ident": "oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8",
  "e_keytype": "BQ",
  "e_regid": "AAAwOQ",
  "e_skey_id": "AAFB",
  "e_skey_val": "HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ",
  "e_skey_sig": "EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUVBNQiAS2KyvwU1nah1PFSjB4yMm04B-qjHygUlWToo4Lg",
  "token": "4s+/5KpaPw6zhEmWAm+hHBGzUqQ=",
  "gpia": "YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgjHFieOwStufC1uiKjjsxyNToHILBB3K08sPOY2axX0v7IBwAUuswhuJjMcgL1C3IKDjDuSO6S9MAz6xVLkCY5Hkml6PwMFNfBH76S6worM8dutJRyqxYwZUR//pYsqc52guScZVilvVUlBB9SPYPW1JObkw8M0zuGBrYvpEIBOarw==",
  "_gg": "s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X",
  "_gi": "F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnnwGfuSLgigcDdTWFOLQWiKDHL1zM0EXK57kskKL022A3l00A9IZ0VQVszK2zZgY3fKc6CVCaXJ5IMo8LT2XjiwwPafZ9DW3adcfw8jXUZ/sofnkgLhuOPgrv9bb4qGZ8vxs2AggITRaaiITPLroYXB0lxD1sf9gbFc3kiQTvjmAMQ==",
  "_gp": "kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE=",
  "_gs": "{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_NhtinJDP13Vch4tShkkqXHtHcFu8WHdSYkx9bojyK-LYjHe2V8pcXyauRVsTcueJnUSX3n1Tbt1Fgsl6J7zIDJfpUFhyM2CiPVhk7cE3Pd4ESLSYOEomGk0VEQKBwgY7uv4SDPDUsPr-7qjfJ3z_iCaDdGy7jTcafh7e3X2cmGswg90l71XdabOvGmXXTPJzSb2wjKFZTI\"}",
  "_ge": "{\"sb\":false,\"sv\":false}",
  "_ga": "{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2AMI3JZGLVB66SVOydUJd5ONOWnXZqCdGj1bkP3IzGBjh9NMWzUtQ==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}",
  "mcc": "310",
  "mnc": "260",
  "sim_mcc": "310",
  "sim_mnc": "260",
  "network_radio_type": "1",
  "simnum": "0",
  "pid": "23456",
  "device_ram": "6",
  "db": "0",
  "sim_type": "0",
  "airplane_mode_type": "0",
  "cellular_strength": "5",
  "roaming_type": "0",
  "mistyped": "7",
  "reason": "",
  "hasav": "1",
  "client_metrics": "{\"attempts\":2,\"app_campaign_download_source\":\"unknown|unknown\"}",
  "education_screen_displayed": "false",
  "prefer_sms_over_flash": "false",
  "feo2_query_status": "did_not_query",
  "advertising_id": "98ea2934-49f4-4245-ae15-339d0d3c6145",
  "push_code": "example-push-code",
  "context": "invite_registration",
  "client_start_message": "SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA",
  "clicked_education_link": "false",
  "manage_call_permission": "true",
  "call_log_permission": "false",
  "sim_status": "1",
  "old_phone_number": "12025550186",
  "first_api_level": "28",
  "recaptcha": "{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}",
  "fid": "9b55089e-f213-4d94-985d-3759a62fe082|7318042560",
  "preloads_app_manager_id": "example-manager-id",
  "preloads_attribution": "example-attribution",
  "tos_version": "5",
  "entrypoint": "suma",
  "cred_token": "example-number-credential",
  "e_pq_last_resort_id": "t1gawg",
  "e_pq_last_resort_val": "_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di",
  "e_pq_last_resort_sig": "oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMnnj980ZhbmuqIpDQDRSX_UoFR3E95MpIw3mVuAhhN-ddw",
  "login": "example-login-material",
  "type": "0"
}
```

`id` 与 `backup_token` 已将原始字节按百分号编码展示，不要再次编码该展示文本；其它值为表单编码前的表示。JSON 型字段保持紧凑 JSON 字符串。PQ 三项仅展示编码外形，实际使用对应算法规定长度与同一条记录。随机包络形状的数据不能解密成有效载荷。

<a id="mock-fields-java"></a>

#### Java 模拟示例

下面的 Android Java 类构造与上方一致的 70 字段展示对象。固定合成值便于重复查看，不实现真实来源读取、安全字段计算或请求认证。

```java
// File: CodeMockFields.java
package wiki.examples;

import android.util.Log;
import org.json.JSONException;
import org.json.JSONObject;

public final class CodeMockFields {
    private CodeMockFields() {}

    // Synthetic display data, not credentials or a sendable request.
    public static JSONObject create() throws JSONException {
        JSONObject fields = new JSONObject();
        fields.put("cc", "1");
        fields.put("in", "2025550147");
        fields.put("lg", "en");
        fields.put("lc", "US");
        fields.put("method", "sms");
        fields.put("rc", "0");
        fields.put("fdid", "ce9ecf75-f380-4676-aa7f-a91c421c4466");
        fields.put("expid", "YucHBVW5kh37FAN49P82LQ");
        fields.put("access_session_id", "hzw8ZMqxkqN-07b-V3NJdg");
        fields.put("aid", "IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY=");
        fields.put("id", "%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF");
        fields.put("backup_token", "%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C");
        fields.put("hasinrc", "1");
        fields.put("authkey", "Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8");
        fields.put("e_ident", "oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8");
        fields.put("e_keytype", "BQ");
        fields.put("e_regid", "AAAwOQ");
        fields.put("e_skey_id", "AAFB");
        fields.put("e_skey_val", "HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ");
        fields.put("e_skey_sig", "EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUVBNQiAS2KyvwU1nah1PFSjB4yMm04B-qjHygUlWToo4Lg");
        fields.put("token", "4s+/5KpaPw6zhEmWAm+hHBGzUqQ=");
        fields.put("gpia", "YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgjHFieOwStufC1uiKjjsxyNToHILBB3K08sPOY2axX0v7IBwAUuswhuJjMcgL1C3IKDjDuSO6S9MAz6xVLkCY5Hkml6PwMFNfBH76S6worM8dutJRyqxYwZUR//pYsqc52guScZVilvVUlBB9SPYPW1JObkw8M0zuGBrYvpEIBOarw==");
        fields.put("_gg", "s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X");
        fields.put("_gi", "F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnnwGfuSLgigcDdTWFOLQWiKDHL1zM0EXK57kskKL022A3l00A9IZ0VQVszK2zZgY3fKc6CVCaXJ5IMo8LT2XjiwwPafZ9DW3adcfw8jXUZ/sofnkgLhuOPgrv9bb4qGZ8vxs2AggITRaaiITPLroYXB0lxD1sf9gbFc3kiQTvjmAMQ==");
        fields.put("_gp", "kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE=");
        fields.put("_gs", "{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_NhtinJDP13Vch4tShkkqXHtHcFu8WHdSYkx9bojyK-LYjHe2V8pcXyauRVsTcueJnUSX3n1Tbt1Fgsl6J7zIDJfpUFhyM2CiPVhk7cE3Pd4ESLSYOEomGk0VEQKBwgY7uv4SDPDUsPr-7qjfJ3z_iCaDdGy7jTcafh7e3X2cmGswg90l71XdabOvGmXXTPJzSb2wjKFZTI\"}");
        fields.put("_ge", "{\"sb\":false,\"sv\":false}");
        fields.put("_ga", "{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2AMI3JZGLVB66SVOydUJd5ONOWnXZqCdGj1bkP3IzGBjh9NMWzUtQ==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}");
        fields.put("mcc", "310");
        fields.put("mnc", "260");
        fields.put("sim_mcc", "310");
        fields.put("sim_mnc", "260");
        fields.put("network_radio_type", "1");
        fields.put("simnum", "0");
        fields.put("pid", "23456");
        fields.put("device_ram", "6");
        fields.put("db", "0");
        fields.put("sim_type", "0");
        fields.put("airplane_mode_type", "0");
        fields.put("cellular_strength", "5");
        fields.put("roaming_type", "0");
        fields.put("mistyped", "7");
        fields.put("reason", "");
        fields.put("hasav", "1");
        fields.put("client_metrics", "{\"attempts\":2,\"app_campaign_download_source\":\"unknown|unknown\"}");
        fields.put("education_screen_displayed", "false");
        fields.put("prefer_sms_over_flash", "false");
        fields.put("feo2_query_status", "did_not_query");
        fields.put("advertising_id", "98ea2934-49f4-4245-ae15-339d0d3c6145");
        fields.put("push_code", "example-push-code");
        fields.put("context", "invite_registration");
        fields.put("client_start_message", "SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA");
        fields.put("clicked_education_link", "false");
        fields.put("manage_call_permission", "true");
        fields.put("call_log_permission", "false");
        fields.put("sim_status", "1");
        fields.put("old_phone_number", "12025550186");
        fields.put("first_api_level", "28");
        fields.put("recaptcha", "{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}");
        fields.put("fid", "9b55089e-f213-4d94-985d-3759a62fe082|7318042560");
        fields.put("preloads_app_manager_id", "example-manager-id");
        fields.put("preloads_attribution", "example-attribution");
        fields.put("tos_version", "5");
        fields.put("entrypoint", "suma");
        fields.put("cred_token", "example-number-credential");
        fields.put("e_pq_last_resort_id", "t1gawg");
        fields.put("e_pq_last_resort_val", "_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di");
        fields.put("e_pq_last_resort_sig", "oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMnnj980ZhbmuqIpDQDRSX_UoFR3E95MpIw3mVuAhhN-ddw");
        fields.put("login", "example-login-material");
        fields.put("type", "0");
        return fields;
    }

    public static void printExample() throws JSONException {
        for (String line : create().toString(2).split("\n")) {
            Log.d("CodeMock", line);
        }
    }
}
```

调用 `CodeMockFields.create()` 取得 `JSONObject`，或调用 `CodeMockFields.printExample()` 逐行输出到 Logcat；调用方处理 `JSONException`。该类不读取手机状态、不写存储，也不发送 HTTP 请求。

### 相关内容

- [注册目录](README.md#zh)：请求顺序与文章索引。
- [exist 参数](exist.md#zh)：号码检查与其自身的字段读取规则。
- [Android 获取代码](android-inputs.md#zh)：公开 API、持久状态与字节编码。
- [security-fields/](security-fields/README.md#zh)：七个安全字段的详细结构与算法。
- [request-encryption.md](request-encryption.md#zh)：序列化与外层封装。
- [sending-requests.md](sending-requests.md#zh)：发送、请求头与响应处理。

[返回注册目录](README.md#zh)

<a id="en"></a>

## English

`/v2/code` requests a verification code or authentication flow through SMS, voice, or another supported route. It carries the number, selected route, client identity, and device state. This is the code-request stage in `exist → code → register`; a later verification request submits the code received by the user.

This reference groups 70 inner parameters by function, including fields gated by collection settings or authentication type. An actual request does not send a fixed set of 70. Use the parameters and state captured for the selected SMS, voice, flash, or autoconf flow. [Android input examples](android-inputs.md#en) cover system reads and some encoding entry points; the code-specific readers and representations below govern this article.

Table and example values are strings. Fields such as `client_metrics`, `_ge`, and `_ga` contain JSON text inside those strings. “Plaintext” means before outer encryption, not before all field-level encoding or encryption. `id` and `backup_token` are raw binary material, displayed here as percent-encoded text.

The phone numbers are examples; identifiers, digests, and cryptographic payloads are synthetic display values. Fixed-width fields preserve representation lengths, while variable-length examples do not specify protocol sizes. These values do not form valid credentials, signatures, or security proofs.

### 1. Number, locale, and verification route

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `cc` | Calling code of the number to verify. | The confirmed number state, captured together with the national number. | Digits without +. Number-based flows include it; a designated authentication type can omit both cc and in. | `"1"` |
| `in` | National number without the calling code. | The same confirmed number state as cc. | Keep it as text and use the same number for token and id. This is not the verification code to submit. | `"2025550147"` |
| `lg` | Current application language. | Locale from the application Resources.Configuration; [locale inputs](android-inputs.md#en-app-locale). | A non-null language-text cache hit can be reused directly. On a miss, use the language manager’s resolved Locale: 2–3 lowercase letters or zz. Not derived from the calling code. | `"en"` |
| `lc` | Region of the application locale. | The country component of the application Locale. | Two uppercase letters or three digits; malformed values use ZZ. Independent of cc. | `"US"` |
| `method` | Verification route selected for this request. | The route captured by the page or business caller. | Text such as sms, voice, flash, or autoconf. The last attempted route is saved before sending, without waiting for sent; suspension does not refresh it from the UI. | `"sms"` |
| `rc` | Release-channel number. | Application release-channel configuration. | Decimal text, for example 0 for the production channel; independent of the Android API level. | `"0"` |

### 2. Device and access-session identifiers

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `fdid` | Persistent phone-id. | The application light preferences: phoneid_id and its associated timestamp. | Hyphenated UUID text. Created and saved when missing; import or synchronization can update it. Reused across requests. | `"ce9ecf75-f380-4676-aa7f-a91c421c4466"` |
| `expid` | Persistent performance-device identifier. | Reuse perf_device_id first; if missing, derive a name UUID from Android ID, or a random UUID for a normal null/blank result. | The UUID as 16 big-endian bytes → unpadded Base64url. A non-null saved value avoids a new Android ID lookup. | `"YucHBVW5kh37FAN49P82LQ"` |
| `access_session_id` | Correlates requests within an access session. | Session-manager memory state and reg_prefs.access_session_id. | UUID16 → unpadded Base64url. Reuse an existing memory wrapper; only a missing wrapper reads preferences and generates when needed. Explicit reset persists/publishes the new value before conditional native synchronization. | `"hzw8ZMqxkqN-07b-V3NJdg"` |
| `aid` | Digest of Android ID. | Android ID from the application ContentResolver; [raw input](android-inputs.md#en-android-id). | JNI modified UTF-8 bytes → SHA-256 → padded standard Base64. Explicit read-failure branches hash signal_error. Cached by the owning security object. | `"IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY="` |

### 3. Recovery and registration material

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `id` | Local recovery material. | Read and decrypt the application files/rc2 record, with the number contributing to its password. | Percent-encode the raw bytes. Only a null reader result leads to generation and attempted persistence; format and decryption errors have separate handling. | `"%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF"` |
| `backup_token` | Backup material retained for registration. | Base64-decode reg_prefs.token_used_during_reg. | Reuse nonempty bytes; generate/save only for zero length, then percent-encode raw bytes. Custom Light backends can default missing/wrong-type values to empty text; malformed nonempty Base64 still fails. No rerun of exist backup selection. | `"%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C"` |
| `hasinrc` | Whether the local recovery file exists. | The rc2 file in the application files directory; [file check](android-inputs.md#en-recovery-record). | 1 when present, otherwise 0. Does not establish valid or decryptable contents. | `"1"` |

### 4. Connection and end-to-end public identity

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `authkey` | Public half of the connection-authentication key. | The current public half in the connection AuthKeyStore. | Stored public32 → unpadded Base64url. Native CBC wrapping separately derives a public key from connection private32; equivalence requires a valid pair and consistent read state. | `"Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8"` |
| `e_ident` | Local Signal identity public key. | The existing local Signal identity record. | raw32 without the type prefix → unpadded Base64url. Do not create a new identity for each code request. | `"oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8"` |
| `e_keytype` | Signal public-key type tag. | The public-key type constant. | Byte 0x05 → Base64url text BQ, not decimal text 5. | `"BQ"` |
| `e_regid` | Local Signal registration identifier. | The current Signal store and its cache. | Four-byte big-endian integer → unpadded Base64url. | `"AAAwOQ"` |
| `e_skey_id` | Identifier of the selected signed prekey. | The selected signed-prekey record. | Low 24 bits of the record id as three big-endian bytes → unpadded Base64url; not simply the SQL row identifier. | `"AAFB"` |
| `e_skey_val` | Public key of that signed prekey. | The same record as e_skey_id. | raw32 without its type prefix → unpadded Base64url. | `"HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ"` |
| `e_skey_sig` | Signature stored with that prekey. | The same record snapshot as its id and public key. | Signature bytes → unpadded Base64url. Neither HTTP H nor a signature from another record is a substitute. | `"EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUV…qjHygUlWToo4Lg"` |

### 5. Client validation and security fields

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `token` | Client validation value bound to application content and the national number. | Package name, the logo resource stream, selected signing certificate, primary classes.dex, and national number. | PBKDF2-HMAC-SHA1 with 128 iterations and a 64-byte derived key feeds HMAC-SHA1; padded standard Base64 output. Distinct from a Google token or verification code. | `"4s+/5KpaPw6zhEmWAm+hHBGzUqQ="` |
| `gpia` | Play Integrity outcome combined with application-integrity metadata. | One asynchronous Play outcome plus application metadata. | JSON → AES-CBC envelope → standard Base64. The wrapping public key comes from the current native connection-key read, not a direct copy of request authkey. Play acquisition is gated and can return a token or error code. [Details](security-fields/gpia.md#en) | `"YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgj…BrYvpEIBOarw=="` |
| `_gg` | Compact Play outcome envelope. | Fresh generation shares an outcome code and optional token with gpia; a public Map reading an older slot need not share that invocation. | Internal _ic and optional _it; CBC/standard Base64 with the current native connection public key and its own IV. Fields publish independently; single-field failure or gating need not clear older values. [Details](security-fields/gg.md#en) | `"s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X"` |
| `_gi` | Integrity information for installed content, signatures, paths, and library directories. | Application/DEX streams, certificates, and actual directories; [package inputs](android-inputs.md#en-package-material). | Composite JSON in a CBC/standard Base64 envelope using the current native connection-key read. Keys depend on configuration, local caches, and asynchronous publication; rebuilding the result owner need not clear all source caches. [Details](security-fields/gi.md#en) | `"F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnn…c3kiQTvjmAMQ=="` |
| `_gp` | Digest of declared permission names. | PackageInfo.requestedPermissions, rather than granted runtime permissions. | Sort JNI modified UTF-8 names by unsigned bytes, retain duplicates, concatenate without separators → SHA-256 → padded standard Base64. Empty concatenation (including all-empty names) or specified API failure uses default_permission; cache miss with no Application can return null. Non-null error digests can also be cached. [Details](security-fields/gp.md#en) | `"kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE="` |
| `_gs` | Security signals from path checks, time, randomness, and execution readiness. | 13 stat checks, time primitives, libc randomness, prepared state, and separate encryption randomness. | Outer JSON contains em: unpadded Base64url of ephemeral public32 and AES-GCM ciphertext/tag16. Refreshes through the enabled collection flow. [Details](security-fields/gs.md#en) | `"{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_Nh…JzSb2wjKFZTI\"}"` |
| `_ge` | Module detection and optional su path checks. | stat results for visible paths and the relevant configuration. | Compact JSON: sv is /sys/module/vmw_pvscsi and sb is /sys/module/vboxsf, each a stat==0 boolean. su is conditional; omission is not false. Non-null cache hits can skip configuration/path checks. No whole-field CBC/Base64. [Details](security-fields/ge.md#en) | `"{\"sb\":false,\"sv\":false}"` |
| `_ga` | Path ages, Monkey checks, and boot-identifier information. | Path mtime and separate time calls, process/system APIs, and the raw boot_id stream. | Outer JSON: ap/ai/ae are seconds and mp/mu booleans. Only bi has CBC, using a public-material snapshot taken when its owner was created. The complete value can remain cached; a request or key change does not automatically resample or re-encrypt it. [Details](security-fields/ga.md#en) | `"{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2A…30,\"mu\":false}"` |

### 6. Operators, device, and network state

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `mcc` | Mobile country code of the current telephony operator. | The complete TelephonyManager.getNetworkOperator() string. | Requires a full match of three MCC digits followed by 2–3 MNC digits; take the first three. Missing or malformed input defaults to 000. | `"310"` |
| `mnc` | Mobile network code of the current telephony operator. | The same parse as mcc. | Parse MNC as an integer, then format with Locale.US %03d; for example 46001 yields 001. | `"260"` |
| `sim_mcc` | Mobile country code of the SIM operator. | TelephonyManager.getSimOperator(). | The same full 5–6 digit format and default 000; not derived from the number being verified. | `"310"` |
| `sim_mnc` | Mobile network code of the SIM operator. | The same parse as sim_mcc. | Three-digit integer formatting, default 000; separate from the current network operator. | `"260"` |
| `network_radio_type` | Application-specific network classification. | Asynchronous network state and cellular subtype; [network inputs](android-inputs.md#en-network-type). | Decimal enum: legacy type 1 alone is Wi-Fi class 1; the initialized SDK≥31 path groups transport 1/3 as 1. Other types use the selected mobile/wifi flags; when neither flag is set, return -1. LTE 111, NR 113, unknown mobile 0. Reads the current snapshot without waiting for 500 ms notifications. | `"1"` |
| `simnum` | Whether an available device number meets the length threshold. | With permission, prefer the first non-null subscription number, then fall back to the telephony-service number. | 1 for non-null text with Java length ≥ 6, otherwise 0. Not a SIM count or a match against the entered number. | `"0"` |
| `pid` | Identifier of the collecting process. | android.os.Process.myPid(). | Decimal text; can change after a process restart. | `"23456"` |
| `device_ram` | Total system memory in GiB. | ActivityManager.MemoryInfo.totalMem ÷ 2³⁰. | Divide totalMem by 1024³ and format GiB using #.## with the actual formatting Locale; non-null text can be cached per instance. Missing service/specified NullPointerException first supplies -1 bytes before division/formatting, not literal "-1". Not availMem. | `"6"` |
| `db` | Read result of the ADB-enabled setting. | Settings.Global.adb_enabled, default 0. | Decimal text only when collection is enabled; disabled collection omits it instead of inserting 0. | `"0"` |

### 7. Optional telephony-environment fields

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `sim_type` | Application SIM-availability class. | Telephony-service availability and getSimState(). | Included when the group gate is enabled: no service 2, SIM absent 0, other states 1. | `"0"` |
| `airplane_mode_type` | Airplane-mode setting. | Settings.Global.airplane_mode_on. | Controlled by the group gate; normalized to 0/1. | `"0"` |
| `cellular_strength` | Cellular signal level or unavailable fallback. | getLevel() when API ≥ 28 and the telephony service and signal object are available. | Group-gated; normally 0..4, with fallback 5 for missing support/service/first signal object. Two reads are not atomic; arbitrary errors are not all mapped to 5. | `"5"` |
| `roaming_type` | Telephony roaming classification. | A network snapshot and isNetworkRoaming(). | Requires the group gate and a fresh network-information read: not roaming 0, roaming 1, no telephony service 2. Omit absent network information; not an atomic snapshot shared with network_radio_type. | `"0"` |

### 8. Automatic receipt, flow metrics, and diagnostics

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `mistyped` | Combined number-correction and phone-permission state. | mistyped_state saved in light preferences by the number-entry flow. | Include non-null text, preserving empty text; omit when custom Light defaults a missing/wrong-type value to null. code does not recompute correction or substitute -1. 7 can represent missing phone-information permission. | `"7"` |
| `reason` | Shared client registration diagnostic. | In-memory diagnostic state, restored/saved by related pages through registration_failure_reason. | Captured before asynchronous preparation; include even an empty string. Reading does not clear it. It can retain earlier request, receipt, or local-validation labels rather than copying the current response reason. | `""` |
| `hasav` | Automatic code-receipt mode/state. | The page SMS Retriever flag and RECEIVE_SMS permission, or a direct parameter from a specific fallback. | Standard page: Retriever true → 2; otherwise granted RECEIVE_SMS → 1, other normal results → 0. A special fallback can pass 0 directly. A present but wrong-type Retriever preference can default to false and override the Intent. Omit null. | `"1"` |
| `client_metrics` | Code-generation attempt metrics and installation attribution. | An attempt snapshot from reg_attempts_generate_code and the attribution component. | Compact JSON with integer attempts. The business producer increments/saves it, not the serializer. Custom Light defaults a wrong-type counter to 0, potentially producing attempts=1. Do not copy exist counters or late-stub additions. | `"{\"attempts\":2,\"app_campaign_download_source\":\"unknown\|unknown\"}"` |
| `education_screen_displayed` | Local flash-call education-page display flag. | light.pref_flash_call_education_screen_displayed, default false. | true/false text. Set when page creation reaches the writer, reset by the relevant number-page creation, and removable by cleanup. Does not mean the page was fully viewed. | `"false"` |
| `prefer_sms_over_flash` | Local preference for SMS over flash calls. | light.pref_prefer_sms_over_flash, default false. | true/false text. Only a qualified SMS-selection path sets true; number-page initialization can reset it. Reading does not consume it. | `"false"` |
| `feo2_query_status` | Saved diagnostic for an associated-service capability query. | light.pref_autoconf_feo2_query_status; missing keys default to did_not_query. | Read-only in code; no provider query. Custom Light defaults missing, Map-null, or wrong-type values to did_not_query; empty/unknown text is retained. Only an actual null delivered to the caller triggers its non-null check. Resetting diagnostics does not clear capabilities. | `"did_not_query"` |

### 9. Additional parameters for specific flows

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `advertising_id` | Resettable advertising identifier. | The advertising-ID provider; [advertising input](android-inputs.md#en-advertising-id). | Include a non-null result when country/flow conditions allow. The newer provider omits for LAT=true; the older direct path does not read LAT. No automatic failure-to-other-branch fallback; do not invent UUIDs or empty values. | `"98ea2934-49f4-4245-ae15-339d0d3c6145"` |
| `push_code` | Registration verification material delivered through push. | The registration_push_notif_code preference. | Include only non-null, nonempty text. Distinct from push_token. | `"example-push-code"` |
| `context` | Authentication context. | The upstream authentication flow. | Non-null text, for example invite_registration; the caller determines the actual value. | `"invite_registration"` |
| `client_start_message` | Autoconf client-start message. | Valid bytes from Autoconf preparation. | Unpadded Base64url. An empty required Autoconf message fails before sending; this is not a mandatory SMS field. | `"SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA"` |
| `clicked_education_link` | Education-link interaction flag. | An integer UI parameter from the flow. | 0 → false, 1 → true, other integers omitted; authentication-type gated. | `"false"` |
| `manage_call_permission` | Call-management permission state carried by the flow. | An integer parameter supplied by the flow. | Only 0/1 become false/true; omit other values rather than sending the raw integer. | `"true"` |
| `call_log_permission` | Call-log permission state carried by the flow. | An integer parameter supplied by the flow. | Only 0/1 become false/true; omit other values. Bind it to the actual permission-reading flow. | `"false"` |
| `sim_status` | Cached auxiliary SIM state. | reg_prefs.pref_cached_sim_status. | Include non-null text; distinct from sim_type and the system sim_state. | `"1"` |
| `old_phone_number` | Previous number of an existing account. | Old calling code and national number from an existing account object. | Concatenate old cc + national without +. Do not initialize a missing account object solely for this field. | `"12025550186"` |
| `first_api_level` | Factory API level of the device. | ro.product.first_api_level. | Include a positive property value when enabled; not the current SDK_INT. | `"28"` |
| `recaptcha` | reCAPTCHA outcome or diagnostic envelope. | Token, length, age, error, and stage state. | Can be diagnostic JSON text; a later nonempty upstream string can overwrite the same key. Not always a bare token. | `"{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}"` |
| `fid` | Composite text from an associated identifier record. | A non-null record from the identifier component. | Join the record members with a vertical bar; the name alone does not establish a Firebase Installation ID. | `"9b55089e-f213-4d94-985d-3759a62fe082\|7318042560"` |
| `preloads_app_manager_id` | Preload application-manager identifier. | The corresponding preload provider. | Include a non-null result; not the package name or system serial number. | `"example-manager-id"` |
| `preloads_attribution` | Preload attribution. | The attribution provider. | Include a non-null result; not interchangeable with client_metrics. | `"example-attribution"` |
| `tos_version` | Terms-flow identifier. | The applicable terms branch. | Decimal text 5 when the branch condition holds. | `"5"` |
| `entrypoint` | Entry point for account addition or a specific account flow. | PAA/suma entry state and account state. | Conditional text such as suma; distinct from the advertising-fetch log label code_entrypoint. | `"suma"` |
| `cred_token` | Number-bound credential. | Lookup by the current number when credential collection is enabled. | Include a non-null credential; a Google token is not a substitute. | `"example-number-credential"` |
| `e_pq_last_resort_id` | Identifier of a post-quantum last-resort record. | One PQ record. | Include with val/sig only when all three are non-null; raw bytes → unpadded Base64url. | `"t1gawg"` |
| `e_pq_last_resort_val` | Public material of that PQ record. | The same record as id/sig. | Unpadded Base64url. Its algorithm/record defines the actual size; do not pad or truncate to match the illustration. | `"_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di"` |
| `e_pq_last_resort_sig` | Signature material of that PQ record. | The signature paired with id/val. | Unpadded Base64url; include the complete triplet. The example does not define its size. | `"oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMn…Iw3mVuAhhN-ddw"` |
| `login` | Login material for a specific authentication branch. | Valid material supplied by that branch. | Conditional text; do not automatically copy a same-named value from an earlier response. | `"example-login-material"` |
| `type` | User/authentication type for a specific flow. | The business-type parameter supplied by the caller. | Decimal text; the type branch can also affect the presence of cc/in and UI parameters. | `"0"` |

### Assembly Rules

- **Number and route snapshots**: `cc/in/method` are captured before asynchronous preparation. UI or preference changes during suspension do not change those captured arguments. `token` uses the national number; the `id` password uses its corresponding normalized number. Keep the number-bound materials consistent.
- **Retained backup material**: decode the registration preference for `backup_token` first. Malformed Base64 is not normal empty material. An earlier exist selection can write this preference before its response; cleanup or number-change tasks can also write empty bytes. A later code read regenerates zero-length material. Individual read/write locks do not make the entire read–generate–write sequence atomic.
- **Interaction and diagnostics**: a captured `reason` does not change with later writers; preserve empty text. Read saved `mistyped` without recomputing it. `hasav=0` can come from a direct fallback and does not establish denied SMS permission. Increment `client_metrics.attempts` when creating its business snapshot, not when reusing or serializing it.
- **Paired identity material**: read the connection public key and Signal identity separately. All three `e_skey_*` fields must share a record; include a PQ triplet only when complete. Reading existing material is not itself a trigger for key initialization, upload, or rotation.
- **Collection and caching**: one group gate controls `sim_type/airplane_mode_type/cellular_strength/roaming_type`; roaming also requires network information. `db` has its own gate. API calls and network callbacks are not an atomic snapshot. Do not replace every read error with 0, false, or empty text.
- **Security-object lifetime**: normal fresh `gpia/_gg` generation shares one Play outcome, but the final Map can read retained older slots. `_gi` contains application material rather than a Play outcome. Handle owner, context, and library caches through the separate lifetimes below.
- **Diagnostics versus queries**: reading `feo2_query_status` does not call the provider. Preparation can write `did_not_query` while retaining cached capabilities. An error string alone cannot identify a particular IPC or permission operation.

#### Preference Reads, Migration, and Cleanup

Preference results depend on the actual backend. Custom Light backends return the caller's default for missing keys, null Map values, and wrong types; they do not coerce strings or integers into booleans. Thus `backup_token` can enter generation after defaulting to empty text, `mistyped` can be omitted through a null default, and `feo2_query_status` can default to `did_not_query`. This does not cover initialization failure, other backends, or decoding malformed nonempty Base64. A named store can also reuse a system backend established earlier.

An active Editor's `clear/apply` changes that instance's memory Map before persistence, and a later disk failure does not automatically roll it back. Merely deleting or replacing a file does not make an already loaded instance reload on every get. Storage cleanup also does not directly clear page Retriever flags, session wrappers, connection-key caches, or captured request values.

Registration-preference migration processes registered keys, including the persistent diagnostic `reason` and last-attempted `method` keys. When a source key exists, an existing destination wins but the source is still removed; copying occurs only for a missing destination. Destination commit, source-removal commit, and migration-version advancement are separate steps; false commit results do not trigger automatic rollback. Phone-id, `perf_device_id`, `access_session_id`, `token_used_during_reg`, code-attempt and attribution keys, and correction/education/Retriever/FEO2 state are outside this migration list and the localSettings backup allowlist. That settings restore writes allowed keys without clearing the whole store. Treat file-level restoration, explicit business writers, and whole-store cleanup separately.

#### Sessions, Keys, and Security Owners

An access-session reset generates and persists/publishes the Java UUID before bootstrap-gated native synchronization. With no native registration record, the synchronization entry does nothing and queues no later replay. A request already started can retain the record it previously obtained. Rebuilding a native account session, protocol-store wrappers, and the security owner is a separate chain: it does not generate this UUID or itself recreate Java identity or connection keys.

Releasing a protocol-store wrapper differs from deleting a signed prekey by ID. ID-based deletion uses the full 32-bit ID, not the low 24 bits sent as `e_skey_id`; successful bridge completion does not establish a positive deleted-row count. Wrapper destruction does not automatically invoke that deletion callback. Do not assume every code request or native reset deletes prekeys; the callback's specific business scheduling conditions remain undetermined.

The security-owner result slots are `_gg/_gi/_gp/_ga/_ge/_gs` and `aid`; `gpia` returns through a separate callback, outside those slots. Normal fresh `gpia/_gg` generation shares one Play outcome, whereas `_gi` describes application material and contains no Play outcome code or token. The three are wrapped and published separately. Field failure can retain an older slot, and the public Map can read an older `_gg`; final fields need not all belong to one generation. Rebuilding an owner does not inherit its old result slots, while the same context's digest dictionary and library-global source slots have separate lifetimes. See [security-field lifetimes](security-fields/README.md#en).

For `_gi`, context key 2 caches only non-null successful full-file digests, while DEX digest key 0 can also cache specified error digests; neither is the field-ciphertext slot. A main callback attempts publication only while library `_is` is null, retaining the first non-null value actually published. `_p` can change after path acquisition and before opening. Owner rebuilding does not clear these layers. An already built Map retains its sampled value references rather than changing when a slot is replaced. HTTP completion, a page ending, or releasing one reference is not global cleanup.

Non-null `_gp/_ge/_ga` cache hits can skip raw reads: permission-error digests can remain cached, module-configuration changes do not automatically resample, and rotation does not automatically re-encrypt `_ga.bi`. A separate `_gs` refresh replaces its value on success. A genuinely new security-library mapping starts prepared at 1337; actual Prepare writes 4242, and queueing does not establish completion. This global differs from the Google provider's prepared slot.

#### Network Notifications and Field Sampling

Lazy network-object creation is separate from system callback registration. An initial callback-registration failure does not automatically install the legacy broadcast receiver; a failed upgrade from an existing broadcast mode can retain that receiver. The 500 ms timer only coalesces notifications and reads current state again when delivering. It is not a registration retry, HTTP retry, or field lifetime.

`network_radio_type` and `roaming_type` each read network state when their getter runs; roaming then queries the telephony service separately. They neither wait for coalesced notification nor share an atomic snapshot. Even volatile raw network fields do not make multiple field, permission, and API reads atomic.

#### Encoding and Encryption Layers

Use the selected transport's form path rather than one safe-character set for all values. The Kotlin builder applies Java `URLEncoder(...,"UTF-8")` to keys and ordinary text values: space becomes `+`, `*` stays literal, and `~` becomes `%7E`. Raw recovery bytes take a preencoded path preserving only letters, digits, and `-._~`, with uppercase `%HH` otherwise. The builder tracks preencoded identity and appends without another escape. Preserve field-specific Base64 alphabets/padding and do not re-encode recovery-byte display examples.

Native WAMsys uses its own typed dictionary and enumeration layout; do not impose Kotlin Map order or assume identical escaping for all text characters. Serialize a JSON-valued object internally, then carry it as one string field through the chosen form path. See [serialization paths](request-encryption.md#en).

The field-level envelope for `gpia`, `_gg`, and `_gi` is:

```text
wrapping_public32 = public_key_derived_by_current_native_connection_key_read
K = SHA256(UTF8(padded_standard_Base64(wrapping_public32)))
field = standard_Base64(IV16 || AES256-CBC-PKCS7(K, IV16, JSON_UTF8))
```

Each field has its own IV. The native connection-key reader derives `wrapping_public32` from private32, whereas request `authkey` and the Play `requestHash` read stored public32. These can match for a valid pair with no intervening rotation, but their sources, timing, and representations differ. Do not assume equivalence for corrupted records or concurrent rotation. The Play `requestHash` is unpadded standard Base64 of that stored public key, without the SHA-256 step above.

`_ge` and the outer `_ga` remain JSON. `_ga.bi` hashes a successfully completed boot-identifier stream, preserving its original newline, into SHA-256/standard Base64, then applies separate CBC. Read failure instead hashes ASCII `failed_sha`, not a partial prefix or a successful empty stream. Its public material is a snapshot taken when its security owner was created, not a fresh read of the current connection public key. A cached complete `_ga` also does not refresh merely because the key rotates.

`_gs.em` uses an independent ephemeral X25519 key. Its 32-byte shared secret directly supplies the AES256-GCM key, with a zero12 nonce, empty AAD, and tag16. The envelope is unpadded Base64url(`ephemeral_public32 || ciphertext || tag16`). Its prepared state belongs to the execution-security component, not the Google provider.

The transport layer subsequently serializes the inner parameters and applies its outer envelope. `ENC`, `H`, and HTTP headers are outside the 70 inner parameters in this article. The selected transport path and configuration determine the signature algorithm and actual headers; one channel's header set does not define every code request.

#### Outer Packaging and Sending Branches

Finish transport merging and required refresh before serializing actual form bytes; neither the field catalog nor display JSON is the final HTTP body. Normal encrypted `ENC` uses fresh ephemeral X25519 shared32 directly as the AES256-GCM key, with zero nonce12, empty AAD, tag16, and no extra KDF. Encode raw public32 followed by ciphertext/tag as unpadded Base64url. When attestation is enabled, `H` signs ENC text; `Authorization` is padded standard Base64 of the matching signer's reversed concatenated DER chain, without a Bearer prefix.

The Kotlin builder's encryption, endpoint-attestation, and signer-internal gates are distinct. Successful encryption can yield ENC-only or ENC/H; the latter can retain empty `H=` when signing is unavailable. Unencrypted/encryption-fallback branches use the original query and sign that query if appending H. Neither these fallbacks nor WAMsys's six application headers define all requests. Signing and certificate reads also lack an atomic snapshot. See [packaging branches](request-encryption.md#en) and [sending](sending-requests.md#en) for their conditions.

Kotlin network retry selects at most 4 attempt entries including initial ORIGINAL. A true parsed-success flag stops immediately, including HTTP 200 with business fail. Retry reuses the parameter builder and preencoded set and can regenerate ENC/H without rerunning upstream UI, Google, or business counters. This does not define WAMsys retry policy. Assess transport, JSON parsing, and business outcome separately.

<a id="plaintext-example-en"></a>

### Full Plaintext Example

The following inventory shows all 70 parameters in table order. It combines conditional branches for lookup, including SMS and Autoconf fields that do not belong together in a normal request. Select only the applicable fields; do not send the entire inventory. Every value is a string, and the object is not the final HTTP body.

```json
{
  "cc": "1",
  "in": "2025550147",
  "lg": "en",
  "lc": "US",
  "method": "sms",
  "rc": "0",
  "fdid": "ce9ecf75-f380-4676-aa7f-a91c421c4466",
  "expid": "YucHBVW5kh37FAN49P82LQ",
  "access_session_id": "hzw8ZMqxkqN-07b-V3NJdg",
  "aid": "IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY=",
  "id": "%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF",
  "backup_token": "%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C",
  "hasinrc": "1",
  "authkey": "Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8",
  "e_ident": "oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8",
  "e_keytype": "BQ",
  "e_regid": "AAAwOQ",
  "e_skey_id": "AAFB",
  "e_skey_val": "HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ",
  "e_skey_sig": "EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUVBNQiAS2KyvwU1nah1PFSjB4yMm04B-qjHygUlWToo4Lg",
  "token": "4s+/5KpaPw6zhEmWAm+hHBGzUqQ=",
  "gpia": "YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgjHFieOwStufC1uiKjjsxyNToHILBB3K08sPOY2axX0v7IBwAUuswhuJjMcgL1C3IKDjDuSO6S9MAz6xVLkCY5Hkml6PwMFNfBH76S6worM8dutJRyqxYwZUR//pYsqc52guScZVilvVUlBB9SPYPW1JObkw8M0zuGBrYvpEIBOarw==",
  "_gg": "s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X",
  "_gi": "F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnnwGfuSLgigcDdTWFOLQWiKDHL1zM0EXK57kskKL022A3l00A9IZ0VQVszK2zZgY3fKc6CVCaXJ5IMo8LT2XjiwwPafZ9DW3adcfw8jXUZ/sofnkgLhuOPgrv9bb4qGZ8vxs2AggITRaaiITPLroYXB0lxD1sf9gbFc3kiQTvjmAMQ==",
  "_gp": "kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE=",
  "_gs": "{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_NhtinJDP13Vch4tShkkqXHtHcFu8WHdSYkx9bojyK-LYjHe2V8pcXyauRVsTcueJnUSX3n1Tbt1Fgsl6J7zIDJfpUFhyM2CiPVhk7cE3Pd4ESLSYOEomGk0VEQKBwgY7uv4SDPDUsPr-7qjfJ3z_iCaDdGy7jTcafh7e3X2cmGswg90l71XdabOvGmXXTPJzSb2wjKFZTI\"}",
  "_ge": "{\"sb\":false,\"sv\":false}",
  "_ga": "{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2AMI3JZGLVB66SVOydUJd5ONOWnXZqCdGj1bkP3IzGBjh9NMWzUtQ==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}",
  "mcc": "310",
  "mnc": "260",
  "sim_mcc": "310",
  "sim_mnc": "260",
  "network_radio_type": "1",
  "simnum": "0",
  "pid": "23456",
  "device_ram": "6",
  "db": "0",
  "sim_type": "0",
  "airplane_mode_type": "0",
  "cellular_strength": "5",
  "roaming_type": "0",
  "mistyped": "7",
  "reason": "",
  "hasav": "1",
  "client_metrics": "{\"attempts\":2,\"app_campaign_download_source\":\"unknown|unknown\"}",
  "education_screen_displayed": "false",
  "prefer_sms_over_flash": "false",
  "feo2_query_status": "did_not_query",
  "advertising_id": "98ea2934-49f4-4245-ae15-339d0d3c6145",
  "push_code": "example-push-code",
  "context": "invite_registration",
  "client_start_message": "SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA",
  "clicked_education_link": "false",
  "manage_call_permission": "true",
  "call_log_permission": "false",
  "sim_status": "1",
  "old_phone_number": "12025550186",
  "first_api_level": "28",
  "recaptcha": "{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}",
  "fid": "9b55089e-f213-4d94-985d-3759a62fe082|7318042560",
  "preloads_app_manager_id": "example-manager-id",
  "preloads_attribution": "example-attribution",
  "tos_version": "5",
  "entrypoint": "suma",
  "cred_token": "example-number-credential",
  "e_pq_last_resort_id": "t1gawg",
  "e_pq_last_resort_val": "_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di",
  "e_pq_last_resort_sig": "oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMnnj980ZhbmuqIpDQDRSX_UoFR3E95MpIw3mVuAhhN-ddw",
  "login": "example-login-material",
  "type": "0"
}
```

`id` and `backup_token` show raw bytes already percent-encoded; do not encode those display strings a second time. Other values are shown before form encoding. JSON-valued fields contain compact JSON strings. The PQ triplet illustrates encoding only: use the actual algorithm-defined sizes and matching record. Random envelope-shaped data does not decrypt into meaningful payloads.

<a id="mock-fields-java-en"></a>

#### Java Mock Builder

This Android Java class creates exactly the same 70-field display object. Its fixed synthetic values make the example repeatable; it does not implement the source readers, derive security fields, or produce an authenticated request.

```java
// File: CodeMockFields.java
package wiki.examples;

import android.util.Log;
import org.json.JSONException;
import org.json.JSONObject;

public final class CodeMockFields {
    private CodeMockFields() {}

    // Synthetic display data, not credentials or a sendable request.
    public static JSONObject create() throws JSONException {
        JSONObject fields = new JSONObject();
        fields.put("cc", "1");
        fields.put("in", "2025550147");
        fields.put("lg", "en");
        fields.put("lc", "US");
        fields.put("method", "sms");
        fields.put("rc", "0");
        fields.put("fdid", "ce9ecf75-f380-4676-aa7f-a91c421c4466");
        fields.put("expid", "YucHBVW5kh37FAN49P82LQ");
        fields.put("access_session_id", "hzw8ZMqxkqN-07b-V3NJdg");
        fields.put("aid", "IaN4m3AC8sdGOkkoOg7blXISWuqXsir6w8ppt7i0leY=");
        fields.put("id", "%BDp%07%B8%B7%0F%26q%A8%A9%5E%EA%82%FB%07%3D%9Fh%F1%CF");
        fields.put("backup_token", "%E6%C5%98%D5%A4%8A%D3V%85%05Ov.%8E%B6%F5%8EQ%07%2C");
        fields.put("hasinrc", "1");
        fields.put("authkey", "Jmq0Sbho4PVnySaBd_0GmkrYx0Mkun6bHAV_9ymthO8");
        fields.put("e_ident", "oh3x9fwwu4D2a4D3jzVWx1PqEKXFE9aeRVtEfVdC1Y8");
        fields.put("e_keytype", "BQ");
        fields.put("e_regid", "AAAwOQ");
        fields.put("e_skey_id", "AAFB");
        fields.put("e_skey_val", "HSwORv1tWoqDyY3l6ezZfl-lDwwRTl2k9yEXCFsjMFQ");
        fields.put("e_skey_sig", "EEMzGyX7pLRXpoenlT9kg11nNn884-3ptftJ6HQJUVBNQiAS2KyvwU1nah1PFSjB4yMm04B-qjHygUlWToo4Lg");
        fields.put("token", "4s+/5KpaPw6zhEmWAm+hHBGzUqQ=");
        fields.put("gpia", "YlLDG4LWtPKKuSLziGRUtYqqZjdoQNQNu/K9SVyKgjHFieOwStufC1uiKjjsxyNToHILBB3K08sPOY2axX0v7IBwAUuswhuJjMcgL1C3IKDjDuSO6S9MAz6xVLkCY5Hkml6PwMFNfBH76S6worM8dutJRyqxYwZUR//pYsqc52guScZVilvVUlBB9SPYPW1JObkw8M0zuGBrYvpEIBOarw==");
        fields.put("_gg", "s7xQcnisngoe2uZJcPM0Eq3IssRafi/Tkh/d4InX55MYfvP7I+Ho2UKliSEtgt+X");
        fields.put("_gi", "F1vfwlNQT+AAhL/LyUpjmv1ll1WDM1xxJsQGBPsHnnwGfuSLgigcDdTWFOLQWiKDHL1zM0EXK57kskKL022A3l00A9IZ0VQVszK2zZgY3fKc6CVCaXJ5IMo8LT2XjiwwPafZ9DW3adcfw8jXUZ/sofnkgLhuOPgrv9bb4qGZ8vxs2AggITRaaiITPLroYXB0lxD1sf9gbFc3kiQTvjmAMQ==");
        fields.put("_gp", "kKZNtVvwXDHjArFyi3JTim5BGa9MYPom3cPRlqX4WNE=");
        fields.put("_gs", "{\"em\":\"I_2P-O17XvKp8IrFmJZgSYKmdbhU5Yal_NhtinJDP13Vch4tShkkqXHtHcFu8WHdSYkx9bojyK-LYjHe2V8pcXyauRVsTcueJnUSX3n1Tbt1Fgsl6J7zIDJfpUFhyM2CiPVhk7cE3Pd4ESLSYOEomGk0VEQKBwgY7uv4SDPDUsPr-7qjfJ3z_iCaDdGy7jTcafh7e3X2cmGswg90l71XdabOvGmXXTPJzSb2wjKFZTI\"}");
        fields.put("_ge", "{\"sb\":false,\"sv\":false}");
        fields.put("_ga", "{\"bi\":\"fi+4pcB0dxsRBDNaNnt0DXVuG1aUrbdPi2AMI3JZGLVB66SVOydUJd5ONOWnXZqCdGj1bkP3IzGBjh9NMWzUtQ==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}");
        fields.put("mcc", "310");
        fields.put("mnc", "260");
        fields.put("sim_mcc", "310");
        fields.put("sim_mnc", "260");
        fields.put("network_radio_type", "1");
        fields.put("simnum", "0");
        fields.put("pid", "23456");
        fields.put("device_ram", "6");
        fields.put("db", "0");
        fields.put("sim_type", "0");
        fields.put("airplane_mode_type", "0");
        fields.put("cellular_strength", "5");
        fields.put("roaming_type", "0");
        fields.put("mistyped", "7");
        fields.put("reason", "");
        fields.put("hasav", "1");
        fields.put("client_metrics", "{\"attempts\":2,\"app_campaign_download_source\":\"unknown|unknown\"}");
        fields.put("education_screen_displayed", "false");
        fields.put("prefer_sms_over_flash", "false");
        fields.put("feo2_query_status", "did_not_query");
        fields.put("advertising_id", "98ea2934-49f4-4245-ae15-339d0d3c6145");
        fields.put("push_code", "example-push-code");
        fields.put("context", "invite_registration");
        fields.put("client_start_message", "SnHtt77YhRvnwx8nUk44xntP5aH_-9ueUxJ3wpY4yaA");
        fields.put("clicked_education_link", "false");
        fields.put("manage_call_permission", "true");
        fields.put("call_log_permission", "false");
        fields.put("sim_status", "1");
        fields.put("old_phone_number", "12025550186");
        fields.put("first_api_level", "28");
        fields.put("recaptcha", "{\"token\":\"example-recaptcha-token\",\"token_length\":23,\"token_age\":0,\"stage\":\"code\"}");
        fields.put("fid", "9b55089e-f213-4d94-985d-3759a62fe082|7318042560");
        fields.put("preloads_app_manager_id", "example-manager-id");
        fields.put("preloads_attribution", "example-attribution");
        fields.put("tos_version", "5");
        fields.put("entrypoint", "suma");
        fields.put("cred_token", "example-number-credential");
        fields.put("e_pq_last_resort_id", "t1gawg");
        fields.put("e_pq_last_resort_val", "_mWwMw_zUuIVZZrDrK5v2RWjIsr9q_JtbbUMWNuCv_RC3_Lsh0SbvV9WxAvpq1di");
        fields.put("e_pq_last_resort_sig", "oHv34Qd1wbRGKBD0A4cGgf1utYCqV7MtZkGlncWIMnnj980ZhbmuqIpDQDRSX_UoFR3E95MpIw3mVuAhhN-ddw");
        fields.put("login", "example-login-material");
        fields.put("type", "0");
        return fields;
    }

    public static void printExample() throws JSONException {
        for (String line : create().toString(2).split("\n")) {
            Log.d("CodeMock", line);
        }
    }
}
```

Call `CodeMockFields.create()` for a `JSONObject`, or `CodeMockFields.printExample()` for line-by-line Logcat output. The caller handles `JSONException`. No phone-state reads, storage writes, or HTTP operations occur.

### Related Reading

- [Registration directory](README.md#en): the request sequence and article map.
- [exist parameters](exist.md#en): number checks and their own source rules.
- [Android input examples](android-inputs.md#en): system APIs, persistent state, and byte encodings.
- [security-fields/](security-fields/README.md#en): detailed structures and algorithms for the seven security fields.
- [request-encryption.md](request-encryption.md#en): serialization and outer envelopes.
- [sending-requests.md](sending-requests.md#en): transmission, headers, and response handling.

[Back to registration](README.md#en)
