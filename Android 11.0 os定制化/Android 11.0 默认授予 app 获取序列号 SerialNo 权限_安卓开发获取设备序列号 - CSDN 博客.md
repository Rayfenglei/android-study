> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126310575)

**目录**

[1. 概述](#1.%20%E6%A6%82%E8%BF%B0)

[2. 默认授予 app 获取序列号 SerialNo 权限相关核心代码:](#t1)

[3. 默认授予 app 获取序列号 SerialNo 权限相关代码核心功能实现分析](#t2)

[3.1 Build.java 关于序列号的相关代码](#t3)

[3.2 DeviceIdentifiersPolicyService 中关于 getSerial() 的相关代码](#3.2%20DeviceIdentifiersPolicyService%E4%B8%AD%E5%85%B3%E4%BA%8EgetSerial%28%29%E7%9A%84%E7%9B%B8%E5%85%B3%E4%BB%A3%E7%A0%81)

[3.3 默认授予 app 获取序列号 SerialNo 权限具体修改为:](#t5)

1. 概述
-----

在 [app 开发](https://so.csdn.net/so/search?q=app%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，会获取序列号等属性，而在 10.0 以后的高版本对于获取系统属性的相关信息要求严格 必须有权限才可以，10.0 以前的 Android 版本中，可以直接通过调用 Build.SERIAL 来获取序列号，在高版本中，为了保护个人隐私， 不让第三方应用轻易获取序列号。所以该 Api 已经过时， 并且它的值也被设置成了 "unknown"

2. 默认授予 app 获取序列号 SerialNo 权限相关核心代码:
------------------------------------

```
frameworks/base/core/java/android/os/Build.java
  frameworks/base/services/core/java/com/android/server/os/DeviceIdentifiersPolicyService.java
```

3. 默认授予 app 获取序列号 SerialNo 权限相关代码核心功能实现分析
-----------------------------------------

#### 3.1 Build.java 关于序列号的相关代码

```
public class Build {
	private static final String TAG = "Build";
	/** Value used for when a build property is unknown. */
	public static final String UNKNOWN = "unknown";
	/** Either a changelist number, or a label like "M4-rc20". */
	public static final String ID = getString("ro.build.id");
	/** A build ID string meant for displaying to the user */
	public static final String DISPLAY = getString("ro.build.display.id");
	/** The name of the overall product. */
	public static final String PRODUCT = getString("ro.product.name");
	/** The name of the industrial design. */
	public static final String DEVICE = getString("ro.product.device");
	/** The name of the underlying board, like "goldfish". */
	public static final String BOARD = getString("ro.product.board");
	/**
* The name of the instruction set (CPU type + ABI convention) of native code.
*
* @deprecated Use {@link #SUPPORTED_ABIS} instead.
*/
	@Deprecated
	public static final String CPU_ABI;
	/**
* The name of the second instruction set (CPU type + ABI convention) of native code.
*
* @deprecated Use {@link #SUPPORTED_ABIS} instead.
*/
	@Deprecated
	public static final String CPU_ABI2;
	/** The manufacturer of the product/hardware. */
	public static final String MANUFACTURER = getString("ro.product.manufacturer");
	/** The consumer-visible brand with which the product/hardware will be associated, if any. */
	public static final String BRAND = getString("ro.product.brand");
	/** The end-user-visible name for the end product. */
	public static final String MODEL = getString("ro.product.model");
	/** The system bootloader version number. */
	public static final String BOOTLOADER = getString("ro.bootloader");
	/**
* The radio firmware version number.
*
* @deprecated The radio firmware version is frequently not
* available when this class is initialized, leading to a blank or
* "unknown" value for this string.  Use
* {@link #getRadioVersion} instead.
*/
	@Deprecated
	public static final String RADIO = joinListOrElse(
	TelephonyProperties.baseband_version(), UNKNOWN);
	/** The name of the hardware (from the kernel command line or /proc). */
	public static final String HARDWARE = getString("ro.hardware");
	/**
* Whether this build was for an emulator device.
* @hide
*/
	@UnsupportedAppUsage
	@TestApi
	public static final Boolean IS_EMULATOR = getString("ro.kernel.qemu").equals("1");
	/**
* A hardware serial number, if available. Alphanumeric only, case-insensitive.
* This field is always set to {@link Build#UNKNOWN}.
*
* @deprecated Use {@link #getSerial()} instead.
**/
	@Deprecated
	// IMPORTANT: This field should be initialized via a function call to
	// prevent its value being inlined in the app during compilation because
	// we will later set it to the value based on the app's target SDK.
	public static final String SERIAL = getString("no.such.thing");
	@SuppressAutoDoc // No support for device / profile owner.
	@RequiresPermission(Manifest.permission.READ_PRIVILEGED_PHONE_STATE)
	public static String getSerial() {
		IDeviceIdentifiersPolicyService service = IDeviceIdentifiersPolicyService.Stub
		asInterface(ServiceManager.getService(Context.DEVICE_IDENTIFIERS_SERVICE));
		try {
			Application application = ActivityThread.currentApplication();
			String callingPackage = application != null ? application.getPackageName() : null;
			return service.getSerialForPackage(callingPackage, null);
		}
		catch (RemoteException e) {
			e.rethrowFromSystemServer();
		}
		return UNKNOWN;
	}
	@UnsupportedAppUsage
	private static String getString(String property) {
		return SystemProperties.get(property, UNKNOWN);
	}
}
	public static final String SERIAL = getString("no.such.thing");
这是获取序列号的api 可以看出已经获取不到序列号了
所以要从getSerial()中获取序列号
而调用getSerial()该api需要READ_PRIVILEGED_PHONE_STATE权限， 
该权限是一个protectLevel为signature的系统级权限，只有系统签名的系统App才能获取该权限， 第三方App是无法获取的。
```

#### 3.2 DeviceIdentifiersPolicyService 中关于 getSerial() 的相关代码

```
  /**
* This service defines the policy for accessing device identifiers.
*/
public final class DeviceIdentifiersPolicyService extends SystemService {
	public DeviceIdentifiersPolicyService(Context context) {
		super(context);
	}
	@Override
	public void onStart() {
		publishBinderService(Context.DEVICE_IDENTIFIERS_SERVICE,
		new DeviceIdentifiersPolicy(getContext()));
	}
	private static final class DeviceIdentifiersPolicy
	extends IDeviceIdentifiersPolicyService.Stub {
		private final @NonNull Context mContext;
		public DeviceIdentifiersPolicy(Context context) {
			mContext = context;
		}
		@Override
		public @Nullable String getSerial() throws RemoteException {
			// Since this invocation is on the server side a null value is used for the
			// callingPackage as the server's package name (typically android) should not be used
			// for any device / profile owner checks. The majority of requests for the serial number
			// should use the getSerialForPackage method with the calling package specified.
			if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
			/* callingPackage */
			null, null, "getSerial")) {
				return Build.UNKNOWN;
			}
			return SystemProperties.get("ro.serialno", Build.UNKNOWN);
		}
		@Override
		public @Nullable String getSerialForPackage(String callingPackage,
		String callingFeatureId) throws RemoteException {
			if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
			callingPackage, callingFeatureId, "getSerial")) {
				return Build.UNKNOWN;
			}
			return SystemProperties.get("ro.serialno", Build.UNKNOWN);
		}
	}
}
```

#### 3.3 默认授予 app 获取序列号 SerialNo 权限具体修改为:

```
private static final class DeviceIdentifiersPolicy
	extends IDeviceIdentifiersPolicyService.Stub {
		private final @NonNull Context mContext;
		public DeviceIdentifiersPolicy(Context context) {
			mContext = context;
		}
		@Override
		public @Nullable String getSerial() throws RemoteException {
			// Since this invocation is on the server side a null value is used for the
			// callingPackage as the server's package name (typically android) should not be used
			// for any device / profile owner checks. The majority of requests for the serial number
			// should use the getSerialForPackage method with the calling package specified.
			+ /*if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
			/* callingPackage */
			null, null, "getSerial")) {
				return Build.UNKNOWN;
			}*/
			return SystemProperties.get("ro.serialno", Build.UNKNOWN);
		}
		@Override
		public @Nullable String getSerialForPackage(String callingPackage,
		String callingFeatureId) throws RemoteException {
			+ /*if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
			callingPackage, callingFeatureId, "getSerial")) {
				return Build.UNKNOWN;
			}*/
			return SystemProperties.get("ro.serialno", Build.UNKNOWN);
		}
	}
注释掉这段代码，无需权限的限制
if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
			/* callingPackage */
			null, null, "getSerial")) {
				return Build.UNKNOWN;
			}
```