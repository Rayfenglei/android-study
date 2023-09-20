> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872558)

### 1. 概述

在 11.0 产品的 laucher3 定制化开发中，在 hotseat 功能中有需求要求添加 allapp 按钮 点击按钮进入所有 app 页面，就是在 hotseat 的几个功能按钮中间放一个 allapp 功能键，实现点击进入 app 列表页

### 2.Hotseat 添加 allapp button 相关代码

```
packages/apps/Launcher3/res/xml/partner_default_layout.xml
packages/apps/Launcher3/src/com/android/launcher3/Hotseat.java
packages/apps/Launcher3/src/com/android/launcher3/Launcher.java

```

### 3.Hotseat 添加 allapp button 相关功能分析和实现

具体定制如下 ：  
添加需要的资源文件如下 ：

1 all_apps_button.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020)

```
<?xml version="1.0" encoding="utf-8"?>
<TextView style="@style/BaseIcon" />

```

```
all_apps_button_icon.xml

```

```
<?xml version="1.0" encoding="utf-8"?>


<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_focused="true" android:drawable="@drawable/ic_allapps_pressed" />
    <item android:state_pressed="true" android:drawable="@drawable/ic_allapps_pressed" />
    <item android:drawable="@drawable/ic_allapps" />
</selector>

```

### 3.1partner_default_layout.xml 中添加 allapp 功能键

```
packages/apps/Launcher3/res/xml/partner_default_layout.xml

```

```
<?xml version="1.0" encoding="utf-8"?>

<favorites xmlns:launcher="http://schemas.android.com/apk/res-auto/com.android.launcher3">


    <appwidget
    launcher:package
    launcher:class
    launcher:screen="0"
    launcher:x="1"
    launcher:y="1"
    launcher:spanX="4"
    launcher:spanY="2" 
  />

    <!-- Hotseat (We use the screen as the position of the item in the hotseat) -->
    <!-- Dialer, Messaging, Browser, Camera -->

    <resolve
        launcher:container="-101"
        launcher:screen="0"
        launcher:x="0"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MAPS;end" />
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MUSIC;end" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="1"
        launcher:x="1"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_GALLERY;end" />
        <favorite launcher:uri="#Intent;type=images/*;end" />
    </resolve>

    <!--favorite
        launcher:container="-101"
        launcher:screen="2"
        launcher:x="2"
        launcher:y="0" 
        launcher:package
        launcher:class/-->


    <resolve
        launcher:container="-101"
        launcher:screen="3"
        launcher:x="3"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.media.action.STILL_IMAGE_CAMERA;end" />
        <favorite launcher:uri="#Intent;action=android.intent.action.CAMERA_BUTTON;end" />
    </resolve>

    <resolve
        launcher:container="-101"
        launcher:screen="4"
        launcher:x="4"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.settings.SETTINGS;category=android.intent.category.DEFAULT;end"/>
    </resolve>

    <!-- <resolve
        launcher:container="-101"
        launcher:screen="3"
        launcher:x="3"
        launcher:y="0" >
        <favorite launcher:uri="#Intent;action=android.media.action.STILL_IMAGE_CAMERA;end" />
        <favorite launcher:uri="#Intent;action=android.intent.action.CAMERA_BUTTON;end" />
    </resolve> -->

    <!-- Bottom row -->
    <!-- Email, Gallery, Music, Settings -->
    <!-- <resolve
        launcher:screen="0"
        launcher:x="0"
        launcher:y="-1" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_EMAIL;end" />
        <favorite launcher:uri="mailto:" />
    </resolve>

    <resolve
        launcher:screen="0"
        launcher:x="1"
        launcher:y="-1" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_GALLERY;end" />
        <favorite launcher:uri="#Intent;type=images/*;end" />
    </resolve>

    <resolve
        launcher:screen="0"
        launcher:x="-2"
        launcher:y="-1" >
        <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MUSIC;end" />
    </resolve>

    <resolve
        launcher:screen="0"
        launcher:x="-1"
        launcher:y="-1" >
        <favorite launcher:uri="#Intent;action=android.settings.SETTINGS;category=android.intent.category.DEFAULT;end" />
    </resolve> -->

    <favorite
        launcher:screen="1"
        launcher:x="0"
        launcher:y="0" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="1"
        launcher:y="0" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="2"
        launcher:y="0" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="3"
        launcher:y="0" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="4"
        launcher:y="0" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="5"
        launcher:y="0" 
        launcher:package
        launcher:class
        />

    <favorite
        launcher:screen="1"
        launcher:x="0"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="1"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="2"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="3"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="4"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="5"
        launcher:y="1" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="0"
        launcher:y="2" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="1"
        launcher:y="2" 
        launcher:package
        launcher:class
        />
    <favorite
        launcher:screen="1"
        launcher:x="2"
        launcher:y="2" 
        launcher:package
        launcher:class
        />

</favorites>

```

### 3.2 最关键的部分 在 Hotseat.java 中添加主要功能

Hotseat.java 添加点击事件 和 加载 allapp 按钮代码  
对于 Hotseat.java 中添加 allapp 首先要在中间的位置添加功能键，然后在添加相关的点击事件  
当点击事件发生时跳转到 allapp 页面来显示 app 列表页实现其功能  
具体实现如下：

