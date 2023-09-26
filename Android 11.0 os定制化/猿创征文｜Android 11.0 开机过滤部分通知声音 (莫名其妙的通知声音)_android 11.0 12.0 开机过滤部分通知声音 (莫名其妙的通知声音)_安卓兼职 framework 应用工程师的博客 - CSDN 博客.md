> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126773022)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 开机过滤部分通知声音 (莫名其妙的通知声音) 核心代码](#2.%E5%BC%80%E6%9C%BA%E8%BF%87%E6%BB%A4%E9%83%A8%E5%88%86%E9%80%9A%E7%9F%A5%E5%A3%B0%E9%9F%B3%28%E8%8E%AB%E5%90%8D%E5%85%B6%E5%A6%99%E7%9A%84%E9%80%9A%E7%9F%A5%E5%A3%B0%E9%9F%B3%29%E6%A0%B8%E5%BF%83%E4%BB%A3%E7%A0%81)

[3. 开机过滤部分通知声音 (莫名其妙的通知声音) 功能分析代码实现](#3.%E5%BC%80%E6%9C%BA%E8%BF%87%E6%BB%A4%E9%83%A8%E5%88%86%E9%80%9A%E7%9F%A5%E5%A3%B0%E9%9F%B3%28%E8%8E%AB%E5%90%8D%E5%85%B6%E5%A6%99%E7%9A%84%E9%80%9A%E7%9F%A5%E5%A3%B0%E9%9F%B3%29%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0)

 [3.1 NotificationManager.java 发送通知流程](#t3)

[3.2 NotificationManagerService.java 关于通知的相关方法](#t4)

1. 概述
-----

 在开发产品的时候，有时候在开机的时候会有一些通知的声音，但是由于系统模块太多，也搞不清楚到底是哪个模块发出的通知声音，所以就需要从通知的流程来屏蔽这些通知声音

2. 开机过滤部分通知声音 (莫名其妙的通知声音) 核心代码
------------------------------

```
   frameworks/base/core/java/android/app/NotificationManager.java
frameworks/base/services/core/java/com/android/server/notification/NotificationManagerService.java
```

3. 开机过滤部分通知声音 (莫名其妙的通知声音) 功能分析代码实现
----------------------------------

###    3.1 NotificationManager.java 发送通知流程

```
  //当通知来临 更新通知
 public void notify(int id, Notification notification)
      {
          notify(null, id, notification);
      }
  
      /**
       * Posts a notification to be shown in the status bar. If a notification with
       * the same tag and id has already been posted by your application and has not yet been
       * canceled, it will be replaced by the updated information.
       *
       * All {@link android.service.notification.NotificationListenerService listener services} will
       * be granted {@link Intent#FLAG_GRANT_READ_URI_PERMISSION} access to any {@link Uri uris}
       * provided on this notification or the
       * {@link NotificationChannel} this notification is posted to using
       * {@link Context#grantUriPermission(String, Uri, int)}. Permission will be revoked when the
       * notification is canceled, or you can revoke permissions with
       * {@link Context#revokeUriPermission(Uri, int)}.
       *
       * @param tag A string identifier for this notification.  May be {@code null}.
       * @param id An identifier for this notification.  The pair (tag, id) must be unique
       *        within your application.
       * @param notification A {@link Notification} object describing what to
       *        show the user. Must not be null.
       */
    //当通知来临 更新通知    
public void notify(String tag, int id, Notification notification)
      {
          notifyAsUser(tag, id, notification, mContext.getUser());
      }
  
      /**
       * Posts a notification as a specified package to be shown in the status bar. If a notification
       * with the same tag and id has already been posted for that package and has not yet been
       * canceled, it will be replaced by the updated information.
       *
       * All {@link android.service.notification.NotificationListenerService listener services} will
       * be granted {@link Intent#FLAG_GRANT_READ_URI_PERMISSION} access to any {@link Uri uris}
       * provided on this notification or the
       * {@link NotificationChannel} this notification is posted to using
       * {@link Context#grantUriPermission(String, Uri, int)}. Permission will be revoked when the
       * notification is canceled, or you can revoke permissions with
       * {@link Context#revokeUriPermission(Uri, int)}.
       *
       * @param targetPackage The package to post the notification as. The package must have granted
       *                      you access to post notifications on their behalf with
       *                      {@link #setNotificationDelegate(String)}.
       * @param tag A string identifier for this notification.  May be {@code null}.
       * @param id An identifier for this notification.  The pair (tag, id) must be unique
       *        within your application.
       * @param notification A {@link Notification} object describing what to
       *        show the user. Must not be null.
      //当通知来临 根据包名更新通知   */
      
public void notifyAsPackage(@NonNull String targetPackage, @Nullable String tag, int id,
              @NonNull Notification notification) {
          INotificationManager service = getService();
          String sender = mContext.getPackageName();
  
          try {
              if (localLOGV) Log.v(TAG, sender + ": notify(" + id + ", " + notification + ")");
      //有NotificationManagerserivice服务来负责更新通知的相关内容        
service.enqueueNotificationWithTag(targetPackage, sender, tag, id,
                      fixNotification(notification), mContext.getUser().getIdentifier());
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
  
      /**
       * @hide
       */
      @UnsupportedAppUsage
      public void notifyAsUser(String tag, int id, Notification notification, UserHandle user)
      {//获取NotificationManagerserivice服务   
          INotificationManager service = getService();
          String pkg = mContext.getPackageName();
  
          try {
              if (localLOGV) Log.v(TAG, pkg + ": notify(" + id + ", " + notification + ")");//有NotificationManagerserivice服务来负责更新通知的相关内容   
              service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id,
                      fixNotification(notification), user.getIdentifier());
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
 
```

### public void notify(int id, [Notification](https://so.csdn.net/so/search?q=Notification&spm=1001.2101.3001.7020) notification) 和

notifyAsUser(tag, id, notification, mContext.getUser());// 负责更新来临的通知

最后在  在 notifyAsUser(String tag, int id, Notification notification, UserHandle user) 中 最终通过  
NotificationManagerService.java 的 enqueueNotificationWithTag 来发送通知

### 3.2 NotificationManagerService.java 关于通知的相关方法

```
          //处理通知的具体方法 @Override
          public void enqueueNotificationWithTag(String pkg, String opPkg, String tag, int id,
                  Notification notification, int userId) throws RemoteException {
//具体的处理方法              
enqueueNotificationInternal(pkg, opPkg, Binder.getCallingUid(),
                      Binder.getCallingPid(), tag, id, notification, userId);
          }
//处理通知的详细方法
void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
             final int callingPid, final String tag, final int id, final Notification notification,
             int incomingUserId) {
         enqueueNotificationInternal(pkg, opPkg, callingUid, callingPid, tag, id, notification,
         incomingUserId, false);
     }
 //继续调用此方法来处理通知
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
 //通知管理类
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
// 由EnqueueNotificationRunnable子线程处理通知 具体看EnqueueNotificationRunnable这个方法
         mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));
     }
 // EnqueueNotificationRunnable这个run方法里面处理相关方法
protected class EnqueueNotificationRunnable implements Runnable {
          private final NotificationRecord r;
          private final int userId;
          private final boolean isAppForeground;
  
          EnqueueNotificationRunnable(int userId, NotificationRecord r, boolean foreground) {
              this.userId = userId;
              this.r = r;
              this.isAppForeground = foreground;
          }
  
          @Override
          public void run() {
              synchronized (mNotificationLock) {
                  final Long snoozeAt =
                          mSnoozeHelper.getSnoozeTimeForUnpostedNotification(
                                  r.getUser().getIdentifier(),
                                  r.getSbn().getPackageName(), r.getSbn().getKey());
                  final long currentTime = System.currentTimeMillis();
                  if (snoozeAt.longValue() > currentTime) {
                      (new SnoozeNotificationRunnable(r.getSbn().getKey(),
                              snoozeAt.longValue() - currentTime, null)).snoozeLocked(r);
                      return;
                  }
  
                  final String contextId =
                          mSnoozeHelper.getSnoozeContextForUnpostedNotification(
                                  r.getUser().getIdentifier(),
                                  r.getSbn().getPackageName(), r.getSbn().getKey());
                  if (contextId != null) {
                      (new SnoozeNotificationRunnable(r.getSbn().getKey(),
                              0, contextId)).snoozeLocked(r);
                      return;
                  }
  //通知队列添加通知
                  mEnqueuedNotifications.add(r);
                  scheduleTimeoutLocked(r);
  
                  final StatusBarNotification n = r.getSbn();
                  if (DBG) Slog.d(TAG, "EnqueueNotificationRunnable.run for: " + n.getKey());
                  NotificationRecord old = mNotificationsByKey.get(n.getKey());
                  if (old != null) {
                      // Retain ranking information from previous record
                      r.copyRankingInformation(old);
                  }
  
                  final int callingUid = n.getUid();
                  final int callingPid = n.getInitialPid();
                  final Notification notification = n.getNotification();
                  final String pkg = n.getPackageName();
                  final int id = n.getId();
                  final String tag = n.getTag();
  
                  // We need to fix the notification up a little for bubbles
                  updateNotificationBubbleFlags(r, isAppForeground);
  
                  // Handle grouped notifications and bail out early if we
                  // can to avoid extracting signals.
                  handleGroupedNotificationLocked(r, old, callingUid, callingPid);
  
                  // if this is a group child, unsnooze parent summary
                  if (n.isGroup() && notification.isGroupChild()) {
                      mSnoozeHelper.repostGroupSummary(pkg, r.getUserId(), n.getGroupKey());
                  }
  
                  // This conditional is a dirty hack to limit the logging done on
                  //     behalf of the download manager without affecting other apps.
                  if (!pkg.equals("com.android.providers.downloads")
                          || Log.isLoggable("DownloadManager", Log.VERBOSE)) {
                      int enqueueStatus = EVENTLOG_ENQUEUE_STATUS_NEW;
                      if (old != null) {
                          enqueueStatus = EVENTLOG_ENQUEUE_STATUS_UPDATE;
                      }
                      EventLogTags.writeNotificationEnqueue(callingUid, callingPid,
                              pkg, id, tag, userId, notification.toString(),
                              enqueueStatus);
                  }
  // 具体由PostNotificationRunnable子线程来发送通知接下来看这个run方法里面处理通知的
                  // tell the assistant service about the notification
                  if (mAssistants.isEnabled()) {
                      mAssistants.onNotificationEnqueuedLocked(r);
                      mHandler.postDelayed(new PostNotificationRunnable(r.getKey()),
                              DELAY_FOR_ASSISTANT_TIME);
                  } else {
                      
                      mHandler.post(new PostNotificationRunnable(r.getKey()));
                  }
              }
          }
      }
 protected class PostNotificationRunnable implements Runnable {
          private final String key;
  
          PostNotificationRunnable(String key) {
              this.key = key;
          }
  
          @Override
          public void run() {//具体处理通知的方法
              synchronized (mNotificationLock) {
                  try {
                      NotificationRecord r = null;
                      int N = mEnqueuedNotifications.size();
//从通知队列里面处理每条 通知
                      for (int i = 0; i < N; i++) {
                          final NotificationRecord enqueued = mEnqueuedNotifications.get(i);
                          if (Objects.equals(key, enqueued.getKey())) {
                              r = enqueued;
                              break;
                          }
                      }
                      if (r == null) {
                          Slog.i(TAG, "Cannot find enqueued record for key: " + key);
                          return;
                      }
  
                      if (isBlocked(r)) {
                          Slog.i(TAG, "notification blocked by assistant request");
                          return;
                      }
  
                      final boolean isPackageSuspended =
                              isPackagePausedOrSuspended(r.getSbn().getPackageName(), r.getUid());
                      r.setHidden(isPackageSuspended);
                      if (isPackageSuspended) {
                          mUsageStats.registerSuspendedByAdmin(r);
                      }
                      NotificationRecord old = mNotificationsByKey.get(key);
                      final StatusBarNotification n = r.getSbn();
                      final Notification notification = n.getNotification();
  
                      // Make sure the SBN has an instance ID for statsd logging.
                      if (old == null || old.getSbn().getInstanceId() == null) {
                          n.setInstanceId(mNotificationInstanceIdSequence.newInstanceId());
                      } else {
                          n.setInstanceId(old.getSbn().getInstanceId());
                      }
  
                      int index = indexOfNotificationLocked(n.getKey());
                      if (index < 0) {
                          mNotificationList.add(r);
                          mUsageStats.registerPostedByApp(r);
                          r.setInterruptive(isVisuallyInterruptive(null, r));
                      } else {
                          old = mNotificationList.get(index);  // Potentially *changes* old
                          mNotificationList.set(index, r);
                          mUsageStats.registerUpdatedByApp(r, old);
                          // Make sure we don't lose the foreground service state.
                          notification.flags |=
                                  old.getNotification().flags & FLAG_FOREGROUND_SERVICE;
                          r.isUpdate = true;
                          final boolean isInterruptive = isVisuallyInterruptive(old, r);
                          r.setTextChanged(isInterruptive);
                          r.setInterruptive(isInterruptive);
                      }
  
                      mNotificationsByKey.put(n.getKey(), r);
  
                      // Ensure if this is a foreground service that the proper additional
                      // flags are set.
                      if ((notification.flags & FLAG_FOREGROUND_SERVICE) != 0) {
                          notification.flags |= FLAG_ONGOING_EVENT
                                  | FLAG_NO_CLEAR;
                      }
  
                      mRankingHelper.extractSignals(r);
                      mRankingHelper.sort(mNotificationList);
                      final int position = mRankingHelper.indexOf(mNotificationList, r);
  
                      int buzzBeepBlinkLoggingCode = 0;
                      if (!r.isHidden()) {
                          buzzBeepBlinkLoggingCode = buzzBeepBlinkLocked(r);
                      }
  
                      if (notification.getSmallIcon() != null) {
                          StatusBarNotification oldSbn = (old != null) ? old.getSbn() : null;
                          mListeners.notifyPostedLocked(r, old);
                          if ((oldSbn == null || !Objects.equals(oldSbn.getGroup(), n.getGroup()))
                                  && !isCritical(r)) {
                              mHandler.post(new Runnable() {
                                  @Override
                                  public void run() {
                                      mGroupHelper.onNotificationPosted(
                                              n, hasAutoGroupSummaryLocked(n));
                                  }
                              });
                          } else if (oldSbn != null) {
                              final NotificationRecord finalRecord = r;
                              mHandler.post(() -> mGroupHelper.onNotificationUpdated(
                                      finalRecord.getSbn(), hasAutoGroupSummaryLocked(n)));
                          }
                      } else {
                          Slog.e(TAG, "Not posting notification without small icon: " + notification);
                          if (old != null && !old.isCanceled) {
                              mListeners.notifyRemovedLocked(r,
                                      NotificationListenerService.REASON_ERROR, r.getStats());
                              mHandler.post(new Runnable() {
                                  @Override
                                  public void run() {
                                      mGroupHelper.onNotificationRemoved(n);
                                  }
                              });
                          }
                          // ATTENTION: in a future release we will bail out here
                          // so that we do not play sounds, show lights, etc. for invalid
                          // notifications
                          Slog.e(TAG, "WARNING: In a future release this will crash the app: "
                                  + n.getPackageName());
                      }
  
                      if (mShortcutHelper != null) {
                          mShortcutHelper.maybeListenForShortcutChangesForBubbles(r,
                                  false /* isRemoved */,
                                  mHandler);
                      }
  
                      maybeRecordInterruptionLocked(r);
                      maybeRegisterMessageSent(r);
  
                      // Log event to statsd
                      mNotificationRecordLogger.maybeLogNotificationPosted(r, old, position,
                              buzzBeepBlinkLoggingCode, getGroupInstanceId(n.getGroupKey()));
                  } finally {
                      int N = mEnqueuedNotifications.size();
                      for (int i = 0; i < N; i++) {
                          final NotificationRecord enqueued = mEnqueuedNotifications.get(i);
                          if (Objects.equals(key, enqueued.getKey())) {
                              mEnqueuedNotifications.remove(i);
                              break;
                          }
                      }
                  }
              }
          }
      }
通过buzzBeepBlinkLocked(NotificationRecord record) 管理通知声音 等功能
        int buzzBeepBlinkLocked(NotificationRecord record) {
          if (mIsAutomotive && !mNotificationEffectsEnabledForAutomotive) {
              return 0;
          }
          boolean buzz = false;
          boolean beep = false;
          boolean blink = false;
  
          final Notification notification = record.getSbn().getNotification();
          final String key = record.getKey();
  
          // Should this notification make noise, vibe, or use the LED?
          final boolean aboveThreshold =
                  mIsAutomotive
                          ? record.getImportance() > NotificationManager.IMPORTANCE_DEFAULT
                          : record.getImportance() >= NotificationManager.IMPORTANCE_DEFAULT;
          // Remember if this notification already owns the notification channels.
          boolean wasBeep = key != null && key.equals(mSoundNotificationKey);
          boolean wasBuzz = key != null && key.equals(mVibrateNotificationKey);
          // These are set inside the conditional if the notification is allowed to make noise.
          boolean hasValidVibrate = false;
          boolean hasValidSound = false;
          boolean sentAccessibilityEvent = false;
  
          // If the notification will appear in the status bar, it should send an accessibility event
          final boolean suppressedByDnd = record.isIntercepted()
                  && (record.getSuppressedVisualEffects() & SUPPRESSED_EFFECT_STATUS_BAR) != 0;
          if (!record.isUpdate
                  && record.getImportance() > IMPORTANCE_MIN
                  && !suppressedByDnd) {
              sendAccessibilityEvent(notification, record.getSbn().getPackageName());
              sentAccessibilityEvent = true;
          }
  
          if (aboveThreshold && isNotificationForCurrentUser(record)) {
              if (mSystemReady && mAudioManager != null) {
                  Uri soundUri = record.getSound();
                  hasValidSound = soundUri != null && !Uri.EMPTY.equals(soundUri);
                  long[] vibration = record.getVibration();
                  // Demote sound to vibration if vibration missing & phone in vibration mode.
                  if (vibration == null
                          && hasValidSound
                          && (mAudioManager.getRingerModeInternal()
                          == AudioManager.RINGER_MODE_VIBRATE)
                          && mAudioManager.getStreamVolume(
                          AudioAttributes.toLegacyStreamType(record.getAudioAttributes())) == 0) {
                      vibration = mFallbackVibrationPattern;
                  }
                  hasValidVibrate = vibration != null;
                  boolean hasAudibleAlert = hasValidSound || hasValidVibrate;
                  if (hasAudibleAlert && !shouldMuteNotificationLocked(record)) {
                      if (!sentAccessibilityEvent) {
                          sendAccessibilityEvent(notification, record.getSbn().getPackageName());
                          sentAccessibilityEvent = true;
                      }
                      if (DBG) Slog.v(TAG, "Interrupting!");
                      if (hasValidSound) {//是否由通知声音 如果由通知声音就来播放通知声音
                          if (isInCall()) {
                              playInCallNotification();
                              beep = true;
                          } else {
                              beep = playSound(record, soundUri);
                          }
                          if(beep) {
                              mSoundNotificationKey = key;
                          }
                      }
  
                      final boolean ringerModeSilent =
                              mAudioManager.getRingerModeInternal()
                                      == AudioManager.RINGER_MODE_SILENT;
                      if (!isInCall() && hasValidVibrate && !ringerModeSilent) {
                          buzz = playVibration(record, vibration, hasValidSound);
                          if(buzz) {
                              mVibrateNotificationKey = key;
                          }
                      }
                  } else if ((record.getFlags() & Notification.FLAG_INSISTENT) != 0) {
                      hasValidSound = false;
                  }
              }
          }
          // If a notification is updated to remove the actively playing sound or vibrate,
          // cancel that feedback now
          if (wasBeep && !hasValidSound) {
              clearSoundLocked();
          }
          if (wasBuzz && !hasValidVibrate) {
              clearVibrateLocked();
          }
  
          // light
          // release the light
          boolean wasShowLights = mLights.remove(key);
          if (canShowLightsLocked(record, aboveThreshold)) {
              mLights.add(key);
              updateLightsLocked();
              if (mUseAttentionLight && mAttentionLight != null) {
                  mAttentionLight.pulse();
              }
              blink = true;
          } else if (wasShowLights) {
              updateLightsLocked();
          }
          final int buzzBeepBlink = (buzz ? 1 : 0) | (beep ? 2 : 0) | (blink ? 4 : 0);
          if (buzzBeepBlink > 0) {
              // Ignore summary updates because we don't display most of the information.
              if (record.getSbn().isGroup() && record.getSbn().getNotification().isGroupSummary()) {
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is not interruptive: summary");
                  }
              } else if (record.canBubble()) {
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is not interruptive: bubble");
                  }
              } else {
                  record.setInterruptive(true);
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is interruptive: alerted");
                  }
              }
              MetricsLogger.action(record.getLogMaker()
                      .setCategory(MetricsEvent.NOTIFICATION_ALERT)
                      .setType(MetricsEvent.TYPE_OPEN)
                      .setSubtype(buzzBeepBlink));
              EventLogTags.writeNotificationAlert(key, buzz ? 1 : 0, beep ? 1 : 0, blink ? 1 : 0);
          }
          record.setAudiblyAlerted(buzz || beep);
          return buzzBeepBlink;
      }
 
所以可以修改为：
        int buzzBeepBlinkLocked(NotificationRecord record) {
          if (mIsAutomotive && !mNotificationEffectsEnabledForAutomotive) {
              return 0;
          }
          boolean buzz = false;
          boolean beep = false;
          boolean blink = false;
  
          final Notification notification = record.getSbn().getNotification();
          final String key = record.getKey();
  
          // Should this notification make noise, vibe, or use the LED?
          final boolean aboveThreshold =
                  mIsAutomotive
                          ? record.getImportance() > NotificationManager.IMPORTANCE_DEFAULT
                          : record.getImportance() >= NotificationManager.IMPORTANCE_DEFAULT;
          // Remember if this notification already owns the notification channels.
          boolean wasBeep = key != null && key.equals(mSoundNotificationKey);
          boolean wasBuzz = key != null && key.equals(mVibrateNotificationKey);
          // These are set inside the conditional if the notification is allowed to make noise.
          boolean hasValidVibrate = false;
          boolean hasValidSound = false;
          boolean sentAccessibilityEvent = false;
  
          // If the notification will appear in the status bar, it should send an accessibility event
          final boolean suppressedByDnd = record.isIntercepted()
                  && (record.getSuppressedVisualEffects() & SUPPRESSED_EFFECT_STATUS_BAR) != 0;
          if (!record.isUpdate
                  && record.getImportance() > IMPORTANCE_MIN
                  && !suppressedByDnd) {
              sendAccessibilityEvent(notification, record.getSbn().getPackageName());
              sentAccessibilityEvent = true;
          }
  
          if (aboveThreshold && isNotificationForCurrentUser(record)) {
              if (mSystemReady && mAudioManager != null) {
                  Uri soundUri = record.getSound();
                  hasValidSound = soundUri != null && !Uri.EMPTY.equals(soundUri);
                  long[] vibration = record.getVibration();
                  // Demote sound to vibration if vibration missing & phone in vibration mode.
                  if (vibration == null
                          && hasValidSound
                          && (mAudioManager.getRingerModeInternal()
                          == AudioManager.RINGER_MODE_VIBRATE)
                          && mAudioManager.getStreamVolume(
                          AudioAttributes.toLegacyStreamType(record.getAudioAttributes())) == 0) {
                      vibration = mFallbackVibrationPattern;
                  }
                  hasValidVibrate = vibration != null;
                  boolean hasAudibleAlert = hasValidSound || hasValidVibrate;
                  if (hasAudibleAlert && !shouldMuteNotificationLocked(record)) {
                      if (!sentAccessibilityEvent) {
                          sendAccessibilityEvent(notification, record.getSbn().getPackageName());
                          sentAccessibilityEvent = true;
                      }
                      if (DBG) Slog.v(TAG, "Interrupting!");
                      if (hasValidSound) {
                          if (isInCall()) {
                              playInCallNotification();
                              beep = true;
                          } else {
   
                        //重点修改部分
                       - beep = playSound(record, soundUri);
                       // 根据soundUri 的路径来播放通知声音
+                       if(soundUri!=null && !soundUri.getPath().equal("通知声音对应路径")) {                    +{
+                          beep = playSound(record, soundUri);
+                       }
 
                          }
                          if(beep) {
                              mSoundNotificationKey = key;
                          }
                      }
  
                      final boolean ringerModeSilent =
                              mAudioManager.getRingerModeInternal()
                                      == AudioManager.RINGER_MODE_SILENT;
                      if (!isInCall() && hasValidVibrate && !ringerModeSilent) {
                          buzz = playVibration(record, vibration, hasValidSound);
                          if(buzz) {
                              mVibrateNotificationKey = key;
                          }
                      }
                  } else if ((record.getFlags() & Notification.FLAG_INSISTENT) != 0) {
                      hasValidSound = false;
                  }
              }
          }
          // If a notification is updated to remove the actively playing sound or vibrate,
          // cancel that feedback now
          if (wasBeep && !hasValidSound) {
              clearSoundLocked();
          }
          if (wasBuzz && !hasValidVibrate) {
              clearVibrateLocked();
          }
  
          // light
          // release the light
          boolean wasShowLights = mLights.remove(key);
          if (canShowLightsLocked(record, aboveThreshold)) {
              mLights.add(key);
              updateLightsLocked();
              if (mUseAttentionLight && mAttentionLight != null) {
                  mAttentionLight.pulse();
              }
              blink = true;
          } else if (wasShowLights) {
              updateLightsLocked();
          }
          final int buzzBeepBlink = (buzz ? 1 : 0) | (beep ? 2 : 0) | (blink ? 4 : 0);
          if (buzzBeepBlink > 0) {
              // Ignore summary updates because we don't display most of the information.
              if (record.getSbn().isGroup() && record.getSbn().getNotification().isGroupSummary()) {
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is not interruptive: summary");
                  }
              } else if (record.canBubble()) {
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is not interruptive: bubble");
                  }
              } else {
                  record.setInterruptive(true);
                  if (DEBUG_INTERRUPTIVENESS) {
                      Slog.v(TAG, "INTERRUPTIVENESS: "
                              + record.getKey() + " is interruptive: alerted");
                  }
              }
              MetricsLogger.action(record.getLogMaker()
                      .setCategory(MetricsEvent.NOTIFICATION_ALERT)
                      .setType(MetricsEvent.TYPE_OPEN)
                      .setSubtype(buzzBeepBlink));
              EventLogTags.writeNotificationAlert(key, buzz ? 1 : 0, beep ? 1 : 0, blink ? 1 : 0);
          }
          record.setAudiblyAlerted(buzz || beep);
          return buzzBeepBlink;
      }
```

在 enqueueNotificationWithTag(）中调用 enqueueNotificationInternal([pkg](https://so.csdn.net/so/search?q=pkg&spm=1001.2101.3001.7020), opPkg, Binder.getCallingUid(),Binder.getCallingPid(), tag, id, notification, userId); 具体负责处理通知消息

// 由 EnqueueNotificationRunnable 子线程处理通知 具体看 EnqueueNotificationRunnable 这个方法  
         mHandler.post(new EnqueueNotificationRunnable(userId, r, isAppForeground));

在 EnqueueNotificationRunnable 中最后由 PostNotificationRunnable 来处理每条[消息队列](https://so.csdn.net/so/search?q=%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97&spm=1001.2101.3001.7020)的通知消息 在 buzzBeepBlinkLocked(NotificationRecord record) 处理通知声音等功能