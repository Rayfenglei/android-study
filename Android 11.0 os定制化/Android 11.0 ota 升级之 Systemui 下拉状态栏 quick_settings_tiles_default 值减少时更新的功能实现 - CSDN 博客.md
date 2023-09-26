> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132701287)

1. 前言
-----

 在 11.0 的系统 rom 定制化开发中，在定制功能需求中，在进行 systemui 的下拉状态栏定制以后，当需要 [ota 升级](https://so.csdn.net/so/search?q=ota%E5%8D%87%E7%BA%A7&spm=1001.2101.3001.7020)的时候，发现在 systemui 下拉状态栏的快捷功能键部分去掉的  
一些快捷功能并没有减少，这是因为 systemui 有缓存造成的只有清理缓存或者[恢复出厂设置](https://so.csdn.net/so/search?q=%E6%81%A2%E5%A4%8D%E5%87%BA%E5%8E%82%E8%AE%BE%E7%BD%AE&spm=1001.2101.3001.7020)后才正常，所以今天就来实现不需要清理缓存或恢复出厂设置  
在 ota 升级后正常使用的功能

2.ota 升级关于 Systemui 下拉状态栏 quick_settings_tiles_default 值减少时更新的功能实现的核心类
----------------------------------------------------------------------

```
    frameworks/base/packages/SystemUI/res/values/config.xml
    frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
    frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSTileHost.java
```

3.ota 升级关于 Systemui 下拉状态栏 quick_settings_tiles_default 值减少时更新的功能实现的核心分析和实现
--------------------------------------------------------------------------

SystemUI 是操作系统中的一个系统级应用程序，负责管理和呈现用户界面的重要元素。  
它提供了与用户交互的界面元素，包括状态栏、导航栏、通知和快捷设置等。  
SystemUI 通过提供用户界面和交互方式，使用户能够轻松访问系统功能和通知  
SystemUI 是 Android 操作系统的一个关键组件，主要负责管理和提供用户界面的核心元素，如状态栏、导航栏和锁屏界面等

SystemUI 路径 与 Packages/apps / 下许多模块不同的是, SystemUI 属于 Android FrameWork 的一部分。这也意味着, SystemUI 在正常情况下不可替代  
SystemUI 下拉状态栏快捷开关是 QSPanel，qs_panel.xml，@+id/quick_settings_panel  
QSPanel 创建是从 StatusBarmakeStatusBarView 开始的。

在 11.0 的系统源码中，在 SystemUI 的 res 的 config 文件中，在 quick_settings_tiles_default 的值，就是对应的 Systemui 下拉状态栏的  
快捷功能按键对应的字符，通过，来分隔开

```
  --- a/frameworks/base/packages/SystemUI/res/values/config.xml
    +++ b/frameworks/base/packages/SystemUI/res/values/config.xml
    @@ -111,7 +111,7 @@
     
         <!-- The default tiles to display in QuickSettings -->
         <string >
    -        volte1,volte2,wifi,bt,dnd,vowifi,lte1,lte2,onehand,flashlight,rotation,battery,cell,airplane,cast
    +        volte1,volte2,wifi,dnd,vowifi,lte1,lte2,onehand,flashlight,rotation,battery,airplane
         </string>
     
         <!-- The minimum number of tiles to display in QuickSettings -->
    @@ -119,7 +119,7 @@
     
         <!-- Tiles native to System UI. Order should match "quick_settings_tiles_default" -->
         <string >
    -        volte1,volte2,wifi,cell,battery,dnd,vowifi,lte1,lte2,flashlight,rotation,bt,airplane,location,hotspot,inversion,saver,dark,work,cast,night,onehand,longscreenshot
    +        volte1,volte2,wifi,battery,dnd,vowifi,lte1,lte2,flashlight,rotation,airplane,hotspot,work,onehand
         </string>
     
         <!-- The tiles to display in QuickSettings -->
    @@ -127,7 +127,7 @@
     
         <!-- The tiles to display in QuickSettings in retail mode -->
         <string >
    -        volte1,volte2,cell,battery,dnd,vowifi,lte1,lte2,flashlight,rotation,location
    +        volte1,volte2,battery,dnd,vowifi,lte1,lte2,flashlight,rotation
         </string>
```

ota 升级关于 Systemui 下拉状态栏 quick_settings_tiles_default 值减少时更新的功能实现中，

在 SystemUI 中的 config.xml 的上述代码中，分析得知，通过在 quick_settings_tiles_stock 和 quick_settings_tiles_default 和 quick_settings_tiles_retail_mode  
中去掉了关于在下拉状态栏快捷功能键部分的对应是字符，从而来减少下拉状态栏快捷功能部分功能，但是在 ota 升级后 发现并没有减少，由于  
在 SystemUI 中缓存了相关的数据后，导致 ota 升级这样改不能解决相关的问题，  
所以接下来分析下 QSTileHost.java 中关于加载快捷功能键的代码

3.2 QSTileHost.java 中关于加载快捷功能键的代码代码分析
-------------------------------------

```
   protected List<String> loadTileSpecs(Context context, String tileList) {
            return loadTileSpecs(context,tileList,false);
        }
     
        protected List<String> loadTileSpecs(Context context, String tileList ,boolean is_special_request) {
            final Resources res = context.getResources();
            String defaultTileList = res.getString(R.string.quick_settings_tiles_default);
            if (TextUtils.isEmpty(tileList)) {
                tileList = res.getString(R.string.quick_settings_tiles);
                if (DEBUG) Log.d(TAG, "Loaded tile specs from config: " + tileList);
            } else {
                if (DEBUG) Log.d(TAG, "Loaded tile specs from setting: " + tileList);
            }
            if (!mIsEnableWifiDisplay) {
                tileList = tileList.replaceAll( ",cast|cast,|cast","");
                defaultTileList = defaultTileList.replaceAll( ",cast|cast,|cast","");
            }
     
            /* UNISOC: Bug 1074234,885650,Super power feature*/
            if (!is_special_request) {
                if(SprdPowerManagerUtil.isSuperPower()) {
                    tileList = "wifi,cell,battery";
                }
            }
            /*@}*/
            /* UNISOC: Bug 1074234,895419,780848,remove battery quick setting label under guest mode */
            if (ActivityManager.getCurrentUser() != UserHandle.USER_OWNER) {
                tileList = tileList.replaceAll("battery","");
                defaultTileList = defaultTileList.replaceAll("battery","");
            }
            /* @} */
            /* UNISOC: Bug 1073208,1127438,longscreenshot feature @{ */
            if (ActivityManager.getCurrentUser() != UserHandle.USER_OWNER) {
                tileList = tileList.replaceAll("longscreenshot","");
                defaultTileList = defaultTileList.replaceAll("longscreenshot","");
            }
            /* @} */
            if (ActivityManager.getCurrentUser() != UserHandle.USER_OWNER) {
                tileList = tileList.replaceAll("lte","");
                defaultTileList = defaultTileList.replaceAll("lte","");
                tileList = tileList.replaceAll("volte","");
                defaultTileList = defaultTileList.replaceAll("volte","");
                tileList = tileList.replaceAll("vowifi","");
                defaultTileList = defaultTileList.replaceAll("vowifi","");
            }
     
            final ArrayList<String> tiles = new ArrayList<String>();
            boolean addedDefault = false;
            for (String tile : tileList.split(",")) {
                tile = tile.trim();
                if (tile.isEmpty()) continue;
                if (tile.equals("default")) {
                    if (!addedDefault) {
                        tiles.addAll(Arrays.asList(defaultTileList.split(",")));
                        if (Build.IS_DEBUGGABLE
                                && GarbageMonitor.MemoryTile.ADD_TO_DEFAULT_ON_DEBUGGABLE_BUILDS) {
                            tiles.add(GarbageMonitor.MemoryTile.TILE_SPEC);
                        }
                        addedDefault = true;
                    }
                } else {
                    tiles.add(tile);
                }
            }
            return tiles;
        }
```

ota 升级关于 Systemui 下拉状态栏 quick_settings_tiles_default 值减少时更新的功能实现中，

在 SystemUI 的源码中，在 QSTileHost.java 中上述源码中，可以看出，在 loadTileSpecs(Context context, String tileList ,boolean is_special_request)  
中主要是处理添加下拉状态栏快捷功能键的加载功能的，在 defaultTileList 就是读取 quick_settings_tiles_default  
的值，然后添加到下拉快捷列表的，从代码中可以看出，是会系统首次 启动的时候，或者恢复出厂设置的时候，从新  
加载 quick_settings_tiles_default 的值，所以在 ota 升级后，减少快捷功能键后这里的值不会变的，接下来在分析下  
QSFactoryImpl.java 关于构建 QSTile 的相关代码分析

3.3 QSFactoryImpl.java 关于构建 QSTile 的相关代码分析
------------------------------------------

```
       private QSTileImpl createTileInternal(String tileSpec) {
            Log.w(TAG, "createTileInternal tileSpec: " + tileSpec);
            // Stock tiles.
            switch (tileSpec) {
                case "wifi":
                    return mWifiTileProvider.get();
                case "volte1":
                case "volte2":
                case "vowifi":
                case "lte": //UNISOC: Modify for bug1263324
                case "lte1":
                case "lte2":
                    return createExtraTile(mHost, tileSpec);
                /*case "bt":
                    return mBluetoothTileProvider.get();
                case "cell":
                    return mCellularTileProvider.get();*/
                case "dnd":
                    return mDndTileProvider.get();
                /*case "inversion":
                    return mColorInversionTileProvider.get();*/
                case "airplane":
                    return mAirplaneModeTileProvider.get();
                case "work":
                    return mWorkModeTileProvider.get();
                case "rotation":
                    return mRotationLockTileProvider.get();
                case "flashlight":
                    return mFlashlightTileProvider.get();
                /*case "longscreenshot":
                    return mLongScreenshotTileProvider.get();*/
                case "onehand":
                    return mOneHandTileProvider.get();
                /*case "location":
                    return mLocationTileProvider.get();*/
                /*case "cast":
                    return mCastTileProvider.get();*/
                case "hotspot":
                    return mHotspotTileProvider.get();
                case "user":
                    return mUserTileProvider.get();
                case "battery":
                    /*UNISOC Bug 1074234, 885650, Super power feature @{ */
                    if (SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE) {
                        Log.w(TAG, "SuperBatteryTile " + tileSpec);
                        return new SuperBatteryTile(mHost);
                    } else {
                        return mBatterySaverTileProvider.get();
                    }
                    /* @} */
                /*case "saver":
                    return mDataSaverTileProvider.get();
                case "night":
                    return mNightDisplayTileProvider.get();*/
                case "nfc":
                    return mNfcTileProvider.get();
                /*case "dark":
                    return mUiModeNightTileProvider.get();*/
            }
 
            return null;
        }
```

ota 升级关于 Systemui 下拉状态栏 quick_settings_tiles_default 值减少时更新的功能实现中，

在 SystemUI 下拉状态栏模块中，在 QSFactoryImpl.java 中的上述源码中，在 createTileInternal(String tileSpec) 中的核心功能部分就是  
根据 titleSpec 的值，来获取对应的 Provider 的值，来获取对应的 QSTile 类，所以可以在这里  
去掉减少的快捷功能键对应的字符，获取 Provider 的相关代码，在获取不到对应的 QSTile 的时候，在  
下拉状态栏的快捷功能部分就不会显示了，这样就实现在 ota 升级以后减少的快捷功能键部分就不会显示了  
就这样实现了功能