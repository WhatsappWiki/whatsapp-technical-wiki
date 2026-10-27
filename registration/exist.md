# exist · 明文参数与取值来源

**Telegram Contact: [@ttjjda](https://t.me/ttjjda)**

[Registration / 注册目录](README.md) · [中文](#zh) · [English](#en) · [完整示意 / Full Example](#plaintext-example) · [Mock Java](#mock-fields-java) · [Android Code](android-inputs.md)

<a id="zh"></a>

## 中文

`exist` 用于注册前的号码检查，携带号码、客户端身份、设备环境和流程状态；不提交验证码。

本文列出 71 个已知内层参数，按功能分组；实际请求只包含当前流程所需字段，并非固定发送 71 项。Android API 获取调用应用自己的数据，持久标识、密钥和服务令牌由对应模块提供。[Android 获取代码](android-inputs.md)

示例保留普通状态值；标识、号码、令牌和密码学载荷使用同格式随机值。固定宽度字段保持字节长度，变长字段的示例不定义协议长度。长值在表中节选，完整内容见下方明文示意。随机示例不构成有效令牌、签名或完整密钥记录。

表格和明文示意中的字段值均按字符串表示；`_ge`、`_ga` 等 JSON 型字段的字符串内容是 JSON。这里的“明文”指外层 ENC 加密之前，字段内部仍可能经过编码或加密。二进制恢复材料的显示约定见[明文示意](#plaintext-example)。

### 1. 号码、语言与流程入口

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `cc` | 待检查号码的电话国家码，不是 ISO 地区代码。 | 用户选择的国家/地区对应拨号码；与 in 一起捕获。 | 数字字符串，不带 +；不由 SIM 国家自动替代。 | `"86"` |
| `in` | 国家码之外的本地号码，不是验证码。 | 号码输入框或用户确认的号码状态。 | 保留字符串；换号后更新依赖该号码的派生材料。 | `"14933330256"` |
| `lg` | 应用当前界面的语言。 | 应用 Context 的 Resources.Configuration；见 [语言与地区](android-inputs.md#app-locale)。 | 通常为小写语言代码；不满足语言格式时使用 zz。 | `"zh"` |
| `lc` | 应用语言环境的地区部分。 | 与应用语言环境对应的 Locale；见 [语言与地区](android-inputs.md#app-locale)。 | 两位大写字母或三位数字；无有效地区时为 ZZ，不等于 cc。 | `"CN"` |
| `rc` | 应用发布渠道编号。 | 应用的发布渠道配置。 | 十进制文本；正式发布渠道可使用 0，不由 SDK_INT 推导。 | `"0"` |
| `entrypoint` | 本次注册或添加账号的入口类别。 | 当前账号入口、添加账号与存量账号状态。 | 条件字段；例如 create_paa、suma；普通入口可以省略。 | `"suma"` |

### 2. 设备与访问会话标识

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `fdid` | 持久保存的客户端 phone-id。 | 应用自己的标识存储；首次建立后按其生命周期复用；见 [持久标识读取](android-inputs.md#persistent-identifiers)。 | 带连字符的 UUID 文本。 | `"5df7a98e-0a78-4c73-9a59-6bbda564a423"` |
| `expid` | 性能/安装设备标识，不应仅凭名称理解成实验编号。 | 应用保存的性能设备 UUID；缺失时以 Android ID 的 UTF-8 字节生成 name-based UUID，ID 为 null 或全空白时使用随机 UUID 并保存。 [性能 UUID 读取](android-inputs.md#performance-identifier) | UUID 的 16 字节大端表示，再做无 padding 的 Base64url；见 [UUID 编码](android-inputs.md#uuid-encoding)。 | `"_TFQzU9PO86vUedM9qsUGg"` |
| `access_session_id` | 关联一段访问会话中的请求。 | 会话管理器当前保存的 UUID；会话重置时更新。 [访问会话读取](android-inputs.md#access-session) | UUID 的 16 字节大端表示，再做无 padding 的 Base64url；不要每次序列化都重建。 | `"MhTn3z5-TNyq7hm6aFuSzQ"` |
| `advertising_id` | 广告服务提供的可重置标识。 | Google Play services 的 AdvertisingIdClient；见 [广告标识接入](android-inputs.md#advertising-id)。 | 字符串；可用性、追踪限制和流程配置决定是否携带，不能用随机 UUID 顶替。 | `"00cc4a0e-77ee-4333-b0c9-3667fe33dd77"` |
| `aid` | Android ID 的摘要表示。 | Settings.Secure.ANDROID_ID，再进行 SHA-256；见 [Android ID 读取](android-inputs.md#android-id)、[aid 摘要编码](android-inputs.md#aid-encoding)。 | 读取成功且 ID 不为 null 时，使用 UTF-8 输入、SHA-256、标准 Base64。读取失败走独立错误分支，不把 null 当空串。 | `"XyeDoz1twFMRk4lDPD67kynrPOlrba4VHNDM1XDsgbI="` |

### 3. 恢复与重装材料

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `id` | 本地恢复材料，不是账号 ID 或 Android ID。 | 优先解析并解密应用私有 files/rc2 恢复记录；恢复读取流程返回 null 时，才创建新材料并尝试保存。 | 二进制材料；按字节做表单百分号编码，不统一转 Base64。解析或解密异常按对应错误分支处理，不统一当作缺失。 | `"%B1%9Fw%05%03%88%A9%97T%8C%40%C1%D0%2F%F64%A6%21%FD%B6"` |
| `backup_token` | 备份/重装流程的辅助材料。 | 本地备份、迁移状态或该流程生成的材料，与号码选择规则关联。 | 二进制材料；按字节做表单百分号编码。 | `"%ED%F5%94%8AT%97B%D8%ABO%D1a%FA%85%F8%A8%EA%BFr%3E"` |
| `backup_token_source` | 本次 backup_token 选择的来源类别。 | 与 backup_token 同一次选择结果。 | 十进制文本；1 为随机兜底，2 为已有材料。不可把材料和来源分开重读。 | `"2"` |
| `backup_token_error` | 备份令牌流程保存的处理状态或错误诊断。 | 备份获取流程的真实回调所保存的诊断。 | 例如 success、null_token 或异常文本；最多 256 个 UTF-16 单元。不是 exist 响应状态。 | `"success"` |
| `hasinrc` | 本地恢复记录是否存在。 | 在应用自己的 files 目录检查 rc2 文件；见 [恢复文件检查](android-inputs.md#recovery-record)。 | 1/0 文本；只表示文件存在，不表示解密成功或材料有效。 | `"1"` |

### 4. 连接与端到端加密公开身份

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `authkey` | 连接认证使用的 Curve25519 公钥。 | 应用连接密钥存储中的当前公钥。 | raw 32 字节公钥 → 无 padding 的 Base64url。不能与 e_ident 或请求签名密钥互换。 | `"4yaaizFmaV08e-VH5faVV5wCF-PoUalZ0O6yRZX9VyM"` |
| `e_keytype` | 端到端公钥类型标签。 | 协议的密钥类型配置。 | 单字节 0x05 → 无 padding 的 Base64url，文本为 BQ；不是十进制字符串 5。 | `"BQ"` |
| `e_ident` | Signal 身份公钥。 | 应用现有 Signal 身份记录。 | 移除存储类型前缀后的 raw 32 字节 → 无 padding 的 Base64url。不要为一次请求新建身份。 | `"Phq_oqMtBwkFmmq86GHQeSeStOZ9q6exfcVD6REAqMo"` |
| `e_regid` | Signal 本地注册编号，与手机号码不同。 | 与当前身份对应的 Signal 注册状态。 | 4 字节大端整数 → 无 padding 的 Base64url；见 [编号编码](android-inputs.md#signal-id-encoding)。 | `"AAAdLQ"` |
| `e_skey_id` | 当前签名预密钥的编号。 | 选中的 signed-prekey 记录。 | 3 字节大端整数 → 无 padding 的 Base64url；与后两项取同一条记录。 | `"KerV"` |
| `e_skey_val` | 该签名预密钥的公钥。 | 与 e_skey_id 同一条记录中的公钥。 | 移除类型前缀后的 raw 32 字节 → 无 padding 的 Base64url。 | `"CMCBNzF6M6QpNfObtJHTzh6Z9PysGCX5YOzOncgK570"` |
| `e_skey_sig` | 身份密钥对该预密钥的签名。 | 与 e_skey_id/e_skey_val 同一条记录保存的签名。 | 签名字节 → 无 padding 的 Base64url；不能用请求签名 H 替代。 | `"vQCjaftbuFsIGt7cXx_nzSc3wq7nGMj79prnxbK6eerqqnBpT66_OHqwHGnq-TGEs4NRdgmT9oYI1FTzCyl55A"` |

### 5. 客户端校验与安全字段

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `token` | 绑定客户端应用材料与待检查号码的校验值。 | 应用包名、应用资源、签名证书、DEX 内容及 national 号码参与派生。 | 标准 Base64；不是 Google token、推送 token 或 backup_token。Android SDK 不直接提供成品。 | `"H05HI8Fb1Yo1aFldguetNlIafyg="` |
| `gpia` | Google Play 完整性结果与应用完整性信息的组合。 | 真实 Play Integrity 异步结果，加上应用信息；见 [Play Integrity 接入](android-inputs.md#play-integrity)。 | 内部 JSON 经字段级加密和 Base64 包装；可能是 token，也可能是错误/超时结果。 | `"pTAApqGZasLEwfdRwW2HOypCqTvgGFSB9FbdAR6HZqhj…nr/s9cG0nij7ENLhy"`（示例节选） |
| `_gg` | Google Play 完整性结果的精简包装。 | 与 gpia 共用同一次调用的结果码和可选 token。 | 内部 JSON 经字段级加密和 Base64 包装；不要为两个字段分别拼接不同调用结果。 | `"mYVMzuhLj8lb5pky1X0EMj/4CwfIoZltinkTWYaouqrW…y+OgTTqA6BxXgAeGz"`（示例节选） |
| `_gi` | 应用安装内容、签名和目录完整性信息。 | 应用安装路径、包/DEX 字节流、签名证书及安装环境；原始 API 见 [安装与签名输入](android-inputs.md#package-material)。 | 复合结构经字段级加密和 Base64 包装。 | `"eHnqtzWQgW9e2IcFRCxh4H3C3dfC7ZvU5RvHbHtWTDVw…CZjT9GJmkEWByrQ=="`（示例节选） |
| `_gp` | 应用声明权限集合的摘要。 | PackageInfo.requestedPermissions；见 [声明权限读取](android-inputs.md#declared-permissions)。 | 按权限名 UTF-8 字节排序，无分隔符拼接，SHA-256 后标准 Base64；不同于运行时授权集合。 | `"q19gmaQaITVtyoBCdLsD7CiZKryGA9zltE8yTd30NdI="` |
| `_gs` | 系统路径检查、执行状态和随机/时间输入组成的安全信号。 | 文件系统 stat 结果、时间、随机数与安全组件的准备状态。 | 外层 JSON 含 em；em 内为临时公钥及 AES-GCM 密文与 tag 的 Base64url 封装。不是单个布尔值。 | `{"em":"f8_clCHf9eCJ5CRfvd_bTug_ZagXp2NjeF0Dht…VIvwpJI2Z-YC8GIo"}`（示例节选） |
| `_ge` | 虚拟化相关模块及条件性的 su 路径检测信息。 | 进程实际可见的模块和路径检查结果。 | JSON 文本；模块/路径检测与检查失败需要区分，不把它整体当作 CBC 密文。 | `{"sb":false,"sv":false}` |
| `_ga` | 目录/安装年龄、自动化测试状态及启动标识等环境信息。 | 实际目录元数据、ActivityManager.isUserAMonkey() 和可读取的启动标识等。 | 外层 JSON；ap/ai/ae 为秒数，mp/mu 为布尔值，bi 有自己的加密包装。文件权限失败不能当作正常空内容。 | `{"mp":false,"mu":false}`（示例节选） |

### 6. 设备、权限和网络状态

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `device_name` | 设备构建代号，不是用户昵称或机型营销名称。 | Build.DEVICE；见 [设备代号读取](android-inputs.md#device-name)。 | 字符串；不要替换成 Build.MODEL。 | `"coral"` |
| `device_ram` | 设备总内存容量。 | ActivityManager.MemoryInfo.totalMem；见 [总内存读取](android-inputs.md#device-ram)。 | 字节除以 1024³，格式化为最多两位小数的 GiB 文本；不是 availMem。 | `"5.34"` |
| `pid` | 发送流程所在应用进程的 PID。 | android.os.Process.myPid()；见 [进程编号读取](android-inputs.md#process-id)。 | 十进制文本。 | `"10751"` |
| `db` | ADB 调试开关状态。 | Settings.Global.ADB_ENABLED；见 [ADB 设置读取](android-inputs.md#adb-setting)。 | 十进制文本；受字段采集配置控制，可以省略；不是 Debug.isDebuggerConnected()。 | `"1"` |
| `read_phone_permission_granted` | 该流程需要的电话权限是否全部满足。 | 检查 READ_PHONE_STATE，并在相应系统条件下检查 READ_PHONE_NUMBERS；见 [电话权限检查](android-inputs.md#phone-permissions)。 | 1/0 文本；声明权限不等于已经获得运行时授权。 | `"0"` |
| `simnum` | 能否取得达到长度门槛的本机号码信息。 | 当前订阅/电话服务返回的号码字符串；见 [本机号码读取](android-inputs.md#device-number)。 | 号码字符串非 null 且长度至少为 6 时为 1，否则为 0；不是 SIM 数量，也不证明号码归属。 | `"0"` |
| `sim_state` | SIM 的详细状态编号。 | TelephonyManager.getSimState()（默认卡）或 getSimState(slotIndex)（指定逻辑卡槽）；见 [SIM 状态读取](android-inputs.md#sim-state)。 | 十进制文本，如 absent/locked/ready 对应常量；与 sim_type 不同。 | `"1"` |
| `sim_operator_name` | SIM 归属运营商名称。 | TelephonyManager.getSimOperatorName()；见 [运营商读取](android-inputs.md#operator-names)。 | 字符串；多 SIM 时应明确订阅。空结果与读取异常是不同情况。 | `""` |
| `network_operator_name` | 当前电话网络运营商名称。 | TelephonyManager.getNetworkOperatorName()；见 [运营商读取](android-inputs.md#operator-names)。 | 字符串；漫游时可能与 SIM 运营商不同。 | `""` |
| `network_radio_type` | 应用对连接类型和蜂窝制式的归类。 | ConnectivityManager/NetworkCapabilities 与所选订阅的数据网络类型；见 [网络分类](android-inputs.md#network-type)。 | 协议枚举，不是直接填 NETWORK_TYPE 常量；例如 Wi-Fi/以太网类为 1，LTE 为 111，NR 为 113。 | `"1"` |
| `is_foa_fdid_app_installed` | 是否存在符合关联应用签名匹配规则的可见应用。 | PackageManager 查询与应用定义的证书匹配规则；原始证书 API 见 [应用签名输入](android-inputs.md#related-app-signatures)。 | true/false 文本；不是只看包名，受包可见性影响。 | `"false"` |

### 7. 推送与可选电话环境字段

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `push_token` | 推送服务发放给应用实例的令牌。 | 推送 SDK 回调并保存的当前令牌；FCM 入口见 [推送令牌接入](android-inputs.md#push-token)。 | 有值且流程需要时加入；不可用或空值可以省略，不使用 UUID 冒充。 | `"ZNMneWyXD336HaogDvqthjeHW6lvJfbH3pHuN7vnuszUbrQpmu1M1ECtfYr7ZIs1DP8eL-MROsjyzEPEOcGu9g"` |
| `sim_type` | 应用归类后的 SIM 可用状态。 | SIM 服务是否存在及 getSimState() 结果。 [SIM 原始状态](android-inputs.md#sim-state) | 条件字段；0 表示 absent，1 表示其它可读取 SIM 状态，2 表示服务不可用；不要与 sim_state 对换。 | `"0"` |
| `airplane_mode_type` | 飞行模式开关。 | Settings.Global.AIRPLANE_MODE_ON；见 [飞行模式读取](android-inputs.md#airplane-mode)。 | 条件字段；归一化为 0/1 文本。 | `"0"` |
| `cellular_strength` | 蜂窝信号强度等级。 | TelephonyManager.getSignalStrength().getLevel()；见 [信号等级读取](android-inputs.md#signal-strength)。 | 条件字段；0..4 为信号等级，5 为不可用兜底，不表示满格。 | `"5"` |
| `roaming_type` | 电话网络漫游状态分类。 | 网络是否存在、所选电话服务和 isNetworkRoaming() 结果。 [漫游原始状态](android-inputs.md#roaming-state) | 条件字段；0 为非漫游，1 为漫游，电话服务对象不存在时为 2；无网络快照时省略。读取异常不直接归为 0 或 2。 | `"0"` |

### 8. 流程统计、交互与诊断

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `client_metrics` | 注册流程统计与安装归因。 | 真实业务尝试计数、安装来源/归因状态、下载引导激活状态。 | 紧凑 JSON 字符串；attempts 在一次业务尝试产生时增加，重复序列化不应再次递增。 | `{"attempts":4,"was_activated_from_stub":false}`（示例节选） |
| `offline_ab` | 本地实验分组、曝光和可选 UI 统计。 | 应用实验分配与曝光存储，及本次流程的 UI 状态。 | 紧凑 JSON 字符串。 | `{"exposure":["xwztmb_bru_okc_uk_zs…v\|qmf_9"],"exp_hash":["860\|yzvwgr_uyg_dhw_n…bftogha"]}`（示例节选） |
| `feo2_query_status` | 关联服务能力查询的处理状态。 | 对应能力查询回调保存的状态。 | 例如 did_not_query、cached、success 或具体错误类别；不等于 client_capabilities 内容。 | `"did_not_query"` |
| `language_selector_clicked_count` | 语言选择流程中按交互规则累计的更改统计。 | 语言通知、提示面板关闭等实际 UI 事件与本地统计。 | 十进制文本；不等于总点击数，也不严格等于 Locale 实际变化次数。 | `"0"` |
| `language_selector_time_spent` | 语言选择面板累计打开时长。 | 面板打开/关闭事件记录的累计毫秒数。 | 毫秒数的十进制文本；与上述计数一起取得，在请求消费点统一处理。 | `"0"` |
| `mistyped` | 号码纠错交互和电话权限的综合状态。 | 当前号码、本机候选、是否展示/采用建议、交互模式及权限检查。 | 枚举字符串；7 可表示电话权限未满足。不是“输错次数”，也不是仅由号码字符串计算。 | `"7"` |

### 9. 特定流程的附加参数

| 字段 | 含义 | 取值来源 | 表示与出现条件 | 示例值 |
| --- | --- | --- | --- | --- |
| `foa_backup_token` | 关联应用备份路径提供的恢复材料。 | 对应账户/备份提供方按号码返回的材料。 | 条件二进制字段，按字节做表单百分号编码；不是普通 backup_token 的别名。 | `"%F3%FD%FC%F3s%AEFmj%14t%CE%0A%F3%DE%B8%D0%15n%3C"` |
| `client_capabilities` | 关联身份服务返回的能力数据。 | 能力服务的真实响应字节。 | 非 null 时使用无 padding 的 Base64url；与 feo2_query_status 分开。 | `"AQID"` |
| `login` | 特定同设备/二次验证流程使用的登录材料。 | 该流程持有的 two-factor login key。 | 条件字符串；不是无条件复制 exist 响应中同名字段。 | `"0-SEEbMgJz6xguYAzxzpseDm1h5V7rSs0C7N573bxps"` |
| `type` | 特定用户/注册类型。 | 调用该请求的业务类型参数。 | 十进制文本；类型分支可能影响 cc/in 是否出现，不能用一个固定模板覆盖所有类型。 | `"0"` |
| `context` | 特殊注册上下文。 | 邀请注册、解封、Web 注册、二次验证轮询等实际流程状态。 | 条件文本，例如 invite_registration、unban_registration、web_registration、poll_2fa。 | `"invite_registration"` |
| `tmobile_exp` | 特定运营环境的分类值。 | 应用的运营环境检测结果。 | 条件十进制文本；没有与 TelephonyManager 网络类型常量一一对应的通用映射。 | `"0"` |
| `sim_status` | 上游流程提供的 SIM 辅助状态。 | SIM 检查流程传入的字符串。 | 条件文本；不是 sim_state 或 sim_type 的别名。 | `"1"` |
| `first_api_level` | 设备最初出厂的 Android API 级别。 | 设备的 ro.product.first_api_level 属性，通过设备提供方允许的接口读取。 | 条件正整数文本；不等于 Build.VERSION.SDK_INT。普通应用没有适用于全部设备的公开等价 getter。 | `"28"` |
| `old_phone_number` | 流程关联的旧号码。 | 已有账户状态中的旧电话国家码与旧本地号码，按此顺序直接拼接。 | 有旧账户状态时才加入；仅含数字，不带 +、空格或分隔符。使用旧国家码，不沿用当前 cc。 | `"8618056485253"` |
| `recaptcha` | reCAPTCHA 校验结果或其诊断包装。 | 真实 reCAPTCHA 服务回调，以及当前校验阶段。 | 可能是 token 字符串，也可能是含 token/token_length/token_age/error/stage 的 JSON；不总是裸 token。 | `"zaKSWs-lzA9ZKKKT9VK69pbeQZS8Nznz_-1FkrhTK1uc…tHYpKaHv2hdCh-mcr"`（示例节选） |
| `fid` | 特定标识提供方返回的辅助身份文本。 | 该身份提供方维护的值和组合规则。 | 条件字符串；不能仅凭缩写当作 Firebase Installation ID，也不能以 fdid 替代。 | `"286de756-0103-40f5-bcc0-90814fae25e8\|1510401559"` |
| `preloads_app_manager_id` | 预装渠道的应用管理标识。 | 设备预装/渠道管理服务。 | 服务有结果时加入；不是安装包名或系统序列号。 | `"795a8dcfcdb6f05267577d725efe3817"` |
| `preloads_attribution` | 预装渠道归因内容。 | 预装归因提供方返回的数据。 | 条件字符串；不是 client_metrics 安装来源字段的简单复制。 | `"5f273ecb4cb1f99a266b0d083762a76a24cfb37ff7a2fc5f"` |
| `tos_version` | 特定条款流程使用的条款编号。 | 当前条款流程的配置与已有同意状态。 | 条件十进制文本；例如特定条款分支使用 5，不由应用版本号推导。 | `"5"` |
| `cred_token` | 与号码关联的凭据。 | 凭据管理器按规约后的号码查找的有效材料。 | 条件字符串；没有 Android 系统 getter，也不以 Google token 或随机字符串顶替。 | `"tdIDs03R_b_aOnk8yxs75qsGVPA4ab3O2pDLsMm6nCpJzt5WNh3xK2MNaVjWpYmJ"` |
| `e_pq_last_resort_id` | 后量子 last-resort 密钥记录标识。 | 加密会话模块选中的同一条 PQ last-resort 记录。 | 原始字节 → 无 padding 的 Base64url；与 val/sig 三项成组。 | `"NHlsOg"` |
| `e_pq_last_resort_val` | 该 PQ last-resort 记录的公钥材料。 | 与 e_pq_last_resort_id 同一条记录。 | 原始字节 → 无 padding 的 Base64url；公钥字节长度由对应 PQ 算法和记录决定。 | `"2jffqARfLr6C1HEDDgF5mJaT7dwIX_1PHyVEru5rCdEN…8sf0HyQKM4Z_FFcfG"`（示例节选） |
| `e_pq_last_resort_sig` | 该 PQ last-resort 记录的签名材料。 | 与 id/val 配对保存的签名。 | 原始字节 → 无 padding 的 Base64url；三项完整时一起加入。 | `"L8RmFk-XvjIMzBBSPPwttNuzwdGZxOiNedZEpSWZP_QgGBKIzF75Dm26G5E1aj3cQX9bfOvek4vjqcGEk0VDdg"` |

三个 `e_pq_last_resort_*` 字段的随机示例仅展示 Base64url 编码形式；示例字节数不定义协议长度。实际值须读取同一条密钥记录，按对应算法规定的长度使用，不按示例长度生成、补齐或截断。

### 组装要点

- `cc/in` 使用同一份号码输入；号码变化时更新相关派生材料。
- `backup_token` / `backup_token_source`、三个 `e_skey_*`、`gpia/_gg` 分别使用同一次选择或调用结果。
- 流程统计在业务事件处累计或消费，序列化重试不重复计数。
- 四个可选电话环境字段由同组开关控制，推送令牌另按推送状态处理。
- 空值、缺键、0 和 false 不可互换；字段编码只做一次，保留各自 Base64 字母表与 padding 规则。

<a id="plaintext-example"></a>

### 明文示意

下例按字段表顺序展示全部 71 个参数，便于查阅，条件字段可能属于不同流程，不应整份照搬发送。所有值均为字符串；标识和载荷使用同格式随机示例，`_ga` 年龄数字为模拟值。JSON 不等于最终 HTTP 正文。

```json
{
  "cc": "86",
  "in": "14933330256",
  "lg": "zh",
  "lc": "CN",
  "rc": "0",
  "entrypoint": "suma",
  "fdid": "5df7a98e-0a78-4c73-9a59-6bbda564a423",
  "expid": "_TFQzU9PO86vUedM9qsUGg",
  "access_session_id": "MhTn3z5-TNyq7hm6aFuSzQ",
  "advertising_id": "00cc4a0e-77ee-4333-b0c9-3667fe33dd77",
  "aid": "XyeDoz1twFMRk4lDPD67kynrPOlrba4VHNDM1XDsgbI=",
  "id": "%B1%9Fw%05%03%88%A9%97T%8C%40%C1%D0%2F%F64%A6%21%FD%B6",
  "backup_token": "%ED%F5%94%8AT%97B%D8%ABO%D1a%FA%85%F8%A8%EA%BFr%3E",
  "backup_token_source": "2",
  "backup_token_error": "success",
  "hasinrc": "1",
  "authkey": "4yaaizFmaV08e-VH5faVV5wCF-PoUalZ0O6yRZX9VyM",
  "e_keytype": "BQ",
  "e_ident": "Phq_oqMtBwkFmmq86GHQeSeStOZ9q6exfcVD6REAqMo",
  "e_regid": "AAAdLQ",
  "e_skey_id": "KerV",
  "e_skey_val": "CMCBNzF6M6QpNfObtJHTzh6Z9PysGCX5YOzOncgK570",
  "e_skey_sig": "vQCjaftbuFsIGt7cXx_nzSc3wq7nGMj79prnxbK6eerqqnBpT66_OHqwHGnq-TGEs4NRdgmT9oYI1FTzCyl55A",
  "token": "H05HI8Fb1Yo1aFldguetNlIafyg=",
  "gpia": "pTAApqGZasLEwfdRwW2HOypCqTvgGFSB9FbdAR6HZqhj4kuw8hOx6eFIqNd29R9DYjF4r40AG7iGExzO16+Ix7HH/q0WUyOOViTsU4POH//l1bIV4wGj7jywfNgJMJVa7BIPyWMTkILFHfN57LP6b/frGHqk7O9bigHMfkOkxYq9On34sZTdwfOwZ9JX7Y6zWvfdTZGb8+WYQNtgVnlwsEM1/4yXweLTd8I9HDtyfHMzt3W6vr3fMInm+iR6QstBkUtdhaYDmtk5n9IwDS3zJNRuHzLqGPyBN0m0Bdxo1Wwuglb1DVP2cuAKnzcd+I5j892knETbW59oqTZui4JWPNZTY29UL2n1luGeYBWN4BEQsfEfWwd6BVcnmRCmMinPDCEmfi/aFn3HR9JwOHh0lNpPYzu/gzAB4mrsiCOll5aNyK4Y+Ned9WxBojJW8BZV1zl5crsv4b656TN7rTrBV6RH7VZSxKU/qwJ0h/3I6VveJGDOft29Pc0Sze9EcLDysgH1jA/lwMKs+pf/iy0o/m//r9BsQOe5g898Wj6wWw7xm23PQ9qfkBMm3WR5Dm1yaUI21caHbDrz831+GzZgcwZCOxpmtRbg69x2fvxLX3rMPFuCM6BMGM+NpVnhv9+MeGkbtYKxDO3JFYdp3LmLEz9nrtk5BJLMEZ1u1d/zT8ehcqctmoUT8d8Uc7kedvLoUDhfq+jIT6nM+TZaanbaEI4DPXLTc6QS/q8CquxLShdY87TMfnp7HZytlBRNu3x37jJoUi/YCFiPGNRMK8KZLvbINugIisXC+N4D1s7B9efPh+EVg82ylD4tLS4e8EnAI5+fp6paIcZOx3mYhodoJxwR3ECBjC81b1bEDt7+zpeAlhA26dTyw8WLVrkKZIunBoKg7z1gBACeSpiWo3jdQQ4c3/ue6eN+p42sYX81RXDaxQLQxIzRlQaXSGyZmcoA+FRRNVBPdIzfAtbMYfzKQ79QOuYP5WW62mAohCID6nTu0mNafwl5ko5iwKNoSJJmwdpBEcG0Y5FIEoChml+urBpxHH+f32caE0zRIlymePG5XoRN89RjqSKwsPNh5e3qPYw0QdVcaNrM0le1Y2hJtSplZPutwsvMzjTaeigNqYxw8/MO1kh1LmW3PbkkP8N/EQIaPUocVb0NRK/ub6VCnPZ1OVE+MbCai395wTWC2pKnxXRweEI891trg1gqr5Zh02IF1W0tqodAlDWt8cKr8s1d5ffSuTWem0lLU6sbxdoabOFv1+PfnX15XbV7Uq5hgfD8ENipTeib51aBYeGmp9LQ8787ASYDR7zEGDB1kdvnihAbT7Ccro9Kexx3BNunMoIYgkRitjE2x8nJAa4ZiGqKH2PNXV0YN9S+aJUDpLQwpBy+qRlIKv2bR2FagvFD/2tUi9BgK6tiCdUCmxPD0d4itAEDueC65qmF/e3JAoKuXJ1nr/s9cG0nij7ENLhy",
  "_gg": "mYVMzuhLj8lb5pky1X0EMj/4CwfIoZltinkTWYaouqrWNxEYUUpjYrWbqy+AND1feAdrLVCMAivg3BRu7bL5rNk6MUCuhW+s2JLfxaBVV1Pk6BhEz3csl12D7UpFlkxv7ThlDUhSzxcjxM5aCa87pjM/HDG2LXJuNxU7LSIoYhUISowQGJmD6a6ihFhigwNOtzCeAxBY2qjUP+0/N7mpvHcBa+MKAOPvjddqc3rd82Qj+O/d9Tv1/3hAZ+E9ZgsWzRMewuPdUUMOsslL74jP7Fy4PW6HO8mLY8rJN1dPdn52t+UystCGuWQXuQsuRnXr3hN0wBVhtGGxLFPhg7PArpWwQZE5liBNoEz9HKuEL5wd+5YiF9ZGSFPKAKMrzhU/ZR2gk0vx58yxPC4GYuCDWNcj/mm0oY8B/Lw+Ypp2OB5j7vMBtPAKtA4nHL4ZlloQWQeNPfYXZhiXtWrpEWQ7bkTVoPs2jfdq3Di+qwcOm70ykjGi48Kb0yPD9ONxmP54vhX/OQgTa5JYxQy1inc7fOL0qAX46E0FSMnTvh6zfCWYe5O0C8ERz95R54hN0l7FP2swpB36lvdagfv+tMDR+k0x+qaqTh5McEkCr4zhp4a/ggmd77j9rbJbbw09qX0D1rb5qlyvceLaVScKWF1BRMyBi8hmfzhRg3NOEXExKnfsJoAnBPbPPdp+wxntvGcjaXkiAhtC4+WX9rwxEnAf0uY8aG08m3Ku8HepFQ4dqmDZ9Vabbjf3EFCbyhaUCn/zcW+M68fFDFowCYcqgnUCvb9v3YTxVi3Yjoln9SPEPASd4CEIZK4cIhQSJJcYLgwjhNC6nKror8d1CwvNldfxP+F3k41JwEzs90lLSi+JombKpcVjs2XGk8vky/f15zq3I1uKQzgiM+8s4WKG0bHIL2QznHn/8CaymkVXxXMTGpAdpjOHQGla5zQD4Sad6jd5Gzl82E4LD/muENBte7baJm8FXX7ms+oIlU2LEzOkRp/pd6Y3ZY+nqCJkKi8hNgJcCDqAn24uR9fmZMHyVZHpoStxH2A24ybuQVMFubiPhsg7FIcy+OgTTqA6BxXgAeGz",
  "_gi": "eHnqtzWQgW9e2IcFRCxh4H3C3dfC7ZvU5RvHbHtWTDVwGfPEdU2h1IcVbI0ZQQdrbyhj1pn6y45qmGBJIbKkAOxLxcXjxuMA3fz7DpM3aA9WOtd+L/yipvT15Zw3uKcNEmnQlffArEtjnN6QB9961m82Y3ryVOPE9MzKFDjVRKChAaxlMksZeOzlsAwXpe6azB72gPwGtfdCDCW9nGNU6AVem3wqmXYs/KbTwfj48YP5xEOp6XIz8sjDp+qqivvTY4NE+SqIBM/hbCj3CmBSBuigimtBoRYgaWlZInPTPV1LiqzuDk8JnxcvEdBe0uFBgEPckaROw9n7FuJ11CJVl8JJtpJSHkAAoXrRAhtHxhhj3Cn33wGHiN4omr8nc3GOAa5aTY36/yeYkTznOe3aGDQOA91xg4jZ8zdYSM6uERyvPAmLoEI6vjbHoiPaMws4Uh4Q9GwBG0HyZWeFr6DBICUnH/7mhobTPqhW+6HP9JJTOZF5dioMXv2a9REnh71Mfp1ldsKIlYBNB6r5b/7ZvZuDpdt0xbO5goDkxn2tTMyJk2b98JdSdET9E4Gq3BBKxoKH2kTlD8gR6ksJFvyz/uZcqmKbrgqR8s8rXCSB+SX6Rdy9WrCdK0XIGkqPu0QF4y2KMH3CZjT9GJmkEWByrQ==",
  "_gp": "q19gmaQaITVtyoBCdLsD7CiZKryGA9zltE8yTd30NdI=",
  "_gs": "{\"em\":\"f8_clCHf9eCJ5CRfvd_bTug_ZagXp2NjeF0DhtLhn3_PAT4NSrz-STLaf3s4VEPC1-_zSDwgzDQNiTZRdU0X22VFTcva8Ad3UPzj3tbOcX4dfrYC3O-O5oNEYdT7cAqvWaWnpFWG-1qwE7maKpWp3gBZgDfEzT4NQhb3amiT30JZfY8Xo9TRPt_keA9D24J8IVw3oTTQNImP5uBVIvwpJI2Z-YC8GIo\"}",
  "_ge": "{\"sb\":false,\"sv\":false}",
  "_ga": "{\"bi\":\"+xz3ADFpfArSWkkdMEnSxQWznNbsipNDN28ymoTBikhQRHSpvkWcJS0yVsSid+kHo/43H7Ui4seU9df8ziUI6A==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}",
  "device_name": "coral",
  "device_ram": "5.34",
  "pid": "10751",
  "db": "1",
  "read_phone_permission_granted": "0",
  "simnum": "0",
  "sim_state": "1",
  "sim_operator_name": "",
  "network_operator_name": "",
  "network_radio_type": "1",
  "is_foa_fdid_app_installed": "false",
  "push_token": "ZNMneWyXD336HaogDvqthjeHW6lvJfbH3pHuN7vnuszUbrQpmu1M1ECtfYr7ZIs1DP8eL-MROsjyzEPEOcGu9g",
  "sim_type": "0",
  "airplane_mode_type": "0",
  "cellular_strength": "5",
  "roaming_type": "0",
  "client_metrics": "{\"attempts\":4,\"app_campaign_download_source\":\"svsemm-utuw|mhuyndi\",\"was_activated_from_stub\":false}",
  "offline_ab": "{\"exposure\":[\"xwztmb_bru_okc_uk_zsctjhpg_hspgzpe|irrtuz_ubj_yhm_ti_zugilvm_plv|qmf_9\"],\"exp_hash\":[\"860|yzvwgr_uyg_dhw_nv_mqbqflws_bftogha\"]}",
  "feo2_query_status": "did_not_query",
  "language_selector_clicked_count": "0",
  "language_selector_time_spent": "0",
  "mistyped": "7",
  "foa_backup_token": "%F3%FD%FC%F3s%AEFmj%14t%CE%0A%F3%DE%B8%D0%15n%3C",
  "client_capabilities": "AQID",
  "login": "0-SEEbMgJz6xguYAzxzpseDm1h5V7rSs0C7N573bxps",
  "type": "0",
  "context": "invite_registration",
  "tmobile_exp": "0",
  "sim_status": "1",
  "first_api_level": "28",
  "old_phone_number": "8618056485253",
  "recaptcha": "zaKSWs-lzA9ZKKKT9VK69pbeQZS8Nznz_-1FkrhTK1ucL08ygXmcX8M7k1ROHGFNlnpEG5kELAfYHVtOEbJyTwokEPoi4B0s47pUQIV_NthuNE1tHYpKaHv2hdCh-mcr",
  "fid": "286de756-0103-40f5-bcc0-90814fae25e8|1510401559",
  "preloads_app_manager_id": "795a8dcfcdb6f05267577d725efe3817",
  "preloads_attribution": "5f273ecb4cb1f99a266b0d083762a76a24cfb37ff7a2fc5f",
  "tos_version": "5",
  "cred_token": "tdIDs03R_b_aOnk8yxs75qsGVPA4ab3O2pDLsMm6nCpJzt5WNh3xK2MNaVjWpYmJ",
  "e_pq_last_resort_id": "NHlsOg",
  "e_pq_last_resort_val": "2jffqARfLr6C1HEDDgF5mJaT7dwIX_1PHyVEru5rCdENG14gbNlQvVRQw3w7fO6DIPl38OaJAog4VliJL3_bk2hDgOz2ERDWLPocv-mI-tWcWNg8sf0HyQKM4Z_FFcfG",
  "e_pq_last_resort_sig": "L8RmFk-XvjIMzBBSPPwttNuzwdGZxOiNedZEpSWZP_QgGBKIzF75Dm26G5E1aj3cQX9bfOvek4vjqcGEk0VDdg"
}
```

`id`、`backup_token`、`foa_backup_token` 的示例以百分号编码显示随机原始字节，不要再次编码。JSON 型字段仍是紧凑 JSON 字符串，`_gs` 保留 `em` 包装。

#### Mock Fields Java

下面的 Android Java 类构造与上方一致的 71 字段示例对象。随机示例值固定写入，便于重复查看和调试，不执行真实身份初始化、令牌获取或安全字段算法。

This Android Java class builds the same 71-field example object. The randomly generated example values are fixed literals for repeatable inspection. It performs no real identity initialization, token acquisition, or security-field computation.

```java
// File: ExistMockFields.java
package wiki.examples;

import android.util.Log;
import org.json.JSONException;
import org.json.JSONObject;

public final class ExistMockFields {
    private ExistMockFields() {}

    // Fixed random examples for local inspection, not valid credentials.
    public static JSONObject create() throws JSONException {
        JSONObject fields = new JSONObject();
        fields.put("cc", "86");
        fields.put("in", "14933330256");
        fields.put("lg", "zh");
        fields.put("lc", "CN");
        fields.put("rc", "0");
        fields.put("entrypoint", "suma");
        fields.put("fdid", "5df7a98e-0a78-4c73-9a59-6bbda564a423");
        fields.put("expid", "_TFQzU9PO86vUedM9qsUGg");
        fields.put("access_session_id", "MhTn3z5-TNyq7hm6aFuSzQ");
        fields.put("advertising_id", "00cc4a0e-77ee-4333-b0c9-3667fe33dd77");
        fields.put("aid", "XyeDoz1twFMRk4lDPD67kynrPOlrba4VHNDM1XDsgbI=");
        fields.put("id", "%B1%9Fw%05%03%88%A9%97T%8C%40%C1%D0%2F%F64%A6%21%FD%B6");
        fields.put("backup_token", "%ED%F5%94%8AT%97B%D8%ABO%D1a%FA%85%F8%A8%EA%BFr%3E");
        fields.put("backup_token_source", "2");
        fields.put("backup_token_error", "success");
        fields.put("hasinrc", "1");
        fields.put("authkey", "4yaaizFmaV08e-VH5faVV5wCF-PoUalZ0O6yRZX9VyM");
        fields.put("e_keytype", "BQ");
        fields.put("e_ident", "Phq_oqMtBwkFmmq86GHQeSeStOZ9q6exfcVD6REAqMo");
        fields.put("e_regid", "AAAdLQ");
        fields.put("e_skey_id", "KerV");
        fields.put("e_skey_val", "CMCBNzF6M6QpNfObtJHTzh6Z9PysGCX5YOzOncgK570");
        fields.put("e_skey_sig", "vQCjaftbuFsIGt7cXx_nzSc3wq7nGMj79prnxbK6eerqqnBpT66_OHqwHGnq-TGEs4NRdgmT9oYI1FTzCyl55A");
        fields.put("token", "H05HI8Fb1Yo1aFldguetNlIafyg=");
        fields.put("gpia", "pTAApqGZasLEwfdRwW2HOypCqTvgGFSB9FbdAR6HZqhj4kuw8hOx6eFIqNd29R9DYjF4r40AG7iGExzO16+Ix7HH/q0WUyOOViTsU4POH//l1bIV4wGj7jywfNgJMJVa7BIPyWMTkILFHfN57LP6b/frGHqk7O9bigHMfkOkxYq9On34sZTdwfOwZ9JX7Y6zWvfdTZGb8+WYQNtgVnlwsEM1/4yXweLTd8I9HDtyfHMzt3W6vr3fMInm+iR6QstBkUtdhaYDmtk5n9IwDS3zJNRuHzLqGPyBN0m0Bdxo1Wwuglb1DVP2cuAKnzcd+I5j892knETbW59oqTZui4JWPNZTY29UL2n1luGeYBWN4BEQsfEfWwd6BVcnmRCmMinPDCEmfi/aFn3HR9JwOHh0lNpPYzu/gzAB4mrsiCOll5aNyK4Y+Ned9WxBojJW8BZV1zl5crsv4b656TN7rTrBV6RH7VZSxKU/qwJ0h/3I6VveJGDOft29Pc0Sze9EcLDysgH1jA/lwMKs+pf/iy0o/m//r9BsQOe5g898Wj6wWw7xm23PQ9qfkBMm3WR5Dm1yaUI21caHbDrz831+GzZgcwZCOxpmtRbg69x2fvxLX3rMPFuCM6BMGM+NpVnhv9+MeGkbtYKxDO3JFYdp3LmLEz9nrtk5BJLMEZ1u1d/zT8ehcqctmoUT8d8Uc7kedvLoUDhfq+jIT6nM+TZaanbaEI4DPXLTc6QS/q8CquxLShdY87TMfnp7HZytlBRNu3x37jJoUi/YCFiPGNRMK8KZLvbINugIisXC+N4D1s7B9efPh+EVg82ylD4tLS4e8EnAI5+fp6paIcZOx3mYhodoJxwR3ECBjC81b1bEDt7+zpeAlhA26dTyw8WLVrkKZIunBoKg7z1gBACeSpiWo3jdQQ4c3/ue6eN+p42sYX81RXDaxQLQxIzRlQaXSGyZmcoA+FRRNVBPdIzfAtbMYfzKQ79QOuYP5WW62mAohCID6nTu0mNafwl5ko5iwKNoSJJmwdpBEcG0Y5FIEoChml+urBpxHH+f32caE0zRIlymePG5XoRN89RjqSKwsPNh5e3qPYw0QdVcaNrM0le1Y2hJtSplZPutwsvMzjTaeigNqYxw8/MO1kh1LmW3PbkkP8N/EQIaPUocVb0NRK/ub6VCnPZ1OVE+MbCai395wTWC2pKnxXRweEI891trg1gqr5Zh02IF1W0tqodAlDWt8cKr8s1d5ffSuTWem0lLU6sbxdoabOFv1+PfnX15XbV7Uq5hgfD8ENipTeib51aBYeGmp9LQ8787ASYDR7zEGDB1kdvnihAbT7Ccro9Kexx3BNunMoIYgkRitjE2x8nJAa4ZiGqKH2PNXV0YN9S+aJUDpLQwpBy+qRlIKv2bR2FagvFD/2tUi9BgK6tiCdUCmxPD0d4itAEDueC65qmF/e3JAoKuXJ1nr/s9cG0nij7ENLhy");
        fields.put("_gg", "mYVMzuhLj8lb5pky1X0EMj/4CwfIoZltinkTWYaouqrWNxEYUUpjYrWbqy+AND1feAdrLVCMAivg3BRu7bL5rNk6MUCuhW+s2JLfxaBVV1Pk6BhEz3csl12D7UpFlkxv7ThlDUhSzxcjxM5aCa87pjM/HDG2LXJuNxU7LSIoYhUISowQGJmD6a6ihFhigwNOtzCeAxBY2qjUP+0/N7mpvHcBa+MKAOPvjddqc3rd82Qj+O/d9Tv1/3hAZ+E9ZgsWzRMewuPdUUMOsslL74jP7Fy4PW6HO8mLY8rJN1dPdn52t+UystCGuWQXuQsuRnXr3hN0wBVhtGGxLFPhg7PArpWwQZE5liBNoEz9HKuEL5wd+5YiF9ZGSFPKAKMrzhU/ZR2gk0vx58yxPC4GYuCDWNcj/mm0oY8B/Lw+Ypp2OB5j7vMBtPAKtA4nHL4ZlloQWQeNPfYXZhiXtWrpEWQ7bkTVoPs2jfdq3Di+qwcOm70ykjGi48Kb0yPD9ONxmP54vhX/OQgTa5JYxQy1inc7fOL0qAX46E0FSMnTvh6zfCWYe5O0C8ERz95R54hN0l7FP2swpB36lvdagfv+tMDR+k0x+qaqTh5McEkCr4zhp4a/ggmd77j9rbJbbw09qX0D1rb5qlyvceLaVScKWF1BRMyBi8hmfzhRg3NOEXExKnfsJoAnBPbPPdp+wxntvGcjaXkiAhtC4+WX9rwxEnAf0uY8aG08m3Ku8HepFQ4dqmDZ9Vabbjf3EFCbyhaUCn/zcW+M68fFDFowCYcqgnUCvb9v3YTxVi3Yjoln9SPEPASd4CEIZK4cIhQSJJcYLgwjhNC6nKror8d1CwvNldfxP+F3k41JwEzs90lLSi+JombKpcVjs2XGk8vky/f15zq3I1uKQzgiM+8s4WKG0bHIL2QznHn/8CaymkVXxXMTGpAdpjOHQGla5zQD4Sad6jd5Gzl82E4LD/muENBte7baJm8FXX7ms+oIlU2LEzOkRp/pd6Y3ZY+nqCJkKi8hNgJcCDqAn24uR9fmZMHyVZHpoStxH2A24ybuQVMFubiPhsg7FIcy+OgTTqA6BxXgAeGz");
        fields.put("_gi", "eHnqtzWQgW9e2IcFRCxh4H3C3dfC7ZvU5RvHbHtWTDVwGfPEdU2h1IcVbI0ZQQdrbyhj1pn6y45qmGBJIbKkAOxLxcXjxuMA3fz7DpM3aA9WOtd+L/yipvT15Zw3uKcNEmnQlffArEtjnN6QB9961m82Y3ryVOPE9MzKFDjVRKChAaxlMksZeOzlsAwXpe6azB72gPwGtfdCDCW9nGNU6AVem3wqmXYs/KbTwfj48YP5xEOp6XIz8sjDp+qqivvTY4NE+SqIBM/hbCj3CmBSBuigimtBoRYgaWlZInPTPV1LiqzuDk8JnxcvEdBe0uFBgEPckaROw9n7FuJ11CJVl8JJtpJSHkAAoXrRAhtHxhhj3Cn33wGHiN4omr8nc3GOAa5aTY36/yeYkTznOe3aGDQOA91xg4jZ8zdYSM6uERyvPAmLoEI6vjbHoiPaMws4Uh4Q9GwBG0HyZWeFr6DBICUnH/7mhobTPqhW+6HP9JJTOZF5dioMXv2a9REnh71Mfp1ldsKIlYBNB6r5b/7ZvZuDpdt0xbO5goDkxn2tTMyJk2b98JdSdET9E4Gq3BBKxoKH2kTlD8gR6ksJFvyz/uZcqmKbrgqR8s8rXCSB+SX6Rdy9WrCdK0XIGkqPu0QF4y2KMH3CZjT9GJmkEWByrQ==");
        fields.put("_gp", "q19gmaQaITVtyoBCdLsD7CiZKryGA9zltE8yTd30NdI=");
        fields.put("_gs", "{\"em\":\"f8_clCHf9eCJ5CRfvd_bTug_ZagXp2NjeF0DhtLhn3_PAT4NSrz-STLaf3s4VEPC1-_zSDwgzDQNiTZRdU0X22VFTcva8Ad3UPzj3tbOcX4dfrYC3O-O5oNEYdT7cAqvWaWnpFWG-1qwE7maKpWp3gBZgDfEzT4NQhb3amiT30JZfY8Xo9TRPt_keA9D24J8IVw3oTTQNImP5uBVIvwpJI2Z-YC8GIo\"}");
        fields.put("_ge", "{\"sb\":false,\"sv\":false}");
        fields.put("_ga", "{\"bi\":\"+xz3ADFpfArSWkkdMEnSxQWznNbsipNDN28ymoTBikhQRHSpvkWcJS0yVsSid+kHo/43H7Ui4seU9df8ziUI6A==\",\"ap\":120,\"ai\":60,\"mp\":false,\"ae\":30,\"mu\":false}");
        fields.put("device_name", "coral");
        fields.put("device_ram", "5.34");
        fields.put("pid", "10751");
        fields.put("db", "1");
        fields.put("read_phone_permission_granted", "0");
        fields.put("simnum", "0");
        fields.put("sim_state", "1");
        fields.put("sim_operator_name", "");
        fields.put("network_operator_name", "");
        fields.put("network_radio_type", "1");
        fields.put("is_foa_fdid_app_installed", "false");
        fields.put("push_token", "ZNMneWyXD336HaogDvqthjeHW6lvJfbH3pHuN7vnuszUbrQpmu1M1ECtfYr7ZIs1DP8eL-MROsjyzEPEOcGu9g");
        fields.put("sim_type", "0");
        fields.put("airplane_mode_type", "0");
        fields.put("cellular_strength", "5");
        fields.put("roaming_type", "0");
        fields.put("client_metrics", "{\"attempts\":4,\"app_campaign_download_source\":\"svsemm-utuw|mhuyndi\",\"was_activated_from_stub\":false}");
        fields.put("offline_ab", "{\"exposure\":[\"xwztmb_bru_okc_uk_zsctjhpg_hspgzpe|irrtuz_ubj_yhm_ti_zugilvm_plv|qmf_9\"],\"exp_hash\":[\"860|yzvwgr_uyg_dhw_nv_mqbqflws_bftogha\"]}");
        fields.put("feo2_query_status", "did_not_query");
        fields.put("language_selector_clicked_count", "0");
        fields.put("language_selector_time_spent", "0");
        fields.put("mistyped", "7");
        fields.put("foa_backup_token", "%F3%FD%FC%F3s%AEFmj%14t%CE%0A%F3%DE%B8%D0%15n%3C");
        fields.put("client_capabilities", "AQID");
        fields.put("login", "0-SEEbMgJz6xguYAzxzpseDm1h5V7rSs0C7N573bxps");
        fields.put("type", "0");
        fields.put("context", "invite_registration");
        fields.put("tmobile_exp", "0");
        fields.put("sim_status", "1");
        fields.put("first_api_level", "28");
        fields.put("old_phone_number", "8618056485253");
        fields.put("recaptcha", "zaKSWs-lzA9ZKKKT9VK69pbeQZS8Nznz_-1FkrhTK1ucL08ygXmcX8M7k1ROHGFNlnpEG5kELAfYHVtOEbJyTwokEPoi4B0s47pUQIV_NthuNE1tHYpKaHv2hdCh-mcr");
        fields.put("fid", "286de756-0103-40f5-bcc0-90814fae25e8|1510401559");
        fields.put("preloads_app_manager_id", "795a8dcfcdb6f05267577d725efe3817");
        fields.put("preloads_attribution", "5f273ecb4cb1f99a266b0d083762a76a24cfb37ff7a2fc5f");
        fields.put("tos_version", "5");
        fields.put("cred_token", "tdIDs03R_b_aOnk8yxs75qsGVPA4ab3O2pDLsMm6nCpJzt5WNh3xK2MNaVjWpYmJ");
        fields.put("e_pq_last_resort_id", "NHlsOg");
        fields.put("e_pq_last_resort_val", "2jffqARfLr6C1HEDDgF5mJaT7dwIX_1PHyVEru5rCdENG14gbNlQvVRQw3w7fO6DIPl38OaJAog4VliJL3_bk2hDgOz2ERDWLPocv-mI-tWcWNg8sf0HyQKM4Z_FFcfG");
        fields.put("e_pq_last_resort_sig", "L8RmFk-XvjIMzBBSPPwttNuzwdGZxOiNedZEpSWZP_QgGBKIzF75Dm26G5E1aj3cQX9bfOvek4vjqcGEk0VDdg");
        return fields;
    }

    public static void printExample() throws JSONException {
        for (String line : create().toString(2).split("\n")) {
            Log.d("ExistMock", line);
        }
    }
}
```

在应用代码中调用 `ExistMockFields.create()` 获取 `JSONObject`，或调用 `ExistMockFields.printExample()` 逐行输出到 Logcat；调用方处理 `JSONException`。这个类不读取手机状态、不保存数据，也不发送请求。

Call `ExistMockFields.create()` for the `JSONObject`, or `printExample()` for line-by-line Logcat output, handling `JSONException` in the caller. The class reads no device state, persists nothing, and sends no requests.

<a id="en"></a>

## English

`exist` checks a number before registration. It carries the number, client identity, device environment, and flow state, not a verification code.

This reference groups 71 known inner parameters by function. An actual request includes only the fields required by its flow, not a fixed set of 71. Android APIs read the calling app's data; identifiers, keys, and service tokens come from their respective modules. [Android code](android-inputs.md)

Ordinary state values are retained; identifiers, numbers, tokens, and cryptographic payloads use random values in the same representation. Fixed-width fields retain their byte lengths; variable-length examples do not define protocol sizes. Tables abbreviate long values, while the full illustration preserves them. Random examples are not valid tokens, signatures, or complete key records.

Values are represented as strings; JSON-valued fields such as `_ge` and `_ga` contain JSON text inside those strings. “Plaintext” means before outer ENC encryption; individual fields can still be encoded or encrypted. See the [full example](#plaintext-example) for the binary recovery-material display convention.

### 1. Number, locale, and entry point

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `cc` | Calling code of the number being checked, distinct from an ISO region code. | The user-selected calling code, captured together with in. | Digit string without +; do not substitute the SIM country. | `"86"` |
| `in` | National number, excluding the calling code; not a verification code. | The number input field or user-confirmed number state. | Preserve the string; update number-dependent material after a change. | `"14933330256"` |
| `lg` | Current application language. | The app Context resources configuration; see [App locale](android-inputs.md#app-locale). | Usually a lowercase language code; zz is the invalid-language fallback. | `"zh"` |
| `lc` | Region component of the application locale. | The app locale; see [App locale](android-inputs.md#app-locale). | Two uppercase letters or three digits; ZZ for an invalid region; distinct from cc. | `"CN"` |
| `rc` | Application release-channel number. | Application release configuration. | Decimal text; 0 denotes the production channel, not an Android API level. | `"0"` |
| `entrypoint` | Entry category for registration or account addition. | Current entry flow and existing/inactive account state. | Conditional; examples include create_paa and suma; ordinary flows may omit it. | `"suma"` |

### 2. Device and access-session identifiers

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `fdid` | Persistent client phone-id. | Application-owned identifier storage; reuse according to its lifecycle; see [Persistent IDs](android-inputs.md#persistent-identifiers). | Hyphenated UUID text. | `"5df7a98e-0a78-4c73-9a59-6bbda564a423"` |
| `expid` | Performance/installation device identifier, not simply an experiment number. | The stored performance-device UUID; initialize from Android ID UTF-8 bytes using a name-based UUID, or persist a random UUID when the ID is null/blank. [Performance UUID](android-inputs.md#performance-identifier) | UUID as 16 big-endian bytes, then unpadded Base64url; see [UUID encoding](android-inputs.md#uuid-encoding). | `"_TFQzU9PO86vUedM9qsUGg"` |
| `access_session_id` | Correlates requests in an access session. | The current UUID retained by the session manager; changed at a session reset. [Access session](android-inputs.md#access-session) | 16 big-endian UUID bytes encoded as unpadded Base64url; do not regenerate during every serialization. | `"MhTn3z5-TNyq7hm6aFuSzQ"` |
| `advertising_id` | Resettable identifier supplied by the advertising service. | Google Play services AdvertisingIdClient; see [Advertising ID](android-inputs.md#advertising-id). | String; availability, tracking preference, and flow configuration govern inclusion; do not substitute a random UUID. | `"00cc4a0e-77ee-4333-b0c9-3667fe33dd77"` |
| `aid` | Digest representation of Android ID. | Settings.Secure.ANDROID_ID followed by SHA-256; see [Android ID](android-inputs.md#android-id) and [aid encoding](android-inputs.md#aid-encoding). | For a present ID: UTF-8, SHA-256, standard Base64. Read failures require separate handling; null is not an empty string. | `"XyeDoz1twFMRk4lDPD67kynrPOlrba4VHNDM1XDsgbI="` |

### 3. Recovery and reinstall material

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `id` | Local recovery material, distinct from an account ID or Android ID. | Parse and decrypt the application-private files/rc2 recovery record first; only when the recovery reader returns null does the module create new material and attempt to persist it. | Binary data percent-encoded at the form layer, not universally Base64. Parsing/decryption errors follow their specific error branches rather than all being treated as missing data. | `"%B1%9Fw%05%03%88%A9%97T%8C%40%C1%D0%2F%F64%A6%21%FD%B6"` |
| `backup_token` | Auxiliary material for backup/reinstall flows. | Local backup or migration state, or material created by that flow, selected with the relevant number. | Binary data percent-encoded at the form layer. | `"%ED%F5%94%8AT%97B%D8%ABO%D1a%FA%85%F8%A8%EA%BFr%3E"` |
| `backup_token_source` | Source category of the selected backup_token. | The same selection result that supplies backup_token. | Decimal text; 1 for random fallback, 2 for existing material. Capture both together. | `"2"` |
| `backup_token_error` | Stored processing status or diagnostic from the backup-token flow. | Diagnostics persisted by the actual backup-fetch callback. | Examples: success, null_token, or exception text; at most 256 UTF-16 units. Not an exist response status. | `"success"` |
| `hasinrc` | Whether the local recovery record exists. | Existence of rc2 under the application files directory; see [Recovery record](android-inputs.md#recovery-record). | 1/0 text; file presence does not prove successful decryption or valid material. | `"1"` |

### 4. Connection and end-to-end public identity

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `authkey` | Curve25519 public key used for connection authentication. | The current public key in the application connection-key store. | Raw 32-byte public key → unpadded Base64url; distinct from e_ident and the request-signing key. | `"4yaaizFmaV08e-VH5faVV5wCF-PoUalZ0O6yRZX9VyM"` |
| `e_keytype` | End-to-end public-key type tag. | Protocol key-type configuration. | Single byte 0x05 → unpadded Base64url, producing BQ; not decimal text 5. | `"BQ"` |
| `e_ident` | Signal identity public key. | The existing Signal identity record. | Raw 32 bytes without the storage type prefix → unpadded Base64url. Do not create a new identity for each request. | `"Phq_oqMtBwkFmmq86GHQeSeStOZ9q6exfcVD6REAqMo"` |
| `e_regid` | Local Signal registration identifier, distinct from the phone number. | Signal registration state associated with the current identity. | 4-byte big-endian integer → unpadded Base64url; see [ID encoding](android-inputs.md#signal-id-encoding). | `"AAAdLQ"` |
| `e_skey_id` | Identifier of the selected signed prekey. | The selected signed-prekey record. | 3-byte big-endian integer → unpadded Base64url; use the same record as the next two fields. | `"KerV"` |
| `e_skey_val` | Public key of that signed prekey. | The public key from the same record as e_skey_id. | Raw 32 bytes without the type prefix → unpadded Base64url. | `"CMCBNzF6M6QpNfObtJHTzh6Z9PysGCX5YOzOncgK570"` |
| `e_skey_sig` | Identity-key signature over that prekey. | The signature stored in the same record as e_skey_id/e_skey_val. | Signature bytes → unpadded Base64url; distinct from the request signature H. | `"vQCjaftbuFsIGt7cXx_nzSc3wq7nGMj79prnxbK6eerqqnBpT66_OHqwHGnq-TGEs4NRdgmT9oYI1FTzCyl55A"` |

### 5. Client validation and security fields

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `token` | Validation value bound to client application material and the number being checked. | Derived using package name, application resources, signing certificate, DEX content, and the national number. | Standard Base64; distinct from Google, push, and backup tokens. | `"H05HI8Fb1Yo1aFldguetNlIafyg="` |
| `gpia` | Combined Play Integrity result and application integrity information. | A real asynchronous Play Integrity result plus application information; see [Play Integrity](android-inputs.md#play-integrity). | Internal JSON with field-level encryption and Base64 packaging; may describe a token, error, or timeout. | `"pTAApqGZasLEwfdRwW2HOypCqTvgGFSB9FbdAR6HZqhj…nr/s9cG0nij7ENLhy"` (example excerpt) |
| `_gg` | Compact packaging of the Play Integrity outcome. | The result code and optional token from the same invocation used by gpia. | Field-encrypted JSON packaged with Base64; do not mix outcomes from separate calls. | `"mYVMzuhLj8lb5pky1X0EMj/4CwfIoZltinkTWYaouqrW…y+OgTTqA6BxXgAeGz"` (example excerpt) |
| `_gi` | Integrity information about installed application content, signatures, and directories. | Installation paths, package/DEX streams, signing certificates, and installation environment; raw APIs in [Package material](android-inputs.md#package-material). | A composite structure with field-level encryption and Base64 packaging. | `"eHnqtzWQgW9e2IcFRCxh4H3C3dfC7ZvU5RvHbHtWTDVw…CZjT9GJmkEWByrQ=="` (example excerpt) |
| `_gp` | Digest of the declared permission set. | PackageInfo.requestedPermissions; see [Declared permissions](android-inputs.md#declared-permissions). | Sort names by UTF-8 bytes, concatenate without separators, SHA-256, then standard Base64; not the granted-permission set. | `"q19gmaQaITVtyoBCdLsD7CiZKryGA9zltE8yTd30NdI="` |
| `_gs` | Security signal combining path checks, execution state, time, and randomness. | Filesystem stat results, time, randomness, and security-component preparation state. | Outer JSON contains em, which packages an ephemeral public key and AES-GCM ciphertext/tag using Base64url; not one boolean. | `{"em":"f8_clCHf9eCJ5CRfvd_bTug_ZagXp2NjeF0Dht…VIvwpJI2Z-YC8GIo"}` (example excerpt) |
| `_ge` | Virtualization-related module checks and conditional su-path checks. | Module and path checks visible to the current process. | JSON text; distinguish detection from a failed check; not an entirely CBC-encrypted field. | `{"sb":false,"sv":false}` |
| `_ga` | Environment information including path ages, automation state, and a boot identifier. | Directory metadata, ActivityManager.isUserAMonkey(), and the readable boot identifier, among other inputs. | Outer JSON; ap/ai/ae are seconds, mp/mu booleans, and bi has its own encryption wrapper. Access failures are not normal empty contents. | `{"mp":false,"mu":false}` (example excerpt) |

### 6. Device, permission, and network state

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `device_name` | Device build codename, not a user nickname or marketing model name. | Build.DEVICE; see [Device codename](android-inputs.md#device-name). | String; do not substitute Build.MODEL. | `"coral"` |
| `device_ram` | Total device RAM. | ActivityManager.MemoryInfo.totalMem; see [Total RAM](android-inputs.md#device-ram). | Bytes divided by 1024³, formatted as GiB text with up to two decimal places; not availMem. | `"5.34"` |
| `pid` | PID of the application process performing the flow. | android.os.Process.myPid(); see [Process ID](android-inputs.md#process-id). | Decimal text. | `"10751"` |
| `db` | ADB setting state. | Settings.Global.ADB_ENABLED; see [ADB setting](android-inputs.md#adb-setting). | Decimal text; collection configuration may omit it; not Debug.isDebuggerConnected(). | `"1"` |
| `read_phone_permission_granted` | Whether all phone permissions needed by the flow are granted. | READ_PHONE_STATE and, where required, READ_PHONE_NUMBERS; see [Phone permissions](android-inputs.md#phone-permissions). | 1/0 text; declaring a permission is different from having runtime approval. | `"0"` |
| `simnum` | Whether a device-number string meets the length threshold. | A number returned by the selected subscription/telephony service; see [Device number](android-inputs.md#device-number). | 1 for a non-null string of length at least 6, otherwise 0; not a SIM count or ownership proof. | `"0"` |
| `sim_state` | Detailed SIM-state number. | TelephonyManager.getSimState() for the default SIM, or getSimState(slotIndex) for a logical slot; see [SIM state](android-inputs.md#sim-state). | Decimal text for states such as absent, locked, or ready; distinct from sim_type. | `"1"` |
| `sim_operator_name` | SIM home-operator name. | TelephonyManager.getSimOperatorName(); see [Operator names](android-inputs.md#operator-names). | String; select the subscription on multi-SIM devices. Empty results and read failures are different. | `""` |
| `network_operator_name` | Current telephony-network operator name. | TelephonyManager.getNetworkOperatorName(); see [Operator names](android-inputs.md#operator-names). | String; may differ from the SIM operator while roaming. | `""` |
| `network_radio_type` | Application classification of transport and cellular radio technology. | ConnectivityManager/NetworkCapabilities and the selected subscription data-network type; see [Network classification](android-inputs.md#network-type). | Protocol enum, not a raw NETWORK_TYPE value; examples: Wi-Fi/Ethernet class 1, LTE 111, NR 113. | `"1"` |
| `is_foa_fdid_app_installed` | Whether visible related applications match the required signing rules. | PackageManager results plus application-defined certificate rules; raw certificate APIs in [Related-app signatures](android-inputs.md#related-app-signatures). | true/false text; package names alone are insufficient, and package visibility limits results. | `"false"` |

### 7. Push and optional telephony-environment fields

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `push_token` | Push-provider token for the application instance. | The current token from a push SDK callback and local storage; FCM entry in [Push token](android-inputs.md#push-token). | Include when available and required by the flow; unavailable/empty values may be omitted, not replaced by a UUID. | `"ZNMneWyXD336HaogDvqthjeHW6lvJfbH3pHuN7vnuszUbrQpmu1M1ECtfYr7ZIs1DP8eL-MROsjyzEPEOcGu9g"` |
| `sim_type` | Application-level SIM-availability category. | SIM-service availability and getSimState(). [Raw SIM state](android-inputs.md#sim-state) | Conditional; 0 for absent, 1 for other readable SIM states, 2 for unavailable service; not sim_state. | `"0"` |
| `airplane_mode_type` | Airplane-mode setting. | Settings.Global.AIRPLANE_MODE_ON; see [Airplane mode](android-inputs.md#airplane-mode). | Conditional; normalized 0/1 text. | `"0"` |
| `cellular_strength` | Cellular signal-strength level. | TelephonyManager.getSignalStrength().getLevel(); see [Signal level](android-inputs.md#signal-strength). | Conditional; 0..4 are signal levels, while 5 is an unavailable fallback, not maximum strength. | `"5"` |
| `roaming_type` | Telephony roaming classification. | Network presence, selected telephony service, and isNetworkRoaming(). [Raw roaming state](android-inputs.md#roaming-state) | Conditional; 0 for not roaming, 1 for roaming, and 2 when the telephony service object is absent. Omit without a network snapshot; do not automatically map read exceptions to 0 or 2. | `"0"` |

### 8. Flow metrics, interaction, and diagnostics

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `client_metrics` | Registration-flow metrics and installation attribution. | Actual business-attempt count, installation/referral state, and downloader activation state. | Compact JSON string; increment attempts at the business-attempt boundary, not on every serialization. | `{"attempts":4,"was_activated_from_stub":false}` (example excerpt) |
| `offline_ab` | Local experiment assignment, exposure, and optional UI metrics. | Application experiment/exposure storage and the current UI-flow state. | Compact JSON string. | `{"exposure":["xwztmb_bru_okc_uk_zs…v\|qmf_9"],"exp_hash":["860\|yzvwgr_uyg_dhw_n…bftogha"]}` (example excerpt) |
| `feo2_query_status` | Processing status of a related-service capability query. | State persisted by the capability-query callback. | Examples: did_not_query, cached, success, or an error category; distinct from client_capabilities content. | `"did_not_query"` |
| `language_selector_clicked_count` | Change-related count accumulated by language-selector interaction rules. | Actual language notifications, dismissal events, and local metrics. | Decimal text; neither total taps nor necessarily actual Locale changes. | `"0"` |
| `language_selector_time_spent` | Accumulated language-selector open duration. | Milliseconds accumulated by actual panel-open/close events. | Decimal millisecond text; capture together with the count and handle consumption at the request boundary. | `"0"` |
| `mistyped` | Combined number-correction interaction and phone-permission state. | Current input, device candidate, hint display/acceptance, interaction mode, and permissions. | Enum string; 7 can mean missing phone permissions. Not a typo count or a function of the number alone. | `"7"` |

### 9. Additional parameters for specific flows

| Field | Meaning | Input source | Representation and presence | Example |
| --- | --- | --- | --- | --- |
| `foa_backup_token` | Recovery material from a related-application backup path. | The relevant account/backup provider, selected by number. | Conditional binary field, percent-encoded at the form layer; not an alias for backup_token. | `"%F3%FD%FC%F3s%AEFmj%14t%CE%0A%F3%DE%B8%D0%15n%3C"` |
| `client_capabilities` | Capability data returned by a related identity service. | Actual response bytes from the capability service. | Unpadded Base64url when non-null; separate from feo2_query_status. | `"AQID"` |
| `login` | Login material for a particular same-device/two-factor flow. | The two-factor login key held by that flow. | Conditional string; not an unconditional copy of a same-named exist response field. | `"0-SEEbMgJz6xguYAzxzpseDm1h5V7rSs0C7N573bxps"` |
| `type` | A particular user/registration type. | The business-type argument of the calling flow. | Decimal text; type can affect cc/in presence, so one template does not cover every branch. | `"0"` |
| `context` | Special registration context. | Actual invitation, unban, Web-registration, or two-factor polling state. | Conditional text, for example invite_registration, unban_registration, web_registration, or poll_2fa. | `"invite_registration"` |
| `tmobile_exp` | A classification for a specific operating/carrier environment. | The application environment-detection result. | Conditional decimal text; no generic one-to-one mapping to TelephonyManager network-type constants. | `"0"` |
| `sim_status` | Supplemental SIM state supplied by the calling flow. | A string supplied by the SIM-check workflow. | Conditional text; distinct from sim_state and sim_type. | `"1"` |
| `first_api_level` | Android API level with which the device originally shipped. | The device ro.product.first_api_level property, exposed through an authorized device-provider interface. | Conditional positive-integer text; not Build.VERSION.SDK_INT. No equivalent public getter works for every ordinary app/device. | `"28"` |
| `old_phone_number` | Old number associated with the flow. | The previous calling code followed directly by the previous national number, both from existing account state. | Include only with relevant existing state. Digits only, without +, spaces, or separators; use the previous calling code, not the current cc. | `"8618056485253"` |
| `recaptcha` | A reCAPTCHA result or diagnostic wrapper. | The actual reCAPTCHA callback and current verification stage. | Can be a token string or JSON containing token/token_length/token_age/error/stage; not always a bare token. | `"zaKSWs-lzA9ZKKKT9VK69pbeQZS8Nznz_-1FkrhTK1uc…tHYpKaHv2hdCh-mcr"` (example excerpt) |
| `fid` | Auxiliary identity text from a specific identifier provider. | Values and composition rules maintained by that provider. | Conditional string; not automatically a Firebase Installation ID and not interchangeable with fdid. | `"286de756-0103-40f5-bcc0-90814fae25e8\|1510401559"` |
| `preloads_app_manager_id` | Application-manager identifier for a preload channel. | A preload/channel-management service. | Include when supplied; not an installer package name or device serial. | `"795a8dcfcdb6f05267577d725efe3817"` |
| `preloads_attribution` | Preload attribution content. | Data returned by a preload-attribution provider. | Conditional string; not a simple copy of installation-source metrics. | `"5f273ecb4cb1f99a266b0d083762a76a24cfb37ff7a2fc5f"` |
| `tos_version` | Terms identifier used by a particular terms flow. | The terms-flow configuration and existing consent state. | Conditional decimal text; a particular branch can use 5, unrelated to the application version number. | `"5"` |
| `cred_token` | Credential associated with a phone number. | Valid material selected by a credential manager using the normalized number. | Conditional string; no Android-wide getter, and not interchangeable with a Google token or random text. | `"tdIDs03R_b_aOnk8yxs75qsGVPA4ab3O2pDLsMm6nCpJzt5WNh3xK2MNaVjWpYmJ"` |
| `e_pq_last_resort_id` | Identifier of a post-quantum last-resort key record. | The same PQ last-resort record selected by the cryptographic session module. | Raw bytes → unpadded Base64url; grouped with val and sig. | `"NHlsOg"` |
| `e_pq_last_resort_val` | Public material of that PQ last-resort record. | The same record as e_pq_last_resort_id. | Raw bytes → unpadded Base64url; the public-material byte length is determined by the PQ algorithm and record. | `"2jffqARfLr6C1HEDDgF5mJaT7dwIX_1PHyVEru5rCdEN…8sf0HyQKM4Z_FFcfG"` (example excerpt) |
| `e_pq_last_resort_sig` | Signature material for that PQ last-resort record. | The signature stored with the matching id and public material. | Raw bytes → unpadded Base64url; include the three components together when complete. | `"L8RmFk-XvjIMzBBSPPwttNuzwdGZxOiNedZEpSWZP_QgGBKIzF75Dm26G5E1aj3cQX9bfOvek4vjqcGEk0VDdg"` |

The random `e_pq_last_resort_*` examples illustrate Base64url representation only; their byte counts do not define protocol sizes. Read all three values from the same key record and retain the lengths required by its algorithm. Do not generate, pad, or truncate material to match the examples.

### Assembly Rules

Use one `cc/in` pair and keep number-dependent material aligned. Capture backup material/source, signed-prekey components, and Play results consistently. Accumulate or consume metrics at business events rather than during serialization retries. The four optional telephony-environment fields share a collection gate; push-token inclusion is separate.

Omission, null, zero, and false differ. Apply each encoding once, preserving its Base64 alphabet and padding. The [full example](#plaintext-example) and [mock Java builder](#mock-fields-java) list all 71 parameters for reference, including fields from different flows; choose fields and real values for the current flow. JSON-valued fields contain compact JSON strings. Recovery-byte examples are already percent-encoded and must not be encoded twice.

## 相关内容 / Related Reading

- [Android 获取代码 / Android input examples](android-inputs.md)：公开 API、应用持久状态与字节编码。 / Public APIs, persistent state, and byte encoding.
- `security-fields/`：七个安全字段的详细结构与算法。 / Security-field structures and algorithms.
- `request-encryption.md`：参数序列化、ENC 与 H。 / Serialization, ENC, and H.
- `sending-requests.md`：Authorization、请求头、HTTP 发送与响应。 / Authorization, headers, transmission, and responses.

[返回注册目录 / Back to registration](README.md)
