# Android Input Examples · Android 字段获取代码

**Telegram Contact: [@ttjjda](https://t.me/ttjjda)**

[exist 明文参数 / exist Parameters](exist.md) · [Registration / 注册目录](README.md) · [Field Lookup](#field-lookup) · [AndroidFields](#androidfields) · [RegistrationState](#registrationstate) · [PackageInputs](#packageinputs) · [FieldEncoding](#fieldencoding)

<a id="android-examples"></a>

本页提供四组 Java 辅助类：系统输入读取、应用状态存储、包信息读取与字段编码，使用 Android 公开 API 和 Java 标准库。编译时需要 API 33 或更高的 `android.jar`，运行基线为 API 28；较新接口通过 API 级别判断后调用。

Four Java helper classes cover system inputs, application state, package information, and field encoding using public Android APIs and Java standard libraries. Compile against API 33 or newer; the runtime baseline is API 28, with guards for newer calls.

以下权限声明放在 `AndroidManifest.xml` 的 `<manifest>` 下、`<application>` 外。电话权限还需按功能在运行时申请；网络状态读取需要 ACCESS_NETWORK_STATE。仅声明权限不等于用户已经授予。代码不会自动申请权限，也不会把 SecurityException 改写成正常状态。

Place these declarations directly under `<manifest>`, outside `<application>`, in `AndroidManifest.xml`. Request the phone permissions required by your feature at runtime; ACCESS_NETWORK_STATE is needed for network reads. The code does not request permissions or convert SecurityException into a normal result.

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_NUMBERS" />
```

## Field Lookup

按字段定位取值方式。`context` 是调用应用的 Context；`state` 是下方 `RegistrationState` 的实例。所有示例辅助类的完整代码均在本页后半部分。

Locate the acquisition step by field. `context` belongs to the calling app; `state` is an instance of the `RegistrationState` class provided below. Full helper implementations follow the lookup sections.

### app-locale

`lg` / `lc`：从应用 Context 的配置读取 Locale，再取语言和地区。 / Read the application locale, then its language and country.

```java
android.os.LocaleList locales = context.getResources().getConfiguration().getLocales();
java.util.Locale locale = locales.isEmpty() ? java.util.Locale.getDefault() : locales.get(0);
String language = locale.getLanguage();
String country = locale.getCountry();
String lg = language.matches("[a-z]{2,3}") ? language : "zz";
String lc = country.matches("[A-Z]{2}|[0-9]{3}") ? country : "ZZ";
```

### persistent-identifiers

`fdid`：读取应用已经保存的标识；`state.readString("fdid")`。键名由应用定义，这不是读取其它应用 ID 的接口。 / Read an identifier already stored by your app; keys are app-defined, not a cross-app ID getter.

### performance-identifier

`expid`：本示例的 `state.getOrCreateExamplePerfUuid()` 演示读取或初始化自己应用的性能 UUID；字段字节表示见 [uuid-encoding](#uuid-encoding)。 / The example method reads or initializes your own app performance UUID; see its encoding separately.

### access-session

`access_session_id`：由会话管理器保存和重置；可通过 `state.readString("access_session_id")` 读取应用已保存的会话值，再做 UUID 字节编码。 / Read the app-owned session value maintained by its session manager, then encode its UUID bytes.

### android-id

`aid` 的原始输入 / Raw input for `aid`：`android.provider.Settings.Secure.getString(context.getContentResolver(), android.provider.Settings.Secure.ANDROID_ID)`。这一步尚未计算摘要。 / This read does not yet compute the digest.

### device-name

`device_name`：`android.os.Build.DEVICE`。对应 `AndroidFields.deviceName()`，不是 `Build.MODEL`。 / Device build codename, exposed by the helper method.

### device-ram

`device_ram`：`ActivityManager.MemoryInfo.totalMem` 经 `getMemoryInfo()` 填充后除以 1024³。`AndroidFields.totalRamGiB(context)` 完成读取与格式化。 / Read total memory and format GiB using the helper.

### process-id

`pid`：`android.os.Process.myPid()`；`AndroidFields.pid()` 返回十进制文本。 / The helper formats the current process ID as decimal text.

### adb-setting

`db`：`Settings.Global.getInt(context.getContentResolver(), Settings.Global.ADB_ENABLED, 0)`；对应 `AndroidFields.adbEnabled(context)`。 / Read the ADB setting through the public Settings API.

### phone-permissions

`read_phone_permission_granted`：用 `Context.checkSelfPermission()` 检查所需电话权限；`AndroidFields.phonePermissionsGranted(context)` 返回 boolean，写入字段时用 `granted ? "1" : "0"`；它不会申请权限。 / Check required phone permissions, then serialize the boolean as `granted ? "1" : "0"`; this does not request permissions.

### device-number

`simnum`：先用 `AndroidFields.firstDeviceNumber(context)` 取得候选号码，再以 `AndroidFields.simnum(number)` 判断长度。前者使用公开 SubscriptionManager / TelephonyManager 接口；后者不再次读取系统。读取器保留首个非 null 返回值（包括空串）；API 33 的 `getPhoneNumber()` 在号码不可用时返回空串，此时不会继续选择后面的订阅。 / Read once, then classify the captured value. This reader keeps the first non-null result, including an empty string. On API 33, an unavailable number is an empty string, so it stops before later subscriptions.

### sim-state

`sim_state`：`AndroidFields.simState(context)` 读取默认 SIM 状态；指定卡槽用 `simState(context, slotIndex)`。`slotIndex` 是逻辑卡槽编号，可取自 `SubscriptionInfo.getSimSlotIndex()`，不能直接传订阅 ID。该结果不是 `sim_type` 分类。 / Read the default SIM state, or pass a logical slot index to the overload. Obtain that index from `SubscriptionInfo.getSimSlotIndex()`; a subscription ID is not a slot index. This is separate from the application SIM category.

### operator-names

`sim_operator_name` / `network_operator_name`：所选订阅的 `getSimOperatorName()` / `getNetworkOperatorName()`；对应 `AndroidFields.simOperator(context)` / `networkOperator(context)`。 / Read home and current operator names for the selected subscription.

### network-type

`network_radio_type`：`AndroidFields.networkRadioType(context)` 读取活动网络与电话制式，再由 `radioCode()` 映射协议编号。它不是直接返回 Android 的 `NETWORK_TYPE_*` 常量。 / Read transport/radio inputs and map them to the protocol enum.

### airplane-mode

`airplane_mode_type`：`Settings.Global.AIRPLANE_MODE_ON`；对应 `AndroidFields.airplaneMode(context)`，返回归一化的 0/1。 / Read the setting and normalize to 0/1.

### signal-strength

`cellular_strength`：所选订阅的 `TelephonyManager.getSignalStrength()`，再取 `SignalStrength.getLevel()`；对应 `AndroidFields.signalLevel(context)`。返回值可为 null；协议字段的不可用兜底为 `5`，不能把 null 自动写成 `0`。 / Read the selected subscription signal level. A null result is unavailable; use the field fallback `5`, not an automatic `0`.

### roaming-state

`roaming_type` 的原始输入 / Raw input：`AndroidFields.roaming(context)` 调用 `isNetworkRoaming()`。有网络快照时，false/true 分别编码为 `"0"`/`"1"`，电话服务对象不存在返回 null 时编码为 `"2"`；无快照则省略字段，读取异常另行处理。 / With a network snapshot, encode false/true as `"0"`/`"1"`, or `"2"` when the absent telephony service yields null. Omit the field without a snapshot; handle read exceptions separately.

### recovery-record

`hasinrc`：`new java.io.File(context.getFilesDir(), "rc2").exists()`；对应 `state.hasRecoveryRecord()`。这里只检查存在性，不读取或解密 `id`。 / Check file existence, not recovery-material content.

### package-material

`_gi` 的原始输入 / Raw inputs：`PackageInputs.installedCodePaths(context)`、`dataDirectory(context)`、`nativeLibraryDirectory(context)` 和 `signingCertificates(context, context.getPackageName())`。这些提供路径与证书，不直接生成 `_gi`。 / These methods provide paths/certificates, not the finished security field.

### declared-permissions

`_gp` 的原始输入 / Raw input：`PackageInputs.declaredPermissions(context)` 使用 `PackageManager.GET_PERMISSIONS` 读取 `PackageInfo.requestedPermissions`；摘要计算在后续安全字段专题展开。 / Read declared permissions; digest computation is separate.

### related-app-signatures

`is_foa_fdid_app_installed`：用 `PackageInputs.signingCertificates(context, packageName)` 获取可见应用证书，再按关联应用的匹配规则判断。这个方法只提供证书，不能单独代表最终布尔结果。 / Read visible-package certificates, then apply the related-app matching rules.

### uuid-encoding

`expid` / `access_session_id` 的表示转换 / Encoding：`FieldEncoding.uuidBytes(uuid)` 把已有 UUID 转成大端 16 字节并做 Base64url。它不获取或创建 UUID。 / This encoder does not acquire or create the UUID.

### signal-id-encoding

`e_regid` / `e_skey_id`：`FieldEncoding.registrationId(id)` / `signedPrekeyId(id)` 分别编码 4 字节和 3 字节编号。输入编号来自已有密钥记录，不由这些方法生成。 / Encode existing record IDs as 4 or 3 bytes.

### aid-encoding

`aid`：先读取 [android-id](#android-id)，再调用 `FieldEncoding.aidForPresentAndroidId(androidId)`，得到 SHA-256 的标准 Base64 文本。 / Read Android ID, then hash and encode the present value.

### advertising-id

`advertising_id`：Google Play services 的 `AdvertisingIdClient.getAdvertisingIdInfo(context)`，读取 ID 与追踪限制；需接入 SDK，并在工作线程调用。 / Integrate the advertising-ID SDK and call it off the main thread.

### push-token

`push_token`：FCM 的 `FirebaseMessaging.getInstance().getToken()` 与 `onNewToken()` 回调；需先完成应用的 Firebase 配置。 / Configure Firebase and obtain/update the token through the SDK.

### play-integrity

`gpia` / `_gg` 的服务输入 / Service input：使用 `StandardIntegrityManager` 准备 provider 后请求 token，同一次结果供两个字段使用。完整接入条件见 [Service Integration](#service-integration)。 / Prepare the provider, request a real outcome, and share it across both fields.

[AndroidFields](#androidfields) · [RegistrationState](#registrationstate) · [PackageInputs](#packageinputs) · [FieldEncoding](#fieldencoding) · [Service Integration](#service-integration)

<a id="android-system"></a>

## AndroidFields

`AndroidFields` 获取调用应用可访问的系统输入。多 SIM 的电话网络读取使用默认数据订阅；SIM 状态单独区分默认卡与指定逻辑卡槽，号码提示遍历活动订阅。订阅、权限和 API 不同可能返回不同号码，不能假设不同入口必然给出相同字符串。返回 null 表示该采集入口没有值，不是直接写入请求的协议兜底。

`AndroidFields` reads system inputs visible to the calling app. It uses the default data subscription for telephony-network reads, distinguishes default-SIM and logical-slot state reads, and iterates active subscriptions for a number hint. Different subscriptions, permissions, and APIs can return different strings. A null return means this reader has no value; it is not an automatic protocol fallback.

`networkRadioType()` 演示当前活动网络的分类；VPN、并行网络和异步回调需要调用方选择对应快照。`totalRamGiB()` 使用当前数字格式环境；请求所需的小数格式应与字段约定一致。`stat()` 保留检查异常，不能把权限不足伪装成文件不存在。

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

Android ID 与应用签名、用户及设备有关，不能把另一应用读取的值当成相同值。[ANDROID_ID](https://developer.android.com/reference/android/provider/Settings.Secure#ANDROID_ID)

Android ID is scoped to signing key, user, and device. [ANDROID_ID](https://developer.android.com/reference/android/provider/Settings.Secure#ANDROID_ID)

本机号码接口不保证提供可用号码；信号与电话网络接口各有权限和设备能力条件，不支持的设备可抛出 UnsupportedOperationException，服务对象非 null 不代表所有电话能力均可用。[SubscriptionManager](https://developer.android.com/reference/android/telephony/SubscriptionManager#getPhoneNumber(int)) · [TelephonyManager](https://developer.android.com/reference/android/telephony/TelephonyManager) · [NetworkCapabilities](https://developer.android.com/reference/android/net/NetworkCapabilities) · [SIM 默认卡与卡槽 / Default SIM and logical slot](https://developer.android.com/reference/android/telephony/TelephonyManager#getSimState(int))

Phone-number availability is not guaranteed. Telephony/signal calls require specific permissions and device features; unsupported devices can throw UnsupportedOperationException even when the service object exists. [SubscriptionManager](https://developer.android.com/reference/android/telephony/SubscriptionManager#getPhoneNumber(int)) · [TelephonyManager](https://developer.android.com/reference/android/telephony/TelephonyManager)

<a id="android-state"></a>

## RegistrationState

这个示例展示如何在自己的应用中读写状态。`wiki_example_registration_state` 是示例自定义的偏好文件名，键名由调用方定义，不是 WhatsApp 的存储文件名或通用字段读取接口。读取 `fdid` 等字段的前提是应用已按相应规则保存了真实值。

This demonstrates state storage in your own app. The preference filename is defined by the example and keys are caller-defined. It is not WhatsApp storage or a universal field getter. Reading a stored field presupposes that its actual producer saved the correct value.

持久标识缺失时才按自己的初始化流程创建并保存；不要在每次请求里为所有 ID 重新生成 UUID。访问会话重置与安装标识初始化也应分别处理。备份和 Signal 材料由其各自模块保存，不能从几个偏好键凭空生成。示例初始化方法没有并发互斥；应用需在自己的状态管理层串行执行初始化，避免并发请求各自生成不同值。

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

`hasRecoveryRecord()` 只检查调用应用自己的文件。存在性不是解密或有效性检查。[应用私有存储 / App-specific storage](https://developer.android.com/training/data-storage/app-specific)

<a id="android-package"></a>

## PackageInputs

这些方法读取原始包信息，供完整性和安装相关字段使用。`declaredPermissions()` 不读取运行时授权结果；安装方包名也不能直接替代完整安装归因 JSON。

These methods read raw package information used by integrity and installation-related fields. Declared permissions differ from runtime grants. An installer package name alone is not complete installation-attribution JSON.

证书数组可能包含签名轮换历史或多个当前签名者；后续字段必须按各自规则选择、排序与处理，不能默认任意第一张证书都可互换。查询其它应用还受包可见性限制。

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

接口说明：[PackageInfo](https://developer.android.com/reference/android/content/pm/PackageInfo) · [SigningInfo](https://developer.android.com/reference/android/content/pm/SigningInfo) · [包可见性 / Package visibility](https://developer.android.com/training/package-visibility)

<a id="android-encoding"></a>

## FieldEncoding

本类只转换调用方已经持有的真实材料，不生成身份公钥、Google token 或安全结果。UUID 文本和 UUID 字节编码要区分；`e_regid` 与 `e_skey_id` 的字节长度不同。

These helpers encode material already held by the caller. They do not generate identity keys, Google tokens, or security verdicts. UUID text differs from UUID-byte encoding; registration and signed-prekey IDs have different widths.

`aidForPresentAndroidId()` 只处理非 null Android ID；读取失败时应进入独立错误处理。`percentEncodeBytes()` 输出已经百分号编码的值，后续组表单不能再对它重复编码。

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

基础接口：[Base64](https://developer.android.com/reference/android/util/Base64) · [ByteBuffer](https://developer.android.com/reference/java/nio/ByteBuffer)

<a id="android-services"></a>

## Service Integration

| 字段 / Field | 可实际接入的入口 / Integration entry | 接入要求 / Requirement |
| --- | --- | --- |
| advertising_id | `AdvertisingIdClient.getAdvertisingIdInfo(context)`，读取返回对象的 `getId()` 和 `isLimitAdTrackingEnabled()` | 添加 Google Play services 广告标识依赖，按所选 SDK 声明相应权限；在工作线程调用，处理服务不可用与追踪限制。 / Add the advertising-ID dependency and required permissions; call off the main thread and handle service availability/tracking preference. |
| push_token | FCM 的 `FirebaseMessaging.getInstance().getToken()`，以及 `onNewToken()` 回调 | 先完成 Firebase 应用配置；异步接收并保存真实 token，随刷新更新。 / Configure Firebase, then asynchronously store/update the actual token. |
| gpia / _gg | Play Integrity 的 `StandardIntegrityManager`，先准备 provider，再请求 token | 需要应用自己的 Play/Cloud 配置与 requestHash；同一次真实结果供两个字段使用。 / Requires the app's Play/Cloud setup and requestHash; share one actual outcome across both fields. |
| recaptcha | reCAPTCHA SDK 的真实校验结果与回调 | 使用应用自己的服务配置，记录 token 或失败结果；不以随机文本代替。 / Use the app's service configuration and actual token/failure outcome. |
| authkey / e_ident / e_regid / e_skey_* / e_pq_last_resort_* | 应用自己的连接认证与 Signal 会话/密钥模块 | Android KeyStore 不是这些协议记录的通用查询器；公开材料必须与已有身份及记录配对。 / Android KeyStore is not a universal getter for these protocol records; public material must match the existing identity/record. |
| id / backup_token / foa_backup_token / cred_token | 对应恢复、备份或凭据模块 | 读取正确账户及号码对应的材料，并执行其自身解析/解密与生命周期。 / Select material for the correct account/number and apply its parsing, decryption, and lifecycle. |
| client_metrics / offline_ab / feo2_query_status / language_selector_* / mistyped | 应用的业务事件与持久状态 | 由实际事件累计、写入或消费；不在序列化时编造交互记录。 / Accumulate, persist, or consume at actual events rather than inventing state during serialization. |

Google/Firebase 返回值属于调用应用、服务项目和对应操作，示例应用取到的 token 不能直接当成另一个客户端的凭据。

Google/Firebase results belong to the calling application, service project, and action; an example application's token is not another client's credential.

实际接口说明：[AdvertisingIdClient](https://developers.google.com/android/reference/com/google/android/gms/ads/identifier/AdvertisingIdClient) · [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging/android/get-started) · [Play Integrity](https://developer.android.com/google/play/integrity/standard)

[返回参数说明 / Back to parameter reference](exist.md)
