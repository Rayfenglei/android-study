> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760619)

### 1. 概述

在 11.0 12.0 的产品定制化开发中，对 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 下拉状态栏的 QuickQSPanel 区域有快捷功能键开关，对于增加各种响铃快捷也是常用功能，有需要增加响铃功能开关功能，接下来就来实现这个功能

### 2. SystemUI 下拉状态栏增加响铃功能的核心类

```
frameworks/base/packages/SystemUI/res/values/config.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java

```

### 3.SystemUI 下拉状态栏增加响铃功能的核心功能实现和分析

实现功能分析  
1. 在 config 中的 quick_settings_tiles_default 的属性值添加响铃字符串  
2. 在增加响铃的 Tile 类实现响铃的快捷功能  
3. 在 QSFactoryImpl 实现响铃的相关调用

### 3.1 config.xml 中 quick_settings_tiles_default 添加 ring 字符串表示响铃功能

```
diff --git a/frameworks/base/packages/SystemUI/res/values/config.xml b/frameworks/base/packages/SystemUI/res/values/config.xml
index b3b2ca4..d59ac48 100755 (executable)
--- a/frameworks/base/packages/SystemUI/res/values/config.xml
+++ b/frameworks/base/packages/SystemUI/res/values/config.xml
@@ -93,7 +93,7 @@
     <bool >true</bool>
 
     <!-- The maximum number of tiles in the QuickQSPanel -->
-    <integer >6</integer>
+    <integer >9</integer>
 
     <!-- Whether QuickSettings is in a phone landscape -->
     <bool >false</bool>
@@ -109,11 +109,11 @@
     <!-- The number of columns that the top level tiles span in the QuickSettings -->
     <integer >1</integer>
 
-    <!-- The default tiles to display in QuickSettings rotation cast lte1,lte2,cell,airplane-->
+    <!-- The default tiles to display in QuickSettings -->
     <string >
-        volte1,volte2,wifi,bt,dnd,vowifi,onehand,flashlight,battery
+        volte1,volte2,wifi,bt,vowifi,onehand,ring,screenshot,cast,rotation,night,airplane,flashlight
     </string>
-
+    <string </string>
     <!-- The minimum number of tiles to display in QuickSettings -->
     <integer >6</integer>

```

### 3.2 在 SystemUI 下增加 RingTile.java 响铃功能开关

实现思路: 根据 AudioManager 的相关 api 来获取当前的响铃模式，然后来设置响铃模式  
具体实现代码如下:

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
import android.media.AudioManager;
import android.content.Context;
/** Quick settings tile: Control flashlight **/
public class RingTile extends QSTileImpl<BooleanState>{

    private final Icon mIcon = ResourceIcon.get(R.drawable.ic_icon_information_h);
    private Handler mHandler;
    private AudioManager mAudioManager;
    @Inject
    public RingTile(QSHost host) {
        super(host);
        mHandler = new Handler();
        mAudioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
    }

    @Override
    public BooleanState newTileState() {
        return new BooleanState();
    }

    @Override
    protected void handleClick() {
        //点击开关后来刷新背景
        boolean newState = !mState.value;
        refreshState(newState);
        //获取当前响铃模式 然后更改响铃模式
        int ringerMode = mAudioManager.getRingerMode();
		if(ringerMode!=AudioManager.RINGER_MODE_SILENT){
			mAudioManager.setRingerMode(AudioManager.RINGER_MODE_SILENT);
		}else{
            mAudioManager.setRingerMode(AudioManager.RINGER_MODE_NORMAL);	
		}
    }

    @Override
    protected void handleUpdateState(BooleanState state, Object arg) {
		int ringerMode = mAudioManager.getRingerMode();
        //根据state.value的值 来更换开关背景
        state.value = ringerMode!=AudioManager.RINGER_MODE_SILENT;
        state.icon = mIcon;
        state.label = mContext.getString(R.string.quick_settings_ring_label);
        state.contentDescription = mContext.getString(R.string.quick_settings_ring_label);
		state.state = state.value ? Tile.STATE_ACTIVE : Tile.STATE_INACTIVE;
    }

    @Override
    public int getMetricsCategory() {
        return 0;
    }

    @Override
    public Intent getLongClickIntent() {
        return new Intent();
    }

    @Override
    protected void handleSetListening(boolean listening) {

    }

    @Override
    public CharSequence getTileLabel() {
        return mContext.getString(R.string.quick_settings_ring_label);
    }
}

```

通过继承 QSTileImpl 构建 QSTile 的基本功能，在 handleUpdateState 中根据点击更好图标在 handleClick 更加点击事件处理相对的事件功能

3.3 在 QSFactoryImpl.java 中添加加载 RingTile 响铃功能的相关配置方法  
路径：  
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
index ffca7b0..291a721 100755 (executable)
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java
@@ -55,7 +55,8 @@ import com.android.systemui.qs.tiles.WorkModeTile;
 import com.android.systemui.util.leak.GarbageMonitor;
 import com.sprd.systemui.qs.tiles.LongScreenshotTile;
 import com.sprd.systemui.qs.tiles.OneHandTile;

+import com.android.systemui.qs.tiles.RingTile;
 import com.sprd.systemui.util.SprdPowerManagerUtil;
 
 import javax.inject.Inject;
@@ -90,7 +91,8 @@ public class QSFactoryImpl implements QSFactory {
     private final Provider<NfcTile> mNfcTileProvider;
     private final Provider<GarbageMonitor.MemoryTile> mMemoryTileProvider;
     private final Provider<UiModeNightTile> mUiModeNightTileProvider;
-

+    private final Provider<RingTile> mRingTileProvider;
     private QSTileHost mHost;
 
     @Inject
@@ -108,6 +110,8 @@ public class QSFactoryImpl implements QSFactory {
             Provider<LocationTile> locationTileProvider,
             Provider<CastTile> castTileProvider,
             Provider<HotspotTile> hotspotTileProvider,
+                       Provider<RingTile> ringTileProvider,
             Provider<UserTile> userTileProvider,
             Provider<BatterySaverTile> batterySaverTileProvider,
             Provider<DataSaverTile> dataSaverTileProvider,
@@ -136,6 +140,8 @@ public class QSFactoryImpl implements QSFactory {
         mNfcTileProvider = nfcTileProvider;
         mMemoryTileProvider = memoryTileProvider;
         mUiModeNightTileProvider = uiModeNightTileProvider;
+               mRingTileProvider = ringTileProvider;
     }
 
     public void setHost(QSTileHost host) {
@@ -183,6 +189,10 @@ public class QSFactoryImpl implements QSFactory {
                 return mLongScreenshotTileProvider.get();
             case "onehand":
                 return mOneHandTileProvider.get();

+                       case "ring":
+                return mRingTileProvider.get();
             case "location":
                 return mLocationTileProvider.get();
  }

```

在 QSFactoryImpl 中这一步最重要的一步，setHost(QSTileHost host) 更加响应的字符串，返回构建的对应的下拉快捷的功能类实现下拉快捷的功能