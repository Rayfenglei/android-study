> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128027208)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 根据包名授予 WRITE_SETTINGS 权限的核心类](#t1)

[3. 根据包名授予 WRITE_SETTINGS 权限的核心功能分析和实现](#t2)

[3.2 PhoneWindowManager.java 中在开机完成后根据包名授权](#t3)

1. 概述
-----

  在 11.0 的系统开发中，对于申请一些特殊权限比如悬浮窗，WRITE_SETTINGS 权限等等，  
这些特殊权限，在 app 中申请以后，不会像普通权限一样，首次启动 app 会弹出动态权限  
弹窗，然后在系统设置里面申请授予权限即可，这就需要分析系统设置授权的流程，然后  
在系统启动过程中授予权限即可

2. 根据包名授予 WRITE_SETTINGS 权限的核心类
-------------------------------

```
packages\apps\Settings\src\com\android\settings\applications\appinfo\WriteSettingsDetails.java
frameworks\base\services\core\java\com\android\server\policy\PhoneWindowManager.java
```

3. 根据包名授予 WRITE_SETTINGS 权限的核心功能分析和实现
-------------------------------------

  在系统设置授权 WRITE_SETTINGS 的页面根据代码分析，发现在每个 app 中授权开启  
WRITE_SETTINGS 权限的处理页面就是在 WriteSettingsDetails.java 中处理的，接下来分析  
WriteSettingsDetails.java 中相关开启 WRITE_SETTINGS 权限的相关代码

```
  public class WriteSettingsDetails extends AppInfoWithHeader implements OnPreferenceChangeListener,
        OnPreferenceClickListener {
 
    private static final String KEY_APP_OPS_PREFERENCE_SCREEN = "app_ops_preference_screen";
    private static final String KEY_APP_OPS_SETTINGS_SWITCH = "app_ops_settings_switch";
    private static final String LOG_TAG = "WriteSettingsDetails";
 
    private static final int [] APP_OPS_OP_CODE = {
            AppOpsManager.OP_WRITE_SETTINGS
    };
 
    // Use a bridge to get the overlay details but don't initialize it to connect with all state.
    // TODO: Break out this functionality into its own class.
    private AppStateWriteSettingsBridge mAppBridge;
    private AppOpsManager mAppOpsManager;
    private SwitchPreference mSwitchPref;
    private Intent mSettingsIntent;
    private WriteSettingsState mWriteSettingsState;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
 
        Context context = getActivity();
        mAppBridge = new AppStateWriteSettingsBridge(context, mState, null);
        mAppOpsManager = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
 
        addPreferencesFromResource(R.xml.write_system_settings_permissions_details);
        mSwitchPref = (SwitchPreference) findPreference(KEY_APP_OPS_SETTINGS_SWITCH);
 
        mSwitchPref.setOnPreferenceChangeListener(this);
 
        mSettingsIntent = new Intent(Intent.ACTION_MAIN)
                .addCategory(Settings.INTENT_CATEGORY_USAGE_ACCESS_CONFIG)
                .setPackage(mPackageName);
    }
```

在 WriteSettingsDetails.java 中的 onCreate(Bundle savedInstanceState) 中 mSwitchPref 就是  
WRITE_SETTINGS 权限的授权开关，而 mAppOpsManager 就是系统 AppOpsManager 的权限  
管理类，负责提供接口来实现对权限的授权

```
   @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        if (preference == mSwitchPref) {
            if (mWriteSettingsState != null && (Boolean) newValue != mWriteSettingsState
                    .isPermissible()) {
                setCanWriteSettings(!mWriteSettingsState.isPermissible());
                refreshUi();
            }
            return true;
        }
        return false;
    }
```

在 WriteSettingsDetails.java 中的 onPreferenceChange 中就是 mSwitchPref 开关以后，调用的  
然后开启关闭设置权限打开和关闭的

```
  private void setCanWriteSettings(boolean newState) {
        logSpecialPermissionChange(newState, mPackageName);
        mAppOpsManager.setMode(AppOpsManager.OP_WRITE_SETTINGS,
                mPackageInfo.applicationInfo.uid, mPackageName, newState
                ? AppOpsManager.MODE_ALLOWED : AppOpsManager.MODE_ERRORED);
    }
 
    void logSpecialPermissionChange(boolean newState, String packageName) {
        int logCategory = newState ? SettingsEnums.APP_SPECIAL_PERMISSION_SETTINGS_CHANGE_ALLOW
                : SettingsEnums.APP_SPECIAL_PERMISSION_SETTINGS_CHANGE_DENY;
        FeatureFactory.getFactory(getContext()).getMetricsFeatureProvider().action(getContext(),
                logCategory, packageName);
    }
 
    private boolean canWriteSettings(String pkgName) {
        int result = mAppOpsManager.checkOpNoThrow(AppOpsManager.OP_WRITE_SETTINGS,
                mPackageInfo.applicationInfo.uid, pkgName);
        if (result == AppOpsManager.MODE_ALLOWED) {
            return true;
        }
 
        return false;
    }
```

在 WriteSettingsDetails.java 中的 setCanWriteSettings(boolean newState) 可以发现  
主要是通过 mAppOpsManager.setMode 来设置权限 AppOpsManager.OP_WRITE_SETTINGS 的  
根据 newState 的值设置 AppOpsManager.MODE_ALLOWED : AppOpsManager.MODE_ERRORED  
来开启和关闭 WRITE_SETTINGS 的权限

```
   public static CharSequence getSummary(Context context, AppEntry entry) {
        WriteSettingsState state;
        if (entry.extraInfo instanceof WriteSettingsState) {
            state = (WriteSettingsState) entry.extraInfo;
        } else if (entry.extraInfo instanceof PermissionState) {
            state = new WriteSettingsState((PermissionState) entry.extraInfo);
        } else {
            state = new AppStateWriteSettingsBridge(context, null, null).getWriteSettingsInfo(
                    entry.info.packageName, entry.info.uid);
        }
 
        return getSummary(context, state);
    }
 
    public static CharSequence getSummary(Context context, WriteSettingsState writeSettingsState) {
        return context.getString(writeSettingsState.isPermissible()
                ? R.string.app_permission_summary_allowed
                : R.string.app_permission_summary_not_allowed);
    }
```

在 WriteSettingsDetails.java 中的 getSummary 通过获取当前权限是否开启来设置显示的文字  
为允许或者不允许

3.2 PhoneWindowManager.java 中在开机完成后根据包名授权
-----------------------------------------

```
 /** {@inheritDoc} */
    @Override
    public void systemReady() {
        // In normal flow, systemReady is called before other system services are ready.
        // So it is better not to bind keyguard here.
        mKeyguardDelegate.onSystemReady();
 
        mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
        if (mVrManagerInternal != null) {
            mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
        }
 
        readCameraLensCoverState();
        updateUiMode();
        mDefaultDisplayRotation.updateOrientationListener();
        synchronized (mLock) {
            mSystemReady = true;
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    updateSettings();
                }
            });
            // If this happens, for whatever reason, systemReady came later than systemBooted.
            // And keyguard should be already bound from systemBooted
            if (mSystemBooted) {
                mKeyguardDelegate.onBootCompleted();
            }
        }
+ allowAppWriteSettingsPermission("包名")；
        mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
    }
```

在 PhoneWindowManager.java 中的 systemReady() 是在开机完成后就会调用的，所以可以在这里  
根据包名设置 WRITE_SETTINGS 的权限，添加根据包名授权方法  
具体实现如下:

```
    @Override
    public void allowAppWriteSettingsPermission(String pkg) throws RemoteException {
        final long ident = Binder.clearCallingIdentity();
        try {
            if (!TextUtils.isEmpty(pkg)) {
                AppOpsManager mAppOpsManager = (AppOpsManager) mContext.getSystemService(Context.APP_OPS_SERVICE);
                PackageManager pm = mContext.getPackageManager();
                ApplicationInfo ai = pm.getApplicationInfo(pkg, PackageManager.GET_ACTIVITIES);
                mAppOpsManager.setMode(AppOpsManager.OP_WRITE_SETTINGS, ai.uid, pkg, AppOpsManager.MODE_ALLOWED);
            }
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

通过 allowAppWriteSettingsPermission(String pkg) 来根据包名设置 WRITE_SETTINGS 的权限  
然后添加到 systemReady() 中就保证在开机后授予 app 的 WRITE_SETTINGS 的权限