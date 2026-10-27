# Android 字段获取代码 · Android Input Examples

**Telegram Contact: [@ttjjda](https://t.me/ttjjda)**

[注册目录](README.md#zh) · [中文](#zh) · [English](#en) · [字段索引](#field-lookup) · [Field Lookup](#en-field-lookup)

<a id="android-input-examples--android-字段获取代码"></a>

<a id="android-examples"></a>

<a id="zh"></a>

## 中文

本页提供四组 Java 辅助类：系统输入读取、应用状态存储、包信息读取与字段编码，使用 Android 公开 API 和 Java 标准库。编译时需要 API 33 或更高的 `android.jar`，运行基线为 API 28；较新接口通过 API 级别判断后调用。

以下权限声明放在 `AndroidManifest.xml` 的 `<manifest>` 下、`<application>` 外。电话权限还需按功能在运行时申请；网络状态读取需要 ACCESS_NETWORK_STATE。仅声明权限不等于用户已经授予。代码不会自动申请权限，也不会把 SecurityException 改写成正常状态。

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

`lg` / `lc`：从应用 Context 的配置读取 Locale，再取语言和地区。

```java
android.os.LocaleList locales = context.getResources().getConfiguration().getLocales();
java.util.Locale locale = locales.isEmpty() ? java.util.Locale.getDefault() : locales.get(0);
String language = locale.getLanguage();
String country = locale.getCountry();
String lg = language.matches("[a-z]{2,3}") ? language : "zz";
String lc = country.matches("[A-Z]{2}|[0-9]{3}") ? country : "ZZ";
```

<a id="persistent-identifiers"></a>

#### 持久标识

`fdid`：读取应用已经保存的标识；`state.readString("fdid")`。键名由应用定义，这不是读取其它应用 ID 的接口。

<a id="performance-identifier"></a>

#### 性能设备标识

`expid`：本示例的 `state.getOrCreateExamplePerfUuid()` 演示读取或初始化自己应用的性能 UUID；字段字节表示见 [UUID 字节编码](#uuid-encoding)。

<a id="access-session"></a>

#### 访问会话

`access_session_id`：由会话管理器保存和重置；可通过 `state.readString("access_session_id")` 读取应用已保存的会话值，再做 UUID 字节编码。

<a id="android-id"></a>

#### Android ID 读取

`aid` 的原始输入：`android.provider.Settings.Secure.getString(context.getContentResolver(), android.provider.Settings.Secure.ANDROID_ID)`。这一步尚未计算摘要。

<a id="device-name"></a>

#### 设备代号

`device_name`：`android.os.Build.DEVICE`。对应 `AndroidFields.deviceName()`，不是 `Build.MODEL`。

<a id="device-ram"></a>

#### 设备总内存

`device_ram`：`ActivityManager.MemoryInfo.totalMem` 经 `getMemoryInfo()` 填充后除以 1024³。`AndroidFields.totalRamGiB(context)` 完成读取与格式化。

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

`simnum`：先用 `AndroidFields.firstDeviceNumber(context)` 取得候选号码，再以 `AndroidFields.simnum(number)` 判断长度。前者使用公开 SubscriptionManager / TelephonyManager 接口；后者不再次读取系统。读取器保留首个非 null 返回值（包括空串）；API 33 的 `getPhoneNumber()` 在号码不可用时返回空串，此时不会继续选择后面的订阅。

<a id="sim-state"></a>

#### SIM 状态

`sim_state`：`AndroidFields.simState(context)` 读取默认 SIM 状态；指定卡槽用 `simState(context, slotIndex)`。`slotIndex` 是逻辑卡槽编号，可取自 `SubscriptionInfo.getSimSlotIndex()`，不能直接传订阅 ID。该结果不是 `sim_type` 分类。

<a id="operator-names"></a>

#### 运营商名称

`sim_operator_name` / `network_operator_name`：所选订阅的 `getSimOperatorName()` / `getNetworkOperatorName()`；对应 `AndroidFields.simOperator(context)` / `networkOperator(context)`。

<a id="network-type"></a>

#### 网络分类

`network_radio_type`：`AndroidFields.networkRadioType(context)` 读取活动网络与电话制式，再由 `radioCode()` 映射协议编号。它不是直接返回 Android 的 `NETWORK_TYPE_*` 常量。

<a id="airplane-mode"></a>

#### 飞行模式

`airplane_mode_type`：`Settings.Global.AIRPLANE_MODE_ON`；对应 `AndroidFields.airplaneMode(context)`，返回归一化的 0/1。

<a id="signal-strength"></a>

#### 蜂窝信号等级

`cellular_strength`：所选订阅的 `TelephonyManager.getSignalStrength()`，再取 `SignalStrength.getLevel()`；对应 `AndroidFields.signalLevel(context)`。返回值可为 null；协议字段的不可用兜底为 `5`，不能把 null 自动写成 `0`。

<a id="roaming-state"></a>

#### 漫游状态

`roaming_type` 的原始输入：`AndroidFields.roaming(context)` 调用 `isNetworkRoaming()`。有网络快照时，false/true 分别编码为 `"0"`/`"1"`，电话服务对象不存在返回 null 时编码为 `"2"`；无快照则省略字段，读取异常另行处理。

<a id="recovery-record"></a>

#### 恢复文件存在性

`hasinrc`：`new java.io.File(context.getFilesDir(), "rc2").exists()`；对应 `state.hasRecoveryRecord()`。这里只检查存在性，不读取或解密 `id`。

<a id="package-material"></a>

#### 安装包原始输入

`_gi` 的原始输入：`PackageInputs.installedCodePaths(context)`、`dataDirectory(context)`、`nativeLibraryDirectory(context)` 和 `signingCertificates(context, context.getPackageName())`。这些提供路径与证书，不直接生成 `_gi`。

<a id="declared-permissions"></a>

#### 声明权限集合

`_gp` 的原始输入：`PackageInputs.declaredPermissions(context)` 使用 `PackageManager.GET_PERMISSIONS` 读取 `PackageInfo.requestedPermissions`；摘要计算在后续安全字段专题展开。

<a id="related-app-signatures"></a>

#### 关联应用签名

`is_foa_fdid_app_installed`：用 `PackageInputs.signingCertificates(context, packageName)` 获取可见应用证书，再按关联应用的匹配规则判断。这个方法只提供证书，不能单独代表最终布尔结果。

<a id="uuid-encoding"></a>

#### UUID 字节编码

`expid` / `access_session_id` 的表示转换：`FieldEncoding.uuidBytes(uuid)` 把已有 UUID 转成大端 16 字节并做 Base64url。它不获取或创建 UUID。

<a id="signal-id-encoding"></a>

#### Signal 编号编码

`e_regid` / `e_skey_id`：`FieldEncoding.registrationId(id)` / `signedPrekeyId(id)` 分别编码 4 字节和 3 字节编号。输入编号来自已有密钥记录，不由这些方法生成。

<a id="aid-encoding"></a>

#### Android ID 摘要编码

`aid`：先读取 [Android ID 读取](#android-id)，再调用 `FieldEncoding.aidForPresentAndroidId(androidId)`，得到 SHA-256 的标准 Base64 文本。

<a id="advertising-id"></a>

#### 广告标识

`advertising_id`：Google Play services 的 `AdvertisingIdClient.getAdvertisingIdInfo(context)`，读取 ID 与追踪限制；需接入 SDK，并在工作线程调用。

<a id="push-token"></a>

#### 推送令牌

`push_token`：FCM 的 `FirebaseMessaging.getInstance().getToken()` 与 `onNewToken()` 回调；需先完成应用的 Firebase 配置。

<a id="play-integrity"></a>

#### Play Integrity 服务输入

`gpia` / `_gg` 的服务输入：使用 `StandardIntegrityManager` 准备 provider 后请求 token，同一次结果供两个字段使用。完整接入条件见 [服务接入](#service-integration)。

<a id="android-system"></a>

### AndroidFields

`AndroidFields` 获取调用应用可访问的系统输入。多 SIM 的电话网络读取使用默认数据订阅；SIM 状态单独区分默认卡与指定逻辑卡槽，号码提示遍历活动订阅。订阅、权限和 API 不同可能返回不同号码，不能假设不同入口必然给出相同字符串。返回 null 表示该采集入口没有值，不是直接写入请求的协议兜底。

`networkRadioType()` 演示当前活动网络的分类；VPN、并行网络和异步回调需要调用方选择对应快照。`totalRamGiB()` 使用当前数字格式环境；请求所需的小数格式应与字段约定一致。`stat()` 保留检查异常，不能把权限不足伪装成文件不存在。

```java
// File: AndroidFields.java
package wiki.examples;

import android.Manifest;
import android.app.ActivityManager;
import android.content.Context;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.os.Build;
import android.provider.Settings;
import android.system.ErrnoException;
import android.system.Os;
import android.system.StructStat;
import android.telephony.SignalStrength;
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
        return locales.isEmpty() ? Locale.getDefault() : locales.get(0);
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
        if (service == null) return null;
        ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();
        service.getMemoryInfo(info);
        return new DecimalFormat("#.##").format(info.totalMem / 1073741824.0);
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

    public static TelephonyManager dataTelephony(Context context) {
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        if (service == null) return null;
        int id = SubscriptionManager.getDefaultDataSubscriptionId();
        return id < 0 ? service : service.createForSubscriptionId(id);
    }

    @SuppressWarnings("deprecation")
    public static String firstDeviceNumber(Context context) {
        if (!phonePermissionsGranted(context)) {
            throw new SecurityException("Phone permissions are not granted");
        }
        SubscriptionManager subscriptions =
                context.getSystemService(SubscriptionManager.class);
        List<SubscriptionInfo> active = subscriptions == null ? null
                : subscriptions.getActiveSubscriptionInfoList();
        if (active != null) {
            for (SubscriptionInfo item : active) {
                String value = Build.VERSION.SDK_INT >= 33
                        ? subscriptions.getPhoneNumber(item.getSubscriptionId())
                        : item.getNumber();
                if (value != null) return value;
            }
        }
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getLine1Number();
    }

    public static String simnum(String number) {
        return number != null && number.length() >= 6 ? "1" : "0";
    }

    public static Integer simState(Context context) {
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState();
    }

    public static Integer simState(Context context, int slotIndex) {
        if (slotIndex < 0) {
            throw new IllegalArgumentException("Expected a logical SIM slot index");
        }
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState(slotIndex);
    }

    public static String simOperator(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getSimOperatorName();
    }

    public static String networkOperator(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getNetworkOperatorName();
    }

    public static Integer signalLevel(Context context) {
        TelephonyManager service = dataTelephony(context);
        if (service == null) return null;
        SignalStrength strength = service.getSignalStrength();
        return strength == null ? null : strength.getLevel();
    }

    public static Boolean roaming(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.isNetworkRoaming();
    }

    public static Integer networkRadioType(Context context) {
        ConnectivityManager service =
                context.getSystemService(ConnectivityManager.class);
        if (service == null) return null;
        Network network = service.getActiveNetwork();
        if (network == null) return -1;
        NetworkCapabilities caps = service.getNetworkCapabilities(network);
        if (caps == null) return null;
        if (caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)
                || caps.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET)) return 1;
        if (!caps.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)) return -1;
        TelephonyManager phone = dataTelephony(context);
        return phone == null ? null : radioCode(phone.getDataNetworkType());
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

持久标识缺失时才按自己的初始化流程创建并保存；不要在每次请求里为所有 ID 重新生成 UUID。访问会话重置与安装标识初始化也应分别处理。备份和 Signal 材料由其各自模块保存，不能从几个偏好键凭空生成。示例初始化方法没有并发互斥；应用需在自己的状态管理层串行执行初始化，避免并发请求各自生成不同值。

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

证书数组可能包含签名轮换历史或多个当前签名者；后续字段必须按各自规则选择、排序与处理，不能默认任意第一张证书都可互换。查询其它应用还受包可见性限制。

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
                ? new String[0] : info.requestedPermissions.clone();
    }

    @SuppressWarnings("deprecation")
    public static byte[][] signingCertificates(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                packageName, PackageManager.GET_SIGNING_CERTIFICATES);
        SigningInfo signing = info.signingInfo;
        if (signing == null) return new byte[0][];
        Signature[] certificates = signing.hasMultipleSigners()
                ? signing.getApkContentsSigners()
                : signing.getSigningCertificateHistory();
        if (certificates == null) return new byte[0][];
        byte[][] result = new byte[certificates.length][];
        for (int i = 0; i < certificates.length; i++) {
            result[i] = certificates[i].toByteArray();
        }
        return result;
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

`aidForPresentAndroidId()` 只处理非 null Android ID；读取失败时应进入独立错误处理。`percentEncodeBytes()` 输出已经百分号编码的值，后续组表单不能再对它重复编码。

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
        if (value < 0 || value > 0xFFFFFF) {
            throw new IllegalArgumentException("Expected an unsigned 24-bit ID");
        }
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
                androidId.getBytes(StandardCharsets.UTF_8));
        return Base64.encodeToString(digest, Base64.NO_WRAP);
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
| gpia / _gg | Play Integrity 的 `StandardIntegrityManager`，先准备 provider，再请求 token | 需要应用自己的 Play/Cloud 配置与 requestHash；同一次真实结果供两个字段使用。 |
| recaptcha | reCAPTCHA SDK 的真实校验结果与回调 | 使用应用自己的服务配置，记录 token 或失败结果；不以随机文本代替。 |
| authkey / e_ident / e_regid / e_skey_* / e_pq_last_resort_* | 应用自己的连接认证与 Signal 会话/密钥模块 | Android KeyStore 不是这些协议记录的通用查询器；公开材料必须与已有身份及记录配对。 |
| id / backup_token / foa_backup_token / cred_token | 对应恢复、备份或凭据模块 | 读取正确账户及号码对应的材料，并执行其自身解析/解密与生命周期。 |
| client_metrics / offline_ab / feo2_query_status / language_selector_* / mistyped | 应用的业务事件与持久状态 | 由实际事件累计、写入或消费；不在序列化时编造交互记录。 |

Google/Firebase 返回值属于调用应用、服务项目和对应操作，示例应用取到的 token 不能直接当成另一个客户端的凭据。

实际接口说明：[AdvertisingIdClient](https://developers.google.com/android/reference/com/google/android/gms/ads/identifier/AdvertisingIdClient) · [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/android/get-started) · [Play Integrity](https://developer.android.com/google/play/integrity/standard)

[返回参数说明](exist.md#zh)

<a id="en"></a>

## English

Four Java helper classes cover system inputs, application state, package information, and field encoding using public Android APIs and Java standard libraries. Compile against API 33 or newer; the runtime baseline is API 28, with guards for newer calls.

Place these declarations directly under `<manifest>`, outside `<application>`, in `AndroidManifest.xml`. Request the phone permissions required by your feature at runtime; ACCESS_NETWORK_STATE is needed for network reads. The code does not request permissions or convert SecurityException into a normal result.

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

`lg` / `lc`: read the application configuration locale, then apply the language/country format checks shown below.

```java
android.os.LocaleList locales = context.getResources().getConfiguration().getLocales();
java.util.Locale locale = locales.isEmpty() ? java.util.Locale.getDefault() : locales.get(0);
String language = locale.getLanguage();
String country = locale.getCountry();
String lg = language.matches("[a-z]{2,3}") ? language : "zz";
String lc = country.matches("[A-Z]{2}|[0-9]{3}") ? country : "ZZ";
```

<a id="en-persistent-identifiers"></a>

#### Application Persistent Identifiers

`fdid`: `state.readString("fdid")` reads an identifier already stored by the calling app. Keys are app-defined; this is not a cross-app identifier getter.

<a id="en-performance-identifier"></a>

#### Performance Device Identifier

`expid`: `state.getOrCreateExamplePerfUuid()` reads or initializes the example app's performance UUID. Encode the result with [UUID encoding](#en-uuid-encoding).

<a id="en-access-session"></a>

#### Current Access Session

`access_session_id`: read the value maintained by the session manager with `state.readString("access_session_id")`, then encode its UUID bytes. Session resets determine when this stored value changes.

<a id="en-android-id"></a>

#### Android ID Input

The raw input for `aid` is `android.provider.Settings.Secure.getString(context.getContentResolver(), android.provider.Settings.Secure.ANDROID_ID)`, also exposed by `AndroidFields.androidId(context)`. This read does not compute a digest.

<a id="en-device-name"></a>

#### Device Codename

`device_name`: `AndroidFields.deviceName()` reads `android.os.Build.DEVICE`, the build codename rather than `Build.MODEL`.

<a id="en-device-ram"></a>

#### Total Device Memory

`device_ram`: `AndroidFields.totalRamGiB(context)` fills `ActivityManager.MemoryInfo` using `getMemoryInfo()`, divides `totalMem` by 1024³, and formats GiB text.

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

`simnum`: call `AndroidFields.firstDeviceNumber(context)` once, then classify that captured string with `AndroidFields.simnum(number)`. The reader keeps the first non-null result, including an empty string. On API 33, `SubscriptionManager.getPhoneNumber()` returns an empty string when unavailable, so this reader stops before later subscriptions.

<a id="en-sim-state"></a>

#### SIM State Input

`sim_state`: `AndroidFields.simState(context)` reads the default SIM state. Use `simState(context, slotIndex)` for a logical slot, obtaining the index from `SubscriptionInfo.getSimSlotIndex()`; a subscription ID is not a slot index. This is separate from `sim_type` classification.

<a id="en-operator-names"></a>

#### Operator Name Inputs

`sim_operator_name` / `network_operator_name`: `AndroidFields.simOperator(context)` / `networkOperator(context)` call `getSimOperatorName()` / `getNetworkOperatorName()` on the selected subscription service.

<a id="en-network-type"></a>

#### Network Classification

`network_radio_type`: `AndroidFields.networkRadioType(context)` reads transport and radio inputs, then `radioCode()` maps them to the protocol enum. It does not return a raw Android `NETWORK_TYPE_*` value.

<a id="en-airplane-mode"></a>

#### Airplane Mode Input

`airplane_mode_type`: `AndroidFields.airplaneMode(context)` reads `Settings.Global.AIRPLANE_MODE_ON` and normalizes it to 0/1.

<a id="en-signal-strength"></a>

#### Cellular Signal Input

`cellular_strength`: `AndroidFields.signalLevel(context)` reads the selected subscription's `TelephonyManager.getSignalStrength()` and calls `SignalStrength.getLevel()`. A null result is unavailable; the field fallback is `5`, not an automatic `0`.

<a id="en-roaming-state"></a>

#### Roaming State Input

`roaming_type`: `AndroidFields.roaming(context)` calls `isNetworkRoaming()`. With a network snapshot, encode false/true as `"0"`/`"1"`, or `"2"` when the absent telephony service yields null. Omit the field without a snapshot; handle read exceptions separately.

<a id="en-recovery-record"></a>

#### Recovery Record Check

`hasinrc`: `state.hasRecoveryRecord()` checks `new java.io.File(context.getFilesDir(), "rc2").exists()`. It checks existence without reading or decrypting `id`.

<a id="en-package-material"></a>

#### Installation Material Inputs

Raw inputs for `_gi`: `PackageInputs.installedCodePaths(context)`, `dataDirectory(context)`, `nativeLibraryDirectory(context)`, and `signingCertificates(context, context.getPackageName())`. These methods provide paths and certificates, not a finished security payload.

<a id="en-declared-permissions"></a>

#### Declared Permission Inputs

Raw input for `_gp`: `PackageInputs.declaredPermissions(context)` reads `PackageInfo.requestedPermissions` with `PackageManager.GET_PERMISSIONS`. Digest computation is separate.

<a id="en-related-app-signatures"></a>

#### Related Application Certificates

`is_foa_fdid_app_installed`: read certificates of visible packages with `PackageInputs.signingCertificates(context, packageName)`, then apply the related-app matching rules. The certificate array alone is not the final boolean result.

<a id="en-uuid-encoding"></a>

#### UUID Byte Representation

`expid` / `access_session_id`: `FieldEncoding.uuidBytes(uuid)` encodes an existing UUID as 16 big-endian bytes using unpadded Base64url. It does not acquire or create the UUID.

<a id="en-signal-id-encoding"></a>

#### Signal Record ID Representation

`e_regid` / `e_skey_id`: `FieldEncoding.registrationId(id)` / `signedPrekeyId(id)` encode existing record IDs as 4 and 3 big-endian bytes respectively; the helpers do not generate these IDs.

<a id="en-aid-encoding"></a>

#### Android ID Digest Representation

`aid`: read [Android ID](#en-android-id), then call `FieldEncoding.aidForPresentAndroidId(androidId)` to obtain SHA-256 in standard Base64.

<a id="en-advertising-id"></a>

#### Advertising Identifier Input

`advertising_id`: integrate Google Play services and call `AdvertisingIdClient.getAdvertisingIdInfo(context)` off the main thread. Read both the ID and its tracking preference.

<a id="en-push-token"></a>

#### Push Provider Token

`push_token`: configure Firebase, obtain the token with `FirebaseMessaging.getInstance().getToken()`, and handle refreshes through `onNewToken()`.

<a id="en-play-integrity"></a>

#### Play Integrity Service Input

`gpia` / `_gg`: prepare a provider using `StandardIntegrityManager`, request an actual outcome, and share it across both fields. See [Service Integration](#en-service-integration).

<a id="en-androidfields"></a>

### AndroidFields

`AndroidFields` reads system inputs visible to the calling app. It uses the default data subscription for telephony-network reads, distinguishes default-SIM and logical-slot state reads, and iterates active subscriptions for a number hint. Different subscriptions, permissions, and APIs can return different strings. A null return means this reader has no value; it is not an automatic protocol fallback.

`networkRadioType()` illustrates active-network classification; VPNs, parallel networks, and callbacks require selecting the corresponding snapshot. RAM formatting uses the current formatting locale. `stat()` preserves errors rather than treating permission denial as absence.

```java
// File: AndroidFields.java
package wiki.examples;

import android.Manifest;
import android.app.ActivityManager;
import android.content.Context;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.os.Build;
import android.provider.Settings;
import android.system.ErrnoException;
import android.system.Os;
import android.system.StructStat;
import android.telephony.SignalStrength;
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
        return locales.isEmpty() ? Locale.getDefault() : locales.get(0);
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
        if (service == null) return null;
        ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();
        service.getMemoryInfo(info);
        return new DecimalFormat("#.##").format(info.totalMem / 1073741824.0);
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

    public static TelephonyManager dataTelephony(Context context) {
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        if (service == null) return null;
        int id = SubscriptionManager.getDefaultDataSubscriptionId();
        return id < 0 ? service : service.createForSubscriptionId(id);
    }

    @SuppressWarnings("deprecation")
    public static String firstDeviceNumber(Context context) {
        if (!phonePermissionsGranted(context)) {
            throw new SecurityException("Phone permissions are not granted");
        }
        SubscriptionManager subscriptions =
                context.getSystemService(SubscriptionManager.class);
        List<SubscriptionInfo> active = subscriptions == null ? null
                : subscriptions.getActiveSubscriptionInfoList();
        if (active != null) {
            for (SubscriptionInfo item : active) {
                String value = Build.VERSION.SDK_INT >= 33
                        ? subscriptions.getPhoneNumber(item.getSubscriptionId())
                        : item.getNumber();
                if (value != null) return value;
            }
        }
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getLine1Number();
    }

    public static String simnum(String number) {
        return number != null && number.length() >= 6 ? "1" : "0";
    }

    public static Integer simState(Context context) {
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState();
    }

    public static Integer simState(Context context, int slotIndex) {
        if (slotIndex < 0) {
            throw new IllegalArgumentException("Expected a logical SIM slot index");
        }
        TelephonyManager service = context.getSystemService(TelephonyManager.class);
        return service == null ? null : service.getSimState(slotIndex);
    }

    public static String simOperator(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getSimOperatorName();
    }

    public static String networkOperator(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.getNetworkOperatorName();
    }

    public static Integer signalLevel(Context context) {
        TelephonyManager service = dataTelephony(context);
        if (service == null) return null;
        SignalStrength strength = service.getSignalStrength();
        return strength == null ? null : strength.getLevel();
    }

    public static Boolean roaming(Context context) {
        TelephonyManager service = dataTelephony(context);
        return service == null ? null : service.isNetworkRoaming();
    }

    public static Integer networkRadioType(Context context) {
        ConnectivityManager service =
                context.getSystemService(ConnectivityManager.class);
        if (service == null) return null;
        Network network = service.getActiveNetwork();
        if (network == null) return -1;
        NetworkCapabilities caps = service.getNetworkCapabilities(network);
        if (caps == null) return null;
        if (caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)
                || caps.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET)) return 1;
        if (!caps.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)) return -1;
        TelephonyManager phone = dataTelephony(context);
        return phone == null ? null : radioCode(phone.getDataNetworkType());
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

Initialize missing identifiers through their own lifecycle and persist them; do not regenerate every ID for every request. Session resets differ from installation-ID initialization. Backup and Signal material require their respective modules. The example initializer has no concurrency lock; serialize initialization in your state-management layer to avoid competing values.

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

Certificates can represent signing history or multiple current signers. Apply the selection/order rules of each consuming field rather than treating any first certificate as interchangeable. Other-package queries are subject to package visibility.

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
                ? new String[0] : info.requestedPermissions.clone();
    }

    @SuppressWarnings("deprecation")
    public static byte[][] signingCertificates(Context context, String packageName)
            throws PackageManager.NameNotFoundException {
        PackageInfo info = context.getPackageManager().getPackageInfo(
                packageName, PackageManager.GET_SIGNING_CERTIFICATES);
        SigningInfo signing = info.signingInfo;
        if (signing == null) return new byte[0][];
        Signature[] certificates = signing.hasMultipleSigners()
                ? signing.getApkContentsSigners()
                : signing.getSigningCertificateHistory();
        if (certificates == null) return new byte[0][];
        byte[][] result = new byte[certificates.length][];
        for (int i = 0; i < certificates.length; i++) {
            result[i] = certificates[i].toByteArray();
        }
        return result;
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

The aid helper only accepts a present Android ID; handle read failures separately. The percent-encoding helper returns an already encoded value, which must not be encoded again when building a form.

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
        if (value < 0 || value > 0xFFFFFF) {
            throw new IllegalArgumentException("Expected an unsigned 24-bit ID");
        }
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
                androidId.getBytes(StandardCharsets.UTF_8));
        return Base64.encodeToString(digest, Base64.NO_WRAP);
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
| gpia / _gg | Prepare a provider through `StandardIntegrityManager`, then request a token. | Requires the app's Play/Cloud setup and requestHash; share one actual outcome across both fields. |
| recaptcha | Use the real result and callback from the reCAPTCHA SDK. | Use the app's service configuration and actual token/failure outcome. |
| authkey / e_ident / e_regid / e_skey_* / e_pq_last_resort_* | Use the application's connection-authentication and Signal session/key modules. | Android KeyStore is not a universal getter for these protocol records; public material must match the existing identity/record. |
| id / backup_token / foa_backup_token / cred_token | Use the appropriate recovery, backup, or credential module. | Select material for the correct account/number and apply its parsing, decryption, and lifecycle. |
| client_metrics / offline_ab / feo2_query_status / language_selector_* / mistyped | Use application business events and persisted state. | Accumulate, persist, or consume at actual events rather than inventing state during serialization. |

Google/Firebase results belong to the calling application, service project, and action; an example application's token is not another client's credential.

API reference: [AdvertisingIdClient](https://developers.google.com/android/reference/com/google/android/gms/ads/identifier/AdvertisingIdClient) · [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/android/get-started) · [Play Integrity](https://developer.android.com/google/play/integrity/standard)

[Back to parameter reference](exist.md#en)
