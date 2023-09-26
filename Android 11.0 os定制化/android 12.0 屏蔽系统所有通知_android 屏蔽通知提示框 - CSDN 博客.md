> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124972390)

### 1. 概述

在 12.0 的产品开发中最近公司项目要求 屏蔽系统所有通知 不需要在下拉状态栏显示通知功能实现  
要控制系统通知的开关功能, 需要屏蔽系统通知，而系统通知都是由 NoticationManagerServices.java 来管理的，这个 [NMS](https://so.csdn.net/so/search?q=NMS&spm=1001.2101.3001.7020) 服务管理通知就需要在 NotificationManagerService.java 来实现需求

### 2. 屏蔽系统所有通知的相关代码

```
frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java

```

### 3. 屏蔽系统所有通知的核心代码分析

接下来分析下 NMS 源码:  
NMS 服务也是在 systemserver 进程中启动的，然后在 onStart() 中初始化关于通知管理类的相关功能，  
所以先来看 onStart 的相关方法

```
@Override
public void onStart() {
SnoozeHelper snoozeHelper = new SnoozeHelper(getContext(), new SnoozeHelper.Callback() {
@Override
public void repost(int userId, NotificationRecord r) {
try {
if (DBG) {
Slog.d(TAG, "Reposting " + r.getKey());
}
enqueueNotificationInternal(r.sbn.getPackageName(), r.sbn.getOpPkg(),
r.sbn.getUid(), r.sbn.getInitialPid(), r.sbn.getTag(), r.sbn.getId(),
r.sbn.getNotification(), userId);
} catch (Exception e) {
Slog.e(TAG, "Cannot un-snooze notification", e);
}
}
}, mUserProfiles);

final File systemDir = new File(Environment.getDataDirectory(), "system");

init(Looper.myLooper(),
AppGlobals.getPackageManager(), getContext().getPackageManager(),
getLocalService(LightsManager.class),
new NotificationListeners(AppGlobals.getPackageManager()),
new NotificationAssistants(getContext(), mNotificationLock, mUserProfiles,
AppGlobals.getPackageManager()),
new ConditionProviders(getContext(), mUserProfiles, AppGlobals.getPackageManager()),
null, snoozeHelper, new NotificationUsageStats(getContext()),
new AtomicFile(new File(
systemDir, "notification_policy.xml"), "notification-policy"),
(ActivityManager) getContext().getSystemService(Context.ACTIVITY_SERVICE),
getGroupHelper(), ActivityManager.getService(),
LocalServices.getService(UsageStatsManagerInternal.class),
LocalServices.getService(DevicePolicyManagerInternal.class),
UriGrantsManager.getService(),
LocalServices.getService(UriGrantsManagerInternal.class),
(AppOpsManager) getContext().getSystemService(Context.APP_OPS_SERVICE),
getContext().getSystemService(UserManager.class));

// register for various Intents
IntentFilter filter = new IntentFilter();
filter.addAction(Intent.ACTION_SCREEN_ON);
filter.addAction(Intent.ACTION_SCREEN_OFF);
filter.addAction(TelephonyManager.ACTION_PHONE_STATE_CHANGED);
filter.addAction(Intent.ACTION_USER_PRESENT);
filter.addAction(Intent.ACTION_USER_STOPPED);
filter.addAction(Intent.ACTION_USER_SWITCHED);
filter.addAction(Intent.ACTION_USER_ADDED);
filter.addAction(Intent.ACTION_USER_REMOVED);
filter.addAction(Intent.ACTION_USER_UNLOCKED);
filter.addAction(Intent.ACTION_MANAGED_PROFILE_UNAVAILABLE);
getContext().registerReceiverAsUser(mIntentReceiver, UserHandle.ALL, filter, null, null);

IntentFilter pkgFilter = new IntentFilter();
pkgFilter.addAction(Intent.ACTION_PACKAGE_ADDED);
pkgFilter.addAction(Intent.ACTION_PACKAGE_REMOVED);
pkgFilter.addAction(Intent.ACTION_PACKAGE_CHANGED);
pkgFilter.addAction(Intent.ACTION_PACKAGE_RESTARTED);
pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
pkgFilter.addDataScheme("package");
getContext().registerReceiverAsUser(mPackageIntentReceiver, UserHandle.ALL, pkgFilter, null,
null);

IntentFilter suspendedPkgFilter = new IntentFilter();
suspendedPkgFilter.addAction(Intent.ACTION_PACKAGES_SUSPENDED);
suspendedPkgFilter.addAction(Intent.ACTION_PACKAGES_UNSUSPENDED);
suspendedPkgFilter.addAction(Intent.ACTION_DISTRACTING_PACKAGES_CHANGED);
getContext().registerReceiverAsUser(mPackageIntentReceiver, UserHandle.ALL,
suspendedPkgFilter, null, null);

IntentFilter sdFilter = new IntentFilter(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
getContext().registerReceiverAsUser(mPackageIntentReceiver, UserHandle.ALL, sdFilter, null,
null);

IntentFilter timeoutFilter = new IntentFilter(ACTION_NOTIFICATION_TIMEOUT);
timeoutFilter.addDataScheme(SCHEME_TIMEOUT);
getContext().registerReceiver(mNotificationTimeoutReceiver, timeoutFilter);

IntentFilter settingsRestoredFilter = new IntentFilter(Intent.ACTION_SETTING_RESTORED);
getContext().registerReceiver(mRestoreReceiver, settingsRestoredFilter);

IntentFilter localeChangedFilter = new IntentFilter(Intent.ACTION_LOCALE_CHANGED);
getContext().registerReceiver(mLocaleChangeReceiver, localeChangedFilter);

publishBinderService(Context.NOTIFICATION_SERVICE, mService, /* allowIsolated= */ false,
DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL);
publishLocalService(NotificationManagerInternal.class, mInternalService);
}

```

NotificationManagerInternal.class 属于 NMS 的本地服务，正在的处理系统通知的相关工作  
在 onstart 中启动这个本地服务  
在启动这个服务的时候

```
publishLocalService(NotificationManagerInternal.class, mInternalService);

```

这里  
绑定本地服务 mInternalService 来处理系统通知接下来看 NotificationManagerInternal 的相关代码

```
/**
* The private API only accessible to the system process.
*/
private final NotificationManagerInternal mInternalService = new NotificationManagerInternal() {
@Override
public NotificationChannel getNotificationChannel(String pkg, int uid, String
channelId) {
return mPreferencesHelper.getNotificationChannel(pkg, uid, channelId, false);
}

@Override
public void enqueueNotification(String pkg, String opPkg, int callingUid, int callingPid,
String tag, int id, Notification notification, int userId) {
enqueueNotificationInternal(pkg, opPkg, callingUid, callingPid, tag, id, notification,
userId);
}

@Override
public void removeForegroundServiceFlagFromNotification(String pkg, int notificationId,
int userId) {
checkCallerIsSystem();
mHandler.post(() -> {
synchronized (mNotificationLock) {
// strip flag from all enqueued notifications. listeners will be informed
// in post runnable.
List<NotificationRecord> enqueued = findNotificationsByListLocked(
mEnqueuedNotifications, pkg, null, notificationId, userId);
for (int i = 0; i < enqueued.size(); i++) {
removeForegroundServiceFlagLocked(enqueued.get(i));
}

// if posted notification exists, strip its flag and tell listeners
NotificationRecord r = findNotificationByListLocked(
mNotificationList, pkg, null, notificationId, userId);
if (r != null) {
removeForegroundServiceFlagLocked(r);
mRankingHelper.sort(mNotificationList);
mListeners.notifyPostedLocked(r, r);
}
}
});
}

@GuardedBy("mNotificationLock")
private void removeForegroundServiceFlagLocked(NotificationRecord r) {
if (r == null) {
return;
}
StatusBarNotification sbn = r.sbn;
// NoMan adds flags FLAG_NO_CLEAR and FLAG_ONGOING_EVENT when it sees
// FLAG_FOREGROUND_SERVICE. Hence it's not enough to remove
// FLAG_FOREGROUND_SERVICE, we have to revert to the flags we received
// initially *and* force remove FLAG_FOREGROUND_SERVICE.
sbn.getNotification().flags =
(r.mOriginalFlags & ~FLAG_FOREGROUND_SERVICE);
}
};


      void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
          final int callingPid, final String tag, final int id, final Notification notification,
          int incomingUserId, boolean postSilently) {
          if (DBG) {
              Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id
                      + " notification=" + notification);
          }
  
          if (pkg == null || notification == null) {
              throw new IllegalArgumentException("null not allowed: pkg=" + pkg
                      + " id=" + id + " notification=" + notification);
          }
  
          final int userId = ActivityManager.handleIncomingUser(callingPid,
                  callingUid, incomingUserId, true, false, "enqueueNotification", pkg);
          final UserHandle user = UserHandle.of(userId);
  
          // Can throw a SecurityException if the calling uid doesn't have permission to post
          // as "pkg"
          final int notificationUid = resolveNotificationUid(opPkg, pkg, callingUid, userId);
  
          if (notificationUid == INVALID_UID) {
              throw new SecurityException("Caller " + opPkg + ":" + callingUid
                      + " trying to post for invalid pkg " + pkg + " in user " + incomingUserId);
          }
  
          checkRestrictedCategories(notification);
  
          // Fix the notification as best we can.
          try {
              fixNotification(notification, pkg, tag, id, userId);
  
          } catch (Exception e) {
              Slog.e(TAG, "Cannot fix notification", e);
              return;
          }
  
          mUsageStats.registerEnqueuedByApp(pkg);
  
          final StatusBarNotification n = new StatusBarNotification(
                  pkg, opPkg, id, tag, notificationUid, callingPid, notification,
                  user, null, System.currentTimeMillis());
  
          // setup local book-keeping
          String channelId = notification.getChannelId();
          if (mIsTelevision && (new Notification.TvExtender(notification)).getChannelId() != null) {
              channelId = (new Notification.TvExtender(notification)).getChannelId();
          }
          String shortcutId = n.getShortcutId();
          final NotificationChannel channel = mPreferencesHelper.getConversationNotificationChannel(
                  pkg, notificationUid, channelId, shortcutId,
                  true /* parent ok */, false /* includeDeleted */);
          if (channel == null) {
              final String noChannelStr = "No Channel found for "
                      + "pkg=" + pkg
                      + ", channelId=" + channelId
                      + ", id=" + id
                      + ", tag=" + tag
                      + ", opPkg=" + opPkg
                      + ", callingUid=" + callingUid
                      + ", userId=" + userId
                      + ", incomingUserId=" + incomingUserId
                      + ", notificationUid=" + notificationUid
                      + ", notification=" + notification;
              Slog.e(TAG, noChannelStr);
              boolean appNotificationsOff = mPreferencesHelper.getImportance(pkg, notificationUid)
                      == NotificationManager.IMPORTANCE_NONE;
  
              if (!appNotificationsOff) {
                  doChannelWarningToast("Developer warning for package \"" + pkg + "\"\n" +
                          "Failed to post notification on channel \"" + channelId + "\"\n" +
                          "See log for more details");
              }
              return;
          }
  
          final NotificationRecord r = new NotificationRecord(getContext(), n, channel);
          r.setIsAppImportanceLocked(mPreferencesHelper.getIsAppImportanceLocked(pkg, callingUid));
          r.setPostSilently(postSilently);
          r.setFlagBubbleRemoved(false);
          r.setPkgAllowedAsConvo(mMsgPkgsAllowedAsConvos.contains(pkg));
  
          if ((notification.flags & Notification.FLAG_FOREGROUND_SERVICE) != 0) {
              final boolean fgServiceShown = channel.isFgServiceShown();
              if (((channel.getUserLockedFields() & NotificationChannel.USER_LOCKED_IMPORTANCE) == 0
                          || !fgServiceShown)
                      && (r.getImportance() == IMPORTANCE_MIN
                              || r.getImportance() == IMPORTANCE_NONE)) {
                  // Increase the importance of foreground service notifications unless the user had
                  // an opinion otherwise (and the channel hasn't yet shown a fg service).
                  if (TextUtils.isEmpty(channelId)
                          || NotificationChannel.DEFAULT_CHANNEL_ID.equals(channelId)) {
                      r.setSystemImportance(IMPORTANCE_LOW);
                  } else {
                      channel.setImportance(IMPORTANCE_LOW);
                      r.setSystemImportance(IMPORTANCE_LOW);
                      if (!fgServiceShown) {
                          channel.unlockFields(NotificationChannel.USER_LOCKED_IMPORTANCE);
                          channel.setFgServiceShown(true);
                      }
                      mPreferencesHelper.updateNotificationChannel(
                              pkg, notificationUid, channel, false);
                      r.updateNotificationChannel(channel);
                  }
              } else if (!fgServiceShown && !TextUtils.isEmpty(channelId)
                      && !NotificationChannel.DEFAULT_CHANNEL_ID.equals(channelId)) {
                  channel.setFgServiceShown(true);
                  r.updateNotificationChannel(channel);
              }
          }
  
          ShortcutInfo info = mShortcutHelper != null
                  ? mShortcutHelper.getValidShortcutInfo(notification.getShortcutId(), pkg, user)
                  : null;
          if (notification.getShortcutId() != null && info == null) {
              Slog.w(TAG, "notification " + r.getKey() + " added an invalid shortcut");
          }
          r.setShortcutInfo(info);
          r.setHasSentValidMsg(mPreferencesHelper.hasSentValidMsg(pkg, notificationUid));
          r.userDemotedAppFromConvoSpace(
                  mPreferencesHelper.hasUserDemotedInvalidMsgApp(pkg, notificationUid));
  
          if (!checkDisqualifyingFeatures(userId, notificationUid, id, tag, r,
                  r.getSbn().getOverrideGroupKey() != null)) {
              return;
          }
  
          if (info != null) {
              // Cache the shortcut synchronously after the associated notification is posted in case
              // the app unpublishes this shortcut immediately after posting the notification. If the
              // user does not modify the notification settings on this conversation, the shortcut
              // will be uncached by People Service when all the associated notifications are removed.
              mShortcutHelper.cacheShortcut(info, user);
          }
  
          // Whitelist pending intents.
          if (notification.allPendingIntents != null) {
              final int intentCount = notification.allPendingIntents.size();
              if (intentCount > 0) {
                  final ActivityManagerInternal am = LocalServices
                          .getService(ActivityManagerInternal.class);
                  final long duration = LocalServices.getService(
                          DeviceIdleInternal.class).getNotificationWhitelistDuration();
                  for (int i = 0; i < intentCount; i++) {
                      PendingIntent pendingIntent = notification.allPendingIntents.valueAt(i);
                      if (pendingIntent != null) {
                          am.setPendingIntentWhitelistDuration(pendingIntent.getTarget(),
                                  WHITELIST_TOKEN, duration);
                          am.setPendingIntentAllowBgActivityStarts(pendingIntent.getTarget(),
                                  WHITELIST_TOKEN, (FLAG_ACTIVITY_SENDER | FLAG_BROADCAST_SENDER
                                          | FLAG_SERVICE_SENDER));
                      }
                  }
              }
          }
  
          // Need escalated privileges to get package importance
          final long token = Binder.clearCallingIdentity();
          boolean isAppForeground;
          try {
              isAppForeground = mActivityManager.getPackageImportance(pkg) == IMPORTANCE_FOREGROUND;
          } finally {
              Binder.restoreCallingIdentity(token);
          }
          mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));
      }

```

真正的处理服务是 enqueueNotification（）负责处理通知而在 enqueueNotification（）中最终  
从以上代码分析其实就是 mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));; 来发送通知的  
所以屏蔽通知可以通过属性来控制是否发送 EnqueueNotificationRunnable 这个消息通知  
具体修改如下：

