> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126110758)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.usb 键盘和软键盘兼容的核心代码](#t1)

[3.usb 键盘和软键盘兼容的核心代码](#t2)

 [3.1 SettingsProvider.java 软键盘禁用的核心代码](#t3)

[而这里设置 final SettingsState secureSettings = getSecureSettingsLocked(userId);Setting currentSetting = secureSettings.getSettingLocked(Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD);if (currentSetting.isNull()) {secureSettings.insertSettingOverrideableByRestoreLocked(Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD,getContext().getResources().getBoolean(R.bool.def_show_ime_with_hard_keyboard) ? "1" : "0",null, true, SettingsState.SYSTEM_PACKAGE_NAME);} 是否禁用软键盘 由 def_show_ime_with_hard_keyboard 属性决定的 默认是禁用软键盘的 所以开启就可以了](#%E8%80%8C%E8%BF%99%E9%87%8C%E8%AE%BE%E7%BD%AEfinal%20SettingsState%20secureSettings%20%3D%20getSecureSettingsLocked%28userId%29%3BSetting%20currentSetting%20%3D%20secureSettings.getSettingLocked%28Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD%29%3Bif%20%28currentSetting.isNull%28%29%29%20%7BsecureSettings.insertSettingOverrideableByRestoreLocked%28Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD%2CgetContext%28%29.getResources%28%29.getBoolean%28R.bool.def_show_ime_with_hard_keyboard%29%20%3F%20%221%22%20%3A%20%220%22%2Cnull%2C%20true%2C%20SettingsState.SYSTEM_PACKAGE_NAME%29%3B%7D%E6%98%AF%E5%90%A6%E7%A6%81%E7%94%A8%E8%BD%AF%E9%94%AE%E7%9B%98%20%E7%94%B1def_show_ime_with_hard_keyboard%E5%B1%9E%E6%80%A7%E5%86%B3%E5%AE%9A%E7%9A%84%20%E9%BB%98%E8%AE%A4%E6%98%AF%E7%A6%81%E7%94%A8%E8%BD%AF%E9%94%AE%E7%9B%98%E7%9A%84%20%E6%89%80%E4%BB%A5%E5%BC%80%E5%90%AF%E5%B0%B1%E5%8F%AF%E4%BB%A5%E4%BA%86)

[3.2 defaults.xml 关于是否禁用软键盘](#t5)

1. 概述
-----

在 11.0 产品开发中. 产品 usb 口有时候会使用到 usb 键盘，但是在使用 usb 键盘的过程中会发现软键盘使用不了两种相互冲突. 当插入 usb 物理键盘时，系统默认会禁用软键盘，所以需要做下兼容性，让物理键盘和软键盘都可以使用

2.usb 键盘和软键盘兼容的核心代码
-------------------

```
   /frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java
   /frameworks/base/packages/SettingsProvider/res/values/defaults.xml
```

3.usb 键盘和软键盘兼容的核心代码
-------------------

####    3.1 SettingsProvider.java 软键盘禁用的核心代码

```
private final class UpgradeController {
private static final int SETTINGS_VERSION = 191;
 
private final int mUserId;
 
public UpgradeController(int userId) {
mUserId = userId;
}
 
public void upgradeIfNeededLocked() {
// The version of all settings for a user is the same (all users have secure).
SettingsState secureSettings = getSettingsLocked(
SETTINGS_TYPE_SECURE, mUserId);
 
// Try an update from the current state.
final int oldVersion = secureSettings.getVersionLocked();
final int newVersion = SETTINGS_VERSION;
 
// If up do date - done.
if (oldVersion == newVersion) {
return;
}
 
// Try to upgrade.
final int curVersion = onUpgradeLocked(mUserId, oldVersion, newVersion);
 
// If upgrade failed start from scratch and upgrade.
if (curVersion != newVersion) {
// Drop state we have for this user.
removeUserStateLocked(mUserId, true);
 
// Recreate the database.
DatabaseHelper dbHelper = new DatabaseHelper(getContext(), mUserId);
SQLiteDatabase database = dbHelper.getWritableDatabase();
dbHelper.recreateDatabase(database, newVersion, curVersion, oldVersion);
 
// Migrate the settings for this user.
migrateLegacySettingsForUserLocked(dbHelper, database, mUserId);
 
// Now upgrade should work fine.
onUpgradeLocked(mUserId, oldVersion, newVersion);
 
// Make a note what happened, so we don't wonder why data was lost
String reason = "Settings rebuilt! Current version: "
+ curVersion + " while expected: " + newVersion;
getGlobalSettingsLocked().insertSettingLocked(
Settings.Global.DATABASE_DOWNGRADE_REASON,
reason, null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
// Set the global settings version if owner.
if (mUserId == UserHandle.USER_SYSTEM) {
SettingsState globalSettings = getSettingsLocked(
SETTINGS_TYPE_GLOBAL, mUserId);
globalSettings.setVersionLocked(newVersion);
}
// Set the secure settings version.
secureSettings.setVersionLocked(newVersion);
// Set the system settings version.
SettingsState systemSettings = getSettingsLocked(
SETTINGS_TYPE_SYSTEM, mUserId);
systemSettings.setVersionLocked(newVersion);
}
private SettingsState getGlobalSettingsLocked() {
return getSettingsLocked(SETTINGS_TYPE_GLOBAL, UserHandle.USER_SYSTEM);
}
private SettingsState getSecureSettingsLocked(int userId) {
return getSettingsLocked(SETTINGS_TYPE_SECURE, userId);
}
private SettingsState getSsaidSettingsLocked(int userId) {
return getSettingsLocked(SETTINGS_TYPE_SSAID, userId);
}
private SettingsState getSystemSettingsLocked(int userId) {
return getSettingsLocked(SETTINGS_TYPE_SYSTEM, userId);
}
/**
* You must perform all necessary mutations to bring the settings
* for this user from the old to the new version. When you add a new
* upgrade step you *must* update SETTINGS_VERSION.
*
* All settings modifications should be made through
* {@link SettingsState#insertSettingOverrideableByRestoreLocked(String, String, String,
* boolean, String)} so that restore can override those values if needed.
*
* This is an example of moving a setting from secure to global.
*
* // v119: Example settings changes.
* if (currentVersion == 118) {
*     if (userId == UserHandle.USER_OWNER) {
*         // Remove from the secure settings.
*         SettingsState secureSettings = getSecureSettingsLocked(userId);
*         String name = "example_setting_to_move";
*         String value = secureSettings.getSetting(name);
*         secureSettings.deleteSetting(name);
*
*         // Add to the global settings.
*         SettingsState globalSettings = getGlobalSettingsLocked();
*         globalSettings.insertSetting(name, value, SettingsState.SYSTEM_PACKAGE_NAME);
*     }
*
*     // Update the current version.
*     currentVersion = 119;
* }
*/
private int onUpgradeLocked(int userId, int oldVersion, int newVersion) {
if (DEBUG) {
Slog.w(LOG_TAG, "Upgrading settings for user: " + userId + " from version: "
+ oldVersion + " to version: " + newVersion);
}
int currentVersion = oldVersion;
// v119: Reset zen + ringer mode.
if (currentVersion == 118) {
if (userId == UserHandle.USER_SYSTEM) {
final SettingsState globalSettings = getGlobalSettingsLocked();
globalSettings.updateSettingLocked(Settings.Global.ZEN_MODE,
Integer.toString(Settings.Global.ZEN_MODE_OFF), null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
globalSettings.updateSettingLocked(Settings.Global.MODE_RINGER,
Integer.toString(AudioManager.RINGER_MODE_NORMAL), null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 119;
}
// v120: Add double tap to wake setting.
if (currentVersion == 119) {
SettingsState secureSettings = getSecureSettingsLocked(userId);
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.DOUBLE_TAP_TO_WAKE,
getContext().getResources().getBoolean(
R.bool.def_double_tap_to_wake) ? "1" : "0", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
currentVersion = 120;
}
if (currentVersion == 120) {
// Before 121, we used a different string encoding logic.  We just bump the
// version here; SettingsState knows how to handle pre-version 120 files.
currentVersion = 121;
}
if (currentVersion == 121) {
// Version 122: allow OEMs to set a default payment component in resources.
// Note that we only write the default if no default has been set;
// if there is, we just leave the default at whatever it currently is.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
String defaultComponent = (getContext().getResources().getString(
R.string.def_nfc_payment_component));
Setting currentSetting = secureSettings.getSettingLocked(
Settings.Secure.NFC_PAYMENT_DEFAULT_COMPONENT);
if (defaultComponent != null && !defaultComponent.isEmpty() &&
currentSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.NFC_PAYMENT_DEFAULT_COMPONENT,
defaultComponent, null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 122;
}
if (currentVersion == 122) {
// Version 123: Adding a default value for the ability to add a user from
// the lock screen.
if (userId == UserHandle.USER_SYSTEM) {
final SettingsState globalSettings = getGlobalSettingsLocked();
Setting currentSetting = globalSettings.getSettingLocked(
Settings.Global.ADD_USERS_WHEN_LOCKED);
if (currentSetting.isNull()) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.ADD_USERS_WHEN_LOCKED,
getContext().getResources().getBoolean(
R.bool.def_add_users_from_lockscreen) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 123;
}
if (currentVersion == 123) {
final SettingsState globalSettings = getGlobalSettingsLocked();
String defaultDisabledProfiles = (getContext().getResources().getString(
R.string.def_bluetooth_disabled_profiles));
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.BLUETOOTH_DISABLED_PROFILES, defaultDisabledProfiles,
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
currentVersion = 124;
}
if (currentVersion == 124) {
// Version 124: allow OEMs to set a default value for whether IME should be
// shown when a physical keyboard is connected.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
Setting currentSetting = secureSettings.getSettingLocked(
Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD);
if (currentSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD,
getContext().getResources().getBoolean(
R.bool.def_show_ime_with_hard_keyboard) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 125;
}
if (currentVersion == 125) {
// Version 125: Allow OEMs to set the default VR service.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
Setting currentSetting = secureSettings.getSettingLocked(
Settings.Secure.ENABLED_VR_LISTENERS);
if (currentSetting.isNull()) {
ArraySet<ComponentName> l =
SystemConfig.getInstance().getDefaultVrComponents();
if (l != null && !l.isEmpty()) {
StringBuilder b = new StringBuilder();
boolean start = true;
for (ComponentName c : l) {
if (!start) {
b.append(':');
}
b.append(c.flattenToString());
start = false;
}
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.ENABLED_VR_LISTENERS, b.toString(),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 126;
}
if (currentVersion == 126) {
// Version 126: copy the primary values of LOCK_SCREEN_SHOW_NOTIFICATIONS and
// LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS into managed profile.
if (mUserManager.isManagedProfile(userId)) {
final SettingsState systemSecureSettings =
getSecureSettingsLocked(UserHandle.USER_SYSTEM);
final Setting showNotifications = systemSecureSettings.getSettingLocked(
Settings.Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS);
if (!showNotifications.isNull()) {
final SettingsState secureSettings = getSecureSettingsLocked(userId);
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS,
showNotifications.getValue(), null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
final Setting allowPrivate = systemSecureSettings.getSettingLocked(
Settings.Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS);
if (!allowPrivate.isNull()) {
final SettingsState secureSettings = getSecureSettingsLocked(userId);
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS,
allowPrivate.getValue(), null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 127;
}
if (currentVersion == 127) {
// version 127 is no longer used.
currentVersion = 128;
}
if (currentVersion == 128) {
// Version 128: Removed
currentVersion = 129;
}
if (currentVersion == 129) {
// default longpress timeout changed from 500 to 400. If unchanged from the old
// default, update to the new default.
final SettingsState systemSecureSettings =
getSecureSettingsLocked(userId);
final String oldValue = systemSecureSettings.getSettingLocked(
Settings.Secure.LONG_PRESS_TIMEOUT).getValue();
if (TextUtils.equals("500", oldValue)) {
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.LONG_PRESS_TIMEOUT,
String.valueOf(getContext().getResources().getInteger(
R.integer.def_long_press_timeout_millis)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 130;
}
if (currentVersion == 130) {
// Split Ambient settings
final SettingsState secureSettings = getSecureSettingsLocked(userId);
boolean dozeExplicitlyDisabled = "0".equals(secureSettings.
getSettingLocked(Settings.Secure.DOZE_ENABLED).getValue());
if (dozeExplicitlyDisabled) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.DOZE_PICK_UP_GESTURE, "0", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.DOZE_DOUBLE_TAP_GESTURE, "0", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 131;
}
if (currentVersion == 131) {
// Initialize new multi-press timeout to default value
final SettingsState systemSecureSettings = getSecureSettingsLocked(userId);
final String oldValue = systemSecureSettings.getSettingLocked(
Settings.Secure.MULTI_PRESS_TIMEOUT).getValue();
if (TextUtils.equals(null, oldValue)) {
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.MULTI_PRESS_TIMEOUT,
String.valueOf(getContext().getResources().getInteger(
R.integer.def_multi_press_timeout_millis)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 132;
}
if (currentVersion == 132) {
// Version 132: Allow managed profile to optionally use the parent's ringtones
final SettingsState systemSecureSettings = getSecureSettingsLocked(userId);
String defaultSyncParentSounds = (getContext().getResources()
getBoolean(R.bool.def_sync_parent_sounds) ? "1" : "0");
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.SYNC_PARENT_SOUNDS, defaultSyncParentSounds,
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
currentVersion = 133;
}
 
if (currentVersion == 133) {
// Version 133: Add default end button behavior
final SettingsState systemSettings = getSystemSettingsLocked(userId);
if (systemSettings.getSettingLocked(Settings.System.END_BUTTON_BEHAVIOR)
isNull()) {
String defaultEndButtonBehavior = Integer.toString(getContext()
getResources().getInteger(R.integer.def_end_button_behavior));
systemSettings.insertSettingOverrideableByRestoreLocked(
Settings.System.END_BUTTON_BEHAVIOR, defaultEndButtonBehavior, null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 134;
}
 
if (currentVersion == 134) {
// Remove setting that specifies if magnification values should be preserved.
// This setting defaulted to true and never has a UI.
getSecureSettingsLocked(userId).deleteSettingLocked(
Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_AUTO_UPDATE);
currentVersion = 135;
}
 
if (currentVersion == 135) {
// Version 135 no longer used.
currentVersion = 136;
}
 
if (currentVersion == 136) {
// Version 136: Store legacy SSAID for all apps currently installed on the
// device as first step in migrating SSAID to be unique per application.
 
final boolean isUpgrade;
try {
isUpgrade = mPackageManager.isDeviceUpgrading();
} catch (RemoteException e) {
throw new IllegalStateException("Package manager not available");
}
// Only retain legacy ssaid if the device is performing an OTA. After wiping
// user data or first boot on a new device should use new ssaid generation.
if (isUpgrade) {
// Retrieve the legacy ssaid from the secure settings table.
final Setting legacySsaidSetting = getSettingLocked(SETTINGS_TYPE_SECURE,
userId, Settings.Secure.ANDROID_ID);
if (legacySsaidSetting == null || legacySsaidSetting.isNull()
|| legacySsaidSetting.getValue() == null) {
throw new IllegalStateException("Legacy ssaid not accessible");
}
final String legacySsaid = legacySsaidSetting.getValue();
 
// Fill each uid with the legacy ssaid to be backwards compatible.
final List<PackageInfo> packages;
try {
packages = mPackageManager.getInstalledPackages(
PackageManager.MATCH_UNINSTALLED_PACKAGES,
userId).getList();
} catch (RemoteException e) {
throw new IllegalStateException("Package manager not available");
}
 
final SettingsState ssaidSettings = getSsaidSettingsLocked(userId);
for (PackageInfo info : packages) {
// Check if the UID already has an entry in the table.
final String uid = Integer.toString(info.applicationInfo.uid);
final Setting ssaid = ssaidSettings.getSettingLocked(uid);
 
if (ssaid.isNull() || ssaid.getValue() == null) {
// Android Id doesn't exist for this package so create it.
ssaidSettings.insertSettingOverrideableByRestoreLocked(uid,
legacySsaid, null, true, info.packageName);
if (DEBUG) {
Slog.d(LOG_TAG, "Keep the legacy ssaid for uid=" + uid);
}
}
}
}
currentVersion = 137;
}
if (currentVersion == 137) {
// Version 138: Settings.Secure#INSTALL_NON_MARKET_APPS is deprecated and its
// default value set to 1. The user can no longer change the value of this
// setting through the UI.
final SettingsState secureSetting = getSecureSettingsLocked(userId);
if (!mUserManager.hasUserRestriction(
UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES, UserHandle.of(userId))
&& secureSetting.getSettingLocked(
Settings.Secure.INSTALL_NON_MARKET_APPS).getValue().equals("0")) {
secureSetting.insertSettingOverrideableByRestoreLocked(
Settings.Secure.INSTALL_NON_MARKET_APPS, "1", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
// For managed profiles with profile owners, DevicePolicyManagerService
// may want to set the user restriction in this case
secureSetting.insertSettingOverrideableByRestoreLocked(
Settings.Secure.UNKNOWN_SOURCES_DEFAULT_REVERSED, "1", null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 138;
}
if (currentVersion == 138) {
// Version 139: Removed.
currentVersion = 139;
}
if (currentVersion == 139) {
// Version 140: Settings.Secure#ACCESSIBILITY_SPEAK_PASSWORD is deprecated and
// the user can no longer change the value of this setting through the UI.
// Force to true.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
secureSettings.updateSettingLocked(Settings.Secure.ACCESSIBILITY_SPEAK_PASSWORD,
"1", null, true, SettingsState.SYSTEM_PACKAGE_NAME);
currentVersion = 140;
}
if (currentVersion == 140) {
// Version 141: Removed
currentVersion = 141;
}
if (currentVersion == 141) {
// This implementation was incorrectly setting the current value of
// settings changed by non-system packages as the default which default
// is set by the system. We add a new upgrade step at the end to properly
// handle this case which would also fix incorrect changes made by the
// old implementation of this step.
currentVersion = 142;
}
if (currentVersion == 142) {
// Version 143: Set a default value for Wi-Fi wakeup feature.
if (userId == UserHandle.USER_SYSTEM) {
final SettingsState globalSettings = getGlobalSettingsLocked();
Setting currentSetting = globalSettings.getSettingLocked(
Settings.Global.WIFI_WAKEUP_ENABLED);
if (currentSetting.isNull()) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.WIFI_WAKEUP_ENABLED,
getContext().getResources().getBoolean(
R.bool.def_wifi_wakeup_enabled) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 143;
}
if (currentVersion == 143) {
// Version 144: Set a default value for Autofill service.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting currentSetting = secureSettings
getSettingLocked(Settings.Secure.AUTOFILL_SERVICE);
if (currentSetting.isNull()) {
final String defaultValue = getContext().getResources().getString(
com.android.internal.R.string.config_defaultAutofillService);
if (defaultValue != null) {
Slog.d(LOG_TAG, "Setting [" + defaultValue + "] as Autofill Service "
+ "for user " + userId);
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.AUTOFILL_SERVICE, defaultValue, null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 144;
}
if (currentVersion == 144) {
// Version 145: Removed
currentVersion = 145;
}
if (currentVersion == 145) {
// Version 146: In step 142 we had a bug where incorrectly
// some settings were considered system set and as a result
// made the default and marked as the default being set by
// the system. Here reevaluate the default and default system
// set flags. This would both fix corruption by the old impl
// of step 142 and also properly handle devices which never
// run 142.
if (userId == UserHandle.USER_SYSTEM) {
SettingsState globalSettings = getGlobalSettingsLocked();
ensureLegacyDefaultValueAndSystemSetUpdatedLocked(globalSettings, userId);
globalSettings.persistSyncLocked();
}
SettingsState secureSettings = getSecureSettingsLocked(mUserId);
ensureLegacyDefaultValueAndSystemSetUpdatedLocked(secureSettings, userId);
secureSettings.persistSyncLocked();
SettingsState systemSettings = getSystemSettingsLocked(mUserId);
ensureLegacyDefaultValueAndSystemSetUpdatedLocked(systemSettings, userId);
systemSettings.persistSyncLocked();
currentVersion = 146;
}
if (currentVersion == 146) {
// Version 147: Removed. (This version previously allowed showing the
// "wifi_wakeup_available" setting).
// The setting that was added here is deleted in 153.
currentVersion = 147;
}
if (currentVersion == 147) {
// Version 148: Set the default value for DEFAULT_RESTRICT_BACKGROUND_DATA.
if (userId == UserHandle.USER_SYSTEM) {
final SettingsState globalSettings = getGlobalSettingsLocked();
final Setting currentSetting = globalSettings.getSettingLocked(
Global.DEFAULT_RESTRICT_BACKGROUND_DATA);
if (currentSetting.isNull()) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.DEFAULT_RESTRICT_BACKGROUND_DATA,
getContext().getResources().getBoolean(
R.bool.def_restrict_background_data) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 148;
}
if (currentVersion == 148) {
// Version 149: Set the default value for BACKUP_MANAGER_CONSTANTS.
final SettingsState systemSecureSettings = getSecureSettingsLocked(userId);
final String oldValue = systemSecureSettings.getSettingLocked(
Settings.Secure.BACKUP_MANAGER_CONSTANTS).getValue();
if (TextUtils.equals(null, oldValue)) {
final String defaultValue = getContext().getResources().getString(
R.string.def_backup_manager_constants);
if (!TextUtils.isEmpty(defaultValue)) {
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.BACKUP_MANAGER_CONSTANTS, defaultValue, null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 149;
}
if (currentVersion == 149) {
// Version 150: Set a default value for mobile data always on
final SettingsState globalSettings = getGlobalSettingsLocked();
final Setting currentSetting = globalSettings.getSettingLocked(
Settings.Global.MOBILE_DATA_ALWAYS_ON);
if (currentSetting.isNull()) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.MOBILE_DATA_ALWAYS_ON,
getContext().getResources().getBoolean(
R.bool.def_mobile_data_always_on) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 150;
}
if (currentVersion == 150) {
// Version 151: Removed.
currentVersion = 151;
}
if (currentVersion == 151) {
// Version 152: Removed. (This version made the setting for wifi_wakeup enabled
// by default but it is now no longer configurable).
// The setting updated here is deleted in 153.
currentVersion = 152;
}
if (currentVersion == 152) {
getGlobalSettingsLocked().deleteSettingLocked("wifi_wakeup_available");
currentVersion = 153;
}
if (currentVersion == 153) {
// Version 154: Read notification badge configuration from config.
// If user has already set the value, don't do anything.
final SettingsState systemSecureSettings = getSecureSettingsLocked(userId);
final Setting showNotificationBadges = systemSecureSettings.getSettingLocked(
Settings.Secure.NOTIFICATION_BADGING);
if (showNotificationBadges.isNull()) {
final boolean defaultValue = getContext().getResources().getBoolean(
com.android.internal.R.bool.config_notificationBadging);
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Secure.NOTIFICATION_BADGING,
defaultValue ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 154;
}
 
if (currentVersion == 154) {
// Version 155: Set the default value for BACKUP_LOCAL_TRANSPORT_PARAMETERS.
final SettingsState systemSecureSettings = getSecureSettingsLocked(userId);
final String oldValue = systemSecureSettings.getSettingLocked(
Settings.Secure.BACKUP_LOCAL_TRANSPORT_PARAMETERS).getValue();
if (TextUtils.equals(null, oldValue)) {
final String defaultValue = getContext().getResources().getString(
R.string.def_backup_local_transport_parameters);
if (!TextUtils.isEmpty(defaultValue)) {
systemSecureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.BACKUP_LOCAL_TRANSPORT_PARAMETERS, defaultValue,
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
}
currentVersion = 155;
}
 
if (currentVersion == 155) {
// Version 156: migrated to version 184
currentVersion = 156;
}
 
if (currentVersion == 156) {
// Version 157: Set a default value for zen duration,
// in version 169, zen duration is moved to secure settings
final SettingsState globalSettings = getGlobalSettingsLocked();
final Setting currentSetting = globalSettings.getSettingLocked(
Global.ZEN_DURATION);
if (currentSetting.isNull()) {
String defaultZenDuration = Integer.toString(getContext()
getResources().getInteger(R.integer.def_zen_duration));
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.ZEN_DURATION, defaultZenDuration,
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 157;
}
 
if (currentVersion == 157) {
// Version 158: Set default value for BACKUP_AGENT_TIMEOUT_PARAMETERS.
final SettingsState globalSettings = getGlobalSettingsLocked();
final String oldValue = globalSettings.getSettingLocked(
Settings.Global.BACKUP_AGENT_TIMEOUT_PARAMETERS).getValue();
if (TextUtils.equals(null, oldValue)) {
final String defaultValue = getContext().getResources().getString(
R.string.def_backup_agent_timeout_parameters);
if (!TextUtils.isEmpty(defaultValue)) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.BACKUP_AGENT_TIMEOUT_PARAMETERS, defaultValue,
null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
}
currentVersion = 158;
}
 
if (currentVersion == 158) {
// Remove setting that specifies wifi bgscan throttling params
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_scan_background_throttle_interval_ms");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_scan_background_throttle_package_whitelist");
currentVersion = 159;
}
 
if (currentVersion == 159) {
// Version 160: Hiding notifications from the lockscreen is only available as
// primary user option, profiles can only make them redacted. If a profile was
// configured to not show lockscreen notifications, ensure that at the very
// least these will be come hidden.
if (mUserManager.isManagedProfile(userId)) {
final SettingsState secureSettings = getSecureSettingsLocked(userId);
Setting showNotifications = secureSettings.getSettingLocked(
Settings.Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS);
// The default value is "1", check if user has turned it off.
if ("0".equals(showNotifications.getValue())) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS, "0",
null /* tag */, false /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
// The setting is no longer valid for managed profiles, it should be
// treated as if it was set to "1".
secureSettings.deleteSettingLocked(Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS);
}
currentVersion = 160;
}
 
if (currentVersion == 160) {
// Version 161: Set the default value for
// MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY and
// SOUND_TRIGGER_DETECTION_SERVICE_OP_TIMEOUT
final SettingsState globalSettings = getGlobalSettingsLocked();
 
String oldValue = globalSettings.getSettingLocked(
Global.MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY).getValue();
if (TextUtils.equals(null, oldValue)) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY,
Integer.toString(getContext().getResources().getInteger(
R.integer.def_max_sound_trigger_detection_service_ops_per_day)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
oldValue = globalSettings.getSettingLocked(
Global.SOUND_TRIGGER_DETECTION_SERVICE_OP_TIMEOUT).getValue();
if (TextUtils.equals(null, oldValue)) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.SOUND_TRIGGER_DETECTION_SERVICE_OP_TIMEOUT,
Integer.toString(getContext().getResources().getInteger(
R.integer.def_sound_trigger_detection_service_op_timeout)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 161;
}
 
if (currentVersion == 161) {
// Version 161: Add a gesture for silencing phones
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting currentSetting = secureSettings.getSettingLocked(
Secure.VOLUME_HUSH_GESTURE);
if (currentSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.VOLUME_HUSH_GESTURE,
Integer.toString(Secure.VOLUME_HUSH_VIBRATE),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
currentVersion = 162;
}
 
if (currentVersion == 162) {
// Version 162: REMOVED: Add a gesture for silencing phones
currentVersion = 163;
}
 
if (currentVersion == 163) {
// Version 163: Update default value of
// MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY from old to new default
final SettingsState settings = getGlobalSettingsLocked();
final Setting currentSetting = settings.getSettingLocked(
Global.MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY);
if (currentSetting.isDefaultFromSystem()) {
settings.insertSettingOverrideableByRestoreLocked(
Settings.Global.MAX_SOUND_TRIGGER_DETECTION_SERVICE_OPS_PER_DAY,
Integer.toString(getContext().getResources().getInteger(
R.integer
def_max_sound_trigger_detection_service_ops_per_day)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
currentVersion = 164;
}
 
if (currentVersion == 164) {
// Version 164: REMOVED: show zen upgrade notification
currentVersion = 165;
}
 
if (currentVersion == 165) {
// Version 165: MOVED: Show zen settings suggestion and zen updated settings
// moved to secure settings and are set in version 169
currentVersion = 166;
}
 
if (currentVersion == 166) {
// Version 166: add default values for hush gesture used and manual ringer
// toggle
final SettingsState secureSettings = getSecureSettingsLocked(userId);
Setting currentHushUsedSetting = secureSettings.getSettingLocked(
Secure.HUSH_GESTURE_USED);
if (currentHushUsedSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.HUSH_GESTURE_USED, "0", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
 
Setting currentRingerToggleCountSetting = secureSettings.getSettingLocked(
Secure.MANUAL_RINGER_TOGGLE_COUNT);
if (currentRingerToggleCountSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Settings.Secure.MANUAL_RINGER_TOGGLE_COUNT, "0", null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 167;
}
 
if (currentVersion == 167) {
// Version 167: MOVED - Settings.Global.CHARGING_VIBRATION_ENABLED moved to
// Settings.Secure.CHARGING_VIBRATION_ENABLED, set in version 170
currentVersion = 168;
}
 
if (currentVersion == 168) {
// Version 168: by default, vibrate for phone calls
final SettingsState systemSettings = getSystemSettingsLocked(userId);
final Setting currentSetting = systemSettings.getSettingLocked(
Settings.System.VIBRATE_WHEN_RINGING);
if (currentSetting.isNull()) {
systemSettings.insertSettingOverrideableByRestoreLocked(
Settings.System.VIBRATE_WHEN_RINGING,
getContext().getResources().getBoolean(
R.bool.def_vibrate_when_ringing) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 169;
}
 
if (currentVersion == 169) {
// Version 169: Set the default value for Secure Settings ZEN_DURATION,
// SHOW_ZEN_SETTINGS_SUGGESTION, ZEN_SETTINGS_UPDATE and
// ZEN_SETTINGS_SUGGESTION_VIEWED
 
final SettingsState globalSettings = getGlobalSettingsLocked();
final Setting globalZenDuration = globalSettings.getSettingLocked(
Global.ZEN_DURATION);
 
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting secureZenDuration = secureSettings.getSettingLocked(
Secure.ZEN_DURATION);
 
// ZEN_DURATION
if (!globalZenDuration.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.ZEN_DURATION, globalZenDuration.getValue(), null, false,
SettingsState.SYSTEM_PACKAGE_NAME);
 
// set global zen duration setting to null since it's deprecated
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.ZEN_DURATION, null, null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
} else if (secureZenDuration.isNull()) {
String defaultZenDuration = Integer.toString(getContext()
getResources().getInteger(R.integer.def_zen_duration));
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.ZEN_DURATION, defaultZenDuration, null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
}
// SHOW_ZEN_SETTINGS_SUGGESTION
final Setting currentShowZenSettingSuggestion = secureSettings.getSettingLocked(
Secure.SHOW_ZEN_SETTINGS_SUGGESTION);
if (currentShowZenSettingSuggestion.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.SHOW_ZEN_SETTINGS_SUGGESTION, "1",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
// ZEN_SETTINGS_UPDATED
final Setting currentUpdatedSetting = secureSettings.getSettingLocked(
Secure.ZEN_SETTINGS_UPDATED);
if (currentUpdatedSetting.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.ZEN_SETTINGS_UPDATED, "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
// ZEN_SETTINGS_SUGGESTION_VIEWED
final Setting currentSettingSuggestionViewed = secureSettings.getSettingLocked(
Secure.ZEN_SETTINGS_SUGGESTION_VIEWED);
if (currentSettingSuggestionViewed.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.ZEN_SETTINGS_SUGGESTION_VIEWED, "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 170;
}
if (currentVersion == 170) {
// Version 170: Set the default value for Secure Settings:
// CHARGING_SOUNDS_ENABLED and CHARGING_VIBRATION_ENABLED
final SettingsState globalSettings = getGlobalSettingsLocked();
final SettingsState secureSettings = getSecureSettingsLocked(userId);
// CHARGING_SOUNDS_ENABLED
final Setting globalChargingSoundEnabled = globalSettings.getSettingLocked(
Global.CHARGING_SOUNDS_ENABLED);
final Setting secureChargingSoundsEnabled = secureSettings.getSettingLocked(
Secure.CHARGING_SOUNDS_ENABLED);
if (!globalChargingSoundEnabled.isNull()) {
if (secureChargingSoundsEnabled.isNull()) {
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.CHARGING_SOUNDS_ENABLED,
globalChargingSoundEnabled.getValue(), null, false,
SettingsState.SYSTEM_PACKAGE_NAME);
}
// set global charging_sounds_enabled setting to null since it's deprecated
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.CHARGING_SOUNDS_ENABLED, null, null, true,
SettingsState.SYSTEM_PACKAGE_NAME);
} else if (secureChargingSoundsEnabled.isNull()) {
String defChargingSoundsEnabled = getContext().getResources()
getBoolean(R.bool.def_charging_sounds_enabled) ? "1" : "0";
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.CHARGING_SOUNDS_ENABLED, defChargingSoundsEnabled, null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
// CHARGING_VIBRATION_ENABLED
final Setting secureChargingVibrationEnabled = secureSettings.getSettingLocked(
Secure.CHARGING_VIBRATION_ENABLED);
 
if (secureChargingVibrationEnabled.isNull()) {
String defChargingVibrationEnabled = getContext().getResources()
getBoolean(R.bool.def_charging_vibration_enabled) ? "1" : "0";
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.CHARGING_VIBRATION_ENABLED, defChargingVibrationEnabled,
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
currentVersion = 171;
}
 
if (currentVersion == 171) {
// Version 171: by default, add STREAM_VOICE_CALL to list of streams that can
// be muted.
final SettingsState systemSettings = getSystemSettingsLocked(userId);
final Setting currentSetting = systemSettings.getSettingLocked(
Settings.System.MUTE_STREAMS_AFFECTED);
if (!currentSetting.isNull()) {
try {
int currentSettingIntegerValue = Integer.parseInt(
currentSetting.getValue());
if ((currentSettingIntegerValue
& (1 << AudioManager.STREAM_VOICE_CALL)) == 0) {
systemSettings.insertSettingOverrideableByRestoreLocked(
Settings.System.MUTE_STREAMS_AFFECTED,
Integer.toString(
currentSettingIntegerValue
| (1 << AudioManager.STREAM_VOICE_CALL)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
} catch (NumberFormatException e) {
// remove the setting in case it is not a valid integer
Slog.w("Failed to parse integer value of MUTE_STREAMS_AFFECTED"
+ "setting, removing setting", e);
systemSettings.deleteSettingLocked(
Settings.System.MUTE_STREAMS_AFFECTED);
}
 
}
currentVersion = 172;
}
 
if (currentVersion == 172) {
// Version 172: Set the default value for Secure Settings: LOCATION_MODE
 
final SettingsState secureSettings = getSecureSettingsLocked(userId);
 
final Setting locationMode = secureSettings.getSettingLocked(
Secure.LOCATION_MODE);
 
if (locationMode.isNull()) {
final Setting locationProvidersAllowed = secureSettings.getSettingLocked(
Secure.LOCATION_PROVIDERS_ALLOWED);
 
final int defLocationMode;
if (locationProvidersAllowed.isNull()) {
defLocationMode = getContext().getResources().getInteger(
R.integer.def_location_mode);
} else {
defLocationMode =
!TextUtils.isEmpty(locationProvidersAllowed.getValue())
? Secure.LOCATION_MODE_ON
: Secure.LOCATION_MODE_OFF;
}
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.LOCATION_MODE, Integer.toString(defLocationMode),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
currentVersion = 173;
}
 
if (currentVersion == 173) {
// Version 173: Set the default value for Secure Settings: NOTIFICATION_BUBBLES
// Removed. Moved NOTIFICATION_BUBBLES to Global Settings.
currentVersion = 174;
}
 
if (currentVersion == 174) {
// Version 174: Set the default value for Global Settings: APPLY_RAMPING_RINGER
 
final SettingsState globalSettings = getGlobalSettingsLocked();
 
Setting currentRampingRingerSetting = globalSettings.getSettingLocked(
Settings.Global.APPLY_RAMPING_RINGER);
if (currentRampingRingerSetting.isNull()) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Settings.Global.APPLY_RAMPING_RINGER,
getContext().getResources().getBoolean(
R.bool.def_apply_ramping_ringer) ? "1" : "0", null,
true, SettingsState.SYSTEM_PACKAGE_NAME);
}
 
currentVersion = 175;
}
 
if (currentVersion == 175) {
// Version 175: Set the default value for System Settings:
// RING_VIBRATION_INTENSITY. If the notification vibration intensity has been
// set and ring vibration intensity hasn't, the ring vibration intensity should
// followed notification vibration intensity.
final SettingsState systemSettings = getSystemSettingsLocked(userId);
Setting notificationVibrationIntensity = systemSettings.getSettingLocked(
Settings.System.NOTIFICATION_VIBRATION_INTENSITY);
Setting ringVibrationIntensity = systemSettings.getSettingLocked(
Settings.System.RING_VIBRATION_INTENSITY);
if (!notificationVibrationIntensity.isNull()
&& ringVibrationIntensity.isNull()) {
systemSettings.insertSettingOverrideableByRestoreLocked(
Settings.System.RING_VIBRATION_INTENSITY,
notificationVibrationIntensity.getValue(),
null , true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 176;
}
if (currentVersion == 176) {
// Version 176: Migrate the existing swipe up setting into the resource overlay
//              for the navigation bar interaction mode.  We do so only if the
//              setting is set.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting swipeUpSetting = secureSettings.getSettingLocked(
"swipe_up_to_switch_apps_enabled");
if (swipeUpSetting != null && !swipeUpSetting.isNull()
&& swipeUpSetting.getValue().equals("1")) {
final IOverlayManager overlayManager = IOverlayManager.Stub.asInterface(
ServiceManager.getService(Context.OVERLAY_SERVICE));
try {
overlayManager.setEnabledExclusiveInCategory(
NAV_BAR_MODE_2BUTTON_OVERLAY, UserHandle.USER_CURRENT);
} catch (SecurityException | IllegalStateException | RemoteException e) {
throw new IllegalStateException(
"Failed to set nav bar interaction mode overlay");
}
}
currentVersion = 177;
}
if (currentVersion == 177) {
// Version 177: Set the default value for Secure Settings: AWARE_ENABLED
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting awareEnabled = secureSettings.getSettingLocked(
Secure.AWARE_ENABLED);
if (awareEnabled.isNull()) {
final boolean defAwareEnabled = getContext().getResources().getBoolean(
R.bool.def_aware_enabled);
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.AWARE_ENABLED, defAwareEnabled ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 178;
}
if (currentVersion == 178) {
// Version 178: Set the default value for Secure Settings:
// SKIP_GESTURE & SILENCE_GESTURE
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting skipGesture = secureSettings.getSettingLocked(
Secure.SKIP_GESTURE);
if (skipGesture.isNull()) {
final boolean defSkipGesture = getContext().getResources().getBoolean(
R.bool.def_skip_gesture);
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.SKIP_GESTURE, defSkipGesture ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
final Setting silenceGesture = secureSettings.getSettingLocked(
Secure.SILENCE_GESTURE);
if (silenceGesture.isNull()) {
final boolean defSilenceGesture = getContext().getResources().getBoolean(
R.bool.def_silence_gesture);
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.SILENCE_GESTURE, defSilenceGesture ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 179;
}
if (currentVersion == 179) {
// Version 178: Reset the default for Secure Settings: NOTIFICATION_BUBBLES
// This is originally set in version 173, however, the default value changed
// so this step is to ensure the value is updated to the correct default.
// Removed. Moved NOTIFICATION_BUBBLES to Global Settings.
currentVersion = 180;
}
if (currentVersion == 180) {
// Version 180: Set the default value for Secure Settings: AWARE_LOCK_ENABLED
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting awareLockEnabled = secureSettings.getSettingLocked(
Secure.AWARE_LOCK_ENABLED);
if (awareLockEnabled.isNull()) {
final boolean defAwareLockEnabled = getContext().getResources().getBoolean(
R.bool.def_aware_lock_enabled);
secureSettings.insertSettingOverrideableByRestoreLocked(
Secure.AWARE_LOCK_ENABLED, defAwareLockEnabled ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 181;
}
if (currentVersion == 181) {
// Version cd : by default, add STREAM_BLUETOOTH_SCO to list of streams that can
// be muted.
final SettingsState systemSettings = getSystemSettingsLocked(userId);
final Setting currentSetting = systemSettings.getSettingLocked(
Settings.System.MUTE_STREAMS_AFFECTED);
if (!currentSetting.isNull()) {
try {
int currentSettingIntegerValue = Integer.parseInt(
currentSetting.getValue());
if ((currentSettingIntegerValue
& (1 << AudioManager.STREAM_BLUETOOTH_SCO)) == 0) {
systemSettings.insertSettingOverrideableByRestoreLocked(
Settings.System.MUTE_STREAMS_AFFECTED,
Integer.toString(
currentSettingIntegerValue
| (1 << AudioManager.STREAM_BLUETOOTH_SCO)),
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
} catch (NumberFormatException e) {
// remove the setting in case it is not a valid integer
Slog.w("Failed to parse integer value of MUTE_STREAMS_AFFECTED"
+ "setting, removing setting", e);
systemSettings.deleteSettingLocked(
Settings.System.MUTE_STREAMS_AFFECTED);
}
}
currentVersion = 182;
}
if (currentVersion == 182) {
// Remove secure bubble settings; it's in global now.
getSecureSettingsLocked(userId).deleteSettingLocked("notification_bubbles");
 
// Removed. Updated NOTIFICATION_BUBBLES to be true by default, see 184.
currentVersion = 183;
}
 
if (currentVersion == 183) {
// Version 183: Set default values for WIRELESS_CHARGING_STARTED_SOUND
// and CHARGING_STARTED_SOUND
final SettingsState globalSettings = getGlobalSettingsLocked();
 
final String oldValueWireless = globalSettings.getSettingLocked(
Global.WIRELESS_CHARGING_STARTED_SOUND).getValue();
final String oldValueWired = globalSettings.getSettingLocked(
Global.CHARGING_STARTED_SOUND).getValue();
 
final String defaultValueWireless = getContext().getResources().getString(
R.string.def_wireless_charging_started_sound);
final String defaultValueWired = getContext().getResources().getString(
R.string.def_charging_started_sound);
 
// wireless charging sound
if (oldValueWireless == null
|| TextUtils.equals(oldValueWireless, defaultValueWired)) {
if (!TextUtils.isEmpty(defaultValueWireless)) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.WIRELESS_CHARGING_STARTED_SOUND, defaultValueWireless,
null /* tag */, true /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
} else if (!TextUtils.isEmpty(defaultValueWired)) {
// if the wireless sound is empty, use the wired charging sound
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.WIRELESS_CHARGING_STARTED_SOUND, defaultValueWired,
null /* tag */, true /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
}
 
// wired charging sound
if (oldValueWired == null && !TextUtils.isEmpty(defaultValueWired)) {
globalSettings.insertSettingOverrideableByRestoreLocked(
Global.CHARGING_STARTED_SOUND, defaultValueWired,
null /* tag */, true /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 184;
}
 
if (currentVersion == 184) {
// Version 184: Reset the default for Global Settings: NOTIFICATION_BUBBLES
// This is originally set in version 182, however, the default value changed
// so this step is to ensure the value is updated to the correct default.
getGlobalSettingsLocked().insertSettingOverrideableByRestoreLocked(
Global.NOTIFICATION_BUBBLES, getContext().getResources().getBoolean(
R.bool.def_notification_bubbles) ? "1" : "0", null /* tag */,
true /* makeDefault */, SettingsState.SYSTEM_PACKAGE_NAME);
 
currentVersion = 185;
}
 
if (currentVersion == 185) {
// Deprecate ACCESSIBILITY_DISPLAY_MAGNIFICATION_NAVBAR_ENABLED, and migrate it
// to ACCESSIBILITY_BUTTON_TARGETS.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting magnifyNavbarEnabled = secureSettings.getSettingLocked(
Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_NAVBAR_ENABLED);
if ("1".equals(magnifyNavbarEnabled.getValue())) {
secureSettings.insertSettingLocked(
Secure.ACCESSIBILITY_BUTTON_TARGETS,
ACCESSIBILITY_SHORTCUT_TARGET_MAGNIFICATION_CONTROLLER,
null /* tag */, false /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
secureSettings.deleteSettingLocked(
Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_NAVBAR_ENABLED);
currentVersion = 186;
}
 
if (currentVersion == 186) {
// Remove unused wifi settings
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_rtt_background_exec_gap_ms");
getGlobalSettingsLocked().deleteSettingLocked(
"network_recommendation_request_timeout_ms");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_suspend_optimizations_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_is_unusable_event_metrics_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_data_stall_min_tx_bad");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_data_stall_min_tx_success_without_rx");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_link_speed_metrics_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_pno_frequency_culling_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_pno_recency_sorting_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_link_probing_enabled");
getGlobalSettingsLocked().deleteSettingLocked(
"wifi_saved_state");
currentVersion = 187;
}
 
if (currentVersion == 187) {
// Migrate adaptive sleep setting from System to Secure.
if (userId == UserHandle.USER_OWNER) {
// Remove from the system settings.
SettingsState systemSettings = getSystemSettingsLocked(userId);
String name = Settings.System.ADAPTIVE_SLEEP;
Setting setting = systemSettings.getSettingLocked(name);
systemSettings.deleteSettingLocked(name);
 
// Add to the secure settings.
SettingsState secureSettings = getSecureSettingsLocked(userId);
secureSettings.insertSettingLocked(name, setting.getValue(), null /* tag */,
false /* makeDefault */, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 188;
}
 
if (currentVersion == 188) {
// Deprecate ACCESSIBILITY_SHORTCUT_ENABLED, and migrate it
// to ACCESSIBILITY_SHORTCUT_TARGET_SERVICE.
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting shortcutEnabled = secureSettings.getSettingLocked(
"accessibility_shortcut_enabled");
if ("0".equals(shortcutEnabled.getValue())) {
// Clear shortcut key targets list setting.
secureSettings.insertSettingLocked(
Secure.ACCESSIBILITY_SHORTCUT_TARGET_SERVICE,
"", null /* tag */, false /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
secureSettings.deleteSettingLocked("accessibility_shortcut_enabled");
currentVersion = 189;
}
 
if (currentVersion == 189) {
final SettingsState secureSettings = getSecureSettingsLocked(userId);
final Setting showNotifications = secureSettings.getSettingLocked(
Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS);
final Setting allowPrivateNotifications = secureSettings.getSettingLocked(
Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS);
if ("1".equals(showNotifications.getValue())
&& "1".equals(allowPrivateNotifications.getValue())) {
secureSettings.insertSettingLocked(
Secure.POWER_MENU_LOCKED_SHOW_CONTENT,
"1", null /* tag */, false /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
} else if ("0".equals(showNotifications.getValue())
|| "0".equals(allowPrivateNotifications.getValue())) {
secureSettings.insertSettingLocked(
Secure.POWER_MENU_LOCKED_SHOW_CONTENT,
"0", null /* tag */, false /* makeDefault */,
SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 190;
}
 
if (currentVersion == 190) {
// Version 190: get HDMI auto device off from overlay
final SettingsState globalSettings = getGlobalSettingsLocked();
final Setting currentSetting = globalSettings.getSettingLocked(
Global.HDMI_CONTROL_AUTO_DEVICE_OFF_ENABLED);
if (currentSetting.isNull()) {
globalSettings.insertSettingLocked(
Global.HDMI_CONTROL_AUTO_DEVICE_OFF_ENABLED,
getContext().getResources().getBoolean(
R.bool.def_hdmiControlAutoDeviceOff) ? "1" : "0",
null, true, SettingsState.SYSTEM_PACKAGE_NAME);
}
currentVersion = 191;
}
 
// vXXX: Add new settings above this point.
 
if (currentVersion != newVersion) {
Slog.wtf("SettingsProvider", "warning: upgrading settings database to version "
+ newVersion + " left it at "
+ currentVersion +
" instead; this is probably a bug. Did you update SETTINGS_VERSION?",
new Throwable());
if (DEBUG) {
throw new RuntimeException("db upgrade error");
}
}
 
// Return the current version.
return currentVersion;
}
}
 
```

#### 而这里设置  
final SettingsState secureSettings = getSecureSettingsLocked(userId);  
Setting currentSetting = secureSettings.getSettingLocked(  
Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD);  
if (currentSetting.isNull()) {  
secureSettings.insertSettingOverrideableByRestoreLocked(  
Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD,  
getContext().getResources().getBoolean(  
R.bool.def_show_ime_with_hard_keyboard) ? "1" : "0",  
null, true, SettingsState.SYSTEM_PACKAGE_NAME);  
}  
是否禁用软键盘 由 def_show_ime_with_hard_keyboard 属性决定的 默认是禁用软键盘的 所以开启就可以了

#### 3.2 defaults.xml 关于是否禁用软键盘

```
路径：
/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
    设置是否禁用软键盘
        <!-- Default for Settings.Secure.SHOW_IME_WITH_HARD_KEYBOARD -->
     <bool >false</bool>
默认禁用
所以修改为：
    <bool >true</bool>
```