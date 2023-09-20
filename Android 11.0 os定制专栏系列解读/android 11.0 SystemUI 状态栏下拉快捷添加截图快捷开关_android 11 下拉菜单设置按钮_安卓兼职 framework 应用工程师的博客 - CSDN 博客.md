> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870024)

1. 概述
-----

在 11.0 的产品开发中，对 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的定制需求也是挺多的，在下拉状态栏中 添加截图快捷开关，也是常有的开发功能，下面就以添加 截图功能为例功能的实现

2.SystemUI 状态栏下拉快捷添加截图快捷开关的核心代码
-------------------------------

```
frameworks/base/packages/SystemUI/res/values/config.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java

```

3.SystemUI 状态栏下拉快捷添加截图快捷开关的功能分析和实现
----------------------------------

3.1 关于下拉快捷图标相关 config 分析
------------------------

在 systemUI 的 res 下的 config 中的 quick_settings_tiles_default 和 quick_settings_tiles_stock 是默认添加下拉快捷的字符资源，在下拉状态栏布局的时候会从这里读取 tiles 的名称，然后找相关的 tiles 进行布局天的  
在 quick_settings_tiles_default 和 quick_settings_tiles_stock 中添加 screenshot 截图

修改如下:

```
diff --git a/frameworks/base/packages/SystemUI/res/values/config.xml b/frameworks/base/packages/SystemUI/res/values/config.xml
index 2ddc56ada3..ff13d52f67 100755
--- a/frameworks/base/packages/SystemUI/res/values/config.xml
+++ b/frameworks/base/packages/SystemUI/res/values/config.xml
@@ -111,7 +111,7 @@
 
     <!-- The default tiles to display in QuickSettings -->
     <string >
-        volte1,volte2,wifi,bt,dnd,vowifi,lte1,lte2,onehand,flashlight,rotation,battery,cell,cast
+        volte1,volte2,wifi,bt,dnd,vowifi,lte1,lte2,onehand,flashlight,screenshot,rotation,battery,cell,cast
     </string>
 
     <!-- The minimum number of tiles to display in QuickSettings -->
@@ -119,7 +119,7 @@
 
     <!-- Tiles native to System UI. Order should match "quick_settings_tiles_default" -->
     <string >
-        volte1,volte2,wifi,cell,battery,dnd,vowifi,lte1,lte2,flashlight,rotation,bt,airplane,location,hotspot,inversion,saver,dark,work,cast,night,onehand,longscreenshot
+        volte1,volte2,wifi,cell,battery,dnd,vowifi,lte1,lte2,flashlight,screenshot,rotation,bt,airplane,location,hotspot,inversion,saver,dark,work,cast,night,onehand,longscreenshot
     </string>

```

通过以上方法添加截图的 tiles，然后在实现截图快捷功能

