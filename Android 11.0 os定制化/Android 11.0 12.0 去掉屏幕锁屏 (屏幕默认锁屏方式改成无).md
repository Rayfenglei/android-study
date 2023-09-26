> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124992865)

### 1. 概述

在 11.0 12.0 定制化开发中, 需求要求去掉屏幕锁屏功能，默认无锁屏功能，所以要去掉系统默认锁屏功能  
分两步：  
1.1 在 SettingProvider 数据库加载时默认无锁屏  
1.2 去掉 Settings 中关于选择锁屏的方式

### 2. 去掉屏幕锁屏 (屏幕默认锁屏方式改成无) 核心代码

```
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
packages/apps/Settings/res/xml/security_settings_picker.xml

```

### 3. 去掉屏幕锁屏 (屏幕默认锁屏方式改成无) 功能分析和实现

### 3.1 关于 SettingProvider 关于去掉锁屏功能的分析

第一部分去掉系统数据库关于锁屏的方式默认无锁屏  
属性修改 SettingProvider 关于去掉锁屏功能的分析

def_lockscreen_disabled 属性就是在第一次开机的时候默认设置是否需要锁屏  
在 loadSecureSettings(SQLiteDatabase db) 中加载这个默认值如下

```
                loadBooleanSetting(stmt, Settings.System.LOCKSCREEN_DISABLED,
                        R.bool.def_lockscreen_disabled);
  具体修改部分：                      
--- a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
+++ b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
@@ -84,7 +84,7 @@
<integer >1000</integer>
<integer >15000</integer>

-    <bool >false</bool>
+    <bool >true</bool>
<bool >false</bool>
<integer >1</integer>

```

def_lockscreen_disabled 就是是否开启锁屏 设为 false 即可

### 3.2 Settings 中去掉其他锁屏的方式

Settings 去掉其他锁屏方式  
security_settings_picker.xml 为屏幕锁屏页面对应的 xml 文件  
xml 中去掉锁屏方式  
保留第一项 unlock_set_off 就可以了 这就是无锁屏选项

```
<PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
        android:title="@string/lock_settings_picker_title"
        android:key="lock_settings_picker">

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_off"
            android:title="@string/unlock_set_unlock_off_title"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_none"
            android:title="@string/unlock_set_unlock_none_title"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_pattern"
            android:title="@string/unlock_set_unlock_pattern_title"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_pin"
            android:title="@string/unlock_set_unlock_pin_title"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_password"
            android:title="@string/unlock_set_unlock_password_title"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_set_managed"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_skip_fingerprint"
            android:title="@string/fingerprint_unlock_skip_fingerprint"
            android:persistent="false"/>

    <com.android.settingslib.RestrictedPreference
            android:key="unlock_skip_face"
            android:title="@string/face_unlock_skip_face"
            android:persistent="false"/>

</PreferenceScreen>
具体修改：
diff --git a/packages/apps/Settings/res/xml/security_settings_picker.xml b/packages/apps/Settings/res/xml/security_settings_picker.xml
old mode 100644
new mode 100755
index 2e6361aca4..2fc36b184d
--- a/packages/apps/Settings/res/xml/security_settings_picker.xml
+++ b/packages/apps/Settings/res/xml/security_settings_picker.xml
@@ -23,38 +23,38 @@
android:title="@string/unlock_set_unlock_off_title"
android:persistent="false"/>

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_set_none"
android:title="@string/unlock_set_unlock_none_title"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_set_pattern"
android:title="@string/unlock_set_unlock_pattern_title"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_set_pin"
android:title="@string/unlock_set_unlock_pin_title"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_set_password"
android:title="@string/unlock_set_unlock_password_title"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_set_managed"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_skip_fingerprint"
android:title="@string/fingerprint_unlock_skip_fingerprint"
-            android:persistent="false"/>
+            android:persistent="false"/-->

-    <com.android.settingslib.RestrictedPreference
+    <!--com.android.settingslib.RestrictedPreference
android:key="unlock_skip_face"
android:title="@string/face_unlock_skip_face"
-            android:persistent="false"/>
+            android:persistent="false"/-->

</PreferenceScreen>

```

### 3.3ChooseLockGeneric.java 中去掉锁屏方式的加载

去掉 ChooseLockGeneric 选择锁屏加载方式的加载

