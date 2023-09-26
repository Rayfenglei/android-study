> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124806874)

### 1. 概述

在 11.0 12.0 系统开发中，修改系统默认[输入法](https://so.csdn.net/so/search?q=%E8%BE%93%E5%85%A5%E6%B3%95&spm=1001.2101.3001.7020)也是经常需要修改的功能，但是替换为搜狗输入法以后，点击输入框时，会弹出  
授权权限对话框 感觉是特别麻烦的，所以在 framework 中要授予搜狗 app 运行时权限

### 2. 修改搜狗为默认输入法，并且授予运行权限的核心类

```
device/google/atv/overlay/frameworks/base/core/res/res/values/config.xml
frameworks\base\services\core\java\com\android\server\pm\permission\DefaultPermissionGrantPolicy.java

```

### 3. 修改搜狗为默认输入法，并且授予运行权限的功能分析和实现

### 3.1 把搜狗输入法 id 设置为默认输入法 id

设置搜狗为默认输入法 相对来简单些需要首选 内置搜狗 app 到系统中  
接下来将添加的第三方搜狗输入法设置为默认输入法，  
修改如下：

```
--- a/device/google/atv/overlay/frameworks/base/core/res/res/values/config.xml
+++ b/device/google/atv/overlay/frameworks/base/core/res/res/values/config.xml
@@ -84,12 +84,12 @@
          And the disabled_until_used state for an IME is released by InputMethodManagerService
          when the IME is selected as an enabled IME. -->
     <string-array >
+        <item>com.sohu.inputmethod.sogou</item>
         <item>com.google.android.inputmethod.latin</item>
         <item>com.google.android.apps.inputmethod.hindi</item>
         <item>com.google.android.apps.inputmethod.zhuyin</item>
         <item>com.google.android.inputmethod.japanese</item>
         <item>com.google.android.inputmethod.korean</item>
         <item>com.google.android.inputmethod.pinyin</item>
     </string-array>
 
     <!-- Whether to keep background restricted profiles running after exiting. If set to false,
diff --git a/frameworks/base/core/res/res/values/config.xml b/frameworks/base/core/res/res/values/config.xml
index fa65de109e..ca57a9b260 100755
--- a/frameworks/base/core/res/res/values/config.xml
+++ b/frameworks/base/core/res/res/values/config.xml
@@ -2712,7 +2712,7 @@
          And the disabled_until_used state for an IME is released by InputMethodManagerService
          when the IME is selected as an enabled IME. -->
     <string-array >
-        <item>com.android.inputmethod.latin</item>
+        <item>com.sohu.inputmethod.sogou</item>
     </string-array>

```

通过上面两个 config 设置搜狗 id 为默认输入法的 id

### 3.2 授予搜狗 app 运行时权限

在 android 9.0 以后 运行时权限都是在 DefaultPermissionGrantPolicy.java 中授予的  
路径：frameworks\base\services\core\java\com\android\server\pm\permission\DefaultPermissionGrantPolicy.java

```
public void grantDefaultPermissions(int userId) {
    grantPermissionsToSysComponentsAndPrivApps(userId);
    grantDefaultSystemHandlerPermissions(userId);
    grantDefaultPermissionExceptions(userId);
    synchronized (mLock) {
        mDefaultPermissionsGrantedUsers.put(userId, userId);
    }
}

private void grantDefaultSystemHandlerPermissions(int userId) {
        Log.i(TAG, "Granting permissions to default platform handlers for user " + userId);

        final PackagesProvider locationPackagesProvider;
        final PackagesProvider locationExtraPackagesProvider;
        final PackagesProvider voiceInteractionPackagesProvider;
        final PackagesProvider smsAppPackagesProvider;
        final PackagesProvider dialerAppPackagesProvider;
        final PackagesProvider simCallManagerPackagesProvider;
        final PackagesProvider useOpenWifiAppPackagesProvider;
        final SyncAdapterPackagesProvider syncAdapterPackagesProvider;

        synchronized (mLock) {
            locationPackagesProvider = mLocationPackagesProvider;
            locationExtraPackagesProvider = mLocationExtraPackagesProvider;
            voiceInteractionPackagesProvider = mVoiceInteractionPackagesProvider;
            smsAppPackagesProvider = mSmsAppPackagesProvider;
            dialerAppPackagesProvider = mDialerAppPackagesProvider;
            simCallManagerPackagesProvider = mSimCallManagerPackagesProvider;
            useOpenWifiAppPackagesProvider = mUseOpenWifiAppPackagesProvider;
            syncAdapterPackagesProvider = mSyncAdapterPackagesProvider;
        }

        String[] voiceInteractPackageNames = (voiceInteractionPackagesProvider != null)
                ? voiceInteractionPackagesProvider.getPackages(userId) : null;
        String[] locationPackageNames = (locationPackagesProvider != null)
                ? locationPackagesProvider.getPackages(userId) : null;
        String[] locationExtraPackageNames = (locationExtraPackagesProvider != null)
                ? locationExtraPackagesProvider.getPackages(userId) : null;
        String[] smsAppPackageNames = (smsAppPackagesProvider != null)
                ? smsAppPackagesProvider.getPackages(userId) : null;
        String[] dialerAppPackageNames = (dialerAppPackagesProvider != null)
                ? dialerAppPackagesProvider.getPackages(userId) : null;
        String[] simCallManagerPackageNames = (simCallManagerPackagesProvider != null)
                ? simCallManagerPackagesProvider.getPackages(userId) : null;
        String[] useOpenWifiAppPackageNames = (useOpenWifiAppPackagesProvider != null)
                ? useOpenWifiAppPackagesProvider.getPackages(userId) : null;
        String[] contactsSyncAdapterPackages = (syncAdapterPackagesProvider != null) ?
                syncAdapterPackagesProvider.getPackages(ContactsContract.AUTHORITY, userId) : null;
        String[] calendarSyncAdapterPackages = (syncAdapterPackagesProvider != null) ?
                syncAdapterPackagesProvider.getPackages(CalendarContract.AUTHORITY, userId) : null;

        // Installer
        grantSystemFixedPermissionsToSystemPackage(
                getKnownPackage(PackageManagerInternal.PACKAGE_INSTALLER, userId),
                userId, STORAGE_PERMISSIONS);

        // Verifier
        final String verifier = getKnownPackage(PackageManagerInternal.PACKAGE_VERIFIER, userId);
        grantSystemFixedPermissionsToSystemPackage(verifier, userId, STORAGE_PERMISSIONS);
        grantPermissionsToSystemPackage(verifier, userId, PHONE_PERMISSIONS, SMS_PERMISSIONS);

        // SetupWizard
        grantPermissionsToSystemPackage(
                getKnownPackage(PackageManagerInternal.PACKAGE_SETUP_WIZARD, userId), userId,
                PHONE_PERMISSIONS, CONTACTS_PERMISSIONS, ALWAYS_LOCATION_PERMISSIONS,
                CAMERA_PERMISSIONS);

        // Camera
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(MediaStore.ACTION_IMAGE_CAPTURE, userId),
                userId, CAMERA_PERMISSIONS, MICROPHONE_PERMISSIONS, STORAGE_PERMISSIONS);

        // Sound recorder
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        MediaStore.Audio.Media.RECORD_SOUND_ACTION, userId),
                userId, MICROPHONE_PERMISSIONS);

        // Media provider
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultProviderAuthorityPackage(MediaStore.AUTHORITY, userId), userId,
                STORAGE_PERMISSIONS, PHONE_PERMISSIONS);

        // Downloads provider
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultProviderAuthorityPackage("downloads", userId), userId,
                STORAGE_PERMISSIONS);

        // Downloads UI
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        DownloadManager.ACTION_VIEW_DOWNLOADS, userId),
                userId, STORAGE_PERMISSIONS);

        // Storage provider
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultProviderAuthorityPackage("com.android.externalstorage.documents", userId),
                userId, STORAGE_PERMISSIONS);

        // CertInstaller
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(Credentials.INSTALL_ACTION, userId), userId,
                STORAGE_PERMISSIONS);

        // Dialer
        if (dialerAppPackageNames == null) {
            String dialerPackage =
                    getDefaultSystemHandlerActivityPackage(Intent.ACTION_DIAL, userId);
            grantDefaultPermissionsToDefaultSystemDialerApp(dialerPackage, userId);
        } else {
            for (String dialerAppPackageName : dialerAppPackageNames) {
                grantDefaultPermissionsToDefaultSystemDialerApp(dialerAppPackageName, userId);
            }
        }

        // Sim call manager
        if (simCallManagerPackageNames != null) {
            for (String simCallManagerPackageName : simCallManagerPackageNames) {
                grantDefaultPermissionsToDefaultSystemSimCallManager(
                        simCallManagerPackageName, userId);
            }
        }

        // Use Open Wifi
        if (useOpenWifiAppPackageNames != null) {
            for (String useOpenWifiPackageName : useOpenWifiAppPackageNames) {
                grantDefaultPermissionsToDefaultSystemUseOpenWifiApp(
                        useOpenWifiPackageName, userId);
            }
        }

        // SMS
        if (smsAppPackageNames == null) {
            String smsPackage = getDefaultSystemHandlerActivityPackageForCategory(
                    Intent.CATEGORY_APP_MESSAGING, userId);
            grantDefaultPermissionsToDefaultSystemSmsApp(smsPackage, userId);
        } else {
            for (String smsPackage : smsAppPackageNames) {
                grantDefaultPermissionsToDefaultSystemSmsApp(smsPackage, userId);
            }
        }

        // Cell Broadcast Receiver
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(Intents.SMS_CB_RECEIVED_ACTION, userId),
                userId, SMS_PERMISSIONS);

        // Carrier Provisioning Service
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerServicePackage(Intents.SMS_CARRIER_PROVISION_ACTION, userId),
                userId, SMS_PERMISSIONS);

        // Calendar
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackageForCategory(
                        Intent.CATEGORY_APP_CALENDAR, userId),
                userId, CALENDAR_PERMISSIONS, CONTACTS_PERMISSIONS);

        // Calendar provider
        String calendarProvider =
                getDefaultProviderAuthorityPackage(CalendarContract.AUTHORITY, userId);
        grantPermissionsToSystemPackage(calendarProvider, userId,
                CONTACTS_PERMISSIONS, STORAGE_PERMISSIONS);
        grantSystemFixedPermissionsToSystemPackage(calendarProvider, userId, CALENDAR_PERMISSIONS);

        // Calendar provider sync adapters
        grantPermissionToEachSystemPackage(
                getHeadlessSyncAdapterPackages(calendarSyncAdapterPackages, userId),
                userId, CALENDAR_PERMISSIONS);

        // Contacts
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackageForCategory(
                        Intent.CATEGORY_APP_CONTACTS, userId),
                userId, CONTACTS_PERMISSIONS, PHONE_PERMISSIONS);

        // Contacts provider sync adapters
        grantPermissionToEachSystemPackage(
                getHeadlessSyncAdapterPackages(contactsSyncAdapterPackages, userId),
                userId, CONTACTS_PERMISSIONS);

        // Contacts provider
        String contactsProviderPackage =
                getDefaultProviderAuthorityPackage(ContactsContract.AUTHORITY, userId);
        grantSystemFixedPermissionsToSystemPackage(contactsProviderPackage, userId,
                CONTACTS_PERMISSIONS, PHONE_PERMISSIONS);
        grantPermissionsToSystemPackage(contactsProviderPackage, userId, STORAGE_PERMISSIONS);

        // Device provisioning
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        DevicePolicyManager.ACTION_PROVISION_MANAGED_DEVICE, userId),
                userId, CONTACTS_PERMISSIONS);

        // Maps
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackageForCategory(Intent.CATEGORY_APP_MAPS, userId),
                userId, ALWAYS_LOCATION_PERMISSIONS);

        // Gallery
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackageForCategory(
                        Intent.CATEGORY_APP_GALLERY, userId),
                userId, STORAGE_PERMISSIONS);

        // Email
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackageForCategory(
                        Intent.CATEGORY_APP_EMAIL, userId),
                userId, CONTACTS_PERMISSIONS, CALENDAR_PERMISSIONS);

        // Browser
        String browserPackage = getKnownPackage(PackageManagerInternal.PACKAGE_BROWSER, userId);
        if (browserPackage == null) {
            browserPackage = getDefaultSystemHandlerActivityPackageForCategory(
                    Intent.CATEGORY_APP_BROWSER, userId);
            if (!isSystemPackage(browserPackage)) {
                browserPackage = null;
            }
        }
        grantPermissionsToPackage(browserPackage, userId, false /* ignoreSystemPackage */,
                true /*whitelistRestrictedPermissions*/, ALWAYS_LOCATION_PERMISSIONS);

        // Voice interaction
        if (voiceInteractPackageNames != null) {
            for (String voiceInteractPackageName : voiceInteractPackageNames) {
                grantPermissionsToSystemPackage(voiceInteractPackageName, userId,
                        CONTACTS_PERMISSIONS, CALENDAR_PERMISSIONS, MICROPHONE_PERMISSIONS,
                        PHONE_PERMISSIONS, SMS_PERMISSIONS, ALWAYS_LOCATION_PERMISSIONS);
            }
        }

        if (ActivityManager.isLowRamDeviceStatic()) {
            // Allow voice search on low-ram devices
            grantPermissionsToSystemPackage(
                    getDefaultSystemHandlerActivityPackage(
                            SearchManager.INTENT_ACTION_GLOBAL_SEARCH, userId),
                    userId, MICROPHONE_PERMISSIONS, ALWAYS_LOCATION_PERMISSIONS);
        }

        // Voice recognition
        Intent voiceRecoIntent = new Intent(RecognitionService.SERVICE_INTERFACE)
                .addCategory(Intent.CATEGORY_DEFAULT);
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerServicePackage(voiceRecoIntent, userId), userId,
                MICROPHONE_PERMISSIONS);

        // Location
        if (locationPackageNames != null) {
            for (String packageName : locationPackageNames) {
                grantPermissionsToSystemPackage(packageName, userId,
                        CONTACTS_PERMISSIONS, CALENDAR_PERMISSIONS, MICROPHONE_PERMISSIONS,
                        PHONE_PERMISSIONS, SMS_PERMISSIONS, CAMERA_PERMISSIONS,
                        SENSORS_PERMISSIONS, STORAGE_PERMISSIONS);
                grantSystemFixedPermissionsToSystemPackage(packageName, userId,
                        ALWAYS_LOCATION_PERMISSIONS, ACTIVITY_RECOGNITION_PERMISSIONS);
            }
        }
        if (locationExtraPackageNames != null) {
            // Also grant location permission to location extra packages.
            for (String packageName : locationExtraPackageNames) {
                grantPermissionsToSystemPackage(packageName, userId, ALWAYS_LOCATION_PERMISSIONS);
            }
        }

        // Music
        Intent musicIntent = new Intent(Intent.ACTION_VIEW)
                .addCategory(Intent.CATEGORY_DEFAULT)
                .setDataAndType(Uri.fromFile(new File("foo.mp3")), AUDIO_MIME_TYPE);
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(musicIntent, userId), userId,
                STORAGE_PERMISSIONS);

        // Home
        Intent homeIntent = new Intent(Intent.ACTION_MAIN)
                .addCategory(Intent.CATEGORY_HOME)
                .addCategory(Intent.CATEGORY_LAUNCHER_APP);
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(homeIntent, userId), userId,
                ALWAYS_LOCATION_PERMISSIONS);

        // Watches
        if (mContext.getPackageManager().hasSystemFeature(PackageManager.FEATURE_WATCH, 0)) {
            // Home application on watches

            String wearPackage = getDefaultSystemHandlerActivityPackageForCategory(
                    Intent.CATEGORY_HOME_MAIN, userId);
            grantPermissionsToSystemPackage(wearPackage, userId,
                    CONTACTS_PERMISSIONS, MICROPHONE_PERMISSIONS, ALWAYS_LOCATION_PERMISSIONS);
            grantSystemFixedPermissionsToSystemPackage(wearPackage, userId, PHONE_PERMISSIONS);

            // Fitness tracking on watches
            grantPermissionsToSystemPackage(
                    getDefaultSystemHandlerActivityPackage(ACTION_TRACK, userId), userId,
                    SENSORS_PERMISSIONS, ALWAYS_LOCATION_PERMISSIONS);
        }

        // Print Spooler
        grantSystemFixedPermissionsToSystemPackage(PrintManager.PRINT_SPOOLER_PACKAGE_NAME, userId,
                ALWAYS_LOCATION_PERMISSIONS);

        // EmergencyInfo
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        TelephonyManager.ACTION_EMERGENCY_ASSISTANCE, userId),
                userId, CONTACTS_PERMISSIONS, PHONE_PERMISSIONS);

        // NFC Tag viewer
        Intent nfcTagIntent = new Intent(Intent.ACTION_VIEW)
                .setType("vnd.android.cursor.item/ndef_msg");
        grantPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(nfcTagIntent, userId), userId,
                CONTACTS_PERMISSIONS, PHONE_PERMISSIONS);

        // Storage Manager
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        StorageManager.ACTION_MANAGE_STORAGE, userId),
                userId, STORAGE_PERMISSIONS);

        // Companion devices
        grantSystemFixedPermissionsToSystemPackage(
                CompanionDeviceManager.COMPANION_DEVICE_DISCOVERY_PACKAGE_NAME, userId,
                ALWAYS_LOCATION_PERMISSIONS);

        // Ringtone Picker
        grantSystemFixedPermissionsToSystemPackage(
                getDefaultSystemHandlerActivityPackage(
                        RingtoneManager.ACTION_RINGTONE_PICKER, userId),
                userId, STORAGE_PERMISSIONS);

        // TextClassifier Service
        String textClassifierPackageName =
                mContext.getPackageManager().getSystemTextClassifierPackageName();
        if (!TextUtils.isEmpty(textClassifierPackageName)) {
            grantPermissionsToSystemPackage(textClassifierPackageName, userId,
                    PHONE_PERMISSIONS, SMS_PERMISSIONS, CALENDAR_PERMISSIONS,
                    ALWAYS_LOCATION_PERMISSIONS, CONTACTS_PERMISSIONS);
        }

        // Atthention Service
        String attentionServicePackageName =
                mContext.getPackageManager().getAttentionServicePackageName();
        if (!TextUtils.isEmpty(attentionServicePackageName)) {
            grantPermissionsToSystemPackage(attentionServicePackageName, userId,
                    CAMERA_PERMISSIONS);
        }

        // There is no real "marker" interface to identify the shared storage backup, it is
        // hardcoded in BackupManagerService.SHARED_BACKUP_AGENT_PACKAGE.
        grantSystemFixedPermissionsToSystemPackage("com.android.sharedstoragebackup", userId,
                STORAGE_PERMISSIONS);

        // System Captions Service
        String systemCaptionsServicePackageName =
                mContext.getPackageManager().getSystemCaptionsServicePackageName();
        if (!TextUtils.isEmpty(systemCaptionsServicePackageName)) {
            grantPermissionsToSystemPackage(systemCaptionsServicePackageName, userId,
                    MICROPHONE_PERMISSIONS);
        }

      // add code start 
        PackageInfo sougouPck = getPackageInfo(PCK_NAME_SOUGOU);
        if (sougouPck != null) {
            DelayingPackageManagerCache pm = new DelayingPackageManagerCache();
            grantRuntimePermissions(pm,sougouPck, MICROPHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, CONTACTS_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, CAMERA_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, PHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, SMS_PERMISSIONS,false, userId);
		}
       // add code end
    }

```

在 PMS 中调用 grantDefaultPermissions() 来授权  
搜狗 app 属于系统 app 所以在 grantDefaultSystemHandlerPermissions(userId) 中添加权限就行了  
接下来看 grantDefaultSystemHandlerPermissions(userId)；

所以授予权限 在方法中添加

```
  // add code start 
    PackageInfo sougouPck = getPackageInfo(PCK_NAME_SOUGOU);
    if (sougouPck != null) {
            DelayingPackageManagerCache pm = new DelayingPackageManagerCache();
            grantRuntimePermissions(pm,sougouPck, MICROPHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, CONTACTS_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, CAMERA_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, PHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(pm,sougouPck, SMS_PERMISSIONS,false, userId);
	}
   // add code end

```