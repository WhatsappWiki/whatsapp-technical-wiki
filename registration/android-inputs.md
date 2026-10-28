# Android 字段获取代码 · Android Input Examples

**Telegram Contact: [@ttjjda](https://t.me/ttjjda)**

[注册目录](README.md#zh) · [中文](#zh) · [English](#en) · [字段索引](#field-lookup) · [Field Lookup](#en-field-lookup)

<a id="android-input-examples--android-字段获取代码"></a>

<a id="android-examples"></a>

<a id="zh"></a>

## 中文

本页提供四组 Java 辅助类：系统输入读取、应用状态存储、包信息读取与字段编码，使用 Android 公开 API 和 Java 标准库。编译时需要 API 33 或更高的 `android.jar`，运行基线为 API 28；较新接口通过 API 级别判断后调用。

以下权限声明放在 `AndroidManifest.xml` 的 `<manifest>` 下、`<application>` 外。电话权限还需按功能在运行时申请；网络状态读取需要 ACCESS_NETWORK_STATE。仅声明权限不等于用户已经授予。代码不会自动申请权限。权限前置失败、特定读取段捕获的异常与其它异常各按下文分支处理，不把所有错误统一改成正常状态。

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_NUMBERS" />
```

<a id="field-lookup"></a>

### 字段索引

按字段定位取值方式。`context` 是调用应用的 Context；`state` 是下方 `RegistrationState` 的实例。所有示例辅助类的完整代码均在本页后半部分。

<a id="app-locale"></a>

#### 语言与地区

`lg` / `lc`：先由应用语言管理器解析 Locale。其 Locale 缓存可直接复用；语言文本还有独立的条件缓存，非null命中时不重读资源或校验格式，地区没有这份文本缓存。下面的 `appLocale()` 演示资源读取和空值fallback，`language()` / `localeCountry()` 演示缓存未命中后的格式转换；它们不自行恢复强制语言、缓存开关或失效回调。

```java
java.util.Locale locale = AndroidFields.appLocale(context);
String language = locale.getLanguage();
String country = locale.getCountry();
String lg = language.matches("[a-z]{2,3}") ? language : "zz";
String lc = country.matches("[A-Z]{2}|[0-9]{3}") ? country : "ZZ";
```

<a id="persistent-identifiers"></a>

#### 持久标识

`fdid`：light 中的 `phoneid_id` 与 `phoneid_timestamp` 成对读取；ID 为 null 或时间为 -1 才进入生成/保存分支，已有空串也不自动按缺失处理。下方 `RegistrationState` 是自定义示例存储，`readString("fdid")` 不是此字段的真实 getter。[标识与缓存](exist.md#zh)

<a id="performance-identifier"></a>

#### 性能设备标识

`expid`：实际读取 light 的 `perf_device_id`；任何非 null 文本先复用，包括空串或非法 UUID，后续编码解析仍可失败。缺失才延迟读取 Android ID：正常非空白值按完整普通 UTF-8 生成 nameUUID，正常 null/Unicode 全空白才随机生成；读取异常不等于正常 null。`state.getOrCreateExamplePerfUuid()` 只演示自定义存储的初始化，不绑定实际 resolver 活槽或偏好后端。[UUID 字节编码](#uuid-encoding)

<a id="access-session"></a>

#### 访问会话

`access_session_id`：先读会话管理器的内存 wrapper；已有 wrapper 即复用，包括其中 UUID 为 null 的状态。缺 wrapper 才读取 `reg_prefs.access_session_id`，按空白/UUID 解析分支决定生成；生成后先持久化再发布缓存。普通读取不触发原生同步，显式 reset 才另按 bootstrap 与 record 状态处理；自定义 `state` 偏好读取不能替代这段生命周期。[会话说明](code.md#zh)

<a id="android-id"></a>

#### Android ID 读取

`aid` 的原始输入：`android.provider.Settings.Secure.getString(context.getContentResolver(), android.provider.Settings.Secure.ANDROID_ID)`。这一步尚未计算摘要。

<a id="device-name"></a>

#### 设备代号

`device_name`：`android.os.Build.DEVICE`。对应 `AndroidFields.deviceName()`，不是 `Build.MODEL`。

<a id="device-ram"></a>

#### 设备总内存

`device_ram`：`getMemoryInfo()` 填充 `totalMem` 后除以 1024³，用实际格式化 Locale 的 `DecimalFormat("#.##")` 输出。服务缺失或该读取的 NullPointerException 分支先取 -1 字节，再执行相同除法/格式化，不直接发送 `"-1"`；其它异常不统一吞掉。`totalRamGiB()` 演示一次读取/变换，实际字段所属实例可缓存非 null 文本。

<a id="process-id"></a>

#### 进程编号

`pid`：`android.os.Process.myPid()`；`AndroidFields.pid()` 返回十进制文本。

<a id="adb-setting"></a>

#### ADB 开关

`db`：`Settings.Global.getInt(context.getContentResolver(), Settings.Global.ADB_ENABLED, 0)`；对应 `AndroidFields.adbEnabled(context)`。

<a id="phone-permissions"></a>

#### 电话权限

`read_phone_permission_granted`：用 `Context.checkSelfPermission()` 检查所需电话权限；`AndroidFields.phonePermissionsGranted(context)` 返回 boolean，写入字段时用 `granted ? "1" : "0"`；它不会申请权限。

<a id="device-number"></a>

#### 本机号码

`simnum`：权限前置不满足时，号码读取返回 null，长度分类为0；允许时遍历活动订阅的 `SubscriptionInfo.getNumber()`，保留首个非 null 值（包括空串），之后才进入默认电话服务的 `getLine1Number()` 兜底。不能因系统升级而改用 `getPhoneNumber()` 并声称是同一选择规则。订阅枚举异常传播；只有电话服务兜底段的 Exception 返回 null。`simnum(number)` 只判断已捕获字符串的 Java length≥6，不重读系统。

<a id="sim-state"></a>

#### SIM 状态

`sim_state`：`AndroidFields.simState(context)` 读取默认SIM状态，服务不存在时为-1。指定卡槽的 `simState(context, slotIndex)` 是额外原始API示例；slotIndex是逻辑卡槽编号，不能直接传订阅ID，也不替代请求的默认卡读取。两者均不是 `sim_type` 分类。

<a id="operator-names"></a>

#### 运营商名称

`sim_operator_name` / `network_operator_name`：本字段路径使用默认电话服务的 `getSimOperatorName()` / `getNetworkOperatorName()`；服务或正常返回值为 null 时归为空串，异常仍传播。`simOperator()` / `networkOperator()` 不自行创建默认数据订阅范围的服务来替换该来源。

<a id="network-type"></a>

#### 网络分类

`network_radio_type`：`AndroidFields.networkRadioType(snapshot)` 只映射已选定的 mobile/wifi/subtype 输入。旧快照路径由 `legacyNetworkSnapshot(context)` 读取，只有 type0 是 mobile、type1 是 Wi-Fi，其它类型不是以太网自动归1。SDK≥31且网络管理器已初始化的新路径，transport1/3 归Wi-Fi类1；其它分类再按 capability18 区分2/3，其中3没有 mobile 位。新路径的 callback、权限、主线程 subtype 缓存须由同一网络管理器提供，不能用一次同步 activeNetwork 读取替代。null 或既非 mobile 又非 wifi 为 -1；未知 mobile subtype 为0。500ms只合并通知，不是字段有效期。

<a id="airplane-mode"></a>

#### 飞行模式

`airplane_mode_type`：`Settings.Global.AIRPLANE_MODE_ON`；对应 `AndroidFields.airplaneMode(context)`，返回归一化的 0/1。

<a id="signal-strength"></a>

#### 蜂窝信号等级

`cellular_strength`：同组开关允许时，SDK≥28、电话服务存在且第一次 `getSignalStrength()` 非 null，才在第二次调用结果上取 `getLevel()`。缺支持/服务/首次对象时为5；第二次返回 null、权限或设备能力异常不统一转5。`signalLevel()` 保留两次读取；两次读取不构成原子快照，不能用保存第一次对象来替换。

<a id="roaming-state"></a>

#### 漫游状态

`roaming_type` 的原始输入：`AndroidFields.roaming(context)` 调用 `isNetworkRoaming()`。有网络快照时，false/true 分别编码为 `"0"`/`"1"`，电话服务对象不存在返回 null 时编码为 `"2"`；无快照则省略字段，读取异常另行处理。

<a id="recovery-record"></a>

#### 恢复文件存在性

`hasinrc`：`new java.io.File(context.getFilesDir(), "rc2").exists()`；对应 `state.hasRecoveryRecord()`。这里只检查存在性，不读取或解密 `id`。

<a id="package-material"></a>

#### 安装包原始输入

`_gi` 的文件输入分开读取：完整/前缀摘要和大小使用主 `sourceDir`，DEX摘要读取 `getPackageCodePath()` 的 `classes.dex`；`installedCodePaths()` 的 split 列表不是统一摘要输入。`dataDirectory()` 与 `nativeLibraryDirectory()` 提供目录；`_icr` 由实际初始 Application 的 PackageManager 以 GET_SIGNATURES（64）读取 signatures 首项DER，再SHA-1/标准Base64。`firstLegacyCertificate(initialApplication)` 只演示这次证书读取，不改用 SigningInfo/history；调用方须传入对应的初始 Application。[_gi详解](security-fields/gi.md#zh)

<a id="declared-permissions"></a>

#### 声明权限集合

`_gp` 的原始输入：对本次采集参数中的同一个 Application，用 GET_PERMISSIONS（4096）读取 `requestedPermissions`。`declaredPermissions()` 保留数组 null、空名称与重复，不把它换成已授权集合。后续按 JNI modified UTF-8、无符号排序、无分隔拼接求摘要；空拼接或指定 API 失败使用 `default_permission`，cache miss且Application缺失则可返回null，两者不同。非null结果槽命中时不再查询PackageManager。[_gp详解](security-fields/gp.md#zh)

<a id="related-app-signatures"></a>

#### 关联应用签名

`is_foa_fdid_app_installed`：对可见的其它应用，`relatedAppFirstCertificate(context, packageName)` 按字段所用的 SigningInfo/history 或 signatures 分支取首项；后续做 SHA-1/无padding Base64url，并按关联应用匹配规则检查。它不使用 `_gi._icr` 的固定 flags64 策略，读取一张DER也不等于得到最终布尔值。

<a id="uuid-encoding"></a>

#### UUID 字节编码

`expid` / `access_session_id` 的表示转换：`FieldEncoding.uuidBytes(uuid)` 把已有 UUID 转成大端 16 字节并做 Base64url。它不获取或创建 UUID。

<a id="signal-id-encoding"></a>

#### Signal 编号编码

`e_regid` 编码已有编号的4字节大端表示；`e_skey_id` 编码同一选定预密钥 protobuf id 的低24位、3字节大端表示，不额外拒绝高位非零或负的 Java int。SQL行号只参与记录选择，不替代该protobuf id。三个 `e_skey_*` 从一次记录快照读取，签名长度按记录保留，不由请求 H 生成。

<a id="aid-encoding"></a>

#### Android ID 摘要编码

`aid`：正常 Android ID 按 JNI modified UTF-8 求 SHA-256，再做标准Base64含padding；NUL为 C0 80，补充字符按两个UTF-16单元各自编码。`aidForPresentAndroidId()` 只做非null字符串变换；已确认的读取失败分支用 `aidForSignalError()`，不把任意异常或缺少接入输入都伪装成正常结果。实际读取从当前线程的 Application 获取 resolver，与 expid 的 resolver 活槽不同；结果随安全owner缓存。expid 的 nameUUID 仍使用普通UTF-8，不能复用此字节编码。

<a id="advertising-id"></a>

#### 广告标识

`advertising_id`：Google Play services 的 `AdvertisingIdClient.getAdvertisingIdInfo(context)` 提供ID与追踪限制，需在工作线程调用。字段所选的新provider分支在LAT=true时省略；旧直接读取分支不读取LAT，不能给它强加同一省略条件。国家/流程开关另行决定是否获取，没有失败后自动切换另一分支或此调用链的广告ID持久缓存。

<a id="push-token"></a>

#### 推送令牌

`push_token`：FCM 的 `FirebaseMessaging.getInstance().getToken()` 与 `onNewToken()` 回调；需先完成应用的 Firebase 配置。

<a id="play-integrity"></a>

#### Play Integrity 服务输入

`gpia` / `_gg` 的服务输入：获取开关允许后使用 `StandardIntegrityManager`，已有provider可跳过prepare；正常本次生成共用一次结果码/可选token。requestHash取此前连接存储public32的标准Base64无padding，不先SHA-256。字段包装晚些读取派生公钥，独立IV/发布；公共Map复用旧槽时不保证仍与本次gpia对应。[服务接入](#service-integration) · [Play结果生命周期](security-fields/gpia.md#zh)

<a id="android-system"></a>

### AndroidFields

`AndroidFields` 获取调用应用可访问的系统输入。字段所用的默认电话服务与显式创建订阅范围的服务不同；SIM卡槽示例也是额外原始API入口，不替换请求的默认卡读取。号码提示保留活动订阅顺序和首个非null值。服务对象可由实际owner缓存，但不因此缓存它返回的号码、运营商或信号。

`networkRadioType()` 只负责协议映射；`legacyNetworkSnapshot()` 仅用于已选定的旧快照分支，新分支须读取实际callback状态。`totalRamGiB()` 是单次计算，不建立字段owner的文本缓存。`stat()` 抛出读取异常，由 `_ge/_gs` 等采集器各自将实际返回/失败按规则处理；`isMonkey()` 只提供 `_ga.mu` 的API输入，不计算 `_ga.mp` 或整份 `_ga`。

```java
// File: AndroidFields.java
package wiki.examples;

import android.Manifest;
import android.app.ActivityManager;
import android.content.Context;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.os.Build;
import android.provider.Settings;
import android.system.ErrnoException;
import android.system.Os;
import android.system.StructStat;
import android.telephony.SubscriptionInfo;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import java.text.DecimalFormat;
import java.util.List;
import java.util.Locale;

public final class AndroidFields {
    private AndroidFields() {}

    public static Locale appLocale(Context context) {
        android.os.LocaleList locales =
                context.getResources().getConfiguration().getLocales();
        Locale value = locales.isEmpty() ? Locale.getDefault() : locales.get(0);
        if (value != null) return value;
        Locale fallback = Locale.getDefault();
        return fallback != null ? fallback : Locale.US;
    }

    public static String language(Context context) {
        String value = appLocale(context).getLanguage();
        return value.matches("[a-z]{2,3}") ? value : "zz";
    }

    public static String localeCountry(Context context) {
        String value = appLocale(context).getCountry();
        return value.matches("[A-Z]{2}|[0-9]{3}") ? value : "ZZ";
    }

    public static String deviceName() { return Build.DEVICE; }
    public static String pid() {
        return Integer.toString(android.os.Process.myPid());
    }

    public static String totalRamGiB(Context context) {
        ActivityManager service = context.getSystemService(ActivityManager.class);
        long total = -1L;
        if (service != null) {
            ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();
            try {
                service.getMemoryInfo(info);
                total = info.totalMem;
            } catch (NullPointerException unavailable) {
                total = -1L;
            }
        }
        return new DecimalFormat("#.##").format(total / 1073741824.0);
    }

    public static String androidId(Context context) {
        return Settings.Secure.getString(context.getContentResolver(),
                Settings.Secure.ANDROID_ID);
    }

    public static int adbEnabled(Context context) {
        return Settings.Global.getInt(context.getContentResolver(),
                Settings.Global.ADB_ENABLED, 0);
    }

    public static int airplaneMode(Context context) {
        int value = Settings.Global.getInt(context.getContentResolver(),
                Settings.Global.AIRPLANE_MODE_ON, 0);
        return value == 0 ? 0 : 1;
    }

    public static boolean phonePermissionsGranted(Context context) {
        if (context.checkSelfPermission(Manifest.permission.READ_PHONE_STATE)
                != PackageManager.PERMISSION_GRANTED) return false;
        return Build.VERSION.SDK_INT < 30
                || context.checkSelfPermission(Manifest.permission.READ_PHONE_NUMBERS)
                == PackageManager.PERMISSION_GRANTED;
    }

    public static TelephonyManager defaultTelephony(Context context) {
        return context.getSystemService(TelephonyManager.class);
    }

    @SuppressWarnings("deprecation")
    public static String firstDeviceNumber(Context context) {
        if (!phonePermissionsGranted(context)) return null;
        SubscriptionManager subscriptions =
                context.getSystemService(SubscriptionManager.class);
        if (subscriptions == null) throw new NullPointerException("SubscriptionManager");
        List<SubscriptionInfo> active = subscriptions.getActiveSubscriptionInfoList();
        if (active != null) {
            for (SubscriptionInfo item : active) {
                String value = item.getNumber();
                if (value != null) return value;
            }
        }
        try {
            TelephonyManager service = defaultTelephony(context);
            return service == null ? null : service.getLine1Number();
        } catch (Exception unavailableFallback) {
            return null;
        }
    }

    public static String simnum(String number) {
        return number != null && number.length() >= 6 ? "1" : "0";
    }

    public static int simState(Context context) {
        TelephonyManager service = defaultTelephony(context);
        return service == null ? -1 : service.getSimState();
    }

    public static Integer simState(Context context, int slotIndex) {
        if (slotIndex < 0) {
            throw new IllegalArgumentException("Expected a logical SIM slot index");
        }
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState(slotIndex);
    }

    public static String simOperator(Context context) {
        TelephonyManager service = defaultTelephony(context);
        String value = service == null ? null : service.getSimOperatorName();
        return value == null ? "" : value;
    }

    public static String networkOperator(Context context) {
        TelephonyManager service = defaultTelephony(context);
        String value = service == null ? null : service.getNetworkOperatorName();
        return value == null ? "" : value;
    }

    public static int signalLevel(Context context) {
        TelephonyManager service = defaultTelephony(context);
        if (Build.VERSION.SDK_INT < 28 || service == null
                || service.getSignalStrength() == null) return 5;
        // Preserve the second read: it can change or fail independently.
        return service.getSignalStrength().getLevel();
    }

    public static Boolean roaming(Context context) {
        TelephonyManager service = defaultTelephony(context);
        return service == null ? null : service.isNetworkRoaming();
    }

    public static final class NetworkSnapshot {
        public final boolean mobile;
        public final boolean wifi;
        public final int subtype;
        public NetworkSnapshot(boolean mobile, boolean wifi, int subtype) {
            this.mobile = mobile;
            this.wifi = wifi;
            this.subtype = subtype;
        }
    }

    @SuppressWarnings("deprecation")
    public static NetworkSnapshot legacyNetworkSnapshot(Context context) {
        ConnectivityManager service = context.getSystemService(ConnectivityManager.class);
        android.net.NetworkInfo info = service == null ? null : service.getActiveNetworkInfo();
        return info == null ? null : new NetworkSnapshot(
                info.getType() == 0, info.getType() == 1, info.getSubtype());
    }

    public static int networkRadioType(NetworkSnapshot snapshot) {
        if (snapshot == null) return -1;
        if (!snapshot.mobile) return snapshot.wifi ? 1 : -1;
        return radioCode(snapshot.subtype);
    }

    public static int radioCode(int androidType) {
        switch (androidType) {
            case TelephonyManager.NETWORK_TYPE_GPRS: return 104;
            case TelephonyManager.NETWORK_TYPE_EDGE: return 100;
            case TelephonyManager.NETWORK_TYPE_UMTS: return 102;
            case TelephonyManager.NETWORK_TYPE_CDMA: return 108;
            case TelephonyManager.NETWORK_TYPE_EVDO_0:
            case TelephonyManager.NETWORK_TYPE_EVDO_A:
            case TelephonyManager.NETWORK_TYPE_EVDO_B: return 103;
            case TelephonyManager.NETWORK_TYPE_1xRTT: return 109;
            case TelephonyManager.NETWORK_TYPE_HSDPA: return 105;
            case TelephonyManager.NETWORK_TYPE_HSUPA: return 106;
            case TelephonyManager.NETWORK_TYPE_HSPA: return 107;
            case TelephonyManager.NETWORK_TYPE_IDEN: return 101;
            case TelephonyManager.NETWORK_TYPE_LTE: return 111;
            case TelephonyManager.NETWORK_TYPE_EHRPD: return 110;
            case TelephonyManager.NETWORK_TYPE_HSPAP: return 112;
            case TelephonyManager.NETWORK_TYPE_TD_SCDMA: return 116;
            case TelephonyManager.NETWORK_TYPE_IWLAN: return 115;
            case TelephonyManager.NETWORK_TYPE_NR: return 113;
            default: return 0;
        }
    }

    public static boolean isMonkey() { return ActivityManager.isUserAMonkey(); }
    public static StructStat stat(String path) throws ErrnoException {
        return Os.stat(path);
    }
}
```

Android ID 与应用签名、用户及设备有关，不能把另一应用读取的值当成相同值。[ANDROID_ID](https://developer.android.com/reference/android/provider/Settings.Secure#ANDROID_ID)

本机号码接口不保证提供可用号码；信号与电话网络接口各有权限和设备能力条件，不支持的设备可抛出 UnsupportedOperationException，服务对象非 null 不代表所有电话能力均可用。[SubscriptionManager](https://developer.android.com/reference/android/telephony/SubscriptionManager#getPhoneNumber(int)) · [TelephonyManager](https://developer.android.com/reference/android/telephony/TelephonyManager) · [NetworkCapabilities](https://developer.android.com/reference/android/net/NetworkCapabilities) · [SIM 默认卡与卡槽](https://developer.android.com/reference/android/telephony/TelephonyManager#getSimState(int))

<a id="android-state"></a>

### RegistrationState

这个示例展示如何在自己的应用中读写状态。`wiki_example_registration_state` 是示例自定义的偏好文件名，键名由调用方定义，不是 WhatsApp 的存储文件名或通用字段读取接口。读取 `fdid` 等字段的前提是应用已按相应规则保存了真实值。

持久标识缺失时才按自己的初始化流程创建并保存；不要在每次请求里为所有 ID 重新生成 UUID。访问会话重置与安装标识初始化也应分别处理。备份和 Signal 材料由其各自模块保存，不能从几个偏好键凭空生成。示例初始化方法没有并发互斥。实际 perf_device_id 的写前重查/写后再读也不等于原子CAS；额外串行化是调用方的设计选择，不能把它写成字段原有保证。此类使用系统 SharedPreferences，错误类型可能抛 ClassCastException；自建 Light 后端的缺键/Map null/错类型默认值规则不由此类模拟。会话wrapper、源摘要和安全结果槽也不由 `clear/apply` 一并清除。

```java
// File: RegistrationState.java
package wiki.examples;

import android.content.Context;
import android.content.SharedPreferences;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

public final class RegistrationState {
    private final Context context;
    private final SharedPreferences preferences;

    public RegistrationState(Context context) {
        this.context = context.getApplicationContext();
        preferences = this.context.getSharedPreferences(
                "wiki_example_registration_state", Context.MODE_PRIVATE);
    }

    public String readString(String key) {
        return preferences.getString(key, null);
    }

    public void saveString(String key, String value) {
        preferences.edit().putString(key, value).apply();
    }

    public String getOrCreateExamplePerfUuid() {
        String existing = readString("example_perf_uuid");
        if (existing != null) return existing;
        String androidId = AndroidFields.androidId(context);
        boolean blank = androidId == null || androidId.isEmpty();
        if (!blank) {
            blank = true;
            for (int i = 0; i < androidId.length(); i++) {
                char value = androidId.charAt(i);
                if (!Character.isWhitespace(value) && !Character.isSpaceChar(value)) {
                    blank = false;
                    break;
                }
            }
        }
        UUID created = blank ? UUID.randomUUID()
                : UUID.nameUUIDFromBytes(androidId.getBytes(StandardCharsets.UTF_8));
        if (readString("example_perf_uuid") == null) {
            saveString("example_perf_uuid", created.toString());
        }
        return readString("example_perf_uuid");
    }

    public int readCount(String key) {
        return preferences.getInt(key, 0);
    }

    public long readDurationMillis(String key) {
        return preferences.getLong(key, 0L);
    }

    public boolean hasRecoveryRecord() {
        return new File(context.getFilesDir(), "rc2").exists();
    }
}
```

`hasRecoveryRecord()` 只检查调用应用自己的文件。存在性不是解密或有效性检查。[应用私有存储](https://developer.android.com/training/data-storage/app-specific)

<a id="android-package"></a>

### PackageInputs

这些方法读取原始包信息，供完整性和安装相关字段使用。`declaredPermissions()` 不读取运行时授权结果；安装方包名也不能直接替代完整安装归因 JSON。

`firstLegacyCertificate()` 用于传入实际初始Application后读取flags64首证书；`clientFirstCertificate()` 在API≥28的SigningInfo非null且不是多签名者时选history，否则选signatures；只转换选中数组的首项。关联应用的首项选择也可用后一规则，随后仍需完成SHA-1/匹配。它们与HTTP Authorization的KeyStore逆序证书链不同，不能互换。查询其它应用还受包可见性限制。

```java
// File: PackageInputs.java
package wiki.examples;

import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.pm.Signature;
import android.content.pm.SigningInfo;
import android.os.Build;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class PackageInputs {
    private PackageInputs() {}

    @SuppressWarnings("deprecation")
    public static String[] declaredPermissions(Context context)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                context.getPackageName(), PackageManager.GET_PERMISSIONS);
        return info.requestedPermissions == null
                ? null : info.requestedPermissions.clone();
    }

    @SuppressWarnings("deprecation")
    public static byte[] clientFirstCertificate(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                packageName, PackageManager.GET_SIGNING_CERTIFICATES);
        SigningInfo signing = info.signingInfo;
        Signature[] certificates = info.signatures;
        if (signing != null && !signing.hasMultipleSigners()) {
            certificates = signing.getSigningCertificateHistory();
        }
        return certificates == null || certificates.length == 0
                ? null : certificates[0].toByteArray();
    }

    @SuppressWarnings("deprecation")
    public static byte[] firstLegacyCertificate(Context initialApplication)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = initialApplication.getPackageManager().getPackageInfo(
                initialApplication.getPackageName(), PackageManager.GET_SIGNATURES);
        Signature[] signatures = info.signatures;
        return signatures == null || signatures.length == 0
                ? null : signatures[0].toByteArray();
    }

    public static byte[] relatedAppFirstCertificate(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        return clientFirstCertificate(context, packageName);
    }

    public static String primaryPackageCodePath(Context context) {
        return context.getPackageCodePath();
    }

    public static String primarySourcePath(Context context) {
        return context.getApplicationInfo().sourceDir;
    }

    public static List<String> installedCodePaths(Context context) {
        ApplicationInfo info = context.getApplicationInfo();
        List<String> result = new ArrayList<>();
        result.add(info.sourceDir);
        if (info.splitSourceDirs != null) {
            Collections.addAll(result, info.splitSourceDirs);
        }
        return result;
    }

    public static String dataDirectory(Context context) {
        return context.getApplicationInfo().dataDir;
    }

    public static String nativeLibraryDirectory(Context context) {
        return context.getApplicationInfo().nativeLibraryDir;
    }

    @SuppressWarnings("deprecation")
    public static String installerPackage(Context context)
            throws PackageManager.NameNotFoundException {
        PackageManager manager = context.getPackageManager();
        String name = context.getPackageName();
        return Build.VERSION.SDK_INT >= 30
                ? manager.getInstallSourceInfo(name).getInstallingPackageName()
                : manager.getInstallerPackageName(name);
    }
}
```

接口说明：[PackageInfo](https://developer.android.com/reference/android/content/pm/PackageInfo) · [SigningInfo](https://developer.android.com/reference/android/content/pm/SigningInfo) · [包可见性](https://developer.android.com/training/package-visibility)

<a id="android-encoding"></a>

### FieldEncoding

本类只转换调用方已经持有的真实材料，不生成身份公钥、Google token 或安全结果。UUID 文本和 UUID 字节编码要区分；`e_regid` 与 `e_skey_id` 的字节长度不同。

`aidForPresentAndroidId()` 使用modified UTF-8，`RegistrationState` 中 expid 示例使用普通UTF-8，两者不可合并。`signedPrekeyId()` 仅取低24位，不验证整个记录。`percentEncodeBytes()` 只保留 `-._~` 与字母数字；它不等于普通文本的Java URLEncoder（空格为+、*保留、~编码）。Kotlin构造器须记录预编码身份，避免二次转义；原生字典枚举与序列化规则见[请求封装](request-encryption.md#zh)。

```java
// File: FieldEncoding.java
package wiki.examples;

