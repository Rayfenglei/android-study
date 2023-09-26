> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126128861)

1. 概述
-----

  在 6.0 以前读写权限是默认授予的，app 不需要申请权限 在 10.0 之前需要 android.permission.WRITE_EXTERNAL_STORAGE 和 android.permission.READ_EXTERNAL_STORAGE  
权限就可以了而在安卓 11 的时候继续强化对 [SD 卡](https://so.csdn.net/so/search?q=SD%E5%8D%A1&spm=1001.2101.3001.7020)读写的管理，引入了 MANAGE_EXTERNAL_STORAGE 权限，而之前的 WRITE_EXTERNAL_STORAGE 已经失效了。  
并且 MANAGE_EXTERNAL_STORAGE 权限只能跳转设置页面申请。  
11.0 需要这样申请权限

app 申请权限  
 

```
 if (sdk_Int >= 30) {
                if (!Environment.isExternalStorageManager()) {
                    Intent intent = new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION);
                    startActivity(intent);
                    return;
                }
                writeAndReaderFile();
                return;
            }
```

2. 系统授予第三方 app 读写权限的核心代码部分
--------------------------

```
核心代码:
  frameworks/base/core/java/android/app/AppOpsManager.java
```

3. 系统授予第三方 app 读写权限的核心代码部分分析以及功能实现
----------------------------------

####   3.1AppOpsManager.java 关于系统权限的核心代码分析

```
public class AppOpsManager {
/**
* This is a subtle behavior change to {@link #startWatchingMode}.
*
* Before this change the system called back for the switched op. After the change the system
* will call back for the actually requested op or all switched ops if no op is specified.
*
* @hide
*/
@ChangeId
@EnabledAfter(targetSdkVersion = Build.VERSION_CODES.Q)
public static final long CALL_BACK_ON_CHANGED_LISTENER_WITH_SWITCHED_OP_CHANGE = 148180766L;
 
private static final int MAX_UNFORWARDED_OPS = 10;
 
final Context mContext;
 
@UnsupportedAppUsage
final IAppOpsService mService;
 
/**
* Service for the application context, to be used by static methods via
* {@link #getService()}
*/
@GuardedBy("sLock")
static IAppOpsService sService;
 
@GuardedBy("mModeWatchers")
private final ArrayMap<OnOpChangedListener, IAppOpsCallback> mModeWatchers =
new ArrayMap<>();
 
@GuardedBy("mActiveWatchers")
private final ArrayMap<OnOpActiveChangedListener, IAppOpsActiveCallback> mActiveWatchers =
new ArrayMap<>();
 
@GuardedBy("mStartedWatchers")
private final ArrayMap<OnOpStartedListener, IAppOpsStartedCallback> mStartedWatchers =
new ArrayMap<>();
 
@GuardedBy("mNotedWatchers")
private final ArrayMap<OnOpNotedListener, IAppOpsNotedCallback> mNotedWatchers =
new ArrayMap<>();
 
private static final Object sLock = new Object();
 
/** Current {@link OnOpNotedCallback}. Change via {@link #setOnOpNotedCallback} */
@GuardedBy("sLock")
private static @Nullable OnOpNotedCallback sOnOpNotedCallback;
 
/**
* Sync note-ops collected from {@link #readAndLogNotedAppops(Parcel)} that have not been
* delivered to a callback yet.
*
* Similar to {@link com.android.server.appop.AppOpsService#mUnforwardedAsyncNotedOps} for
* {@link COLLECT_ASYNC}. Used in situation when AppOpsManager asks to collect stacktrace with
* {@link #sMessageCollector}, which forces {@link COLLECT_SYNC} mode.
*/
@GuardedBy("sLock")
private static ArrayList<AsyncNotedAppOp> sUnforwardedOps = new ArrayList<>();
 
private static int[] sOpToSwitch = new int[] {
OP_COARSE_LOCATION,                 // COARSE_LOCATION
OP_COARSE_LOCATION,                 // FINE_LOCATION
OP_COARSE_LOCATION,                 // GPS
OP_VIBRATE,                         // VIBRATE
OP_READ_CONTACTS,                   // READ_CONTACTS
OP_WRITE_CONTACTS,                  // WRITE_CONTACTS
OP_READ_CALL_LOG,                   // READ_CALL_LOG
OP_WRITE_CALL_LOG,                  // WRITE_CALL_LOG
OP_READ_CALENDAR,                   // READ_CALENDAR
OP_WRITE_CALENDAR,                  // WRITE_CALENDAR
OP_COARSE_LOCATION,                 // WIFI_SCAN
OP_POST_NOTIFICATION,               // POST_NOTIFICATION
OP_COARSE_LOCATION,                 // NEIGHBORING_CELLS
OP_CALL_PHONE,                      // CALL_PHONE
OP_READ_SMS,                        // READ_SMS
OP_WRITE_SMS,                       // WRITE_SMS
OP_RECEIVE_SMS,                     // RECEIVE_SMS
OP_RECEIVE_SMS,                     // RECEIVE_EMERGECY_SMS
OP_RECEIVE_MMS,                     // RECEIVE_MMS
OP_RECEIVE_WAP_PUSH,                // RECEIVE_WAP_PUSH
OP_SEND_SMS,                        // SEND_SMS
OP_READ_SMS,                        // READ_ICC_SMS
OP_WRITE_SMS,                       // WRITE_ICC_SMS
OP_WRITE_SETTINGS,                  // WRITE_SETTINGS
OP_SYSTEM_ALERT_WINDOW,             // SYSTEM_ALERT_WINDOW
OP_ACCESS_NOTIFICATIONS,            // ACCESS_NOTIFICATIONS
OP_CAMERA,                          // CAMERA
OP_RECORD_AUDIO,                    // RECORD_AUDIO
OP_PLAY_AUDIO,                      // PLAY_AUDIO
OP_READ_CLIPBOARD,                  // READ_CLIPBOARD
OP_WRITE_CLIPBOARD,                 // WRITE_CLIPBOARD
OP_TAKE_MEDIA_BUTTONS,              // TAKE_MEDIA_BUTTONS
OP_TAKE_AUDIO_FOCUS,                // TAKE_AUDIO_FOCUS
OP_AUDIO_MASTER_VOLUME,             // AUDIO_MASTER_VOLUME
OP_AUDIO_VOICE_VOLUME,              // AUDIO_VOICE_VOLUME
OP_AUDIO_RING_VOLUME,               // AUDIO_RING_VOLUME
OP_AUDIO_MEDIA_VOLUME,              // AUDIO_MEDIA_VOLUME
OP_AUDIO_ALARM_VOLUME,              // AUDIO_ALARM_VOLUME
OP_AUDIO_NOTIFICATION_VOLUME,       // AUDIO_NOTIFICATION_VOLUME
OP_AUDIO_BLUETOOTH_VOLUME,          // AUDIO_BLUETOOTH_VOLUME
OP_WAKE_LOCK,                       // WAKE_LOCK
OP_COARSE_LOCATION,                 // MONITOR_LOCATION
OP_COARSE_LOCATION,                 // MONITOR_HIGH_POWER_LOCATION
OP_GET_USAGE_STATS,                 // GET_USAGE_STATS
OP_MUTE_MICROPHONE,                 // MUTE_MICROPHONE
OP_TOAST_WINDOW,                    // TOAST_WINDOW
OP_PROJECT_MEDIA,                   // PROJECT_MEDIA
OP_ACTIVATE_VPN,                    // ACTIVATE_VPN
OP_WRITE_WALLPAPER,                 // WRITE_WALLPAPER
OP_ASSIST_STRUCTURE,                // ASSIST_STRUCTURE
OP_ASSIST_SCREENSHOT,               // ASSIST_SCREENSHOT
OP_READ_PHONE_STATE,                // READ_PHONE_STATE
OP_ADD_VOICEMAIL,                   // ADD_VOICEMAIL
OP_USE_SIP,                         // USE_SIP
OP_PROCESS_OUTGOING_CALLS,          // PROCESS_OUTGOING_CALLS
OP_USE_FINGERPRINT,                 // USE_FINGERPRINT
OP_BODY_SENSORS,                    // BODY_SENSORS
OP_READ_CELL_BROADCASTS,            // READ_CELL_BROADCASTS
OP_MOCK_LOCATION,                   // MOCK_LOCATION
OP_READ_EXTERNAL_STORAGE,           // READ_EXTERNAL_STORAGE
OP_WRITE_EXTERNAL_STORAGE,          // WRITE_EXTERNAL_STORAGE
OP_TURN_SCREEN_ON,                  // TURN_SCREEN_ON
OP_GET_ACCOUNTS,                    // GET_ACCOUNTS
OP_RUN_IN_BACKGROUND,               // RUN_IN_BACKGROUND
OP_AUDIO_ACCESSIBILITY_VOLUME,      // AUDIO_ACCESSIBILITY_VOLUME
OP_READ_PHONE_NUMBERS,              // READ_PHONE_NUMBERS
OP_REQUEST_INSTALL_PACKAGES,        // REQUEST_INSTALL_PACKAGES
OP_PICTURE_IN_PICTURE,              // ENTER_PICTURE_IN_PICTURE_ON_HIDE
OP_INSTANT_APP_START_FOREGROUND,    // INSTANT_APP_START_FOREGROUND
OP_ANSWER_PHONE_CALLS,              // ANSWER_PHONE_CALLS
OP_RUN_ANY_IN_BACKGROUND,           // OP_RUN_ANY_IN_BACKGROUND
OP_CHANGE_WIFI_STATE,               // OP_CHANGE_WIFI_STATE
OP_REQUEST_DELETE_PACKAGES,         // OP_REQUEST_DELETE_PACKAGES
OP_BIND_ACCESSIBILITY_SERVICE,      // OP_BIND_ACCESSIBILITY_SERVICE
OP_ACCEPT_HANDOVER,                 // ACCEPT_HANDOVER
OP_MANAGE_IPSEC_TUNNELS,            // MANAGE_IPSEC_HANDOVERS
OP_START_FOREGROUND,                // START_FOREGROUND
OP_COARSE_LOCATION,                 // BLUETOOTH_SCAN
OP_USE_BIOMETRIC,                   // BIOMETRIC
OP_ACTIVITY_RECOGNITION,            // ACTIVITY_RECOGNITION
OP_SMS_FINANCIAL_TRANSACTIONS,      // SMS_FINANCIAL_TRANSACTIONS
OP_READ_MEDIA_AUDIO,                // READ_MEDIA_AUDIO
OP_WRITE_MEDIA_AUDIO,               // WRITE_MEDIA_AUDIO
OP_READ_MEDIA_VIDEO,                // READ_MEDIA_VIDEO
OP_WRITE_MEDIA_VIDEO,               // WRITE_MEDIA_VIDEO
OP_READ_MEDIA_IMAGES,               // READ_MEDIA_IMAGES
OP_WRITE_MEDIA_IMAGES,              // WRITE_MEDIA_IMAGES
OP_LEGACY_STORAGE,                  // LEGACY_STORAGE
OP_ACCESS_ACCESSIBILITY,            // ACCESS_ACCESSIBILITY
OP_READ_DEVICE_IDENTIFIERS,         // READ_DEVICE_IDENTIFIERS
OP_ACCESS_MEDIA_LOCATION,           // ACCESS_MEDIA_LOCATION
OP_QUERY_ALL_PACKAGES,              // QUERY_ALL_PACKAGES
OP_MANAGE_EXTERNAL_STORAGE,         // MANAGE_EXTERNAL_STORAGE
OP_INTERACT_ACROSS_PROFILES,        //INTERACT_ACROSS_PROFILES
OP_ACTIVATE_PLATFORM_VPN,           // ACTIVATE_PLATFORM_VPN
OP_LOADER_USAGE_STATS,              // LOADER_USAGE_STATS
OP_DEPRECATED_1,                    // deprecated
OP_AUTO_REVOKE_PERMISSIONS_IF_UNUSED, //AUTO_REVOKE_PERMISSIONS_IF_UNUSED
OP_AUTO_REVOKE_MANAGED_BY_INSTALLER, //OP_AUTO_REVOKE_MANAGED_BY_INSTALLER
OP_NO_ISOLATED_STORAGE,             // NO_ISOLATED_STORAGE
OP_PHONE_CALL_MICROPHONE,           // OP_PHONE_CALL_MICROPHONE
OP_PHONE_CALL_CAMERA,               // OP_PHONE_CALL_CAMERA
OP_RECORD_AUDIO_HOTWORD,            // RECORD_AUDIO_HOTWORD
};
 
/**
* This maps each operation to the public string constant for it.
*/
private static String[] sOpToString = new String[]{
OPSTR_COARSE_LOCATION,
OPSTR_FINE_LOCATION,
OPSTR_GPS,
OPSTR_VIBRATE,
OPSTR_READ_CONTACTS,
OPSTR_WRITE_CONTACTS,
OPSTR_READ_CALL_LOG,
OPSTR_WRITE_CALL_LOG,
OPSTR_READ_CALENDAR,
OPSTR_WRITE_CALENDAR,
OPSTR_WIFI_SCAN,
OPSTR_POST_NOTIFICATION,
OPSTR_NEIGHBORING_CELLS,
OPSTR_CALL_PHONE,
OPSTR_READ_SMS,
OPSTR_WRITE_SMS,
OPSTR_RECEIVE_SMS,
OPSTR_RECEIVE_EMERGENCY_BROADCAST,
OPSTR_RECEIVE_MMS,
OPSTR_RECEIVE_WAP_PUSH,
OPSTR_SEND_SMS,
OPSTR_READ_ICC_SMS,
OPSTR_WRITE_ICC_SMS,
OPSTR_WRITE_SETTINGS,
OPSTR_SYSTEM_ALERT_WINDOW,
OPSTR_ACCESS_NOTIFICATIONS,
OPSTR_CAMERA,
OPSTR_RECORD_AUDIO,
OPSTR_PLAY_AUDIO,
OPSTR_READ_CLIPBOARD,
OPSTR_WRITE_CLIPBOARD,
OPSTR_TAKE_MEDIA_BUTTONS,
OPSTR_TAKE_AUDIO_FOCUS,
OPSTR_AUDIO_MASTER_VOLUME,
OPSTR_AUDIO_VOICE_VOLUME,
OPSTR_AUDIO_RING_VOLUME,
OPSTR_AUDIO_MEDIA_VOLUME,
OPSTR_AUDIO_ALARM_VOLUME,
OPSTR_AUDIO_NOTIFICATION_VOLUME,
OPSTR_AUDIO_BLUETOOTH_VOLUME,
OPSTR_WAKE_LOCK,
OPSTR_MONITOR_LOCATION,
OPSTR_MONITOR_HIGH_POWER_LOCATION,
OPSTR_GET_USAGE_STATS,
OPSTR_MUTE_MICROPHONE,
OPSTR_TOAST_WINDOW,
OPSTR_PROJECT_MEDIA,
OPSTR_ACTIVATE_VPN,
OPSTR_WRITE_WALLPAPER,
OPSTR_ASSIST_STRUCTURE,
OPSTR_ASSIST_SCREENSHOT,
OPSTR_READ_PHONE_STATE,
OPSTR_ADD_VOICEMAIL,
OPSTR_USE_SIP,
OPSTR_PROCESS_OUTGOING_CALLS,
OPSTR_USE_FINGERPRINT,
OPSTR_BODY_SENSORS,
OPSTR_READ_CELL_BROADCASTS,
OPSTR_MOCK_LOCATION,
OPSTR_READ_EXTERNAL_STORAGE,
OPSTR_WRITE_EXTERNAL_STORAGE,
OPSTR_TURN_SCREEN_ON,
OPSTR_GET_ACCOUNTS,
OPSTR_RUN_IN_BACKGROUND,
OPSTR_AUDIO_ACCESSIBILITY_VOLUME,
OPSTR_READ_PHONE_NUMBERS,
OPSTR_REQUEST_INSTALL_PACKAGES,
OPSTR_PICTURE_IN_PICTURE,
OPSTR_INSTANT_APP_START_FOREGROUND,
OPSTR_ANSWER_PHONE_CALLS,
OPSTR_RUN_ANY_IN_BACKGROUND,
OPSTR_CHANGE_WIFI_STATE,
OPSTR_REQUEST_DELETE_PACKAGES,
OPSTR_BIND_ACCESSIBILITY_SERVICE,
OPSTR_ACCEPT_HANDOVER,
OPSTR_MANAGE_IPSEC_TUNNELS,
OPSTR_START_FOREGROUND,
OPSTR_BLUETOOTH_SCAN,
OPSTR_USE_BIOMETRIC,
OPSTR_ACTIVITY_RECOGNITION,
OPSTR_SMS_FINANCIAL_TRANSACTIONS,
OPSTR_READ_MEDIA_AUDIO,
OPSTR_WRITE_MEDIA_AUDIO,
OPSTR_READ_MEDIA_VIDEO,
OPSTR_WRITE_MEDIA_VIDEO,
OPSTR_READ_MEDIA_IMAGES,
OPSTR_WRITE_MEDIA_IMAGES,
OPSTR_LEGACY_STORAGE,
OPSTR_ACCESS_ACCESSIBILITY,
OPSTR_READ_DEVICE_IDENTIFIERS,
OPSTR_ACCESS_MEDIA_LOCATION,
OPSTR_QUERY_ALL_PACKAGES,
OPSTR_MANAGE_EXTERNAL_STORAGE,
OPSTR_INTERACT_ACROSS_PROFILES,
OPSTR_ACTIVATE_PLATFORM_VPN,
OPSTR_LOADER_USAGE_STATS,
"", // deprecated
OPSTR_AUTO_REVOKE_PERMISSIONS_IF_UNUSED,
OPSTR_AUTO_REVOKE_MANAGED_BY_INSTALLER,
OPSTR_NO_ISOLATED_STORAGE,
OPSTR_PHONE_CALL_MICROPHONE,
OPSTR_PHONE_CALL_CAMERA,
OPSTR_RECORD_AUDIO_HOTWORD,
};
 
/**
* This provides a simple name for each operation to be used
* in debug output.
*/
private static String[] sOpNames = new String[] {
"COARSE_LOCATION",
"FINE_LOCATION",
"GPS",
"VIBRATE",
"READ_CONTACTS",
"WRITE_CONTACTS",
"READ_CALL_LOG",
"WRITE_CALL_LOG",
"READ_CALENDAR",
"WRITE_CALENDAR",
"WIFI_SCAN",
"POST_NOTIFICATION",
"NEIGHBORING_CELLS",
"CALL_PHONE",
"READ_SMS",
"WRITE_SMS",
"RECEIVE_SMS",
"RECEIVE_EMERGECY_SMS",
"RECEIVE_MMS",
"RECEIVE_WAP_PUSH",
"SEND_SMS",
"READ_ICC_SMS",
"WRITE_ICC_SMS",
"WRITE_SETTINGS",
"SYSTEM_ALERT_WINDOW",
"ACCESS_NOTIFICATIONS",
"CAMERA",
"RECORD_AUDIO",
"PLAY_AUDIO",
"READ_CLIPBOARD",
"WRITE_CLIPBOARD",
"TAKE_MEDIA_BUTTONS",
"TAKE_AUDIO_FOCUS",
"AUDIO_MASTER_VOLUME",
"AUDIO_VOICE_VOLUME",
"AUDIO_RING_VOLUME",
"AUDIO_MEDIA_VOLUME",
"AUDIO_ALARM_VOLUME",
"AUDIO_NOTIFICATION_VOLUME",
"AUDIO_BLUETOOTH_VOLUME",
"WAKE_LOCK",
"MONITOR_LOCATION",
"MONITOR_HIGH_POWER_LOCATION",
"GET_USAGE_STATS",
"MUTE_MICROPHONE",
"TOAST_WINDOW",
"PROJECT_MEDIA",
"ACTIVATE_VPN",
"WRITE_WALLPAPER",
"ASSIST_STRUCTURE",
"ASSIST_SCREENSHOT",
"READ_PHONE_STATE",
"ADD_VOICEMAIL",
"USE_SIP",
"PROCESS_OUTGOING_CALLS",
"USE_FINGERPRINT",
"BODY_SENSORS",
"READ_CELL_BROADCASTS",
"MOCK_LOCATION",
"READ_EXTERNAL_STORAGE",
"WRITE_EXTERNAL_STORAGE",
"TURN_ON_SCREEN",
"GET_ACCOUNTS",
"RUN_IN_BACKGROUND",
"AUDIO_ACCESSIBILITY_VOLUME",
"READ_PHONE_NUMBERS",
"REQUEST_INSTALL_PACKAGES",
"PICTURE_IN_PICTURE",
"INSTANT_APP_START_FOREGROUND",
"ANSWER_PHONE_CALLS",
"RUN_ANY_IN_BACKGROUND",
"CHANGE_WIFI_STATE",
"REQUEST_DELETE_PACKAGES",
"BIND_ACCESSIBILITY_SERVICE",
"ACCEPT_HANDOVER",
"MANAGE_IPSEC_TUNNELS",
"START_FOREGROUND",
"BLUETOOTH_SCAN",
"USE_BIOMETRIC",
"ACTIVITY_RECOGNITION",
"SMS_FINANCIAL_TRANSACTIONS",
"READ_MEDIA_AUDIO",
"WRITE_MEDIA_AUDIO",
"READ_MEDIA_VIDEO",
"WRITE_MEDIA_VIDEO",
"READ_MEDIA_IMAGES",
"WRITE_MEDIA_IMAGES",
"LEGACY_STORAGE",
"ACCESS_ACCESSIBILITY",
"READ_DEVICE_IDENTIFIERS",
"ACCESS_MEDIA_LOCATION",
"QUERY_ALL_PACKAGES",
"MANAGE_EXTERNAL_STORAGE",
"INTERACT_ACROSS_PROFILES",
"ACTIVATE_PLATFORM_VPN",
"LOADER_USAGE_STATS",
"deprecated",
"AUTO_REVOKE_PERMISSIONS_IF_UNUSED",
"AUTO_REVOKE_MANAGED_BY_INSTALLER",
"NO_ISOLATED_STORAGE",
"PHONE_CALL_MICROPHONE",
"PHONE_CALL_CAMERA",
"RECORD_AUDIO_HOTWORD",
};
 
/**
* This optionally maps a permission to an operation.  If there
* is no permission associated with an operation, it is null.
*/
@UnsupportedAppUsage
private static String[] sOpPerms = new String[] {
android.Manifest.permission.ACCESS_COARSE_LOCATION,
android.Manifest.permission.ACCESS_FINE_LOCATION,
null,
android.Manifest.permission.VIBRATE,
android.Manifest.permission.READ_CONTACTS,
android.Manifest.permission.WRITE_CONTACTS,
android.Manifest.permission.READ_CALL_LOG,
android.Manifest.permission.WRITE_CALL_LOG,
android.Manifest.permission.READ_CALENDAR,
android.Manifest.permission.WRITE_CALENDAR,
android.Manifest.permission.ACCESS_WIFI_STATE,
null, // no permission required for notifications
null, // neighboring cells shares the coarse location perm
android.Manifest.permission.CALL_PHONE,
android.Manifest.permission.READ_SMS,
null, // no permission required for writing sms
android.Manifest.permission.RECEIVE_SMS,
android.Manifest.permission.RECEIVE_EMERGENCY_BROADCAST,
android.Manifest.permission.RECEIVE_MMS,
android.Manifest.permission.RECEIVE_WAP_PUSH,
android.Manifest.permission.SEND_SMS,
android.Manifest.permission.READ_SMS,
null, // no permission required for writing icc sms
android.Manifest.permission.WRITE_SETTINGS,
android.Manifest.permission.SYSTEM_ALERT_WINDOW,
android.Manifest.permission.ACCESS_NOTIFICATIONS,
android.Manifest.permission.CAMERA,
android.Manifest.permission.RECORD_AUDIO,
null, // no permission for playing audio
null, // no permission for reading clipboard
null, // no permission for writing clipboard
null, // no permission for taking media buttons
null, // no permission for taking audio focus
null, // no permission for changing master volume
null, // no permission for changing voice volume
null, // no permission for changing ring volume
null, // no permission for changing media volume
null, // no permission for changing alarm volume
null, // no permission for changing notification volume
null, // no permission for changing bluetooth volume
android.Manifest.permission.WAKE_LOCK,
null, // no permission for generic location monitoring
null, // no permission for high power location monitoring
android.Manifest.permission.PACKAGE_USAGE_STATS,
null, // no permission for muting/unmuting microphone
null, // no permission for displaying toasts
null, // no permission for projecting media
null, // no permission for activating vpn
null, // no permission for supporting wallpaper
null, // no permission for receiving assist structure
null, // no permission for receiving assist screenshot
Manifest.permission.READ_PHONE_STATE,
Manifest.permission.ADD_VOICEMAIL,
Manifest.permission.USE_SIP,
Manifest.permission.PROCESS_OUTGOING_CALLS,
Manifest.permission.USE_FINGERPRINT,
Manifest.permission.BODY_SENSORS,
Manifest.permission.READ_CELL_BROADCASTS,
null,
Manifest.permission.READ_EXTERNAL_STORAGE,
Manifest.permission.WRITE_EXTERNAL_STORAGE,
null, // no permission for turning the screen on
Manifest.permission.GET_ACCOUNTS,
null, // no permission for running in background
null, // no permission for changing accessibility volume
Manifest.permission.READ_PHONE_NUMBERS,
Manifest.permission.REQUEST_INSTALL_PACKAGES,
null, // no permission for entering picture-in-picture on hide
Manifest.permission.INSTANT_APP_FOREGROUND_SERVICE,
Manifest.permission.ANSWER_PHONE_CALLS,
null, // no permission for OP_RUN_ANY_IN_BACKGROUND
Manifest.permission.CHANGE_WIFI_STATE,
Manifest.permission.REQUEST_DELETE_PACKAGES,
Manifest.permission.BIND_ACCESSIBILITY_SERVICE,
Manifest.permission.ACCEPT_HANDOVER,
Manifest.permission.MANAGE_IPSEC_TUNNELS,
Manifest.permission.FOREGROUND_SERVICE,
null, // no permission for OP_BLUETOOTH_SCAN
Manifest.permission.USE_BIOMETRIC,
Manifest.permission.ACTIVITY_RECOGNITION,
Manifest.permission.SMS_FINANCIAL_TRANSACTIONS,
null,
null, // no permission for OP_WRITE_MEDIA_AUDIO
null,
null, // no permission for OP_WRITE_MEDIA_VIDEO
null,
null, // no permission for OP_WRITE_MEDIA_IMAGES
null, // no permission for OP_LEGACY_STORAGE
null, // no permission for OP_ACCESS_ACCESSIBILITY
null, // no direct permission for OP_READ_DEVICE_IDENTIFIERS
Manifest.permission.ACCESS_MEDIA_LOCATION,
null, // no permission for OP_QUERY_ALL_PACKAGES
Manifest.permission.MANAGE_EXTERNAL_STORAGE,
android.Manifest.permission.INTERACT_ACROSS_PROFILES,
null, // no permission for OP_ACTIVATE_PLATFORM_VPN
android.Manifest.permission.LOADER_USAGE_STATS,
null, // deprecated operation
null, // no permission for OP_AUTO_REVOKE_PERMISSIONS_IF_UNUSED
null, // no permission for OP_AUTO_REVOKE_MANAGED_BY_INSTALLER
null, // no permission for OP_NO_ISOLATED_STORAGE
null, // no permission for OP_PHONE_CALL_MICROPHONE
null, // no permission for OP_PHONE_CALL_CAMERA
null, // no permission for OP_RECORD_AUDIO_HOTWORD
};
 
/**
* This specifies the default mode for each operation.
*/
private static int[] sOpDefaultMode = new int[] {
AppOpsManager.MODE_ALLOWED, // COARSE_LOCATION
AppOpsManager.MODE_ALLOWED, // FINE_LOCATION
AppOpsManager.MODE_ALLOWED, // GPS
AppOpsManager.MODE_ALLOWED, // VIBRATE
AppOpsManager.MODE_ALLOWED, // READ_CONTACTS
AppOpsManager.MODE_ALLOWED, // WRITE_CONTACTS
AppOpsManager.MODE_ALLOWED, // READ_CALL_LOG
AppOpsManager.MODE_ALLOWED, // WRITE_CALL_LOG
AppOpsManager.MODE_ALLOWED, // READ_CALENDAR
AppOpsManager.MODE_ALLOWED, // WRITE_CALENDAR
AppOpsManager.MODE_ALLOWED, // WIFI_SCAN
AppOpsManager.MODE_ALLOWED, // POST_NOTIFICATION
AppOpsManager.MODE_ALLOWED, // NEIGHBORING_CELLS
AppOpsManager.MODE_ALLOWED, // CALL_PHONE
AppOpsManager.MODE_ALLOWED, // READ_SMS
AppOpsManager.MODE_IGNORED, // WRITE_SMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_SMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_EMERGENCY_BROADCAST
AppOpsManager.MODE_ALLOWED, // RECEIVE_MMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_WAP_PUSH
AppOpsManager.MODE_ALLOWED, // SEND_SMS
AppOpsManager.MODE_ALLOWED, // READ_ICC_SMS
AppOpsManager.MODE_ALLOWED, // WRITE_ICC_SMS
AppOpsManager.MODE_DEFAULT, // WRITE_SETTINGS
getSystemAlertWindowDefault(), // SYSTEM_ALERT_WINDOW
AppOpsManager.MODE_ALLOWED, // ACCESS_NOTIFICATIONS
AppOpsManager.MODE_ALLOWED, // CAMERA
AppOpsManager.MODE_ALLOWED, // RECORD_AUDIO
AppOpsManager.MODE_ALLOWED, // PLAY_AUDIO
AppOpsManager.MODE_ALLOWED, // READ_CLIPBOARD
AppOpsManager.MODE_ALLOWED, // WRITE_CLIPBOARD
AppOpsManager.MODE_ALLOWED, // TAKE_MEDIA_BUTTONS
AppOpsManager.MODE_ALLOWED, // TAKE_AUDIO_FOCUS
AppOpsManager.MODE_ALLOWED, // AUDIO_MASTER_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_VOICE_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_RING_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_MEDIA_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_ALARM_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_NOTIFICATION_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_BLUETOOTH_VOLUME
AppOpsManager.MODE_ALLOWED, // WAKE_LOCK
AppOpsManager.MODE_ALLOWED, // MONITOR_LOCATION
AppOpsManager.MODE_ALLOWED, // MONITOR_HIGH_POWER_LOCATION
AppOpsManager.MODE_DEFAULT, // GET_USAGE_STATS
AppOpsManager.MODE_ALLOWED, // MUTE_MICROPHONE
AppOpsManager.MODE_ALLOWED, // TOAST_WINDOW
AppOpsManager.MODE_IGNORED, // PROJECT_MEDIA
AppOpsManager.MODE_IGNORED, // ACTIVATE_VPN
AppOpsManager.MODE_ALLOWED, // WRITE_WALLPAPER
AppOpsManager.MODE_ALLOWED, // ASSIST_STRUCTURE
AppOpsManager.MODE_ALLOWED, // ASSIST_SCREENSHOT
AppOpsManager.MODE_ALLOWED, // READ_PHONE_STATE
AppOpsManager.MODE_ALLOWED, // ADD_VOICEMAIL
AppOpsManager.MODE_ALLOWED, // USE_SIP
AppOpsManager.MODE_ALLOWED, // PROCESS_OUTGOING_CALLS
AppOpsManager.MODE_ALLOWED, // USE_FINGERPRINT
AppOpsManager.MODE_ALLOWED, // BODY_SENSORS
AppOpsManager.MODE_ALLOWED, // READ_CELL_BROADCASTS
AppOpsManager.MODE_ERRORED, // MOCK_LOCATION
AppOpsManager.MODE_ALLOWED, // READ_EXTERNAL_STORAGE
AppOpsManager.MODE_ALLOWED, // WRITE_EXTERNAL_STORAGE
AppOpsManager.MODE_ALLOWED, // TURN_SCREEN_ON
AppOpsManager.MODE_ALLOWED, // GET_ACCOUNTS
AppOpsManager.MODE_ALLOWED, // RUN_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED, // AUDIO_ACCESSIBILITY_VOLUME
AppOpsManager.MODE_ALLOWED, // READ_PHONE_NUMBERS
AppOpsManager.MODE_DEFAULT, // REQUEST_INSTALL_PACKAGES
AppOpsManager.MODE_ALLOWED, // PICTURE_IN_PICTURE
AppOpsManager.MODE_DEFAULT, // INSTANT_APP_START_FOREGROUND
AppOpsManager.MODE_ALLOWED, // ANSWER_PHONE_CALLS
AppOpsManager.MODE_ALLOWED, // RUN_ANY_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED, // CHANGE_WIFI_STATE
AppOpsManager.MODE_ALLOWED, // REQUEST_DELETE_PACKAGES
AppOpsManager.MODE_ALLOWED, // BIND_ACCESSIBILITY_SERVICE
AppOpsManager.MODE_ALLOWED, // ACCEPT_HANDOVER
AppOpsManager.MODE_ERRORED, // MANAGE_IPSEC_TUNNELS
AppOpsManager.MODE_ALLOWED, // START_FOREGROUND
AppOpsManager.MODE_ALLOWED, // BLUETOOTH_SCAN
AppOpsManager.MODE_ALLOWED, // USE_BIOMETRIC
AppOpsManager.MODE_ALLOWED, // ACTIVITY_RECOGNITION
AppOpsManager.MODE_DEFAULT, // SMS_FINANCIAL_TRANSACTIONS
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_AUDIO
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_AUDIO
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_VIDEO
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_VIDEO
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_IMAGES
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_IMAGES
AppOpsManager.MODE_DEFAULT, // LEGACY_STORAGE
AppOpsManager.MODE_ALLOWED, // ACCESS_ACCESSIBILITY
AppOpsManager.MODE_ERRORED, // READ_DEVICE_IDENTIFIERS
AppOpsManager.MODE_ALLOWED, // ALLOW_MEDIA_LOCATION
AppOpsManager.MODE_DEFAULT, // QUERY_ALL_PACKAGES
AppOpsManager.MODE_DEFAULT, // MANAGE_EXTERNAL_STORAGE
AppOpsManager.MODE_DEFAULT, // INTERACT_ACROSS_PROFILES
AppOpsManager.MODE_IGNORED, // ACTIVATE_PLATFORM_VPN
AppOpsManager.MODE_DEFAULT, // LOADER_USAGE_STATS
AppOpsManager.MODE_IGNORED, // deprecated operation
AppOpsManager.MODE_DEFAULT, // OP_AUTO_REVOKE_PERMISSIONS_IF_UNUSED
AppOpsManager.MODE_ALLOWED, // OP_AUTO_REVOKE_MANAGED_BY_INSTALLER
AppOpsManager.MODE_ERRORED, // OP_NO_ISOLATED_STORAGE
AppOpsManager.MODE_ALLOWED, // PHONE_CALL_MICROPHONE
AppOpsManager.MODE_ALLOWED, // PHONE_CALL_CAMERA
AppOpsManager.MODE_ALLOWED, // OP_RECORD_AUDIO_HOTWORD
};
....
}
  
```

#### 在 sOpDefaultMode 中 是默认授予对于的权限 每个权限后面有注释  
AppOpsManager.MODE_DEFAULT, // MANAGE_EXTERNAL_STORAGE 就是 11.0 新增的读写访问权限  
默认是需要申请授权的 所以需要修改为允许

#### 3.2 系统授予第三方 app 读写权限的具体实现

```
/**
* This specifies the default mode for each operation.
*/
private static int[] sOpDefaultMode = new int[] {
AppOpsManager.MODE_ALLOWED, // COARSE_LOCATION
AppOpsManager.MODE_ALLOWED, // FINE_LOCATION
AppOpsManager.MODE_ALLOWED, // GPS
AppOpsManager.MODE_ALLOWED, // VIBRATE
AppOpsManager.MODE_ALLOWED, // READ_CONTACTS
AppOpsManager.MODE_ALLOWED, // WRITE_CONTACTS
AppOpsManager.MODE_ALLOWED, // READ_CALL_LOG
AppOpsManager.MODE_ALLOWED, // WRITE_CALL_LOG
AppOpsManager.MODE_ALLOWED, // READ_CALENDAR
AppOpsManager.MODE_ALLOWED, // WRITE_CALENDAR
AppOpsManager.MODE_ALLOWED, // WIFI_SCAN
AppOpsManager.MODE_ALLOWED, // POST_NOTIFICATION
AppOpsManager.MODE_ALLOWED, // NEIGHBORING_CELLS
AppOpsManager.MODE_ALLOWED, // CALL_PHONE
AppOpsManager.MODE_ALLOWED, // READ_SMS
AppOpsManager.MODE_IGNORED, // WRITE_SMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_SMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_EMERGENCY_BROADCAST
AppOpsManager.MODE_ALLOWED, // RECEIVE_MMS
AppOpsManager.MODE_ALLOWED, // RECEIVE_WAP_PUSH
AppOpsManager.MODE_ALLOWED, // SEND_SMS
AppOpsManager.MODE_ALLOWED, // READ_ICC_SMS
AppOpsManager.MODE_ALLOWED, // WRITE_ICC_SMS
AppOpsManager.MODE_DEFAULT, // WRITE_SETTINGS
getSystemAlertWindowDefault(), // SYSTEM_ALERT_WINDOW
AppOpsManager.MODE_ALLOWED, // ACCESS_NOTIFICATIONS
AppOpsManager.MODE_ALLOWED, // CAMERA
AppOpsManager.MODE_ALLOWED, // RECORD_AUDIO
AppOpsManager.MODE_ALLOWED, // PLAY_AUDIO
AppOpsManager.MODE_ALLOWED, // READ_CLIPBOARD
AppOpsManager.MODE_ALLOWED, // WRITE_CLIPBOARD
AppOpsManager.MODE_ALLOWED, // TAKE_MEDIA_BUTTONS
AppOpsManager.MODE_ALLOWED, // TAKE_AUDIO_FOCUS
AppOpsManager.MODE_ALLOWED, // AUDIO_MASTER_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_VOICE_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_RING_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_MEDIA_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_ALARM_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_NOTIFICATION_VOLUME
AppOpsManager.MODE_ALLOWED, // AUDIO_BLUETOOTH_VOLUME
AppOpsManager.MODE_ALLOWED, // WAKE_LOCK
AppOpsManager.MODE_ALLOWED, // MONITOR_LOCATION
AppOpsManager.MODE_ALLOWED, // MONITOR_HIGH_POWER_LOCATION
AppOpsManager.MODE_DEFAULT, // GET_USAGE_STATS
AppOpsManager.MODE_ALLOWED, // MUTE_MICROPHONE
AppOpsManager.MODE_ALLOWED, // TOAST_WINDOW
AppOpsManager.MODE_IGNORED, // PROJECT_MEDIA
AppOpsManager.MODE_IGNORED, // ACTIVATE_VPN
AppOpsManager.MODE_ALLOWED, // WRITE_WALLPAPER
AppOpsManager.MODE_ALLOWED, // ASSIST_STRUCTURE
AppOpsManager.MODE_ALLOWED, // ASSIST_SCREENSHOT
AppOpsManager.MODE_ALLOWED, // READ_PHONE_STATE
AppOpsManager.MODE_ALLOWED, // ADD_VOICEMAIL
AppOpsManager.MODE_ALLOWED, // USE_SIP
AppOpsManager.MODE_ALLOWED, // PROCESS_OUTGOING_CALLS
AppOpsManager.MODE_ALLOWED, // USE_FINGERPRINT
AppOpsManager.MODE_ALLOWED, // BODY_SENSORS
AppOpsManager.MODE_ALLOWED, // READ_CELL_BROADCASTS
AppOpsManager.MODE_ERRORED, // MOCK_LOCATION
AppOpsManager.MODE_ALLOWED, // READ_EXTERNAL_STORAGE
AppOpsManager.MODE_ALLOWED, // WRITE_EXTERNAL_STORAGE
AppOpsManager.MODE_ALLOWED, // TURN_SCREEN_ON
AppOpsManager.MODE_ALLOWED, // GET_ACCOUNTS
AppOpsManager.MODE_ALLOWED, // RUN_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED, // AUDIO_ACCESSIBILITY_VOLUME
AppOpsManager.MODE_ALLOWED, // READ_PHONE_NUMBERS
AppOpsManager.MODE_DEFAULT, // REQUEST_INSTALL_PACKAGES
AppOpsManager.MODE_ALLOWED, // PICTURE_IN_PICTURE
AppOpsManager.MODE_DEFAULT, // INSTANT_APP_START_FOREGROUND
AppOpsManager.MODE_ALLOWED, // ANSWER_PHONE_CALLS
AppOpsManager.MODE_ALLOWED, // RUN_ANY_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED, // CHANGE_WIFI_STATE
AppOpsManager.MODE_ALLOWED, // REQUEST_DELETE_PACKAGES
AppOpsManager.MODE_ALLOWED, // BIND_ACCESSIBILITY_SERVICE
AppOpsManager.MODE_ALLOWED, // ACCEPT_HANDOVER
AppOpsManager.MODE_ERRORED, // MANAGE_IPSEC_TUNNELS
AppOpsManager.MODE_ALLOWED, // START_FOREGROUND
AppOpsManager.MODE_ALLOWED, // BLUETOOTH_SCAN
AppOpsManager.MODE_ALLOWED, // USE_BIOMETRIC
AppOpsManager.MODE_ALLOWED, // ACTIVITY_RECOGNITION
AppOpsManager.MODE_DEFAULT, // SMS_FINANCIAL_TRANSACTIONS
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_AUDIO
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_AUDIO
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_VIDEO
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_VIDEO
AppOpsManager.MODE_ALLOWED, // READ_MEDIA_IMAGES
AppOpsManager.MODE_ERRORED, // WRITE_MEDIA_IMAGES
AppOpsManager.MODE_DEFAULT, // LEGACY_STORAGE
AppOpsManager.MODE_ALLOWED, // ACCESS_ACCESSIBILITY
AppOpsManager.MODE_ERRORED, // READ_DEVICE_IDENTIFIERS
AppOpsManager.MODE_ALLOWED, // ALLOW_MEDIA_LOCATION
AppOpsManager.MODE_DEFAULT, // QUERY_ALL_PACKAGES
 
- AppOpsManager.MODE_DEFAULT, // MANAGE_EXTERNAL_STORAGE
+ AppOpsManager.MODE_ALLOWED, // MANAGE_EXTERNAL_STORAGE
 
AppOpsManager.MODE_DEFAULT, // INTERACT_ACROSS_PROFILES
AppOpsManager.MODE_IGNORED, // ACTIVATE_PLATFORM_VPN
AppOpsManager.MODE_DEFAULT, // LOADER_USAGE_STATS
AppOpsManager.MODE_IGNORED, // deprecated operation
AppOpsManager.MODE_DEFAULT, // OP_AUTO_REVOKE_PERMISSIONS_IF_UNUSED
AppOpsManager.MODE_ALLOWED, // OP_AUTO_REVOKE_MANAGED_BY_INSTALLER
AppOpsManager.MODE_ERRORED, // OP_NO_ISOLATED_STORAGE
AppOpsManager.MODE_ALLOWED, // PHONE_CALL_MICROPHONE
AppOpsManager.MODE_ALLOWED, // PHONE_CALL_CAMERA
AppOpsManager.MODE_ALLOWED, // OP_RECORD_AUDIO_HOTWORD
};
```

AppOpsManager.MODE_ALLOWED, // MANAGE_EXTERNAL_STORAGE

对于运行什么权限授予 MODE_ALLOWED 就可以了