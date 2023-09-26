> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125375110)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 核心代码](#t1)

[3. 核心代码分析](#t2)

 [3.1 先看下 StatusBarSignalPolicy.java 这里负责显示图标](#t3)

[3.2StatusBarIconController.java 代码分析](#t4)

[3.2 接下来看 config.xml](#t5)

[4. 功能修改](#t6)

1. 概述
-----

在 11.0 的某些产品 会出现耳机图标一直显示在状态栏上不消失，所以为了解决这个问题 就把耳机图标 加入到图标黑名单里面就好了，改法和 10.0 的稍微有些差别

2. 核心代码
-------

```
下面核心代码
/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarSignalPolicy.java
/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconController.java
 /frameworks/base/packages/SystemUI/res/values/config.xml
```

3. 核心代码分析
---------

####  3.1 先看下 StatusBarSignalPolicy.java 这里负责显示图标

路径:/frameworks/base/packages/[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020)/src/com/android/systemui/statusbar/phone/StatusBarSignalPolicy.java

StatusBarSignalPolicy 构造方法中获取需要显示的图标信息

```
public StatusBarSignalPolicy(Context context, StatusBarIconController iconController) {
          mContext = context;
  
          mSlotAirplane = mContext.getString(com.android.internal.R.string.status_bar_airplane);
          mSlotMobile   = mContext.getString(com.android.internal.R.string.status_bar_mobile);
          mSlotWifi     = mContext.getString(com.android.internal.R.string.status_bar_wifi);
          mSlotEthernet = mContext.getString(com.android.internal.R.string.status_bar_ethernet);
          mSlotVpn      = mContext.getString(com.android.internal.R.string.status_bar_vpn);
          mActivityEnabled = mContext.getResources().getBoolean(R.bool.config_showActivity);
  
          mIconController = iconController;
          mNetworkController = Dependency.get(NetworkController.class);
          mSecurityController = Dependency.get(SecurityController.class);
  
          Dependency.get(TunerService.class).addTunable(this, StatusBarIconController.ICON_BLACKLIST);
          mNetworkController.addCallback(this);
          mSecurityController.addCallback(this);
      }
```

```
 
@Override
public void onTuningChanged(String key, String newValue) {
if (!StatusBarIconController.ICON_BLACKLIST.equals(key)) {
return;
}
ArraySet<String> blockList = StatusBarIconController.getIconBlacklist(mContext, newValue);
boolean blockAirplane = blockList.contains(mSlotAirplane);
boolean blockMobile = blockList.contains(mSlotMobile);
boolean blockWifi = blockList.contains(mSlotWifi);
boolean blockEthernet = blockList.contains(mSlotEthernet);
 
if (blockAirplane != mBlockAirplane || blockMobile != mBlockMobile
|| blockEthernet != mBlockEthernet || blockWifi != mBlockWifi) {
mBlockAirplane = blockAirplane;
mBlockMobile = blockMobile;
mBlockEthernet = blockEthernet;
mBlockWifi = blockWifi || mForceBlockWifi;
// Re-register to get new callbacks.
mNetworkController.removeCallback(this);
mNetworkController.addCallback(this);
}
}
```

StatusBarIconController.getIconBlacklist(mContext, newValue); 就是获取黑名单列表 接下来看 StatusBarIconController.java

#### 3.2StatusBarIconController.java 代码分析

路径: /frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarIconController.java

```
 
 
/** Reads the default blacklist from config value unless blacklistStr is provided. */
static ArraySet<String> getIconBlacklist(Context context, String blackListStr) {
ArraySet<String> ret = new ArraySet<>();
String[] blacklist = blackListStr == null
? context.getResources().getStringArray(R.array.config_statusBarIconBlackList)
: blackListStr.split(",");
for (String slot : blacklist) {
if (!TextUtils.isEmpty(slot)) {
ret.add(slot);
}
}
return ret;
}
```

#### 3.2 接下来看 config.xml

/frameworks/base/packages/SystemUI/res/values/config.xml

```
 
<!-- Defines the blacklist for system icons.  That is to say, the icons in the status bar that
are part of the blacklist are never displayed. Each item in the blacklist must be a string
defined in core/res/res/config.xml to properly blacklist the icon.
-->
<string-array >
<item>@*android:string/status_bar_rotate</item>
</string-array>
 
<!-- A path similar to frameworks/base/core/res/res/values/config.xml
config_mainBuiltInDisplayCutout that describes a path larger than the exact path of a display
cutout. If present as well as config_enableDisplayCutoutProtection is set to true, then
SystemUI will draw this "protection path" instead of the display cutout path that is normally
used for anti-aliasing.
 
This path will only be drawn when the front-facing camera turns on, otherwise the main
DisplayCutout path will be rendered
-->
<string translatable="false" ></string>
 
<!--  ID for the camera that needs extra protection -->
<string translatable="false" ></string>
 
<!-- Comma-separated list of packages to exclude from camera protection e.g.
"com.android.systemui,com.android.xyz" -->
<string translatable="false" ></string>
 
<!--  Flag to turn on the rendering of the above path or not  -->
<bool >false</bool>
 
<!-- Respect drawable/rounded.xml intrinsic size for multiple radius corner path customization -->
<bool >false</bool>
 
<!-- Controls can query 2 preferred applications for limited number of suggested controls.
This config value should contain a list of package names of thoses preferred applications.
-->
<string-array translatable="false"  />
 
<!-- Max number of columns for quick controls area -->
<integer >2</integer>
 
<!-- Max number of columns for power menu -->
<integer >3</integer>
 
<!-- If the dp width of the available space is <= this value, potentially adjust the number
of columns-->
<integer >320</integer>
<!-- If the config font scale is >= this value, potentially adjust the number of columns-->
<item >1.25</item>
</resources>
 
 
```

#### 4. 功能修改

```
 
  修改如下:
  <string-array >
         <item>@*android:string/status_bar_rotate</item>
      +   <item>@*android:string/status_bar_headset</item>
     </string-array>
添加headset耳机图标名称就可以了
```