import android.util.Base64;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.UUID;

public final class FieldEncoding {
    private FieldEncoding() {}

    public static String base64url(byte[] value) {
        return Base64.encodeToString(value,
                Base64.URL_SAFE | Base64.NO_PADDING | Base64.NO_WRAP);
    }

    public static String uuidBytes(UUID value) {
        return base64url(ByteBuffer.allocate(16).order(ByteOrder.BIG_ENDIAN)
                .putLong(value.getMostSignificantBits())
                .putLong(value.getLeastSignificantBits()).array());
    }

    public static String registrationId(int value) {
        return base64url(ByteBuffer.allocate(4).order(ByteOrder.BIG_ENDIAN)
                .putInt(value).array());
    }

    public static String signedPrekeyId(int value) {
        byte[] bytes = {(byte) (value >>> 16),
                (byte) (value >>> 8), (byte) value};
        return base64url(bytes);
    }

    public static String aidForPresentAndroidId(String androidId)
            throws NoSuchAlgorithmException {
        if (androidId == null) {
            throw new IllegalArgumentException("Android ID is unavailable");
        }
        byte[] digest = MessageDigest.getInstance("SHA-256").digest(
                modifiedUtf8(androidId));
        return Base64.encodeToString(digest, Base64.NO_WRAP);
    }

