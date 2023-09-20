> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872800)

11.0 默认开启相关权限  
AppOpsManager 介绍  
  AppOpsManager 是 Google 在 Android4.3 里面引进的应用程序操作（权限）的管理类，核心实现类为 AppOpsService。  
app op（应用操作）的出现比运行时权限早，最初在没有出现运行时权限的时候，应用一旦被安装成功，是会被一次性授予所有需要的权限的，所以限制应用权限的唯一方案是使用 AppOpsManager。

AppOpsManager 重要变量

sOpToSwitch

左边的 op code 是开关，右边的注释是左边开关可以控制的 op code。例如，OP_COARSE_LOCATION 这个 op code 可以控制 OP_COARSE_LOCATION，OP_FINE_LOCATION 和 OP_GPS 三个 op code。sOpToSwitch 数组也有 91 个，和 op code 的内容是递增对应的。

sOpPerms

sOpPerms 和 sOpToSwitch 一样，和 op code 的内容时递增对应的。sOpPerms 是一个运行时和签名权限[字符串数组](https://so.csdn.net/so/search?q=%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%95%B0%E7%BB%84&spm=1001.2101.3001.7020)，和 op code 的内容映射。例如，OP_COARSE_LOCATION 映射 android.Manifest.permission.ACCESS_COARSE_LOCATION 权限，而 OP_GPS 映射为 null，说明没有对应的权限。

sOpToString

sOpToString 描述了 op code 和描述字符串的映射。

sOpDefaultMode

sOpDefaultMode 描述了一个 op code 的默认授权情况，例如 OP_COARSE_LOCATION 的默认授权情况总是 MODE_ALLOWED 的。

sOpStrToOp

sOpStrToOp 是 op 描述字符串对 op code 的映射。

sPermToOp

sPermToOp 是权限名对 op code 的映射。

sOpRestrictions

op code 对用户限制的映射，用户限制可以为 null。如果一个 op code 被添加了用户限制，那么在限制用户下使用 startOp/noteOp/unsafeCheckOp 是返回 AppOpsManager.MODE_IGNORED 的。如下面所示，OP_COARSE_LOCATION 这个 op code 映射了 DISALLOW_SHARE_LOCATION，但是这个用户限制不一定生效，还需要使用 DevicePolicyManager#addUserRestriction(ComponentName, String) 设置后才会生效。

sOpAllowSystemRestrictionBypass

sOpAllowSystemRestrictionBypass 描述了是否允许系统组件绕过用户限制（在用户限制被激活的情况下）。

sOpDisableReset

sOpDisableReset 用来指定是否允许在重置所有应用偏好设置后，重置 Operation 的授予情况，true 表示禁止重置，false 表示允许重置。

根据综上所述 每个成员的关系后，默认授予权限 就需要在每个成员修改对应的权限就可以了  
所以解决方案如下:

```
+++ b/frameworks/base/core/java/android/app/AppOpsManager.java
@@ -1704,10 +1704,10 @@ public class AppOpsManager {
             false, //SEND_SMS
             false, //READ_ICC_SMS
             false, //WRITE_ICC_SMS
-            false, //WRITE_SETTINGS
+            true, //WRITE_SETTINGS
             true, //SYSTEM_ALERT_WINDOW
             false, //ACCESS_NOTIFICATIONS
-            false, //CAMERA
+            true, //CAMERA
             false, //RECORD_AUDIO
             false, //PLAY_AUDIO
             false, //READ_CLIPBOARD
@@ -1740,14 +1740,14 @@ public class AppOpsManager {
             false, // BODY_SENSORS
             false, // READ_CELL_BROADCASTS
             false, // MOCK_LOCATION
-            false, // READ_EXTERNAL_STORAGE
-            false, // WRITE_EXTERNAL_STORAGE
+            true, // READ_EXTERNAL_STORAGE
+            true, // WRITE_EXTERNAL_STORAGE
             false, // TURN_ON_SCREEN
             false, // GET_ACCOUNTS
             false, // RUN_IN_BACKGROUND
             false, // AUDIO_ACCESSIBILITY_VOLUME
             false, // READ_PHONE_NUMBERS
-            false, // REQUEST_INSTALL_PACKAGES
+            true, // REQUEST_INSTALL_PACKAGES
             false, // ENTER_PICTURE_IN_PICTURE_ON_HIDE
             false, // INSTANT_APP_START_FOREGROUND
             false, // ANSWER_PHONE_CALLS
@@ -1801,8 +1801,8 @@ public class AppOpsManager {
             AppOpsManager.MODE_ALLOWED, // SEND_SMS
             AppOpsManager.MODE_ALLOWED, // READ_ICC_SMS
             AppOpsManager.MODE_ALLOWED, // WRITE_ICC_SMS
-            AppOpsManager.MODE_DEFAULT, // WRITE_SETTINGS
-            getSystemAlertWindowDefault(), // SYSTEM_ALERT_WINDOW
+            AppOpsManager.MODE_ALLOWED, // WRITE_SETTINGS
+            AppOpsManager.MODE_ALLOWED, // SYSTEM_ALERT_WINDOW
             AppOpsManager.MODE_ALLOWED, // ACCESS_NOTIFICATIONS
             AppOpsManager.MODE_ALLOWED, // CAMERA
             AppOpsManager.MODE_ALLOWED, // RECORD_AUDIO
@@ -1844,7 +1844,7 @@ public class AppOpsManager {
             AppOpsManager.MODE_ALLOWED, // RUN_IN_BACKGROUND
             AppOpsManager.MODE_ALLOWED, // AUDIO_ACCESSIBILITY_VOLUME
             AppOpsManager.MODE_ALLOWED, // READ_PHONE_NUMBERS
-            AppOpsManager.MODE_DEFAULT, // REQUEST_INSTALL_PACKAGES
+            AppOpsManager.MODE_ALLOWED, // REQUEST_INSTALL_PACKAGES
             AppOpsManager.MODE_ALLOWED, // PICTURE_IN_PICTURE
             AppOpsManager.MODE_DEFAULT, // INSTANT_APP_START_FOREGROUND
             AppOpsManager.MODE_ALLOWED, // ANSWER_PHONE_CALLS
@@ -1879,8 +1879,8 @@ public class AppOpsManager {
      * for whichever app is selected as the current SMS app).
      */
     private static boolean[] sOpDisableReset = new boolean[] {
-            false, // COARSE_LOCATION
-            false, // FINE_LOCATION
+            true, // COARSE_LOCATION
+            true, // FINE_LOCATION
             false, // GPS
             false, // VIBRATE
             false, // READ_CONTACTS
@@ -1902,10 +1902,10 @@ public class AppOpsManager {
             true, // SEND_SMS
             false, // READ_ICC_SMS
             false, // WRITE_ICC_SMS
-            false, // WRITE_SETTINGS
-            false, // SYSTEM_ALERT_WINDOW
+            true, // WRITE_SETTINGS
+            true, // SYSTEM_ALERT_WINDOW
             false, // ACCESS_NOTIFICATIONS
-            false, // CAMERA
+            true, // CAMERA
             false, // RECORD_AUDIO
             false, // PLAY_AUDIO
             false, // READ_CLIPBOARD
@@ -1938,8 +1938,8 @@ public class AppOpsManager {
             false, // BODY_SENSORS
             true, // READ_CELL_BROADCASTS
             false, // MOCK_LOCATION
-            false, // READ_EXTERNAL_STORAGE
-            false, // WRITE_EXTERNAL_STORAGE
+            true, // READ_EXTERNAL_STORAGE
+            true, // WRITE_EXTERNAL_STORAGE
             false, // TURN_SCREEN_ON
             false, // GET_ACCOUNTS
             false, // RUN_IN_BACKGROUND

```