3.2 添加自定义 [Tile](https://so.csdn.net/so/search?q=Tile&spm=1001.2101.3001.7020) ScreenShotTile.java
--------------------------------------------------------------------------------------------------

```
package com.android.systemui.qs.tiles;

import android.app.ActivityManager;
import android.content.Intent;
import android.service.quicksettings.Tile;
import android.widget.Switch;

import com.android.systemui.R;
import com.android.systemui.plugins.qs.QSTile.BooleanState;
import com.android.systemui.qs.QSHost;
import com.android.systemui.qs.tileimpl.QSTileImpl;
import com.android.internal.util.ScreenshotHelper;
import android.os.RemoteException;
import android.os.Handler;
import android.util.Log;
import android.os.Message;
import javax.inject.Inject;

/** Quick settings tile: Control flashlight **/
public class ScreenShotTile extends QSTileImpl<BooleanState>{

    private final Icon mIcon = ResourceIcon.get(com.android.internal.R.drawable.ic_screenshot);
    private Handler mHandler;
    private ScreenshotHelper mScreenshotHelper;

    //这个Inject还是必须要有的，不然会导致编译不过，一开始就入坑了
    @Inject
    public ScreenShotTile(QSHost host) {
        super(host);
        mHandler = new Handler();
        mScreenshotHelper = new ScreenshotHelper(mContext);
		
    }

    @Override
    public BooleanState newTileState() {
        return new BooleanState();
    }

// 添加点击事件
    @Override
    protected void handleClick() {
        mHost.collapsePanels();

        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                     //点击调用截图功能
				   mScreenshotHelper.takeScreenshot(1, true, true, mHandler);
                }
            }, 1000);

    }

    @Override
    protected void handleUpdateState(BooleanState state, Object arg) {
        state.icon = mIcon;
        state.label = "screen shot";
        state.contentDescription = "screen shot";
    }

    @Override
    public int getMetricsCategory() {
        return 0;
    }

    @Override
    public Intent getLongClickIntent() {
        return null;
    }

    @Override
    protected void handleSetListening(boolean listening) {

    }

    @Override
    public CharSequence getTileLabel() {
    //quick_settings_screenshot_label 这种字符串String.xml中添加就可以了
        return "screen shot";
    }

}

```

在 ScreenShotTile 中模仿其他的 Tile 实现相关的布局 点击事件的功能，从而添加截图快捷功能

3.3 添加 ScreenShotTile 到 QSFactoryImpl 中完成截图快捷的布局功能
--------------------------------------------------

具体功能如下：

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
index ffca7b01d1..140f6405ca 100755
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
@@ -52,6 +52,7 @@ import com.android.systemui.qs.tiles.VolteTile;
 import com.android.systemui.qs.tiles.VoWifiTile;
 import com.android.systemui.qs.tiles.WifiTile;
 import com.android.systemui.qs.tiles.WorkModeTile;
+import com.android.systemui.qs.tiles.ScreenShotTile;
 import com.android.systemui.util.leak.GarbageMonitor;
 import com.sprd.systemui.qs.tiles.LongScreenshotTile;
 import com.sprd.systemui.qs.tiles.OneHandTile;
@@ -90,7 +91,7 @@ public class QSFactoryImpl implements QSFactory {
     private final Provider<NfcTile> mNfcTileProvider;
     private final Provider<GarbageMonitor.MemoryTile> mMemoryTileProvider;
     private final Provider<UiModeNightTile> mUiModeNightTileProvider;
-
+    private final Provider<ScreenShotTile> mScreenShotTileProvider;
     private QSTileHost mHost;
 
     @Inject
@@ -114,6 +115,7 @@ public class QSFactoryImpl implements QSFactory {
             Provider<NightDisplayTile> nightDisplayTileProvider,
             Provider<NfcTile> nfcTileProvider,
             Provider<GarbageMonitor.MemoryTile> memoryTileProvider,
+                       Provider<ScreenShotTile> screenShotTileProvider,
             Provider<UiModeNightTile> uiModeNightTileProvider) {
         mWifiTileProvider = wifiTileProvider;
         mBluetoothTileProvider = bluetoothTileProvider;
@@ -136,6 +138,7 @@ public class QSFactoryImpl implements QSFactory {
         mNfcTileProvider = nfcTileProvider;
         mMemoryTileProvider = memoryTileProvider;
         mUiModeNightTileProvider = uiModeNightTileProvider;
+               mScreenShotTileProvider = screenShotTileProvider;
     }
 
     public void setHost(QSTileHost host) {
@@ -191,6 +194,8 @@ public class QSFactoryImpl implements QSFactory {
                 return mHotspotTileProvider.get();
             case "user":
                 return mUserTileProvider.get();
+            case "screenshot":
+                return mScreenShotTileProvider.get();
             case "battery":
                 /*UNISOC Bug 1074234, 885650, Super power feature @{ */
                 if (SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE) {

```

通过上述的添加步骤发现功能实现了 也就完成了这部分功能要求开发