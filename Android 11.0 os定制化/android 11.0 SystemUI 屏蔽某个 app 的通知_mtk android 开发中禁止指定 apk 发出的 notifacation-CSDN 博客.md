> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124890521)

### 1. 概述

在 11.0 的产品开发中，对于系统的通知部分，要求根据 app 包名来过滤掉一部分通知，就是在接收到系统  
通知时，根据包名判断是否需要接收通知的功能，首选要分析通知流程，然后实现功能

### 2.[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 屏蔽某个 app 的通知相关代码

```
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\notification\NoticationFilter.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/collection/NotificationRankingManager.kt

```

### 3.SystemUI 屏蔽某个 app 的通知核心功能分析

### 3.1 NotificationRankingManager 关于过滤通知的相关方法

```
open class NotificationRankingManager @Inject constructor(
      private val mediaManagerLazy: Lazy<NotificationMediaManager>,
      private val groupManager: NotificationGroupManager,
      private val headsUpManager: HeadsUpManager,
      private val notifFilter: NotificationFilter,
      private val logger: NotificationEntryManagerLogger,
      private val sectionsFeatureManager: NotificationSectionsFeatureManager,
      private val peopleNotificationIdentifier: PeopleNotificationIdentifier,
      private val highPriorityProvider: HighPriorityProvider
    /** Uses the [rankingComparator] to sort notifications which aren't filtered */
      private fun filterAndSortLocked(
          entries: Collection<NotificationEntry>,
          reason: String
      ): List<NotificationEntry> {
          logger.logFilterAndSort(reason)
          val filtered = entries.asSequence()
                  .filterNot(this::filter)
                  .sortedWith(rankingComparator)
                  .toList()
          entries.forEach { it.bucket = getBucketForEntry(it) }
          return filtered
      }
  
      private fun filter(entry: NotificationEntry): Boolean {
          val filtered = notifFilter.shouldFilterOut(entry)
          if (filtered) {
              // notification is removed from the list, so we reset its initialization time
              entry.resetInitializationTime()
          }
          return filtered
      }

```

在 filter(entry: NotificationEntry) 调用 notifFilter.shouldFilterOut(entry) 判断是否过滤接收到的通知  
下面看 NotificationFilter 的 shouldFilterOut(entry) 方法

### 3.2NoticationFilter.java 关于过滤通知的相关代码

根据 SystemUI 源码通知显示流程可以得知 NoticationFilter.java 中可以处理过滤通知  
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\[notification](https://so.csdn.net/so/search?q=notification&spm=1001.2101.3001.7020)\NoticationFilter.java

```
public class NotificationFilter {
  
      private final NotificationGroupManager mGroupManager = Dependency.get(
              NotificationGroupManager.class);
      private final StatusBarStateController mStatusBarStateController;
      private final Boolean mIsMediaFlagEnabled;
  
      private NotificationEntryManager.KeyguardEnvironment mEnvironment;
      private ShadeController mShadeController;
      private ForegroundServiceController mFsc;
      private NotificationLockscreenUserManager mUserManager;
  
      @Inject
      public NotificationFilter(
              StatusBarStateController statusBarStateController,
              MediaFeatureFlag mediaFeatureFlag) {
          mStatusBarStateController = statusBarStateController;
          mIsMediaFlagEnabled = mediaFeatureFlag.getEnabled();
      }
       /**
      * @return true if the provided notification should NOT be shown right now.
      */
     public boolean shouldFilterOut(NotificationEntry entry) {
          final StatusBarNotification sbn = entry.getSbn();
          if (!(getEnvironment().isDeviceProvisioned()
                  || showNotificationEvenIfUnprovisioned(sbn))) {
              return true;
          }
  
          if (!getEnvironment().isNotificationForCurrentProfiles(sbn)) {
              return true;
          }
  
          if (getUserManager().isLockscreenPublicMode(sbn.getUserId())
                  && (sbn.getNotification().visibility == Notification.VISIBILITY_SECRET
                          || getUserManager().shouldHideNotifications(sbn.getUserId())
                          || getUserManager().shouldHideNotifications(sbn.getKey()))) {
              return true;
          }
  
          if (mStatusBarStateController.isDozing() && entry.shouldSuppressAmbient()) {
              return true;
          }
  
          if (!mStatusBarStateController.isDozing() && entry.shouldSuppressNotificationList()) {
              return true;
          }
  
          if (entry.getRanking().isSuspended()) {
              return true;
          }
  
          if (getFsc().isDisclosureNotification(sbn)
                  && !getFsc().isDisclosureNeededForUser(sbn.getUserId())) {
              // this is a foreground-service disclosure for a user that does not need to show one
              return true;
          }
          if (getFsc().isSystemAlertNotification(sbn)) {
              final String[] apps = sbn.getNotification().extras.getStringArray(
                      Notification.EXTRA_FOREGROUND_APPS);
              if (apps != null && apps.length >= 1) {
                  if (!getFsc().isSystemAlertWarningNeeded(sbn.getUserId(), apps[0])) {
                      return true;
                  }
              }
          }
  
          if (mIsMediaFlagEnabled && isMediaNotification(sbn)) {
              return true;
          }
          return false;
      }

```

根据 shouldFilterOut 返回值来实现是否过滤  
所以就需要在此处添加包名返回 true 就可以了，在通知到来时就不会显示通知了

```
    public boolean shouldFilterOut(NotificationEntry entry) {
        final StatusBarNotification sbn = entry.notification;
		// 添加需要屏蔽的app的包名 返回true就可以了
		try {
			if(sbn.getPackageName().equals("com.android.systemdemo")){
				Log.d(TAG,"block notification channel-USB  of package-android.");
				return true;
			}
		}catch (Exception e){
			e.printStackTrace();
			//do nothing.
		}
        if (!(getEnvironment().isDeviceProvisioned()
                || showNotificationEvenIfUnprovisioned(sbn))) {
            return true;
        }

        if (!getEnvironment().isNotificationForCurrentProfiles(sbn)) {
            return true;
        }

        if (getUserManager().isLockscreenPublicMode(sbn.getUserId())
                && (sbn.getNotification().visibility == Notification.VISIBILITY_SECRET
                        || getUserManager().shouldHideNotifications(sbn.getUserId())
                        || getUserManager().shouldHideNotifications(sbn.getKey()))) {
            return true;
        }

        if (getShadeController().isDozing() && entry.shouldSuppressAmbient()) {
            return true;
        }

        if (!getShadeController().isDozing() && entry.shouldSuppressNotificationList()) {
            return true;
        }

        if (entry.suspended) {
            return true;
        }

        if (!StatusBar.ENABLE_CHILD_NOTIFICATIONS
                && mGroupManager.isChildInGroupWithSummary(sbn)) {
            return true;
        }

        if (getFsc().isDisclosureNotification(sbn)
                && !getFsc().isDisclosureNeededForUser(sbn.getUserId())) {
            // this is a foreground-service disclosure for a user that does not need to show one
            return true;
        }
        if (getFsc().isSystemAlertNotification(sbn)) {
            final String[] apps = sbn.getNotification().extras.getStringArray(
                    Notification.EXTRA_FOREGROUND_APPS);
            if (apps != null && apps.length >= 1) {
                if (!getFsc().isSystemAlertWarningNeeded(sbn.getUserId(), apps[0])) {
                    return true;
                }
            }
        }
        return false;
    }

```