```
--- a/frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java
@@ -4827,8 +4827,8 @@ public class NotificationManagerService extends SystemService {
}
void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
          final int callingPid, final String tag, final int id, final Notification notification,
          int incomingUserId, boolean postSilently) {
          if (DBG) {
              Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id
                      + " notification=" + notification);
          }
  
          if (pkg == null || notification == null) {
              throw new IllegalArgumentException("null not allowed: pkg=" + pkg
                      + " id=" + id + " notification=" + notification);
          }
  
          final int userId = ActivityManager.handleIncomingUser(callingPid,
                  callingUid, incomingUserId, true, false, "enqueueNotification", pkg);
          final UserHandle user = UserHandle.of(userId);
  
          // Can throw a SecurityException if the calling uid doesn't have permission to post
          // as "pkg"
          final int notificationUid = resolveNotificationUid(opPkg, pkg, callingUid, userId);
  
          if (notificationUid == INVALID_UID) {
              throw new SecurityException("Caller " + opPkg + ":" + callingUid
                      + " trying to post for invalid pkg " + pkg + " in user " + incomingUserId);
          }
...

          // Need escalated privileges to get package importance
          final long token = Binder.clearCallingIdentity();
          boolean isAppForeground;
          try {
              isAppForeground = mActivityManager.getPackageImportance(pkg) == IMPORTANCE_FOREGROUND;
          } finally {
              Binder.restoreCallingIdentity(token);
          }
-
-        mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));
+               int expand_panel = Settings.Global.getInt(getContext().getContentResolver(),"expand_panel",1);
+        if(expand_panel==1)mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));
}

```

在不需要通知的时候 注释掉就可以了