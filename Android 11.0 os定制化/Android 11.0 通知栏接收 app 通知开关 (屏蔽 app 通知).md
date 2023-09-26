> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124908342)

在 11.0 定制化开发中，需要屏蔽通知栏的通知的需求，而系统所有的通知都有 NotificationManager.java 来负责管理, 所以我们  
只要 NotificationManager 这里面控制通知的发送就可以了

首先来了解下 NoticationManager.java

NotificationManager 常用方法介绍：  
public void cancelAll() 移除所有通知 (只是针对当前 Context 下的 Notification)  
public void cancel(int id) 移除标记为 id 的通知 (只是针对当前 Context 下的所有 Notification)  
public void notify(String tag ,int id, Notification notification) 将通知加入状态栏，标签为 tag，标记为 id  
public void notify(int id, Notification notification) 将通知加入状态栏，标记为 id

// 设置 flag 位  
FLAG_AUTO_CANCEL 该通知能被状态栏的清除按钮给清除掉  
FLAG_NO_CLEAR 该通知能被状态栏的清除按钮给清除掉  
FLAG_ONGOING_EVENT 通知放置在正在运行  
FLAG_INSISTENT 是否一直进行，比如音乐一直播放，知道用户响应

常用字段：  
contentIntent 设置 PendingIntent 对象，点击时发送该 Intent  
defaults 添加默认效果  
flags 设置 flag 位，例如 FLAG_NO_CLEAR 等  
icon 设置图标  
sound 设置声音  
tickerText 显示在状态栏中的文字  
when 发送此通知的时间戳

通过上面的方法了解到 所有的通知都会走 public void notify(String tag, int id, Notification notification) 来更新状态栏的通知  
所有解决方案：  
在 notify 中来添加[标志位](https://so.csdn.net/so/search?q=%E6%A0%87%E5%BF%97%E4%BD%8D&spm=1001.2101.3001.7020) 从而实现控制通知栏通知的显示

```
diff --git a/frameworks/base/core/java/android/app/NotificationManager.java b/frameworks/base/core/java/android/app/NotificationManager.java
old mode 100644
new mode 100755
index dd39376f80..0fdda1a931
--- a/frameworks/base/core/java/android/app/NotificationManager.java
+++ b/frameworks/base/core/java/android/app/NotificationManager.java
@@ -41,6 +41,7 @@ import android.os.RemoteException;
 import android.os.ServiceManager;
 import android.os.StrictMode;
 import android.os.UserHandle;
+import android.provider.Settings;
 import android.provider.Settings.Global;
 import android.service.notification.Adjustment;
 import android.service.notification.Condition;
@@ -441,6 +442,8 @@ public class NotificationManager {
      */
     public void notify(String tag, int id, Notification notification)
     {
+               int expand_panel = Settings.Global.getInt(mContext.getContentResolver(),"expand_panel",1);
+               if(expand_panel==0)return;
         notifyAsUser(tag, id, notification, mContext.getUser());
     }

```

在 notify 中通过标志位来处理通知的发送