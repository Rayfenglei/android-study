> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126168049)

1. 概述
-----

在进行产品开发中，开启 usb 调试模式和开发者模式，也便于开发工作，所以这就需要通过了解开发者调试时，打开开关时做了哪些功能，然后开启开发者模式和 usb 调试模式。

2. 默认开启开发者模式和 usb 调试模式的相关核心代码
-----------------------------

```
/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java

```

3. 默认开启开发者模式和 usb 调试模式的相关核心代码和功能
--------------------------------

#### 1. 打开 usb 调试模式

    通过字段关闭和开启 usb 调试模式  
通过开启开发者模式最终发现开启或者关闭是改变如下字段:  
Settings.Global.putInt(getContentResolver(),Settings.Global.ADB_ENABLED, 1);

####   2. 开启开发者模式

   可以通过 Settings.GlobalDEVELOPMENT_SETTINGS_ENABLED 属性来实现  
   Settings.Global.putInt(getContentResolver(),Settings.Global.ADB_ENABLED, 1);

####    3. 接下来可以在 DatabaseHelper.java 中添加默认开启这两个属性值

```
@Deprecated
class DatabaseHelper extends SQLiteOpenHelper {
private static final String TAG = "SettingsProvider";
private static final String DATABASE_NAME = "settings.db";
 
// Please, please please. If you update the database version, check to make sure the
// database gets upgraded properly. At a minimum, please confirm that 'upgradeVersion'
// is properly propagated through your change.  Not doing so will result in a loss of user
// settings.
private static final int DATABASE_VERSION = 118;
 
private Context mContext;
private int mUserHandle;
 
private static final HashSet<String> mValidTables = new HashSet<String>();
 
private static final String DATABASE_BACKUP_SUFFIX = "-backup";
 
private static final String TABLE_SYSTEM = "system";
private static final String TABLE_SECURE = "secure";
private static final String TABLE_GLOBAL = "global";
 
static {
mValidTables.add(TABLE_SYSTEM);
mValidTables.add(TABLE_SECURE);
mValidTables.add(TABLE_GLOBAL);
 
// These are old.
mValidTables.add("bluetooth_devices");
mValidTables.add("bookmarks");
mValidTables.add("favorites");
mValidTables.add("old_favorites");
mValidTables.add("android_metadata");
}
 
static String dbNameForUser(final int userHandle) {
// The owner gets the unadorned db name;
if (userHandle == UserHandle.USER_SYSTEM) {
return DATABASE_NAME;
} else {
// Place the database in the user-specific data tree so that it's
// cleaned up automatically when the user is deleted.
File databaseFile = new File(
Environment.getUserSystemDirectory(userHandle), DATABASE_NAME);
// If databaseFile doesn't exist, database can be kept in memory. It's safe because the
// database will be migrated and disposed of immediately after onCreate finishes
if (!databaseFile.exists()) {
Log.i(TAG, "No previous database file exists - running in in-memory mode");
return null;
}
return databaseFile.getPath();
}
}
 
public DatabaseHelper(Context context, int userHandle) {
super(context, dbNameForUser(userHandle), null, DATABASE_VERSION);
mContext = context;
mUserHandle = userHandle;
}
 
public static boolean isValidTable(String name) {
return mValidTables.contains(name);
}
 
@Override
public void onCreate(SQLiteDatabase db) {
db.execSQL("CREATE TABLE system (" +
"_id INTEGER PRIMARY KEY AUTOINCREMENT," +
"name TEXT UNIQUE ON CONFLICT REPLACE," +
"value TEXT" +
");");
db.execSQL("CREATE INDEX systemIndex1 ON system (name);");
 
createSecureTable(db);
 
// Only create the global table for the singleton 'owner/system' user
if (mUserHandle == UserHandle.USER_SYSTEM) {
createGlobalTable(db);
}
 
db.execSQL("CREATE TABLE bluetooth_devices (" +
"_id INTEGER PRIMARY KEY," +
"name TEXT," +
"addr TEXT," +
"channel INTEGER," +
"type INTEGER" +
");");
 
db.execSQL("CREATE TABLE bookmarks (" +
"_id INTEGER PRIMARY KEY," +
"title TEXT," +
"folder TEXT," +
"intent TEXT," +
"shortcut INTEGER," +
"ordering INTEGER" +
");");
 
db.execSQL("CREATE INDEX bookmarksIndex1 ON bookmarks (folder);");
db.execSQL("CREATE INDEX bookmarksIndex2 ON bookmarks (shortcut);");
 
// Populate bookmarks table with initial bookmarks
boolean onlyCore = false;
try {
onlyCore = IPackageManager.Stub.asInterface(ServiceManager.getService(
"package")).isOnlyCoreApps();
} catch (RemoteException e) {
}
if (!onlyCore) {
loadBookmarks(db);
}
 
// Load initial volume levels into DB
loadVolumeLevels(db);
 
// Load inital settings values
loadSettings(db);
}
 
private void loadSettings(SQLiteDatabase db) {
loadSystemSettings(db);
loadSecureSettings(db);
// The global table only exists for the 'owner/system' user
if (mUserHandle == UserHandle.USER_SYSTEM) {
loadGlobalSettings(db);
}
}
 
private void loadSystemSettings(SQLiteDatabase db) {
SQLiteStatement stmt = null;
try {
stmt = db.compileStatement("INSERT OR IGNORE INTO system(name,value)"
+ " VALUES(?,?);");
 
loadBooleanSetting(stmt, Settings.System.DIM_SCREEN,
R.bool.def_dim_screen);
loadIntegerSetting(stmt, Settings.System.SCREEN_OFF_TIMEOUT,
R.integer.def_screen_off_timeout);
 
// Set default cdma DTMF type
loadSetting(stmt, Settings.System.DTMF_TONE_TYPE_WHEN_DIALING, 0);
 
// Set default hearing aid
loadSetting(stmt, Settings.System.HEARING_AID, 0);
 
// Set default tty mode
loadSetting(stmt, Settings.System.TTY_MODE, 0);
 
loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS,
R.integer.def_screen_brightness);
 
loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS_FOR_VR,
com.android.internal.R.integer.config_screenBrightnessForVrSettingDefault);
 
loadBooleanSetting(stmt, Settings.System.SCREEN_BRIGHTNESS_MODE,
R.bool.def_screen_brightness_automatic_mode);
 
loadBooleanSetting(stmt, Settings.System.ACCELEROMETER_ROTATION,
R.bool.def_accelerometer_rotation);
 
loadDefaultHapticSettings(stmt);
 
loadBooleanSetting(stmt, Settings.System.NOTIFICATION_LIGHT_PULSE,
R.bool.def_notification_pulse);
 
loadUISoundEffectsSettings(stmt);
 
loadIntegerSetting(stmt, Settings.System.POINTER_SPEED,
R.integer.def_pointer_speed);
 
/*
* IMPORTANT: Do not add any more upgrade steps here as the global,
* secure, and system settings are no longer stored in a database
* but are kept in memory and persisted to XML.
*
* See: SettingsProvider.UpgradeController#onUpgradeLocked
*/
} finally {
if (stmt != null) stmt.close();
}
}
 
private void loadUISoundEffectsSettings(SQLiteStatement stmt) {
loadBooleanSetting(stmt, Settings.System.DTMF_TONE_WHEN_DIALING,
R.bool.def_dtmf_tones_enabled);
loadBooleanSetting(stmt, Settings.System.SOUND_EFFECTS_ENABLED,
R.bool.def_sound_effects_enabled);
loadBooleanSetting(stmt, Settings.System.HAPTIC_FEEDBACK_ENABLED,
R.bool.def_haptic_feedback);
 
loadIntegerSetting(stmt, Settings.System.LOCKSCREEN_SOUNDS_ENABLED,
R.integer.def_lockscreen_sounds_enabled);
}
 
private void loadDefaultAnimationSettings(SQLiteStatement stmt) {
loadFractionSetting(stmt, Settings.System.WINDOW_ANIMATION_SCALE,
R.fraction.def_window_animation_scale, 1);
loadFractionSetting(stmt, Settings.System.TRANSITION_ANIMATION_SCALE,
R.fraction.def_window_transition_scale, 1);
}
 
private void loadDefaultHapticSettings(SQLiteStatement stmt) {
loadBooleanSetting(stmt, Settings.System.HAPTIC_FEEDBACK_ENABLED,
R.bool.def_haptic_feedback);
}
 
private void loadSecureSettings(SQLiteDatabase db) {
SQLiteStatement stmt = null;
try {
stmt = db.compileStatement("INSERT OR IGNORE INTO secure(name,value)"
+ " VALUES(?,?);");
 
// Don't do this.  The SystemServer will initialize ADB_ENABLED from a
// persistent system property instead.
//loadSetting(stmt, Settings.Secure.ADB_ENABLED, 0);
 
// Allow mock locations default, based on build
loadSetting(stmt, Settings.Secure.ALLOW_MOCK_LOCATION,
"1".equals(SystemProperties.get("ro.allow.mock.location")) ? 1 : 0);
 
loadSecure35Settings(stmt);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_PLAY_NOTIFICATION_SND,
R.bool.def_mount_play_notification_snd);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_AUTOSTART,
R.bool.def_mount_ums_autostart);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_PROMPT,
R.bool.def_mount_ums_prompt);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_NOTIFY_ENABLED,
R.bool.def_mount_ums_notify_enabled);
 
loadIntegerSetting(stmt, Settings.Secure.LONG_PRESS_TIMEOUT,
R.integer.def_long_press_timeout_millis);
 
loadBooleanSetting(stmt, Settings.Secure.TOUCH_EXPLORATION_ENABLED,
R.bool.def_touch_exploration_enabled);
 
loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_SPEAK_PASSWORD,
R.bool.def_accessibility_speak_password);
 
if (SystemProperties.getBoolean("ro.lockscreen.disable.default", false) == true) {
loadSetting(stmt, Settings.System.LOCKSCREEN_DISABLED, "1");
} else {
loadBooleanSetting(stmt, Settings.System.LOCKSCREEN_DISABLED,
R.bool.def_lockscreen_disabled);
}
 
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ENABLED,
com.android.internal.R.bool.config_dreamsEnabledByDefault);
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_DOCK,
com.android.internal.R.bool.config_dreamsActivatedOnDockByDefault);
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_SLEEP,
com.android.internal.R.bool.config_dreamsActivatedOnSleepByDefault);
loadStringSetting(stmt, Settings.Secure.SCREENSAVER_COMPONENTS,
com.android.internal.R.string.config_dreamsDefaultComponent);
loadStringSetting(stmt, Settings.Secure.SCREENSAVER_DEFAULT_COMPONENT,
com.android.internal.R.string.config_dreamsDefaultComponent);
 
loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_ENABLED,
R.bool.def_accessibility_display_magnification_enabled);
 
loadFractionSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_SCALE,
R.fraction.def_accessibility_display_magnification_scale, 1);
 
loadBooleanSetting(stmt, Settings.Secure.USER_SETUP_COMPLETE,
R.bool.def_user_setup_complete);
 
loadStringSetting(stmt, Settings.Secure.IMMERSIVE_MODE_CONFIRMATIONS,
R.string.def_immersive_mode_confirmations);
 
loadBooleanSetting(stmt, Settings.Secure.INSTALL_NON_MARKET_APPS,
R.bool.def_install_non_market_apps);
 
loadBooleanSetting(stmt, Settings.Secure.WAKE_GESTURE_ENABLED,
R.bool.def_wake_gesture_enabled);
 
loadIntegerSetting(stmt, Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS,
R.integer.def_lock_screen_show_notifications);
 
loadBooleanSetting(stmt, Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS,
R.bool.def_lock_screen_allow_private_notifications);
 
loadIntegerSetting(stmt, Settings.Secure.SLEEP_TIMEOUT,
R.integer.def_sleep_timeout);
 
/*
* IMPORTANT: Do not add any more upgrade steps here as the global,
* secure, and system settings are no longer stored in a database
* but are kept in memory and persisted to XML.
*
* See: SettingsProvider.UpgradeController#onUpgradeLocked
*/
} finally {
if (stmt != null) stmt.close();
}
}
}
 
在loadGlobalSettings(SQLiteDatabase db) 中
添加开启开发者模式和usb调试属性值
private void loadSecureSettings(SQLiteDatabase db) {
SQLiteStatement stmt = null;
try {
stmt = db.compileStatement("INSERT OR IGNORE INTO secure(name,value)"
+ " VALUES(?,?);");
 
// Don't do this.  The SystemServer will initialize ADB_ENABLED from a
// persistent system property instead.
//loadSetting(stmt, Settings.Secure.ADB_ENABLED, 0);
 
// Allow mock locations default, based on build
loadSetting(stmt, Settings.Secure.ALLOW_MOCK_LOCATION,
"1".equals(SystemProperties.get("ro.allow.mock.location")) ? 1 : 0);
 
 
// add core start
loadSetting(stmt, Settings.Global.ADB_ENABLED, 1);
loadSetting(stmt, Settings.Global.DEVELOPMENT_SETTINGS_ENABLED, 1);
//add core end
 
 
loadSecure35Settings(stmt);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_PLAY_NOTIFICATION_SND,
R.bool.def_mount_play_notification_snd);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_AUTOSTART,
R.bool.def_mount_ums_autostart);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_PROMPT,
R.bool.def_mount_ums_prompt);
 
loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_NOTIFY_ENABLED,
R.bool.def_mount_ums_notify_enabled);
 
loadIntegerSetting(stmt, Settings.Secure.LONG_PRESS_TIMEOUT,
R.integer.def_long_press_timeout_millis);
 
loadBooleanSetting(stmt, Settings.Secure.TOUCH_EXPLORATION_ENABLED,
R.bool.def_touch_exploration_enabled);
 
loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_SPEAK_PASSWORD,
R.bool.def_accessibility_speak_password);
 
if (SystemProperties.getBoolean("ro.lockscreen.disable.default", false) == true) {
loadSetting(stmt, Settings.System.LOCKSCREEN_DISABLED, "1");
} else {
loadBooleanSetting(stmt, Settings.System.LOCKSCREEN_DISABLED,
R.bool.def_lockscreen_disabled);
}
 
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ENABLED,
com.android.internal.R.bool.config_dreamsEnabledByDefault);
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_DOCK,
com.android.internal.R.bool.config_dreamsActivatedOnDockByDefault);
loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_SLEEP,
com.android.internal.R.bool.config_dreamsActivatedOnSleepByDefault);
loadStringSetting(stmt, Settings.Secure.SCREENSAVER_COMPONENTS,
com.android.internal.R.string.config_dreamsDefaultComponent);
loadStringSetting(stmt, Settings.Secure.SCREENSAVER_DEFAULT_COMPONENT,
com.android.internal.R.string.config_dreamsDefaultComponent);
 
loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_ENABLED,
R.bool.def_accessibility_display_magnification_enabled);
 
loadFractionSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_SCALE,
R.fraction.def_accessibility_display_magnification_scale, 1);
 
loadBooleanSetting(stmt, Settings.Secure.USER_SETUP_COMPLETE,
R.bool.def_user_setup_complete);
 
loadStringSetting(stmt, Settings.Secure.IMMERSIVE_MODE_CONFIRMATIONS,
R.string.def_immersive_mode_confirmations);
 
loadBooleanSetting(stmt, Settings.Secure.INSTALL_NON_MARKET_APPS,
R.bool.def_install_non_market_apps);
 
loadBooleanSetting(stmt, Settings.Secure.WAKE_GESTURE_ENABLED,
R.bool.def_wake_gesture_enabled);
 
loadIntegerSetting(stmt, Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS,
R.integer.def_lock_screen_show_notifications);
 
loadBooleanSetting(stmt, Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS,
R.bool.def_lock_screen_allow_private_notifications);
 
loadIntegerSetting(stmt, Settings.Secure.SLEEP_TIMEOUT,
R.integer.def_sleep_timeout);
 
/*
* IMPORTANT: Do not add any more upgrade steps here as the global,
* secure, and system settings are no longer stored in a database
* but are kept in memory and persisted to XML.
*
* See: SettingsProvider.UpgradeController#onUpgradeLocked
*/
} finally {
if (stmt != null) stmt.close();
}
}
}
```