> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124790139)

在内置某些的 app 时，由于 so 库的原因，导致会出现点击 app 进入 app 时会崩溃，但是如果手动安装 app 时又会一切正常，所以在没有其他办法的情况下  
就想着预安装的方法来安装这个 app. 然后在 Launcher3 拖拽卸载时，不让他卸载

接下来就看 Launcher3 app 长按卸载的流程  
在 luncher.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 中

```
<include
    android:id="@+id/drop_target_bar"
    layout="@layout/drop_target_bar" />

 

drop_targe_bar.xml

<?xml version="1.0" encoding="utf-8"?><!--
     Copyright (C) 2018 The Android Open Source Project

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->
<com.android.launcher3.DropTargetBar xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="@dimen/dynamic_grid_drop_target_size"
    android:layout_gravity="center_horizontal|top"
    android:focusable="false"
    android:alpha="0"
    android:theme="@style/HomeScreenElementTheme"
    android:visibility="invisible">

    <!-- Delete target -->
    <com.android.launcher3.DeleteDropTarget
        android:id="@+id/delete_target_text"
        style="@style/DropTargetButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="@string/remove_drop_target_label" />

    <!-- Uninstall target -->
    <com.android.launcher3.SecondaryDropTarget
        android:id="@+id/uninstall_target_text"
        style="@style/DropTargetButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="@string/uninstall_drop_target_label" />

</com.android.launcher3.DropTargetBar>

```

DeleteDropTarget.java 是负责处理长按图标删除 快捷图标和时钟等等的  
而 SecondaryDropTarget 是负责处理长按图标卸载 app 的  
当 app 长按时开始拖拽时，就会响应开始拖拽事件，在 Workspace 的 app 的列表页面 在 SecondaryDropTarget.java 就会判断是否支持卸载，支持卸载 就会出现卸载字样  
当移动到卸载区域内后就会 弹出卸载框 确定卸载就会卸载

先看看 ButtonDropTarget.java 的相关源码

```
public abstract class ButtonDropTarget extends TextView
        implements DropTarget, DragController.DragListener, OnClickListener {

    private static final Property<ButtonDropTarget, Integer> TEXT_COLOR =
            new Property<ButtonDropTarget, Integer>(Integer.TYPE, "textColor") {

                @Override
                public Integer get(ButtonDropTarget target) {
                    return target.getTextColor();
                }

                @Override
                public void set(ButtonDropTarget target, Integer value) {
                    target.setTextColor(value);
                }
            };

              @Override
    public void onDragStart(DropTarget.DragObject dragObject, DragOptions options) {
        mActive = supportsDrop(dragObject.dragInfo);
        mDrawable.setColorFilter(null);
        if (mCurrentColorAnim != null) {
            mCurrentColorAnim.cancel();
            mCurrentColorAnim = null;
        }
        setTextColor(mOriginalTextColor);
        setVisibility(mActive ? View.VISIBLE : View.GONE);

        mAccessibleDrag = options.isAccessibleDrag;
        setOnClickListener(mAccessibleDrag ? this : null);
    }
}

```

在开始拖拽时 会在 onDragStart() 中 处理拖拽相关的事情  
通过 supportsDrop(dragObject.dragInfo); 来处理在拖拽 app 图标时 要不要显示卸载 然后完成卸载功能

接下来看下 SecondaryDropTarget.java 的

```
supportsDrop(dragObject.dragInfo);方法

    @Override
    protected boolean supportsDrop(ItemInfo info) {
        return supportsAccessibilityDrop(info, getViewUnderDrag(info));
    }

    @Override
    public boolean supportsAccessibilityDrop(ItemInfo info, View view) {
        if (view instanceof AppWidgetHostView) {
            if (getReconfigurableWidgetId(view) != INVALID_APPWIDGET_ID) {
                setupUi(RECONFIGURE);
                return true;
            }
            return false;
        }

        setupUi(UNINSTALL);
        Boolean uninstallDisabled = mUninstallDisabledCache.get(info.user);
        if (uninstallDisabled == null) {
            UserManager userManager =
                    (UserManager) getContext().getSystemService(Context.USER_SERVICE);
            Bundle restrictions = userManager.getUserRestrictions(info.user);
            uninstallDisabled = restrictions.getBoolean(UserManager.DISALLOW_APPS_CONTROL, false)
                    || restrictions.getBoolean(UserManager.DISALLOW_UNINSTALL_APPS, false);
            mUninstallDisabledCache.put(info.user, uninstallDisabled);
        }
        // Cancel any pending alarm and set cache expiry after some time
        mCacheExpireAlarm.setAlarm(CACHE_EXPIRE_TIMEOUT);
        if (uninstallDisabled) {
            return false;
        }

        if (info instanceof ItemInfoWithIcon) {
            ItemInfoWithIcon iconInfo = (ItemInfoWithIcon) info;
            if ((iconInfo.runtimeStatusFlags & FLAG_SYSTEM_MASK) != 0) {
                return (iconInfo.runtimeStatusFlags & FLAG_SYSTEM_NO) != 0;
            }
        }
        return getUninstallTarget(info) != null;
    }

```

这里会根据 ItemInfo 的属性判断是否是系统应用  
所以我们要不让卸载第三方 app 可以根据他的 title 或者是 contentDescription 来判断就行了 如果是就返回 false 然后在拖拽时 就不会显示卸载了 就实现了功能

具体修改如下:

```
 @Override
    public boolean supportsAccessibilityDrop(ItemInfo info, View view) {
               
                //add core start
		android.util.Log.e(TAG,"title:"+info.title+"--contentDescription:"+info.contentDescription);
		if(info.title!=null&&info.title.equals("JNITest"))return false;
               // add code end 

        if (view instanceof AppWidgetHostView) {
            if (getReconfigurableWidgetId(view) != INVALID_APPWIDGET_ID) {
                setupUi(RECONFIGURE);
                return true;
            }
            return false;
        }

        setupUi(UNINSTALL);
        Boolean uninstallDisabled = mUninstallDisabledCache.get(info.user);
        if (uninstallDisabled == null) {
            UserManager userManager =
                    (UserManager) getContext().getSystemService(Context.USER_SERVICE);
            Bundle restrictions = userManager.getUserRestrictions(info.user);
            uninstallDisabled = restrictions.getBoolean(UserManager.DISALLOW_APPS_CONTROL, false)
                    || restrictions.getBoolean(UserManager.DISALLOW_UNINSTALL_APPS, false);
            mUninstallDisabledCache.put(info.user, uninstallDisabled);
        }
        // Cancel any pending alarm and set cache expiry after some time
        mCacheExpireAlarm.setAlarm(CACHE_EXPIRE_TIMEOUT);
        if (uninstallDisabled) {
            return false;
        }

        if (info instanceof ItemInfoWithIcon) {
            ItemInfoWithIcon iconInfo = (ItemInfoWithIcon) info;
            if ((iconInfo.runtimeStatusFlags & FLAG_SYSTEM_MASK) != 0) {
                return (iconInfo.runtimeStatusFlags & FLAG_SYSTEM_NO) != 0;
            }
        }
        return getUninstallTarget(info) != null;
    }

```

编译重启 发现实现了功能