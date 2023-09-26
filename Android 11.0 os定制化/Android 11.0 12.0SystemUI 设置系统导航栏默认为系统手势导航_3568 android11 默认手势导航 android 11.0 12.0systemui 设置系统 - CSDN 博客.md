> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124740007)

### 1. 概述

在 11.0 12.0 的原生系统产品开发中，系统导航栏在 10.0 以后可以支持手势导航，但系统导航栏默认的是三键导航，Home Back Recent 键三个键显示在底部  
但是对于一些全屏的 app 感觉操作起来不太方便，所以产品需要要求使用导航栏设置为系统手势导航这时  
系统底部就不会被占用了

### 2.SystemUI 设置系统导航栏默认为系统手势导航核心类

```
frameworks\base\core\res\res\values\config.xml
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java

```

### 3.SystemUI 设置系统导航栏默认为系统手势导航核心功能分析和实现

在 frameworks 中设置默认手势的配置是在 config.xml 中定义的，接下来看 config.xml 中的相关[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)  
这样该怎么设置系统手势为默认的导航方式呢  
第一步在 config.xml 中

路径为：frameworks\base\core\res\res\values\config.xml

```
 <!-- Controls the opacity of the navigation bar depending on the visibility of the
         various workspace stacks.
         0 - Nav bar is always opaque when either the freeform stack or docked stack is visible.
         1 - Nav bar is always translucent when the freeform stack is visible, otherwise always
             opaque.
         2 - Nav bar is never forced opaque.
         -->
    <integer >0</integer>

    <!-- Controls the navigation bar interaction mode:
         0: 3 button mode (back, home, overview buttons)
         1: 2 button mode (back, home buttons + swipe up for overview)
         2: gestures only for back, home and overview -->
    <integer >0</integer>

    <!-- Controls whether the nav bar can move from the bottom to the side in landscape.
         Only applies if the device display is not square. -->
    <bool >true</bool>

    <!-- Controls whether the navigation bar lets through taps. -->
    <bool >false</bool>

    <!-- Controls whether the side edge gestures can always trigger the transient nav bar to
         show. -->
    <bool >false</bool>

```

通过注释我们可以看到 2 就是手势导航所以  
修改 2  
这样就是默认导航就是手势导航

### 3.2DatabaseHelper.java 中设置默认导航栏为手势导航

```
    private void loadSettings(SQLiteDatabase db) {
         loadSystemSettings(db);
         loadSecureSettings(db);
         // The global table only exists for the 'owner/system' user
         if (mUserHandle == UserHandle.USER_SYSTEM) {
             loadGlobalSettings(db);
         }
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

```

在 DatabaseHelper.java 的 onCreate() 中可以看到 loadSettings(db) 来加载系统相关数据的  
到数据库中，所以设置默认手势导航也可以在这里添加就可以了

```
 int navBar_mode = mContext.getResources().getInteger(com.android.internal.R.integer.config_navBarInteractionMode);
  if(navBar_mode == 2){
     try {
               IOverlayManager mOverlayManager = IOverlayManager.Stub.
       asInterface(ServiceManager.getService(Context.OVERLAY_SERVICE));
    mOverlayManager.setEnabledExclusiveInCategory(NAV_BAR_MODE_GESTURAL_OVERLAY, USER_CURRENT);
   } catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
   }

```

具体修改如下：

```
private void loadSettings(SQLiteDatabase db) {
         loadSystemSettings(db);
         loadSecureSettings(db);
         // The global table only exists for the 'owner/system' user
         if (mUserHandle == UserHandle.USER_SYSTEM) {
             loadGlobalSettings(db);
         }
         //add core start
 int navBar_mode = mContext.getResources().getInteger(com.android.internal.R.integer.config_navBarInteractionMode);
  if(navBar_mode == 2){
     try {
               IOverlayManager mOverlayManager = IOverlayManager.Stub.
       asInterface(ServiceManager.getService(Context.OVERLAY_SERVICE));
    mOverlayManager.setEnabledExclusiveInCategory(NAV_BAR_MODE_GESTURAL_OVERLAY, USER_CURRENT);
   } catch (RemoteException e) {
    throw e.rethrowFromSystemServer();
   }
//add core end
     }

```

通过以上两步的修改，然后编译发现功能已经实现了