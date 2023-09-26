> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124596575)

### 1. 概述

在 android 11.0 原生系统中，调用系统截图功能以后，截图会发送一个通知，在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的状态栏显示通知，但是通知里面有三个选项分享 编辑 删除等，由于产品要求去掉分享功能, 所以要去掉分享按钮来满足要求

### 2.SystemUI 截图功能去掉分享按钮 (屏蔽分享功能) 的核心类

```
frameworks\base\packages\SystemUI\src\com\android\systemui\screenshot\GlobalScreenshot.java

```

### 3.SystemUI 截图功能去掉分享按钮 (屏蔽分享功能) 核心功能分析和实现

在系统截屏中对于调用系统截屏 api 截屏功能，然后显示弹窗通知的时候，可以借助 Android studio 开发工具  
来查找相关布局，通过查关键词来实现对相关源码的查找  
通过分析工具发现 GlobalScreenshot.java  
中负责处理的截图的相关功能  
路径为：frameworks\base\packages\SystemUI\src\com\android\systemui\screenshot\GlobalScreenshot.java

```
private void showUiOnActionsReady(SavedImageData imageData) {
          logSuccessOnActionsReady(imageData);
  
          AccessibilityManager accessibilityManager = (AccessibilityManager)
                  mContext.getSystemService(Context.ACCESSIBILITY_SERVICE);
          long timeoutMs = accessibilityManager.getRecommendedTimeoutMillis(
                  SCREENSHOT_CORNER_DEFAULT_TIMEOUT_MILLIS,
                  AccessibilityManager.FLAG_CONTENT_CONTROLS);
  
          mScreenshotHandler.removeMessages(MESSAGE_CORNER_TIMEOUT);
          mScreenshotHandler.sendMessageDelayed(
                  mScreenshotHandler.obtainMessage(MESSAGE_CORNER_TIMEOUT),
                  timeoutMs);
  
          if (imageData.uri != null) {
              mScreenshotHandler.post(() -> {
                  if (mScreenshotAnimation != null && mScreenshotAnimation.isRunning()) {
                      mScreenshotAnimation.addListener(new AnimatorListenerAdapter() {
                          @Override
                          public void onAnimationEnd(Animator animation) {
                              super.onAnimationEnd(animation);
                              createScreenshotActionsShadeAnimation(imageData).start();
                          }
                      });
                  } else {
                      createScreenshotActionsShadeAnimation(imageData).start();
                  }
              });
          }
      }
      
private ValueAnimator createScreenshotActionsShadeAnimation(SavedImageData imageData) {
LayoutInflater inflater = LayoutInflater.from(mContext);
mActionsView.removeAllViews();
mScreenshotLayout.invalidate();
mScreenshotLayout.requestLayout();
mScreenshotLayout.getViewTreeObserver().dispatchOnGlobalLayout();
      // By default the activities won't be able to start immediately; override this to keep
     // the same behavior as if started from a notification
      try {
          ActivityManager.getService().resumeAppSwitches();
      } catch (RemoteException e) {
      }

      ArrayList<ScreenshotActionChip> chips = new ArrayList<>();

      for (Notification.Action smartAction : imageData.smartActions) {
         ScreenshotActionChip actionChip = (ScreenshotActionChip) inflater.inflate(
                  R.layout.global_screenshot_action_chip, mActionsView, false);
          actionChip.setText(smartAction.title);
          actionChip.setIcon(smartAction.getIcon(), false);
          actionChip.setPendingIntent(smartAction.actionIntent,
                  () -> {
                     mUiEventLogger.log(ScreenshotEvent.SCREENSHOT_SMART_ACTION_TAPPED);
                      dismissScreenshot("chip tapped", false);
                      mOnCompleteRunnable.run();
                  });
          mActionsView.addView(actionChip);
          chips.add(actionChip);
      }

      ScreenshotActionChip shareChip = (ScreenshotActionChip) inflater.inflate(
              R.layout.global_screenshot_action_chip, mActionsView, false);
      shareChip.setText(imageData.shareAction.title);
      shareChip.setIcon(imageData.shareAction.getIcon(), true);
      shareChip.setPendingIntent(imageData.shareAction.actionIntent, () -> {
          mUiEventLogger.log(ScreenshotEvent.SCREENSHOT_SHARE_TAPPED);
         dismissScreenshot("chip tapped", false);
          mOnCompleteRunnable.run();
     });
      mActionsView.addView(shareChip);
      chips.add(shareChip);

      ScreenshotActionChip editChip = (ScreenshotActionChip) inflater.inflate(
              R.layout.global_screenshot_action_chip, mActionsView, false);
      editChip.setText(imageData.editAction.title);
      editChip.setIcon(imageData.editAction.getIcon(), true);
      editChip.setPendingIntent(imageData.editAction.actionIntent, () -> {
          mUiEventLogger.log(ScreenshotEvent.SCREENSHOT_EDIT_TAPPED);
          dismissScreenshot("chip tapped", false);
          mOnCompleteRunnable.run();
      });
      mActionsView.addView(editChip);
      chips.add(editChip);

      mScreenshotPreview.setOnClickListener(v -> {
          try {
             imageData.editAction.actionIntent.send();
              mUiEventLogger.log(ScreenshotEvent.SCREENSHOT_PREVIEW_TAPPED);
             dismissScreenshot("screenshot preview tapped", false);
              mOnCompleteRunnable.run();
          } catch (PendingIntent.CanceledException e) {
              Log.e(TAG, "Intent cancelled", e);
          }
      });
      mScreenshotPreview.setContentDescription(imageData.editAction.title);

      // remove the margin from the last chip so that it's correctly aligned with the end
     LinearLayout.LayoutParams params = (LinearLayout.LayoutParams)
              mActionsView.getChildAt(mActionsView.getChildCount() - 1).getLayoutParams();
      params.setMarginEnd(0);

      ValueAnimator animator = ValueAnimator.ofFloat(0, 1);
      animator.setDuration(SCREENSHOT_ACTIONS_EXPANSION_DURATION_MS);
     float alphaFraction = (float) SCREENSHOT_ACTIONS_ALPHA_DURATION_MS
             / SCREENSHOT_ACTIONS_EXPANSION_DURATION_MS;
      mActionsContainer.setAlpha(0f);
     mActionsContainerBackground.setAlpha(0f);
     mActionsContainer.setVisibility(View.VISIBLE);
     mActionsContainerBackground.setVisibility(View.VISIBLE);

      animator.addUpdateListener(animation -> {
          float t = animation.getAnimatedFraction();
          mBackgroundProtection.setAlpha(t);
          float containerAlpha = t < alphaFraction ? t / alphaFraction : 1;
          mActionsContainer.setAlpha(containerAlpha);
          mActionsContainerBackground.setAlpha(containerAlpha);
          float containerScale = SCREENSHOT_ACTIONS_START_SCALE_X
                  + (t * (1 - SCREENSHOT_ACTIONS_START_SCALE_X));
         mActionsContainer.setScaleX(containerScale);
          mActionsContainerBackground.setScaleX(containerScale);
          for (ScreenshotActionChip chip : chips) {
              chip.setAlpha(t);
              chip.setScaleX(1 / containerScale); // invert to keep size of children constant
          }
          mActionsContainer.setScrollX(mDirectionLTR ? 0 : mActionsContainer.getWidth());
          mActionsContainer.setPivotX(mDirectionLTR ? 0 : mActionsContainer.getWidth());
         mActionsContainerBackground.setPivotX(
                 mDirectionLTR ? 0 : mActionsContainerBackground.getWidth());
      });
      return animator;
  }

```

在 GlobalScreenshot.java 的 showUiOnActionsReady(SavedImageData imageData) 的方法在进行弹窗展示的时候调用  
createScreenshotActionsShadeAnimation(SavedImageData imageData) 的方法来具体布局显示哪些选项  
而 ScreenshotActionChip 就是具体的选项布局  
从上面的代码可以看出

```
ScreenshotActionChip shareChip = (ScreenshotActionChip) inflater.inflate(
R.layout.global_screenshot_action_chip, mActionsView, false);
shareChip.setText(imageData.shareAction.title);
shareChip.setIcon(imageData.shareAction.getIcon(), true);
shareChip.setPendingIntent(imageData.shareAction.actionIntent, () -> {
mUiEventLogger.log(ScreenshotEvent.SCREENSHOT_SHARE_TAPPED);
dismissScreenshot("chip tapped", false);
mOnCompleteRunnable.run();
});
mActionsView.addView(shareChip);
chips.add(shareChip);

```

这就是分享按钮的代码  
所以注释掉这部分就可以了