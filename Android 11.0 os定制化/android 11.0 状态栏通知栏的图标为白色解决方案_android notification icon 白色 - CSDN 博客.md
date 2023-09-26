> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828250)

### 1. 概述

在 11.0 进行定制化开发中，对 SystemUI 的相关问题解决也是相当多的 ，最近在测试中发现 app 弹出的通知  
在状态栏显示为白色，从而看不清通知的原来图标，而显示不了正常的背景色，这就跟状态栏通知的背景颜色有关，看下哪里修改了通知背景颜色修改过来就可以了

### 2. 状态栏通知栏的图标为白色解决方案的核心类

```
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarNotificationPresenter.java
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelView.java
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationIconAreaController.java
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/stack/NotificationStackScrollLayout.java

```

### 3. 状态栏通知栏的图标为白色解决方案的核心功能分析和实现

### 3.1StatusBarNotificationPresenter.java 的关于状态栏通知的显示

接下来首选看下通知显示的流程

```
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarNotificationPresenter.java
private final NotificationPanelView mNotificationPanel;

     @Override
      public void onUserSwitched(int newUserId) {
          // Begin old BaseStatusBar.userSwitched
          mHeadsUpManager.setUser(newUserId);
          // End old BaseStatusBar.userSwitched
          if (MULTIUSER_DEBUG) mNotificationPanelDebugText.setText("USER " + newUserId);
          mCommandQueue.animateCollapsePanels();
          if (mReinflateNotificationsOnUserSwitched) {
              updateNotificationsOnDensityOrFontScaleChanged();
              mReinflateNotificationsOnUserSwitched = false;
          }
          if (mDispatchUiModeChangeOnUserSwitched) {
              updateNotificationOnUiModeChanged();
              mDispatchUiModeChangeOnUserSwitched = false;
          }
          updateNotificationViews("user switched");
          mMediaManager.clearCurrentMediaNotification();
          mStatusBar.setLockscreenUser(newUserId);
          updateMediaMetaData(true, false);
      }
  
      @Override
      public void onBindRow(ExpandableNotificationRow row) {
          row.setAboveShelfChangedListener(mAboveShelfObserver);
          row.setSecureStateProvider(mKeyguardStateController::canDismissLockScreen);
      }

```

在收到通知后 StatusBarNotificationPresenter  
调用 NotificationPanel.updateNotificationViews() 来处理通知事件，所以具体通知在 NotificationPanelView.java 中进行处理  
接下来看 NotificationPanelView.java 关于通知的处理

### 3.2NotificationPanelView.java 关于通知的处理流程分析

路径：  
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelView.java  
而在 NotificationPanelView.java 中正在通知的布局是在 mNotificationStackScroller 中

```
protected NotificationStackScrollLayout mNotificationStackScroller;
public void updateNotificationViews() {
mNotificationStackScroller.updateSectionBoundaries();
mNotificationStackScroller.updateSpeedBumpIndex();
mNotificationStackScroller.updateFooter();
updateShowEmptyShadeView();
mNotificationStackScroller.updateIconAreaViews();
}

framework/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/stack/NotificationStackScrollLayout.java
private NotificationIconAreaController mIconAreaController;
public void updateIconAreaViews() {
mIconAreaController.updateNotificationIcons();
}

```

在调用 updateNotificationViews() 中调用 mNotificationStackScroller.updateIconAreaViews() 的方法处理通知事件  
最终是通过 mIconAreaController.updateNotificationIcons(); 来处理通知  
继续往下走：  
在 updateNotificationIcons() 中发现主要的更新通知颜色是在 applyNotificationIconsTint() 接下来看下  
applyNotificationIconsTint() 的相关核心代码对通知的背景色进行着色的处理工作

```
framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationIconAreaController.java
/**
Updates the notifications with the given list of notifications to display.
*/
private static final boolean CHECK_SDK_VERSION = false;
public void updateNotificationIcons() {
updateStatusBarIcons();
updateShelfIcons();
updateCenterIcon();
applyNotificationIconsTint();
}
/**
Applies {@link #mIconTint} to the notification icons.
Applies {@link #mCenteredIconTint} to the center notification icon.
*/
private void applyNotificationIconsTint() {
for (int i = 0; i < mNotificationIcons.getChildCount(); i++) {
final StatusBarIconView iv = (StatusBarIconView) mNotificationIcons.getChildAt(i);
if (iv.getWidth() != 0) {
updateTintForIcon(iv, mIconTint);
} else {
iv.executeOnLayout(() -> updateTintForIcon(iv, mIconTint));
}
}for (int i = 0; i < mCenteredIcon.getChildCount(); i++) {
final StatusBarIconView iv = (StatusBarIconView) mCenteredIcon.getChildAt(i);
if (iv.getWidth() != 0) {
updateTintForIcon(iv, mCenteredIconTint);
} else {
iv.executeOnLayout(() -> updateTintForIcon(iv, mCenteredIconTint));
}
}
}
private void updateTintForIcon(StatusBarIconView v, int tint) {
boolean isPreL = Boolean.TRUE.equals(v.getTag(R.id.icon_is_pre_L));
int color = StatusBarIconView.NO_COLOR;
/Unisoc bug 906697:Notification icon display as a white icon{@/
//boolean colorize = !isPreL || NotificationUtils.isGrayscale(v, mContrastColorUtil);
boolean colorize = CHECK_SDK_VERSION ? !isPreL || NotificationUtils.isGrayscale(v, mContrastColorUtil)
: NotificationUtils.isGrayscale(v, mContrastColorUtil);
/@}/
if (colorize) {
    color = DarkIconDispatcher.getTint(mTintArea, v, tint);
}
v.setStaticDrawableColor(color);
v.setDecorColor(tint);

}

```

[private](https://so.csdn.net/so/search?q=private&spm=1001.2101.3001.7020) int mIconTint = Color.WHITE;  
private int mCenteredIconTint = Color.WHITE;  
而 mIconTint 和 mCenteredIconTint 都代表纯白色的背景  
这就是 app 发送通知的 Icon 会进行判断然后进行着色  
所以修改方案 就是不让着色：

```
private void updateTintForIcon(StatusBarIconView v, int tint) {
boolean isPreL = Boolean.TRUE.equals(v.getTag(R.id.icon_is_pre_L));
int color = StatusBarIconView.NO_COLOR;
/Unisoc bug 906697:Notification icon display as a white icon{@/
//boolean colorize = !isPreL || NotificationUtils.isGrayscale(v, mContrastColorUtil);
boolean colorize = CHECK_SDK_VERSION ? !isPreL || NotificationUtils.isGrayscale(v, mContrastColorUtil)
: NotificationUtils.isGrayscale(v, mContrastColorUtil);
/@}/
    if (colorize) {
        color = DarkIconDispatcher.getTint(mTintArea, v, tint);
    }
 -   v.setStaticDrawableColor(color);
 -   v.setDecorColor(tint);
}

```

去掉着色的 setStaticDrawableColor 和 setDecorColor 就能实现功能需求