    public static String aidForSignalError() throws NoSuchAlgorithmException {
        return aidForPresentAndroidId("signal_error");
    }

    public static byte[] modifiedUtf8(String value) {
        java.io.ByteArrayOutputStream out = new java.io.ByteArrayOutputStream();
        for (int i = 0; i < value.length(); i++) {
            int c = value.charAt(i);
            if (c >= 1 && c <= 0x7F) out.write(c);
            else if (c <= 0x7FF) {
                out.write(0xC0 | (c >>> 6));
                out.write(0x80 | (c & 0x3F));
            } else {
                out.write(0xE0 | (c >>> 12));
                out.write(0x80 | ((c >>> 6) & 0x3F));
                out.write(0x80 | (c & 0x3F));
            }
        }
        return out.toByteArray();
    }

    public static String percentEncodeBytes(byte[] value) {
        char[] hex = "0123456789ABCDEF".toCharArray();
        StringBuilder out = new StringBuilder();
        for (byte item : value) {
            int b = item & 0xFF;
            boolean unreserved = (b >= 'A' && b <= 'Z')
                    || (b >= 'a' && b <= 'z') || (b >= '0' && b <= '9')
                    || b == '-' || b == '.' || b == '_' || b == '~';
            if (unreserved) out.append((char) b);
            else out.append('%').append(hex[b >>> 4]).append(hex[b & 15]);
        }
        return out.toString();
    }
}
```

基础接口：[Base64](https://developer.android.com/reference/android/util/Base64) · [ByteBuffer](https://developer.android.com/reference/java/nio/ByteBuffer)

<a id="android-services"></a>

<a id="service-integration"></a>

### 服务接入

| 字段 | 接入入口 | 接入要求 |
| --- | --- | --- |
| advertising_id | `AdvertisingIdClient.getAdvertisingIdInfo(context)`，读取返回对象的 `getId()` 和 `isLimitAdTrackingEnabled()` | 添加 Google Play services 广告标识依赖，按所选 SDK 声明相应权限；在工作线程调用，处理服务不可用与追踪限制。 |
| push_token | FCM 的 `FirebaseMessaging.getInstance().getToken()`，以及 `onNewToken()` 回调 | 先完成 Firebase 应用配置；异步接收并保存真实 token，随刷新更新。 |
| gpia / _gg | `StandardIntegrityManager` 准备provider后请求结果 | requestHash是此前读取的连接存储public32的标准Base64无padding，不先SHA-256。正常本次生成共用一次结果；最终Map旧槽复用另按生命周期处理。 |
| recaptcha | reCAPTCHA SDK 的真实校验结果与回调 | 使用应用自己的服务配置，记录 token 或失败结果；不以随机文本代替。 |
| authkey / e_ident / e_regid / e_skey_* / e_pq_last_resort_* | 应用自己的连接认证与 Signal 会话/密钥模块 | Android KeyStore 不是这些协议记录的通用查询器；公开材料必须与已有身份及记录配对。 |
| id / backup_token / foa_backup_token / cred_token | 各自的恢复、备份或凭据模块 | id读取rc2；exist按号码选择source1并在发送前保存，code直接解码reg_prefs.token_used_during_reg、非空复用/零长才生成；foa与cred各按自己的条件来源处理。 |
| client_metrics / offline_ab / feo2_query_status / language_selector_* / mistyped | 应用的业务事件与持久状态 | 由实际事件累计、写入或消费；不在序列化时编造交互记录。 |

Google/Firebase 返回值属于调用应用、服务项目和对应操作，示例应用取到的 token 不能直接当成另一个客户端的凭据。

实际接口说明：[AdvertisingIdClient](https://developers.google.com/android/reference/com/google/android/gms/ads/identifier/AdvertisingIdClient) · [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/android/get-started) · [Play Integrity](https://developer.android.com/google/play/integrity/standard)

<a id="input-boundaries"></a>

### 字段组装与专题边界

原始API值、派生字段、缓存槽与最终请求不是同一对象。`RegistrationState` 不接管应用的真实生命周期；电话环境四字段还受同组开关控制，`db` 有独立开关。空串、null、缺键、0与false分别处理，不为缺失的服务结果填造字段。

| 专题 | 与本页代码的衔接 |
| --- | --- |
| [安全字段总览](security-fields/README.md#zh) | gpia走独立callback；其余六个安全字段与aid在安全owner槽中保存。owner重建、context字典、库全局和已构造Map引用分别处理。 |
| [gpia](security-fields/gpia.md#zh) | provider非null可绕过prepare；本地不做in-flight合并或固定TTL。Google阶段timeout不覆盖后续元数据/CBC包装，token与provider缓存分开。 |
| [_gg](security-fields/gg.md#zh) | 正常本次生成将gpia的结果码/可选token投影为_ic/_it，独立IV/发布；公共Map可读旧槽，获取开关关闭不自动清旧值。 |
| [_gi](security-fields/gi.md#zh) | 完整摘要键2、DEX键0、库级_is/_p与最终密文槽分开。首次非null_is发布可保留，路径先于打开发布；主文件与split、签名API选择均不可混用。 |
| [_gp](security-fields/gp.md#zh) | modified UTF-8、无符号排序、保留重复和空拼接默认材料共同决定摘要；原始权限数组不是成品字段，错误摘要也可缓存。 |
| [_gs](security-fields/gs.md#zh) | 13次stat、两次timeval、libc rand、prepared和加密DRBG分别提供输入；prepared初值1337、实际Prepare写4242。通用stat/Java随机数不能代替全部采集与19项CSV。 |
| [_ge](security-fields/ge.md#zh) | sv对应vmw_pvscsi，sb对应vboxsf；按配置决定su是否出现，缓存命中可跳过配置/stat。检查失败的false不证明路径全局不存在。 |
| [_ga](security-fields/ga.md#zh) | mu的API与mp的进程扫描不同。只有bi单独CBC；boot失败先以failed_sha为材料求摘要，owner创建时的公共快照与整份缓存不会随换钥自动更新。 |
| [请求加密](request-encryption.md#zh) | 普通文本与预编码字节分别序列化，内部JSON先成为字符串字段。ENC用新临时X25519/GCM；正常加密时H签ENC，Kotlin明文回退时可签query，证书链另配对。 |
| [发送流程](sending-requests.md#zh) | 最后合并/刷新之后才形成form与body。Kotlin重试可复用参数并重建ENC/H，不重跑业务计数；解析成功与业务成功分开，六项WAMsys应用头不推广到所有路径。 |

`exist` 与 `code` 的备份读取、尝试计数和纠错字段按各自文章处理：code只读已保存的mistyped文本，reason在异步准备前捕获且空串保留；普通页面hasav为Retriever→2、否则RECEIVE_SMS获准→1、其它正常结果→0，特定fallback也可直接传0。FEO2诊断读取不主动query或清能力缓存，不能因为读取统计/诊断而重新执行其producer。

本页编码结果仍需进入所选通道的最终序列化。字段级CBC、ENC、H和Authorization使用不同输入与生命周期；四组辅助类不拼装最终HTTP正文。

[exist 参数](exist.md#zh) · [code 参数](code.md#zh)

<a id="en"></a>

## English

Four Java helper classes cover system inputs, application state, package information, and field encoding using public Android APIs and Java standard libraries. Compile against API 33 or newer; the runtime baseline is API 28, with guards for newer calls.

Place these declarations directly under `<manifest>`, outside `<application>`, in `AndroidManifest.xml`. Request the phone permissions required by your feature at runtime; ACCESS_NETWORK_STATE is needed for network reads. The code does not request permissions. An unmet permission gate, an exception caught by a specific read segment, and other failures follow the separate branches below rather than one universal normal fallback.

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_NUMBERS" />
```

