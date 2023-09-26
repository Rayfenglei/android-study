> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124792269)

在 11.0 12.0 产品中由于项目需要，要添加商标的[水印](https://so.csdn.net/so/search?q=%E6%B0%B4%E5%8D%B0&spm=1001.2101.3001.7020)，所以功能差不多和安全模式的水印相同，进入安全模式的时候 左下角就有 安全模式 的水印  
所以按照[安全模式](https://so.csdn.net/so/search?q=%E5%AE%89%E5%85%A8%E6%A8%A1%E5%BC%8F&spm=1001.2101.3001.7020) 添加水印 完成功能即可

1. 查看 SystemServer 的相关代码找到安全模式的显示入口  
路径:/frameworks/base/services/java/com/android/server/SystemServer.java

private void startOtherServices() 中是启动其他服务的  
而

```
 if (safeMode) {
        mActivityManagerService.showSafeModeOverlay();
 }

```

这里就是显示安全模式的水印

所以就需要在这里添加 自定义系统水印

```
boolean debugMode = SystemProperties.getBoolean("persist.sys.debug.mode", false); // 属性
        if (safeMode) {
            mActivityManagerService.showSafeModeOverlay();
        }

	if (debugMode) { // 水印显示入口
           mActivityManagerService.showWaterMarksOverlay();
        }

```

在 device 下 buildinfo.sh 中定义属性 persist.sys.debug.mode=true;

然后在 AMS 中添加 showWaterMarksOverlay()  
路径为: frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java  
在 window 窗口 添加自定义 View 当做水印

```
 public final void showWaterMarksOverlay() {
		android.util.Log.e("WaterMarks","showWaterMarksOverlay");
        int resorce = com.android.internal.R.layout.water_marks;
        View waterMarksView = android.view.LayoutInflater.from(mContext).inflate(resorce, null);
        WindowManager.LayoutParams lp = new WindowManager.LayoutParams();
        lp.type = WindowManager.LayoutParams.TYPE_SECURE_SYSTEM_OVERLAY;
        lp.width = WindowManager.LayoutParams.WRAP_CONTENT;
        lp.height = WindowManager.LayoutParams.WRAP_CONTENT;
        lp.gravity = Gravity.BOTTOM;
        lp.format = waterMarksView.getBackground().getOpacity();
        lp.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        lp.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_SHOW_FOR_ALL_USERS;
        ((WindowManager)mContext.getSystemService(
                Context.WINDOW_SERVICE)).addView(waterMarksView, lp);
    }

```

然后在 framework/base/core/res/res/layout 下添加 water_marks.xml

```
<?xml version="1.0" encoding="utf-8"?>

<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content" 
	android:layout_height="wrap_content"
    android:gravity="center"
    android:padding="3dp"
    android:background="@android:color/transparent"
    android:text="pnr"
	android:textSize="16sp"
    android:textColor="@android:color/black"
    android:alpha="0.5"
/>

```

最后在 frameworks/base/core/res/res/values/symbols.xml 注册 water_marks.xml

```
<java-symbol type="layout"  />

```

编译发现底部有添加的水印 完成功能由于项目需要，要添加商标的水印，所以功能差不多和安全模式的水印相同，进入安全模式的时候 左下角就有 安全模式 的水印