> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124562947)

### 1. 概述

在 11.0 12.0 的产品定制开发中，在系统开机以后会弹出一个系统通知来，通知内容就是提示 “性能受到影响。  
要停用，请查看引导加载程序” 这么个提示通知，其实对于开机弹出这个检测系统性能非常不友好，所以  
产品要求去掉这个通知，就要看是在哪里发出这个通知，然后屏蔽掉就可以了  
如图:  
![](https://img-blog.csdnimg.cn/7669005f66524b1f8f9574672ea79c61.png#pic_center)

### 2.SystemUI 去掉 性能受到影响。要停用，请查看引导加载程序 的通知核心类

```
/frameworks/base/core/res/res/values-zh-rCN/strings.xml
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```

### 3.SystemUI 去掉 性能受到影响。要停用，请查看引导加载程序 的通知核心功能分析和实现

### 3.1 首选要查询这个通知是哪里发出来的 然后屏蔽掉

开机后要么是 AMS ATMS PMS WMS 会发送些通知什么的 所以就要从这几个方面入手  
对于这个开机检测性能的通知，如果在系统中找相关代码，对系统源码不太了解的话会花很多时间的，其实主要  
就是通过搜索关键字就可以找到这个通知

```
/frameworks/base/core/res/res/values-zh-rCN/strings.xml
<string </string>
<string </string>
<string </string>
<string </string>
<string 新增："</font></string>
<string 由“<xliff:g id="APP_NAME">%1$s</xliff:g>”提供。"</string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string </string>
<string 点按即可查看更多选项。"</string>
<string 正在为连接的设备充电。点按即可查看更多选项。"</string>
<string </string>
<string 连接的设备与此手机不兼容。点按即可了解详情。"</string>
<string </string>
<string </string>
<string 选择即可停用 USB 调试功能。"</string>
<string </string>
<string </string>
<string 选择即可停用无线调试功能。"</string>
<string </string>
<string 恢复出厂设置以停用自动化测试框架模式。"</string>
<string </string>
<string 性能受到影响。要停用，请查看引导加载程序。"</string>
<string </string>
<string USB 端口已自动停用。点按即可了解详情。"</string>
<string </string>
<string 手机不再检测到液体或碎屑。"</string>
<string 正在生成错误报告…"</string>
<string 要分享错误报告吗？"</string>
<string 正在分享错误报告…"</string>
发现<string 性能受到影响。要停用，请查看引导加载程序。"</string>

```

在 string.xml 中发现  
console_running_notification_message 就是消息的内容  
搜索相关信息发现这里是构建通知是在  
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java  
所以接下来看下 ActivityManagerService.java 的相关源码分析

### 3.2ActivityManagerService.java 相关性能通知分析

```
private void showConsoleNotificationIfActive() {
if (!SystemProperties.get("init.svc.console").equals("running")) {
return;
}
String title = mContext
.getString(com.android.internal.R.string.console_running_notification_title);
String message = mContext
.getString(com.android.internal.R.string.console_running_notification_message);
Notification notification =
new Notification.Builder(mContext, SystemNotificationChannels.DEVELOPER)
.setSmallIcon(com.android.internal.R.drawable.stat_sys_adb)
.setWhen(0)
.setOngoing(true)
.setTicker(title)
.setDefaults(0)  // please be quiet
.setColor(mContext.getColor(
com.android.internal.R.color
.system_notification_accent_color))
.setContentTitle(title)
.setContentText(message)
.setVisibility(Notification.VISIBILITY_PUBLIC)
.build();
      NotificationManager notificationManager =
              mContext.getSystemService(NotificationManager.class);
      notificationManager.notifyAsUser(
              null, SystemMessage.NOTE_SERIAL_CONSOLE_ENABLED, notification, UserHandle.ALL);
  }

```

通过上面代码发现 showConsoleNotificationIfActive() 就是在这里发出性能的通知  
在 systemui 的通知栏弹出这个通知，所以屏蔽掉就好了

```
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
final void finishBooting() {
TimingsTraceAndSlog t = new TimingsTraceAndSlog(TAG + "Timing",
Trace.TRACE_TAG_ACTIVITY_MANAGER);
t.traceBegin("FinishBooting");
          ...
          maybeLogUserspaceRebootEvent();
          mUserController.scheduleStartProfiles();
      }
     // UART is on if init's console service is running, send a warning notification.
  //  showConsoleNotificationIfActive();

     t.traceEnd();
  }

```

finishBooting() 这里发出通知的 所以在这里屏蔽掉就可以了