> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125860925)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 屏蔽某些通知的提示音的核心代码](#t1)

[3. 屏蔽某些通知的提示音的核心功能代码分析](#t2)

[3.1NotificationManager.java 关于通知流程分析](#t3)

[3.2 NotificationManagerService.java 中关于通知的相关方法](#t4)

[4. 屏蔽某些通知的提示音的核心功能实现](#t5)

1. 概述
-----

在系统 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 中会发一些通知的声音，但是同时也会在开机的时候，会有一些通知的声音，特别是不想要的一些通知的声音，  
这些对于产品还是有一些影响的，所以为了产品体验，就需要屏蔽掉一些开机的通知的声音

2. 屏蔽某些通知的提示音的核心代码
------------------

```
核心代码：
frameworks\base\services\core\java\com\android\server\notification\NotificationManagerService.java
  frameworks/base/core/java/android/app/NotificationManager.java
```

3. 屏蔽某些通知的提示音的核心功能代码分析
----------------------

#### 3.1NotificationManager.java 关于通知流程分析

```
public class NotificationManager {
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
    public void notify(String tag, int id, Notification notification)
    {
		int expand_panel = Settings.Global.getInt(mContext.getContentResolver(),"expand_panel",1);
		if(expand_panel==0)return;
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
     */
    public void notifyAsPackage(@NonNull String targetPackage, @NonNull String tag, int id,
            @NonNull Notification notification) {
        INotificationManager service = getService();
        String sender = mContext.getPackageName();
 
        try {
            if (localLOGV) Log.v(TAG, sender + ": notify(" + id + ", " + notification + ")");
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
    {
        INotificationManager service = getService();
        String pkg = mContext.getPackageName();
 
        try {
            if (localLOGV) Log.v(TAG, pkg + ": notify(" + id + ", " + notification + ")");
            service.enqueueNotificationWithTag(pkg, mContext.getOpPackageName(), tag, id,
                    fixNotification(notification), user.getIdentifier());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
 
    private Notification fixNotification(Notification notification) {
        String pkg = mContext.getPackageName();
        // Fix the notification as best we can.
        Notification.addFieldsFromContext(mContext, notification);
 
        if (notification.sound != null) {
            notification.sound = notification.sound.getCanonicalUri();
            if (StrictMode.vmFileUriExposureEnabled()) {
                notification.sound.checkFileUriExposed("Notification.sound");
            }
 
        }
        fixLegacySmallIcon(notification, pkg);
        if (mContext.getApplicationInfo().targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
            if (notification.getSmallIcon() == null) {
                throw new IllegalArgumentException("Invalid notification (no valid small icon): "
                        + notification);
            }
        }
 
        notification.reduceImageSizes(mContext);
 
        ActivityManager am = (ActivityManager) mContext.getSystemService(Context.ACTIVITY_SERVICE);
        boolean isLowRam = am.isLowRamDevice();
        return Builder.maybeCloneStrippedForDelivery(notification, isLowRam, mContext);
    }
....
}
 
在notify()中通过监听到发出通知后调用NotificationManagerService.java的enqueueNotificationWithTag(）来发送通知
 
接下来看下NotificationManagerService.java中关于通知的相关方法
```

#### 3.2 NotificationManagerService.java 中关于通知的相关方法

```
public class NotificationManagerService extends SystemService {
        @Override
        public void enqueueNotificationWithTag(String pkg, String opPkg, String tag, int id,
                Notification notification, int userId) throws RemoteException {
            enqueueNotificationInternal(pkg, opPkg, Binder.getCallingUid(),
                    Binder.getCallingPid(), tag, id, notification, userId);
        }
void enqueueNotificationInternal(final String pkg, final String opPkg, final int callingUid,
            final int callingPid, final String tag, final int id, final Notification notification,
            int incomingUserId) {
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
 
        checkRestrictedCategories(notification);
 
        // Fix the notification as best we can.
        try {
            fixNotification(notification, pkg, userId);
 
        } catch (NameNotFoundException e) {
            Slog.e(TAG, "Cannot create a context for sending app", e);
            return;
        }
 
        mUsageStats.registerEnqueuedByApp(pkg);
 
        // setup local book-keeping
        String channelId = notification.getChannelId();
        if (mIsTelevision && (new Notification.TvExtender(notification)).getChannelId() != null) {
            channelId = (new Notification.TvExtender(notification)).getChannelId();
        }
        final NotificationChannel channel = mPreferencesHelper.getNotificationChannel(pkg,
                notificationUid, channelId, false /* includeDeleted */);
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
 
        final StatusBarNotification n = new StatusBarNotification(
                pkg, opPkg, id, tag, notificationUid, callingPid, notification,
                user, null, System.currentTimeMillis());
        final NotificationRecord r = new NotificationRecord(getContext(), n, channel);
        r.setIsAppImportanceLocked(mPreferencesHelper.getIsAppImportanceLocked(pkg, callingUid));
 
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
 
        if (!checkDisqualifyingFeatures(userId, notificationUid, id, tag, r,
                r.sbn.getOverrideGroupKey() != null)) {
            return;
        }
 
        // Whitelist pending intents.
        if (notification.allPendingIntents != null) {
            final int intentCount = notification.allPendingIntents.size();
            if (intentCount > 0) {
                final ActivityManagerInternal am = LocalServices
                        .getService(ActivityManagerInternal.class);
                final long duration = LocalServices.getService(
                        DeviceIdleController.LocalService.class).getNotificationWhitelistDuration();
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
       mHandler.post(new EnqueueNotificationRunnable(userId, r));
    }
 
protected class PostNotificationRunnable implements Runnable {
        private final String key;
 
        PostNotificationRunnable(String key) {
            this.key = key;
        }
 
        @Override
        public void run() {
            synchronized (mNotificationLock) {
                try {
                    NotificationRecord r = null;
                    int N = mEnqueuedNotifications.size();
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
 
                    final boolean isPackageSuspended = isPackageSuspendedLocked(r);
                    r.setHidden(isPackageSuspended);
                    if (isPackageSuspended) {
                        mUsageStats.registerSuspendedByAdmin(r);
                    }
                    NotificationRecord old = mNotificationsByKey.get(key);
                    final StatusBarNotification n = r.sbn;
                    final Notification notification = n.getNotification();
                    int index = indexOfNotificationLocked(n.getKey());
                    if (index < 0) {
                        mNotificationList.add(r);
                        mUsageStats.registerPostedByApp(r);
                        r.setInterruptive(isVisuallyInterruptive(null, r));
                    } else {
                        old = mNotificationList.get(index);
                        mNotificationList.set(index, r);
                        mUsageStats.registerUpdatedByApp(r, old);
                        // Make sure we don't lose the foreground service state.
                        notification.flags |=
                                old.getNotification().flags & FLAG_FOREGROUND_SERVICE;
                        r.isUpdate = true;
                        r.setTextChanged(isVisuallyInterruptive(old, r));
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
 
                    if (!r.isHidden()) {
                        buzzBeepBlinkLocked(r);
                    }
 
                    if (notification.getSmallIcon() != null) {
                        StatusBarNotification oldSbn = (old != null) ? old.sbn : null;
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
 
                    maybeRecordInterruptionLocked(r);
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
 
PostNotificationRunnable中的buzzBeepBlinkLocked()负责震动、响铃或闪光灯的相关功能
 
void buzzBeepBlinkLocked(NotificationRecord record) {
        if (mIsAutomotive && !mNotificationEffectsEnabledForAutomotive) {
            return;
        }
        boolean buzz = false;
        boolean beep = false;
        boolean blink = false;
 
        final Notification notification = record.sbn.getNotification();
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
        // If the notification will appear in the status bar, it should send an accessibility
        // event
        if (!record.isUpdate && record.getImportance() > IMPORTANCE_MIN) {
            sendAccessibilityEvent(notification, record.sbn.getPackageName());
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
                        sendAccessibilityEvent(notification, record.sbn.getPackageName());
                        sentAccessibilityEvent = true;
                    }
                    if (DBG) Slog.v(TAG, "Interrupting!");
                    if (hasValidSound) {
                        if (mInCall) {
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
                    if (!mInCall && hasValidVibrate && !ringerModeSilent) {
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
            if (mUseAttentionLight) {
                mAttentionLight.pulse();
            }
            blink = true;
        } else if (wasShowLights) {
            updateLightsLocked();
        }
        if (buzz || beep || blink) {
            // Ignore summary updates because we don't display most of the information.
            if (record.sbn.isGroup() && record.sbn.getNotification().isGroupSummary()) {
                if (DEBUG_INTERRUPTIVENESS) {
                    Slog.v(TAG, "INTERRUPTIVENESS: "
                            + record.getKey() + " is not interruptive: summary");
                }
            } else {
                if (DEBUG_INTERRUPTIVENESS) {
                    Slog.v(TAG, "INTERRUPTIVENESS: "
                            + record.getKey() + " is interruptive: alerted");
                }
                record.setInterruptive(true);
            }
            MetricsLogger.action(record.getLogMaker()
                    .setCategory(MetricsEvent.NOTIFICATION_ALERT)
                    .setType(MetricsEvent.TYPE_OPEN)
                    .setSubtype((buzz ? 1 : 0) | (beep ? 2 : 0) | (blink ? 4 : 0)));
            EventLogTags.writeNotificationAlert(key, buzz ? 1 : 0, beep ? 1 : 0, blink ? 1 : 0);
        }
        record.setAudiblyAlerted(buzz || beep);
    }
 
而 if (hasValidSound) {
                        if (mInCall) {
                            playInCallNotification();
                            beep = true;
                        } else {
                            beep = playSound(record, soundUri);
                        }
                        if(beep) {
                            mSoundNotificationKey = key;
                        }
                    }
就是负责播放通知声音的
```

4. 屏蔽某些通知的提示音的核心功能实现
--------------------

```
if (hasValidSound) {
                        if (mInCall) {
                            playInCallNotification();
                            beep = true;
                        } else {
                            // 修改部分开始
                            String path = soundUri.getPath();
                            if(path!=null&&!path.equals("声音路径")){
                                beep = playSound(record, soundUri);
                            }
                            // 修改部分结束
                        }
                        if(beep) {
                            mSoundNotificationKey = key;
                        }
                    }
```