<a id="en-field-lookup"></a>

### Field Input Lookup

Locate the acquisition step by field. `context` belongs to the calling app; `state` is an instance of the `RegistrationState` class provided below. Full helper implementations follow the lookup sections.

<a id="en-app-locale"></a>

#### Language and Region

`lg` / `lc`: use the application language manager’s resolved Locale. Its Locale cache can be reused; a separate gated language-text cache skips resources and validation on a non-null hit, while country has no such text cache. `appLocale()` below illustrates resource acquisition/null fallback, and `language()` / `localeCountry()` illustrate formatting on a cache miss. They do not restore forced-language state, cache gates, or invalidation callbacks.

```java
java.util.Locale locale = AndroidFields.appLocale(context);
String language = locale.getLanguage();
String country = locale.getCountry();
String lg = language.matches("[a-z]{2,3}") ? language : "zz";
String lc = country.matches("[A-Z]{2}|[0-9]{3}") ? country : "ZZ";
```

<a id="en-persistent-identifiers"></a>

#### Application Persistent Identifiers

`fdid`: read `phoneid_id` and `phoneid_timestamp` together from light. A null ID or timestamp -1 enters generation/persistence; an existing empty string is not automatically missing. `RegistrationState` below owns an example store; `readString("fdid")` is not this field’s actual getter. See [identifiers and caches](exist.md#en).

<a id="en-performance-identifier"></a>

#### Performance Device Identifier

`expid`: read light `perf_device_id`; reuse any non-null text first, including empty/invalid UUID text that can fail during later encoding. Only a miss lazily reads Android ID: use its complete ordinary UTF-8 bytes for a nameUUID if nonblank, or generate randomly for a normal null/Unicode-blank result. A read exception is not a normal null. `state.getOrCreateExamplePerfUuid()` illustrates initialization in a custom store, without binding the actual resolver slot/backend. See [UUID encoding](#en-uuid-encoding).

<a id="en-access-session"></a>

#### Current Access Session

`access_session_id`: read the session manager’s memory wrapper first, reusing an existing wrapper even when its UUID is null. Only a missing wrapper reads `reg_prefs.access_session_id` and applies blank/UUID parsing before generation; generation persists before publishing the cache. Ordinary reads do not synchronize native state. Explicit reset separately follows bootstrap/record state. Reading the custom `state` store does not replace this lifecycle. See [session behavior](code.md#en).

<a id="en-android-id"></a>

#### Android ID Input

The raw input for `aid` is `android.provider.Settings.Secure.getString(context.getContentResolver(), android.provider.Settings.Secure.ANDROID_ID)`, also exposed by `AndroidFields.androidId(context)`. This read does not compute a digest.

<a id="en-device-name"></a>

#### Device Codename

`device_name`: `AndroidFields.deviceName()` reads `android.os.Build.DEVICE`, the build codename rather than `Build.MODEL`.

<a id="en-device-ram"></a>

#### Total Device Memory

`device_ram`: divide `getMemoryInfo().totalMem` by 1024³ and use `DecimalFormat("#.##")` with the actual formatting Locale. A missing service or the handled NullPointerException supplies -1 bytes before the same division/formatting, rather than directly sending `"-1"`; other errors are not broadly swallowed. `totalRamGiB()` demonstrates one read/transform, while the field owner can cache non-null text.

<a id="en-process-id"></a>

#### Application Process ID

`pid`: `AndroidFields.pid()` returns `android.os.Process.myPid()` as decimal text.

<a id="en-adb-setting"></a>

#### ADB Setting Input

`db`: `AndroidFields.adbEnabled(context)` uses `Settings.Global.getInt(context.getContentResolver(), Settings.Global.ADB_ENABLED, 0)`.

<a id="en-phone-permissions"></a>

#### Phone Permission Checks

`read_phone_permission_granted`: `AndroidFields.phonePermissionsGranted(context)` checks the required permissions using `Context.checkSelfPermission()`. Serialize its boolean result as `granted ? "1" : "0"`. It does not request permissions.

<a id="en-device-number"></a>

#### Device Number Input

`simnum`: an unmet permission gate returns null from the number reader and thus classification 0. Otherwise iterate active subscriptions using `SubscriptionInfo.getNumber()`, retaining the first non-null value, including empty text, before falling back to default telephony `getLine1Number()`. Switching to `getPhoneNumber()` on newer systems is a different selection rule. Subscription-enumeration exceptions propagate; only the telephony fallback catches Exception and yields null. `simnum(number)` checks Java length≥6 on the captured string without rereading the system.

<a id="en-sim-state"></a>

#### SIM State Input

`sim_state`: `AndroidFields.simState(context)` reads default-SIM state and gives -1 for an absent service. The slot-specific overload is an additional raw API: use a logical slot index rather than a subscription ID, without replacing default-SIM request acquisition. Neither result is `sim_type` classification.

<a id="en-operator-names"></a>

#### Operator Name Inputs

`sim_operator_name` / `network_operator_name`: this field path calls default telephony `getSimOperatorName()` / `getNetworkOperatorName()`. A null service/normal API result becomes empty text; exceptions propagate. `simOperator()` / `networkOperator()` do not substitute a service scoped to the default data subscription.

<a id="en-network-type"></a>

#### Network Classification

`network_radio_type`: `AndroidFields.networkRadioType(snapshot)` maps selected mobile/wifi/subtype inputs. `legacyNetworkSnapshot(context)` reads the legacy path: only type 0 is mobile and type 1 Wi-Fi; other types do not automatically map Ethernet to 1. With SDK≥31 and an initialized network manager, the newer path groups transport 1/3 as Wi-Fi class 1; other cases use capability 18 for classes 2/3, and class 3 has no mobile bit. That path requires the same manager’s callback, permissions, and main-thread subtype cache, rather than a replacement synchronous activeNetwork read. Null or neither mobile nor wifi maps to -1; unknown mobile subtype maps to 0. The 500 ms timer coalesces notifications rather than defining field validity.

<a id="en-airplane-mode"></a>

#### Airplane Mode Input

`airplane_mode_type`: `AndroidFields.airplaneMode(context)` reads `Settings.Global.AIRPLANE_MODE_ON` and normalizes it to 0/1.

<a id="en-signal-strength"></a>

#### Cellular Signal Input

`cellular_strength`: when the group gate allows collection, SDK≥28, a telephony service, and a non-null first `getSignalStrength()` are required before taking `getLevel()` from a second call. Missing support/service/first object gives 5. A null second result or permission/device-capability exception is not uniformly converted to 5. `signalLevel()` retains two non-atomic reads rather than saving and reusing the first object.

<a id="en-roaming-state"></a>

#### Roaming State Input

`roaming_type`: `AndroidFields.roaming(context)` calls `isNetworkRoaming()`. With a network snapshot, encode false/true as `"0"`/`"1"`, or `"2"` when the absent telephony service yields null. Omit the field without a snapshot; handle read exceptions separately.

<a id="en-recovery-record"></a>

#### Recovery Record Check

`hasinrc`: `state.hasRecoveryRecord()` checks `new java.io.File(context.getFilesDir(), "rc2").exists()`. It checks existence without reading or decrypting `id`.

<a id="en-package-material"></a>

#### Installation Material Inputs

For `_gi`, full/prefix digests and size use primary `sourceDir`, whereas the DEX digest reads `classes.dex` from `getPackageCodePath()`. The split list in `installedCodePaths()` is not one combined digest input. `dataDirectory()` and `nativeLibraryDirectory()` supply directories. `_icr` uses the actual initial Application’s PackageManager with GET_SIGNATURES (64), taking signatures[0] DER before SHA-1/standard Base64. `firstLegacyCertificate(initialApplication)` illustrates that read without substituting SigningInfo/history; the caller must supply the corresponding initial Application. See [_gi](security-fields/gi.md#en).

<a id="en-declared-permissions"></a>

#### Declared Permission Inputs

For `_gp`, read `requestedPermissions` with GET_PERMISSIONS (4096) on the same Application passed to collection. `declaredPermissions()` preserves a null array, empty names, and duplicates; it is not a granted-permission list. Later processing uses JNI modified UTF-8, unsigned sorting, and separator-free concatenation. Empty concatenation or specified API failure uses `default_permission`, unlike a cache miss with no Application, which can return null. A non-null result-slot hit skips PackageManager. See [_gp](security-fields/gp.md#en).

<a id="en-related-app-signatures"></a>

#### Related Application Certificates

`is_foa_fdid_app_installed`: for visible other packages, `relatedAppFirstCertificate(context, packageName)` selects the first certificate through the field’s SigningInfo/history or signatures branch. Then apply SHA-1/unpadded Base64url and related-app matching rules. This differs from `_gi._icr`’s fixed flags 64 strategy; reading DER alone is not the final boolean result.

<a id="en-uuid-encoding"></a>

#### UUID Byte Representation

`expid` / `access_session_id`: `FieldEncoding.uuidBytes(uuid)` encodes an existing UUID as 16 big-endian bytes using unpadded Base64url. It does not acquire or create the UUID.

<a id="en-signal-id-encoding"></a>

#### Signal Record ID Representation

`e_regid` encodes the existing ID as 4 big-endian bytes. `e_skey_id` encodes the selected prekey’s protobuf ID low 24 bits as 3 big-endian bytes, without extra rejection of high bits or negative Java int values. The SQL row ID selects a record rather than replacing its protobuf ID. Read all three `e_skey_*` from one snapshot, preserving the stored signature length; request H does not generate it.

<a id="en-aid-encoding"></a>

#### Android ID Digest Representation

`aid`: hash normal Android ID as JNI modified UTF-8 with SHA-256, then padded standard Base64. NUL is C0 80; supplementary characters encode each UTF-16 unit separately. `aidForPresentAndroidId()` only transforms non-null text; use `aidForSignalError()` for confirmed read-failure branches, without hiding arbitrary errors or missing integration inputs. Actual acquisition uses the current-thread Application’s resolver, separately from expid’s resolver slot, and the security owner caches the result. expid nameUUID still uses ordinary UTF-8, not this encoding.

<a id="en-advertising-id"></a>

#### Advertising Identifier Input

`advertising_id`: Google Play services `AdvertisingIdClient.getAdvertisingIdInfo(context)` supplies ID/tracking information off the main thread. The selected newer provider branch omits the field for LAT=true; the older direct-read branch does not read LAT and must not inherit that condition. Country/flow gates separately control acquisition. This call chain has neither automatic failure-to-other-branch fallback nor persistent advertising-ID caching.

<a id="en-push-token"></a>

#### Push Provider Token

`push_token`: configure Firebase, obtain the token with `FirebaseMessaging.getInstance().getToken()`, and handle refreshes through `onNewToken()`.

<a id="en-play-integrity"></a>

#### Play Integrity Service Input

`gpia` / `_gg`: when acquisition is enabled, use `StandardIntegrityManager`; an existing provider can skip prepare. Fresh generation shares one outcome code/optional token. requestHash is unpadded standard Base64 of the earlier stored connection public32, without prior SHA-256. Field wrapping reads a derived public key later and has independent IV/publication; a retained public-Map slot need not still match this gpia. See [service integration](#en-service-integration) and [Play-result lifetime](security-fields/gpia.md#en).

<a id="en-androidfields"></a>

### AndroidFields

`AndroidFields` reads inputs visible to the calling app. The default telephony service used by these fields differs from an explicitly subscription-scoped service. The slot-specific example is an additional raw API, not a replacement for default-SIM request acquisition. Number hints preserve subscription order and the first non-null value. An owner can cache a service object without caching its returned number, operator, or signal.

`networkRadioType()` only maps the protocol enum; use `legacyNetworkSnapshot()` only for the selected legacy path and the actual callback state for the newer path. `totalRamGiB()` performs one calculation without establishing the field owner’s text cache. `stat()` throws read errors; `_ge/_gs` collectors apply their own actual-result/failure rules. `isMonkey()` supplies only the `_ga.mu` API input, not `_ga.mp` or the complete `_ga`.

```java
// File: AndroidFields.java
package wiki.examples;

import android.Manifest;
import android.app.ActivityManager;
import android.content.Context;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.os.Build;
import android.provider.Settings;
import android.system.ErrnoException;
import android.system.Os;
import android.system.StructStat;
import android.telephony.SubscriptionInfo;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import java.text.DecimalFormat;
import java.util.List;
import java.util.Locale;

public final class AndroidFields {
    private AndroidFields() {}

    public static Locale appLocale(Context context) {
        android.os.LocaleList locales =
                context.getResources().getConfiguration().getLocales();
        Locale value = locales.isEmpty() ? Locale.getDefault() : locales.get(0);
        if (value != null) return value;
        Locale fallback = Locale.getDefault();
        return fallback != null ? fallback : Locale.US;
    }

    public static String language(Context context) {
        String value = appLocale(context).getLanguage();
        return value.matches("[a-z]{2,3}") ? value : "zz";
    }

    public static String localeCountry(Context context) {
        String value = appLocale(context).getCountry();
        return value.matches("[A-Z]{2}|[0-9]{3}") ? value : "ZZ";
    }

    public static String deviceName() { return Build.DEVICE; }
    public static String pid() {
        return Integer.toString(android.os.Process.myPid());
    }

    public static String totalRamGiB(Context context) {
        ActivityManager service = context.getSystemService(ActivityManager.class);
        long total = -1L;
        if (service != null) {
            ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();
            try {
                service.getMemoryInfo(info);
                total = info.totalMem;
            } catch (NullPointerException unavailable) {
                total = -1L;
            }
        }
        return new DecimalFormat("#.##").format(total / 1073741824.0);
    }

    public static String androidId(Context context) {
        return Settings.Secure.getString(context.getContentResolver(),
                Settings.Secure.ANDROID_ID);
    }

    public static int adbEnabled(Context context) {
        return Settings.Global.getInt(context.getContentResolver(),
                Settings.Global.ADB_ENABLED, 0);
    }

    public static int airplaneMode(Context context) {
        int value = Settings.Global.getInt(context.getContentResolver(),
                Settings.Global.AIRPLANE_MODE_ON, 0);
        return value == 0 ? 0 : 1;
    }

    public static boolean phonePermissionsGranted(Context context) {
        if (context.checkSelfPermission(Manifest.permission.READ_PHONE_STATE)
                != PackageManager.PERMISSION_GRANTED) return false;
        return Build.VERSION.SDK_INT < 30
                || context.checkSelfPermission(Manifest.permission.READ_PHONE_NUMBERS)
                == PackageManager.PERMISSION_GRANTED;
    }

    public static TelephonyManager defaultTelephony(Context context) {
        return context.getSystemService(TelephonyManager.class);
    }

    @SuppressWarnings("deprecation")
    public static String firstDeviceNumber(Context context) {
        if (!phonePermissionsGranted(context)) return null;
        SubscriptionManager subscriptions =
                context.getSystemService(SubscriptionManager.class);
        if (subscriptions == null) throw new NullPointerException("SubscriptionManager");
        List<SubscriptionInfo> active = subscriptions.getActiveSubscriptionInfoList();
        if (active != null) {
            for (SubscriptionInfo item : active) {
                String value = item.getNumber();
                if (value != null) return value;
            }
        }
        try {
            TelephonyManager service = defaultTelephony(context);
            return service == null ? null : service.getLine1Number();
        } catch (Exception unavailableFallback) {
            return null;
        }
    }

    public static String simnum(String number) {
        return number != null && number.length() >= 6 ? "1" : "0";
    }

    public static int simState(Context context) {
        TelephonyManager service = defaultTelephony(context);
        return service == null ? -1 : service.getSimState();
    }

    public static Integer simState(Context context, int slotIndex) {
        if (slotIndex < 0) {
            throw new IllegalArgumentException("Expected a logical SIM slot index");
        }
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState(slotIndex);
    }

    public static String simOperator(Context context) {
        TelephonyManager service = defaultTelephony(context);
        String value = service == null ? null : service.getSimOperatorName();
        return value == null ? "" : value;
    }

    public static String networkOperator(Context context) {
        TelephonyManager service = defaultTelephony(context);
        String value = service == null ? null : service.getNetworkOperatorName();
        return value == null ? "" : value;
    }

    public static int signalLevel(Context context) {
        TelephonyManager service = defaultTelephony(context);
        if (Build.VERSION.SDK_INT < 28 || service == null
                || service.getSignalStrength() == null) return 5;
        // Preserve the second read: it can change or fail independently.
        return service.getSignalStrength().getLevel();
    }

    public static Boolean roaming(Context context) {
        TelephonyManager service = defaultTelephony(context);
        return service == null ? null : service.isNetworkRoaming();
    }

    public static final class NetworkSnapshot {
        public final boolean mobile;
        public final boolean wifi;
        public final int subtype;
        public NetworkSnapshot(boolean mobile, boolean wifi, int subtype) {
            this.mobile = mobile;
            this.wifi = wifi;
            this.subtype = subtype;
        }
    }

    @SuppressWarnings("deprecation")
    public static NetworkSnapshot legacyNetworkSnapshot(Context context) {
        ConnectivityManager service = context.getSystemService(ConnectivityManager.class);
        android.net.NetworkInfo info = service == null ? null : service.getActiveNetworkInfo();
        return info == null ? null : new NetworkSnapshot(
                info.getType() == 0, info.getType() == 1, info.getSubtype());
    }

    public static int networkRadioType(NetworkSnapshot snapshot) {
        if (snapshot == null) return -1;
        if (!snapshot.mobile) return snapshot.wifi ? 1 : -1;
        return radioCode(snapshot.subtype);
    }

    public static int radioCode(int androidType) {
        switch (androidType) {
            case TelephonyManager.NETWORK_TYPE_GPRS: return 104;
            case TelephonyManager.NETWORK_TYPE_EDGE: return 100;
            case TelephonyManager.NETWORK_TYPE_UMTS: return 102;
            case TelephonyManager.NETWORK_TYPE_CDMA: return 108;
            case TelephonyManager.NETWORK_TYPE_EVDO_0:
            case TelephonyManager.NETWORK_TYPE_EVDO_A:
            case TelephonyManager.NETWORK_TYPE_EVDO_B: return 103;
            case TelephonyManager.NETWORK_TYPE_1xRTT: return 109;
            case TelephonyManager.NETWORK_TYPE_HSDPA: return 105;
            case TelephonyManager.NETWORK_TYPE_HSUPA: return 106;
            case TelephonyManager.NETWORK_TYPE_HSPA: return 107;
            case TelephonyManager.NETWORK_TYPE_IDEN: return 101;
            case TelephonyManager.NETWORK_TYPE_LTE: return 111;
            case TelephonyManager.NETWORK_TYPE_EHRPD: return 110;
            case TelephonyManager.NETWORK_TYPE_HSPAP: return 112;
            case TelephonyManager.NETWORK_TYPE_TD_SCDMA: return 116;
            case TelephonyManager.NETWORK_TYPE_IWLAN: return 115;
            case TelephonyManager.NETWORK_TYPE_NR: return 113;
            default: return 0;
        }
    }

    public static boolean isMonkey() { return ActivityManager.isUserAMonkey(); }
    public static StructStat stat(String path) throws ErrnoException {
        return Os.stat(path);
    }
}
```

Android ID is scoped to signing key, user, and device. [ANDROID_ID](https://developer.android.com/reference/android/provider/Settings.Secure#ANDROID_ID)

Phone-number availability is not guaranteed. Telephony/signal calls require specific permissions and device features; unsupported devices can throw UnsupportedOperationException even when the service object exists. [SubscriptionManager](https://developer.android.com/reference/android/telephony/SubscriptionManager#getPhoneNumber(int)) · [TelephonyManager](https://developer.android.com/reference/android/telephony/TelephonyManager)

<a id="en-registrationstate"></a>

### RegistrationState

This demonstrates state storage in your own app. The preference filename is defined by the example and keys are caller-defined. It is not WhatsApp storage or a universal field getter. Reading a stored field presupposes that its actual producer saved the correct value.

Initialize missing identifiers through their own lifecycle and persist them; do not regenerate every ID for every request. Session resets differ from installation-ID initialization. Backup and Signal material require their respective modules. The example initialization has no concurrency lock. The actual perf_device_id recheck-before-write/read-after-write is not atomic CAS either; added serialization is a caller design choice rather than a field guarantee. This class uses system SharedPreferences, whose wrong-type reads can throw ClassCastException. It does not model custom Light defaults for missing keys/Map null/wrong types. Session wrappers, source digests, and security-result slots also are not all cleared by `clear/apply`.

```java
// File: RegistrationState.java
package wiki.examples;

import android.content.Context;
import android.content.SharedPreferences;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

public final class RegistrationState {
    private final Context context;
    private final SharedPreferences preferences;

    public RegistrationState(Context context) {
        this.context = context.getApplicationContext();
        preferences = this.context.getSharedPreferences(
                "wiki_example_registration_state", Context.MODE_PRIVATE);
    }

    public String readString(String key) {
        return preferences.getString(key, null);
    }

    public void saveString(String key, String value) {
        preferences.edit().putString(key, value).apply();
    }

    public String getOrCreateExamplePerfUuid() {
        String existing = readString("example_perf_uuid");
        if (existing != null) return existing;
        String androidId = AndroidFields.androidId(context);
        boolean blank = androidId == null || androidId.isEmpty();
        if (!blank) {
            blank = true;
            for (int i = 0; i < androidId.length(); i++) {
                char value = androidId.charAt(i);
                if (!Character.isWhitespace(value) && !Character.isSpaceChar(value)) {
                    blank = false;
                    break;
                }
            }
        }
        UUID created = blank ? UUID.randomUUID()
                : UUID.nameUUIDFromBytes(androidId.getBytes(StandardCharsets.UTF_8));
        if (readString("example_perf_uuid") == null) {
            saveString("example_perf_uuid", created.toString());
        }
        return readString("example_perf_uuid");
    }

    public int readCount(String key) {
        return preferences.getInt(key, 0);
    }

    public long readDurationMillis(String key) {
        return preferences.getLong(key, 0L);
    }

    public boolean hasRecoveryRecord() {
        return new File(context.getFilesDir(), "rc2").exists();
    }
}
```

`hasRecoveryRecord()` only checks the calling app's own file. Existence is not decryption or validity verification. [App-specific storage](https://developer.android.com/training/data-storage/app-specific)

<a id="en-packageinputs"></a>

### PackageInputs

These methods read raw package information used by integrity and installation-related fields. Declared permissions differ from runtime grants. An installer package name alone is not complete installation-attribution JSON.

`firstLegacyCertificate()` reads the flags 64 first certificate from the supplied actual initial Application. On API≥28, `clientFirstCertificate()` selects history only for non-null SigningInfo without multiple signers, otherwise signatures; it converts only the selected array’s first entry. Related-app first-entry selection can use the latter rule but still needs SHA-1/matching. Neither is HTTP Authorization’s reversed KeyStore chain. Other-app queries remain subject to package visibility.

```java
// File: PackageInputs.java
package wiki.examples;

import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.pm.Signature;
import android.content.pm.SigningInfo;
import android.os.Build;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class PackageInputs {
    private PackageInputs() {}

    @SuppressWarnings("deprecation")
    public static String[] declaredPermissions(Context context)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                context.getPackageName(), PackageManager.GET_PERMISSIONS);
        return info.requestedPermissions == null
                ? null : info.requestedPermissions.clone();
    }

    @SuppressWarnings("deprecation")
    public static byte[] clientFirstCertificate(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                packageName, PackageManager.GET_SIGNING_CERTIFICATES);
        SigningInfo signing = info.signingInfo;
        Signature[] certificates = info.signatures;
        if (signing != null && !signing.hasMultipleSigners()) {
            certificates = signing.getSigningCertificateHistory();
        }
        return certificates == null || certificates.length == 0
                ? null : certificates[0].toByteArray();
    }

    @SuppressWarnings("deprecation")
    public static byte[] firstLegacyCertificate(Context initialApplication)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = initialApplication.getPackageManager().getPackageInfo(
                initialApplication.getPackageName(), PackageManager.GET_SIGNATURES);
        Signature[] signatures = info.signatures;
        return signatures == null || signatures.length == 0
                ? null : signatures[0].toByteArray();
    }

    public static byte[] relatedAppFirstCertificate(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        return clientFirstCertificate(context, packageName);
    }

    public static String primaryPackageCodePath(Context context) {
        return context.getPackageCodePath();
    }

    public static String primarySourcePath(Context context) {
        return context.getApplicationInfo().sourceDir;
    }

    public static List<String> installedCodePaths(Context context) {
        ApplicationInfo info = context.getApplicationInfo();
        List<String> result = new ArrayList<>();
        result.add(info.sourceDir);
        if (info.splitSourceDirs != null) {
            Collections.addAll(result, info.splitSourceDirs);
        }
        return result;
    }

    public static String dataDirectory(Context context) {
        return context.getApplicationInfo().dataDir;
    }

    public static String nativeLibraryDirectory(Context context) {
        return context.getApplicationInfo().nativeLibraryDir;
    }

    @SuppressWarnings("deprecation")
    public static String installerPackage(Context context)
            throws PackageManager.NameNotFoundException {
        PackageManager manager = context.getPackageManager();
        String name = context.getPackageName();
        return Build.VERSION.SDK_INT >= 30
                ? manager.getInstallSourceInfo(name).getInstallingPackageName()
                : manager.getInstallerPackageName(name);
    }
}
```

API reference: [PackageInfo](https://developer.android.com/reference/android/content/pm/PackageInfo) · [SigningInfo](https://developer.android.com/reference/android/content/pm/SigningInfo) · [Package visibility](https://developer.android.com/training/package-visibility)

<a id="en-fieldencoding"></a>

### FieldEncoding

These helpers encode material already held by the caller. They do not generate identity keys, Google tokens, or security verdicts. UUID text differs from UUID-byte encoding; registration and signed-prekey IDs have different widths.

`aidForPresentAndroidId()` uses modified UTF-8; the expid example in `RegistrationState` uses ordinary UTF-8. `signedPrekeyId()` takes only low 24 bits without validating the full record. `percentEncodeBytes()` preserves only `-._~` and alphanumerics, unlike ordinary Java URLEncoder text (space as `+`, literal `*`, escaped `~`). The Kotlin builder must track preencoded identity to avoid a second escape; see [request packaging](request-encryption.md#en) for native dictionary enumeration/serialization.

```java
// File: FieldEncoding.java
package wiki.examples;

import android.util.Base64;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.UUID;

public final class FieldEncoding {
    private FieldEncoding() {}

    public static String base64url(byte[] value) {
        return Base64.encodeToString(value,
                Base64.URL_SAFE | Base64.NO_PADDING | Base64.NO_WRAP);
    }

    public static String uuidBytes(UUID value) {
        return base64url(ByteBuffer.allocate(16).order(ByteOrder.BIG_ENDIAN)
                .putLong(value.getMostSignificantBits())
                .putLong(value.getLeastSignificantBits()).array());
    }

    public static String registrationId(int value) {
        return base64url(ByteBuffer.allocate(4).order(ByteOrder.BIG_ENDIAN)
                .putInt(value).array());
    }

    public static String signedPrekeyId(int value) {
        byte[] bytes = {(byte) (value >>> 16),
                (byte) (value >>> 8), (byte) value};
        return base64url(bytes);
    }

    public static String aidForPresentAndroidId(String androidId)
            throws NoSuchAlgorithmException {
        if (androidId == null) {
            throw new IllegalArgumentException("Android ID is unavailable");
        }
        byte[] digest = MessageDigest.getInstance("SHA-256").digest(
                modifiedUtf8(androidId));
        return Base64.encodeToString(digest, Base64.NO_WRAP);
    }

    public static String aidForSignalError() throws NoSuchAlgorithmException {
        return aidForPresentAndroidId("signal_error");
    }

    public static byte[] modifiedUtf8(String value) {
        java.io.ByteArrayOutputStream out = new java.io.ByteArrayOutputStream();
        for (int i = 0; i < value.length(); i++) {
            int c = value.charAt(i);
            if (c >= 1 && c <= 0x7F) out.write(c);
            else if (c <= 0x7FF) {
                out.write(0xC0 | (c >>> 6));
                out.write(0x80 | (c & 0x3F));
            } else {
                out.write(0xE0 | (c >>> 12));
                out.write(0x80 | ((c >>> 6) & 0x3F));
                out.write(0x80 | (c & 0x3F));
            }
        }
        return out.toByteArray();
    }

    public static String percentEncodeBytes(byte[] value) {
        char[] hex = "0123456789ABCDEF".toCharArray();
        StringBuilder out = new StringBuilder();
        for (byte item : value) {
            int b = item & 0xFF;
            boolean unreserved = (b >= 'A' && b <= 'Z')
                    || (b >= 'a' && b <= 'z') || (b >= '0' && b <= '9')
                    || b == '-' || b == '.' || b == '_' || b == '~';
            if (unreserved) out.append((char) b);
            else out.append('%').append(hex[b >>> 4]).append(hex[b & 15]);
        }
        return out.toString();
    }
}
```

API reference: [Base64](https://developer.android.com/reference/android/util/Base64) · [ByteBuffer](https://developer.android.com/reference/java/nio/ByteBuffer)

<a id="en-service-integration"></a>

### External Service Integration

| Field | Integration Entry | Requirement |
| --- | --- | --- |
| advertising_id | Call `AdvertisingIdClient.getAdvertisingIdInfo(context)`, then read `getId()` and `isLimitAdTrackingEnabled()`. | Add the advertising-ID dependency and required permissions; call off the main thread and handle service availability/tracking preference. |
| push_token | Use `FirebaseMessaging.getInstance().getToken()` and the `onNewToken()` callback. | Configure Firebase, then asynchronously store/update the actual token. |
| gpia / _gg | `StandardIntegrityManager`: prepare a provider, then request an outcome | requestHash is unpadded standard Base64 of the earlier stored connection public32 read, without prior SHA-256. Fresh generation shares one outcome; retained older Map slots follow their own lifetime. |
| recaptcha | Use the real result and callback from the reCAPTCHA SDK. | Use the app's service configuration and actual token/failure outcome. |
| authkey / e_ident / e_regid / e_skey_* / e_pq_last_resort_* | Use the application's connection-authentication and Signal session/key modules. | Android KeyStore is not a universal getter for these protocol records; public material must match the existing identity/record. |
| id / backup_token / foa_backup_token / cred_token | Respective recovery, backup, or credential modules | id reads rc2. exist selects number-bound source1 and saves before sending; code directly decodes reg_prefs.token_used_during_reg, reusing nonempty bytes and generating only for zero length. foa and cred retain their own conditional sources. |
| client_metrics / offline_ab / feo2_query_status / language_selector_* / mistyped | Use application business events and persisted state. | Accumulate, persist, or consume at actual events rather than inventing state during serialization. |

Google/Firebase results belong to the calling application, service project, and action; an example application's token is not another client's credential.

API reference: [AdvertisingIdClient](https://developers.google.com/android/reference/com/google/android/gms/ads/identifier/AdvertisingIdClient) · [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/android/get-started) · [Play Integrity](https://developer.android.com/google/play/integrity/standard)

<a id="en-input-boundaries"></a>

### Field Assembly and Topic Boundaries

Raw API values, derived fields, cache slots, and final requests are different objects. `RegistrationState` does not take over the application’s actual lifecycle. Four telephony-environment fields share a collection gate, while `db` has its own. Handle empty text, null, omission, 0, and false separately rather than inventing missing service outcomes.

| Topic | Connection to These Helpers |
| --- | --- |
| [Security overview](security-fields/README.md#en) | gpia returns through a separate callback; the other six security fields and aid have security-owner slots. Owner replacement, context dictionaries, globals, and built-Map references have distinct lifetimes. |
| [gpia](security-fields/gpia.md#en) | A non-null provider can bypass prepare; this layer has no in-flight coalescing or fixed TTL. Google-stage timeout excludes later metadata/CBC work, and token/provider caching differ. |
| [_gg](security-fields/gg.md#en) | Fresh generation projects the gpia outcome code/optional token into _ic/_it with independent IV/publication. A public Map can read an older slot; disabling acquisition does not automatically clear it. |
| [_gi](security-fields/gi.md#en) | Full-digest key 2, DEX key 0, library _is/_p, and final ciphertext slots differ. The first non-null _is publication can persist, and paths publish before opening. Keep primary/split paths and certificate API selection distinct. |
| [_gp](security-fields/gp.md#en) | Modified UTF-8, unsigned sorting, retained duplicates, and empty-concatenation defaults all affect the digest. The raw array is not a finished field, and error digests can be cached. |
| [_gs](security-fields/gs.md#en) | Thirteen stats, two timevals, libc rand, prepared, and encryption DRBG have separate inputs. Prepared starts at 1337 and actual Prepare writes 4242. Generic stat/Java randomness is not the complete collector or 19-value CSV. |
| [_ge](security-fields/ge.md#en) | sv maps to vmw_pvscsi and sb to vboxsf; configuration controls su presence. A cache hit can skip configuration/stat, and a failed check does not establish global path absence. |
| [_ga](security-fields/ga.md#en) | The mu API differs from mp process scanning. Only bi has CBC; boot failure hashes failed_sha material. The creation-time public snapshot and complete cached result do not automatically refresh on rotation. |
| [Request encryption](request-encryption.md#en) | Serialize ordinary text and preencoded bytes separately; inner JSON first becomes a string field. ENC uses fresh ephemeral X25519/GCM. Normal encrypted H signs ENC, while a Kotlin plaintext fallback can sign the query; pair the certificate chain separately. |
| [Sending](sending-requests.md#en) | Final merging/refresh precedes form/body creation. Kotlin retries can reuse parameters and rebuild ENC/H without rerunning business counters. Parsing success differs from business success; six WAMsys application headers do not define every route. |

Keep each article’s request-specific backup reads, attempt counters, and correction rules. code reads saved mistyped text; reason is captured before asynchronous preparation, preserving empty text. Standard-page hasav is Retriever→2, otherwise granted RECEIVE_SMS→1, other normal results→0; a specific fallback can also pass 0 directly. Reading FEO2 diagnostics does not query or clear capabilities, and reading metrics/diagnostics does not rerun their producers.

Encoded values still require final serialization by the chosen transport. Field CBC, ENC, H, and Authorization have different inputs/lifetimes; these four helper classes do not construct the final HTTP body.

[exist parameters](exist.md#en) · [code parameters](code.md#en)