```
diff --git a/packages/apps/Settings/src/com/android/settings/password/ChooseLockGeneric.java b/packages/apps/Settings/src/com/android/settings/password/C
hooseLockGeneric.java
old mode 100644
new mode 100755
index e9cc9b770a..4a84def4dc
--- a/packages/apps/Settings/src/com/android/settings/password/ChooseLockGeneric.java
+++ b/packages/apps/Settings/src/com/android/settings/password/ChooseLockGeneric.java
@@ -511,10 +511,10 @@ public class ChooseLockGeneric extends SettingsActivity {

// Used for testing purposes
findPreference(ScreenLockType.NONE.preferenceKey).setViewId(R.id.lock_none);
-            findPreference(KEY_SKIP_FINGERPRINT).setViewId(R.id.lock_none);
-            findPreference(KEY_SKIP_FACE).setViewId(R.id.lock_none);
-            findPreference(ScreenLockType.PIN.preferenceKey).setViewId(R.id.lock_pin);
-            findPreference(ScreenLockType.PASSWORD.preferenceKey).setViewId(R.id.lock_password);
+            //findPreference(KEY_SKIP_FINGERPRINT).setViewId(R.id.lock_none);
+            //findPreference(KEY_SKIP_FACE).setViewId(R.id.lock_none);
+            //findPreference(ScreenLockType.PIN.preferenceKey).setViewId(R.id.lock_pin);
+            //findPreference(ScreenLockType.PASSWORD.preferenceKey).setViewId(R.id.lock_password);
}

private String getFooterString() {
@@ -540,24 +540,24 @@ public class ChooseLockGeneric extends SettingsActivity {

private void updatePreferenceText() {
if (mForFingerprint) {
-                setPreferenceTitle(ScreenLockType.PATTERN,
+                /*setPreferenceTitle(ScreenLockType.PATTERN,
R.string.fingerprint_unlock_set_unlock_pattern);
setPreferenceTitle(ScreenLockType.PIN, R.string.fingerprint_unlock_set_unlock_pin);
setPreferenceTitle(ScreenLockType.PASSWORD,
-                        R.string.fingerprint_unlock_set_unlock_password);
+                        R.string.fingerprint_unlock_set_unlock_password);*/
} else if (mForFace) {
-                setPreferenceTitle(ScreenLockType.PATTERN,
+                /*setPreferenceTitle(ScreenLockType.PATTERN,
R.string.face_unlock_set_unlock_pattern);
setPreferenceTitle(ScreenLockType.PIN, R.string.face_unlock_set_unlock_pin);
setPreferenceTitle(ScreenLockType.PASSWORD,
-                        R.string.face_unlock_set_unlock_password);
+                        R.string.face_unlock_set_unlock_password);*/
}

if (mManagedPasswordProvider.isSettingManagedPasswordSupported()) {
-                setPreferenceTitle(ScreenLockType.MANAGED,
-                        mManagedPasswordProvider.getPickerOptionTitle(mForFingerprint));
+                /*setPreferenceTitle(ScreenLockType.MANAGED,
+                        mManagedPasswordProvider.getPickerOptionTitle(mForFingerprint));*/
} else {
-                removePreference(ScreenLockType.MANAGED.preferenceKey);
+                //removePreference(ScreenLockType.MANAGED.preferenceKey);
}

if (!(mForFingerprint && mIsSetNewPassword)) {
@@ -679,10 +679,10 @@ public class ChooseLockGeneric extends SettingsActivity {
return;
}

-            setPreferenceSummary(ScreenLockType.PATTERN, R.string.secure_lock_encryption_warning);
+            /*setPreferenceSummary(ScreenLockType.PATTERN, R.string.secure_lock_encryption_warning);
setPreferenceSummary(ScreenLockType.PIN, R.string.secure_lock_encryption_warning);
setPreferenceSummary(ScreenLockType.PASSWORD, R.string.secure_lock_encryption_warning);
-            setPreferenceSummary(ScreenLockType.MANAGED, R.string.secure_lock_encryption_warning);
+            setPreferenceSummary(ScreenLockType.MANAGED, R.string.secure_lock_encryption_warning);*/
}

protected Intent getLockManagedPasswordIntent(byte[] password) {

```

在 updatePreferenceText() 中去掉选择其他的锁屏方式