```
    packages/apps/Launcher3/src/com/android/launcher3/Hotseat.java

--- a/packages/apps/Launcher3/src/com/android/launcher3/Hotseat.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Hotseat.java
@@ -25,7 +25,7 @@ import android.view.View;
 import android.view.ViewDebug;
 import android.view.ViewGroup;
 import android.widget.FrameLayout;
-
+import com.android.launcher3.Utilities;
 import com.android.launcher3.graphics.RotationMode;
 import com.android.launcher3.logging.StatsLogUtils.LogContainerProvider;
 import com.android.launcher3.userevent.nano.LauncherLogProto;
@@ -33,13 +33,18 @@ import com.android.launcher3.userevent.nano.LauncherLogProto.Target;
 import com.android.launcher3.views.Transposable;
 import com.sprd.ext.LauncherAppMonitor;
 import com.sprd.ext.grid.HotseatController;
-
+import android.view.LayoutInflater;
+import android.widget.TextView;
+import android.graphics.drawable.Drawable;
 public class Hotseat extends CellLayout implements LogContainerProvider, Insettable, Transposable {
 
     @ViewDebug.ExportedProperty(category = "launcher")
     public boolean mHasVerticalHotseat;
     private final HotseatController mController;
-
+    private boolean DISABLE_ALL_APPS = false;
+    private Context mContext = null;
+    private final Launcher mLauncher;
+    private CellLayout mContent;
     public Hotseat(Context context) {
         this(context, null);
     }
@@ -51,6 +56,8 @@ public class Hotseat extends CellLayout implements LogContainerProvider, Insetta
     public Hotseat(Context context, AttributeSet attrs, int defStyle) {
         super(context, attrs, defStyle);
         mController = LauncherAppMonitor.getInstance(context).getHotseatController();
+               this.mContext = context;
+        mLauncher = Launcher.getLauncher(context);
     }
 
     public HotseatController getController() {
@@ -65,7 +72,11 @@ public class Hotseat extends CellLayout implements LogContainerProvider, Insetta
     public int getCellYFromOrder(int rank) {
         return mHasVerticalHotseat ? (getCountY() - (rank + 1)) : 0;
     }
-
+   @Override
+    protected void onFinishInflate() {
+        super.onFinishInflate();
+        mContent = findViewById(R.id.hotseat);
+    }
     public void resetLayout(boolean hasVerticalHotseat) {
         removeAllViewsInLayout();
         mHasVerticalHotseat = hasVerticalHotseat;
@@ -75,8 +86,43 @@ public class Hotseat extends CellLayout implements LogContainerProvider, Insetta
         } else {
             setGridSize(idp.numHotseatIcons, 1);
         }
+        addAllappsLayout();
     }
    加载allapp按钮
+    void addAllappsLayout() {
+ 
+        if (!DISABLE_ALL_APPS) {
+            // Add the Apps button
+            int mAllAppsButtonRank = 2;
+            LayoutInflater inflater = LayoutInflater.from(mContext);
+            TextView allAppsButton = (TextView)
+                    inflater.inflate(R.layout.all_apps_button, mContent, false);
+            Drawable d = mContext.getResources().getDrawable(R.drawable.all_apps_button_icon);
+            mLauncher.resizeIconDrawable(d);
+            allAppsButton.setCompoundDrawables(null, d, null, null);
+ 
+            allAppsButton.setContentDescription(mContext.getString(R.string.label_application));
+            if (mLauncher != null) {
+                //allAppsButton.setOnTouchListener(mLauncher.getHapticFeedbackTouchListener());
+            }
+            allAppsButton.setOnClickListener(new View.OnClickListener() {
+                @Override
+                public void onClick(android.view.View v) {
+                    if (mLauncher != null) {
+                        mLauncher.onClickAllAppsButton(v);
+                    }
+                }
+            });
+ 
+            // Note: We do this to ensure that the hotseat is always laid out in the orientation of
+            // the hotseat in order regardless of which orientation they were added
+            int x = getCellXFromOrder(mAllAppsButtonRank);
+            int y = getCellYFromOrder(mAllAppsButtonRank);
+            CellLayout.LayoutParams lp = new CellLayout.LayoutParams(2,0,1,1);
+            lp.canReorder = false;
+            mContent.addViewToCellLayout(allAppsButton, -1, allAppsButton.getId(), lp, true);
+        }
+    }
     @Override
     public void fillInLogContainerData(View v, ItemInfo info, Target target, Target targetParent) {
         target.gridX = info.cellX;

```

### 3.3 在 Launcher3 添加 allapp 点击事件的处理

当监听到 allapp 的点击事件时，进入 workspace 的第二页显示所有的 app 列表  
从而实现功能需求

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
@@ -80,7 +80,7 @@ import android.view.ViewGroup;
 import android.view.accessibility.AccessibilityEvent;
 import android.view.animation.OvershootInterpolator;
 import android.widget.Toast;
-
+import android.graphics.drawable.Drawable;
 import com.android.launcher3.DropTarget.DragObject;
 import com.android.launcher3.accessibility.LauncherAccessibilityDelegate;
 import com.android.launcher3.allapps.AllAppsContainerView;
@@ -2746,4 +2746,10 @@ public class Launcher extends BaseDraggingActivity implements LauncherExterns,
     public ScrimView getScrimView() {
         return mScrimView;
     }
+    public void onClickAllAppsButton(View v){
+        getStateManager().goToState(ALL_APPS);
+    }
+    public void resizeIconDrawable(Drawable icon) {
+        icon.setBounds(0, 0, mDeviceProfile.iconSizePx, mDeviceProfile.iconSizePx);
+    }
 }

```