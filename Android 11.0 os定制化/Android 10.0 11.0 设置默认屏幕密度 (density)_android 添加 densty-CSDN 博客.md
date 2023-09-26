> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124617371)

定制化开发中需要增加属性来保存默认屏幕密度 density

1. 首选就是要在 SettingsProvider 把默认值添加到数据库中  
先在 SettingsProvider 的 default.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 文件中配置 display_density_forced 的值  
diff --git a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml  
index 3ba4f34…208a26b 100755  
— a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml  
+++ b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml  
@@ -17,6 +17,8 @@  
*/  
–>

```
<resources>
+ <string >320</string>
<bool >true</bool>
<integer >0</integer>
<integer >-1</integer>

```

2. 在加载数据库时把密度值添加到数据库中  
在 DatabaseHelper.java 的 loadSystemSettings() 中添加数据到数据库  
private void loadSystemSettings(SQLiteDatabase db) {  
…  
}

```
diff --git a/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java b/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
index b402e73..1031071 100755
--- a/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
+++ b/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
@@ -2555,6 +2555,14 @@ class DatabaseHelper extends SQLiteOpenHelper {    loadIntegerSetting(stmt, Settings.Secure.SLEEP_TIMEOUT,
            R.integer.def_sleep_timeout);

              // 加载屏幕密度值

              loadStringSetting(stmt, Settings.Secure.DISPLAY_DENSITY_FORCED,

                              R.string.display_density_forced);

```