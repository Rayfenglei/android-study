> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872627)

### 1. 概述

在 11.0 的系统长按关机键，会弹出关机的对话框，关机对话框里面由关机重启截图和紧急呼叫等功能，而由于开发功能需求要求去掉屏幕截图和紧急呼叫等功能，所以就要先找到关机对框的代码  
然后实现功能  
功能分析：  
长按电源键弹出关机对话框，通过 [adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) shell 命令发现  
就是 frameworks/base/packages/[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020)/src/com/android/systemui/globalactions/GlobalActionsDialog.java  
而所有的关机 Actions 事件也是在 GlobalActionDialog 中处理的

### 2. 长按 Power 弹出关机对话框去掉屏幕截图和紧急呼救功能的核心代码

```
frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsDialog.java

```

### 3. 长按 Power 弹出关机对话框去掉屏幕截图和紧急呼救功能分析和实现

去掉截图和紧急呼叫功能首先分析代码  
接下来看下 GlobalActionDialog.java 源码

```
public void showDialog(boolean keyguardShowing, boolean isDeviceProvisioned,
        GlobalActionsPanelPlugin panelPlugin) {
    mKeyguardShowing = keyguardShowing;
    mDeviceProvisioned = isDeviceProvisioned;
    mPanelPlugin = panelPlugin;
    if (mDialog != null) {
        mDialog.dismiss();
        mDialog = null;
        // Show delayed, so that the dismiss of the previous dialog completes
        mHandler.sendEmptyMessage(MESSAGE_SHOW);
    } else {
        handleShow();
    }
}

```

长按 power 键通过调用 showDialog(弹出关机对话框

```
private void handleShow() {
        awakenIfNecessary();
        mDialog = createDialog();
        prepareDialog();

        // If we only have 1 item and it's a simple press action, just do this action.
        if (mAdapter.getCount() == 1
                && mAdapter.getItem(0) instanceof SinglePressAction
                && !(mAdapter.getItem(0) instanceof LongPressAction)) {
            ((SinglePressAction) mAdapter.getItem(0)).onPress();
        } else {
            WindowManager.LayoutParams attrs = mDialog.getWindow().getAttributes();
            attrs.setTitle("ActionsDialog");
            attrs.layoutInDisplayCutoutMode = LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
            mDialog.getWindow().setAttributes(attrs);
            mDialog.show();
            mWindowManagerFuncs.onGlobalActionsShown();

            mHandler.postDelayed(() -> {
                notifyStopLongscreenshotIfNeeded();
            }, 200);
        }
    }

```

在关机对话框页面的 createDialog() 添加各种 Action 事件关机重启和截图等功能

```
 /**
     * Create the global actions dialog.
     *
     * @return A new dialog.
     */
    private ActionsDialog createDialog() {
        // Simple toggle style if there's no vibrator, otherwise use a tri-state
        if (!mHasVibrator) {
            mSilentModeAction = new SilentModeToggleAction();
        } else {
            mSilentModeAction = new SilentModeTriStateAction(mAudioManager, mHandler);
        }
        mAirplaneModeOn = new ToggleAction(
                R.drawable.ic_lock_airplane_mode,
                R.drawable.ic_lock_airplane_mode_off,
                R.string.global_actions_toggle_airplane_mode,
                R.string.global_actions_airplane_mode_on_status,
                R.string.global_actions_airplane_mode_off_status) {

            void onToggle(boolean on) {
                if (mHasTelephony && Boolean.parseBoolean(
                        SystemProperties.get(TelephonyProperties.PROPERTY_INECM_MODE))) {
                    mIsWaitingForEcmExit = true;
                    // Launch ECM exit dialog
                    Intent ecmDialogIntent =
                            new Intent(TelephonyIntents.ACTION_SHOW_NOTICE_ECM_BLOCK_OTHERS, null);
                    ecmDialogIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    mContext.startActivity(ecmDialogIntent);
                } else {
                    changeAirplaneModeSystemSetting(on);
                }
            }

            @Override
            protected void changeStateFromPress(boolean buttonOn) {
                if (!mHasTelephony) return;

                // In ECM mode airplane state cannot be changed
                if (!(Boolean.parseBoolean(
                        SystemProperties.get(TelephonyProperties.PROPERTY_INECM_MODE)))) {
                    mState = buttonOn ? State.TurningOn : State.TurningOff;
                    mAirplaneState = mState;
                }
            }

            public boolean showDuringKeyguard() {
                return true;
            }

            public boolean showBeforeProvisioning() {
                return false;
            }
        };
        onAirplaneModeChanged();

        mItems = new ArrayList<Action>();
        String[] defaultActions = mContext.getResources().getStringArray(
                R.array.config_globalActionsList);

        ArraySet<String> addedKeys = new ArraySet<String>();
        mHasLogoutButton = false;
        mHasLockdownButton = false;
        for (int i = 0; i < defaultActions.length; i++) {
            String actionKey = defaultActions[i];
            if (addedKeys.contains(actionKey)) {
                // If we already have added this, don't add it again.
                continue;
            }
            if (GLOBAL_ACTION_KEY_POWER.equals(actionKey)) {
                mItems.add(new PowerAction());
            } else if (GLOBAL_ACTION_KEY_AIRPLANE.equals(actionKey)) {
                mItems.add(mAirplaneModeOn);
            } else if (GLOBAL_ACTION_KEY_BUGREPORT.equals(actionKey)) {
                if (Settings.Global.getInt(mContext.getContentResolver(),
                        Settings.Global.BUGREPORT_IN_POWER_MENU, 0) != 0 && isCurrentUserOwner()) {
                    mItems.add(new BugReportAction());
                }
            } else if (GLOBAL_ACTION_KEY_SILENT.equals(actionKey)) {
                if (mShowSilentToggle) {
                    mItems.add(mSilentModeAction);
                }
            } else if (GLOBAL_ACTION_KEY_USERS.equals(actionKey)) {
                if (SystemProperties.getBoolean("fw.power_user_switcher", false)) {
                    addUsersToMenu(mItems);
                }
            } else if (GLOBAL_ACTION_KEY_SETTINGS.equals(actionKey)) {
                mItems.add(getSettingsAction());
            } else if (GLOBAL_ACTION_KEY_LOCKDOWN.equals(actionKey)) {
                if (Settings.Secure.getIntForUser(mContext.getContentResolver(),
                            Settings.Secure.LOCKDOWN_IN_POWER_MENU, 0, getCurrentUser().id) != 0
                        && shouldDisplayLockdown()) {
                    mItems.add(getLockdownAction());
                    mHasLockdownButton = true;
                }
            } else if (GLOBAL_ACTION_KEY_VOICEASSIST.equals(actionKey)) {
                mItems.add(getVoiceAssistAction());
            } else if (GLOBAL_ACTION_KEY_ASSIST.equals(actionKey)) {
                mItems.add(getAssistAction());
            } else if (GLOBAL_ACTION_KEY_RESTART.equals(actionKey)) {
                mItems.add(new RestartAction());
            } else if (GLOBAL_ACTION_KEY_SCREENSHOT.equals(actionKey)) {

                //UNISOC: fix for bug 1104146
                if(KeyguardUpdateMonitor.getInstance(mContext).isSecure()
                        && !KeyguardUpdateMonitor.getInstance(mContext).getStrongAuthTracker().hasUserAuthenticatedSinceBoot()) {
                    Log.e(TAG, "Not add screenshot because user hasn't authenticated.");
                    continue;
                }

                /* UNISOC Bug 1074234,939142 remove screenshot when pressing power,in powersave mode @{ */
                if (SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE) {
                   if (!SprdPowerManagerUtil.isSuperPower()) {
                       mItems.add(new ScreenshotAction());
                     }
                } else {
                       mItems.add(new ScreenshotAction());
                }
                /* @} */
            } else if (GLOBAL_ACTION_KEY_LOGOUT.equals(actionKey)) {
                if (mDevicePolicyManager.isLogoutEnabled()
                        && getCurrentUser().id != UserHandle.USER_SYSTEM) {
                    mItems.add(new LogoutAction());
                    mHasLogoutButton = true;
                }
            } else if (GLOBAL_ACTION_KEY_EMERGENCY.equals(actionKey)) {
                if (!mEmergencyAffordanceManager.needsEmergencyAffordance()) {
                    mItems.add(new EmergencyDialerAction());
                }
            } else {
                Log.e(TAG, "Invalid global action key " + actionKey);
            }
            // Add here so we don't add more than one.
            addedKeys.add(actionKey);
        }

        if (mEmergencyAffordanceManager.needsEmergencyAffordance()) {
            mItems.add(new EmergencyAffordanceAction());
        }

        mAdapter = new MyAdapter();

        GlobalActionsPanelPlugin.PanelViewController panelViewController =
                mPanelPlugin != null
                        ? mPanelPlugin.onPanelShown(
                                new GlobalActionsPanelPlugin.Callbacks() {
                                    @Override
                                    public void dismissGlobalActionsMenu() {
                                        if (mDialog != null) {
                                            mDialog.dismiss();
                                        }
                                    }

                                    @Override
                                    public void startPendingIntentDismissingKeyguard(
                                            PendingIntent intent) {
                                        mActivityStarter
                                                .startPendingIntentDismissingKeyguard(intent);
                                    }
                                },
                                mKeyguardManager.isDeviceLocked())
                        : null;

        ActionsDialog dialog = new ActionsDialog(mContext, mAdapter, panelViewController);
        dialog.setCanceledOnTouchOutside(false); // Handled by the custom class.
        dialog.setKeyguardShowing(mKeyguardShowing);

        dialog.setOnDismissListener(this);
        dialog.setOnShowListener(this);

        return dialog;
    }

```

在代码中发现加载了各种功能事件  
ScreenshotAction 就是屏幕截图 action

```
private class ScreenshotAction extends SinglePressAction implements LongPressAction {
        public ScreenshotAction() {
            super(R.drawable.ic_screenshot, R.string.global_action_screenshot);
        }

        @Override
        public void onPress() {
            if (isInRegionOrLongshotMode(mContext.getContentResolver())) {
                Toast.makeText(mContext, com.android.systemui.R.string.long_screen_shot_tips, Toast.LENGTH_SHORT).show();
                return;
            }

            // Add a little delay before executing, to give the
            // dialog a chance to go away before it takes a
            // screenshot.
            // TODO: instead, omit global action dialog layer
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    mScreenshotHelper.takeScreenshot(1, true, true, mHandler);
                    MetricsLogger.action(mContext,
                            MetricsEvent.ACTION_SCREENSHOT_POWER_MENU);
                }
            }, 500);
        }

        @Override
        public boolean showDuringKeyguard() {
            return true;
        }

        @Override
        public boolean showBeforeProvisioning() {
            return false;
        }

        @Override
        public boolean onLongPress() {
            if (FeatureFlagUtils.isEnabled(mContext, FeatureFlagUtils.SCREENRECORD_LONG_PRESS)) {
                mScreenRecordHelper.launchRecordPrompt();
            } else {
                onPress();
            }
            return true;
        }
    }

```

紧急呼救 action

```
private class EmergencyAffordanceAction extends EmergencyAction {
    EmergencyAffordanceAction() {
        super(R.drawable.emergency_icon,
                R.string.global_action_emergency);
    }

    @Override
    public void onPress() {
        mEmergencyAffordanceManager.performEmergencyCall();
    }
}

private class EmergencyDialerAction extends EmergencyAction {
    private EmergencyDialerAction() {
        super(com.android.systemui.R.drawable.ic_emergency_star,
                R.string.global_action_emergency);
    }

    @Override
    public void onPress() {
        MetricsLogger.action(mContext, MetricsEvent.ACTION_EMERGENCY_DIALER_FROM_POWER_MENU);
        Intent intent = new Intent(EmergencyDialerConstants.ACTION_DIAL);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
                | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        intent.putExtra(EmergencyDialerConstants.EXTRA_ENTRY_TYPE,
                EmergencyDialerConstants.ENTRY_TYPE_POWER_MENU);
        mContext.startActivityAsUser(intent, UserHandle.CURRENT);
        mContext.sendBroadcast(new Intent(Intent.ACTION_CLOSE_SYSTEM_DIALOGS));
    }
}

```

通过上面的源码可以知道只需要在 createDialog() 中去掉屏幕截图和紧急救援 action 就可以了

实现案例如下:

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsDialog.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsDialog.java
@@ -408,13 +408,13 @@ public class GlobalActionsDialog implements DialogInterface.OnDismissListener,
                 }
 
                 /* UNISOC Bug 1074234,939142 remove screenshot when pressing power,in powersave mode @{ */
-                if (SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE) {
+                /*if (SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE) {
                    if (!SprdPowerManagerUtil.isSuperPower()) {
                        mItems.add(new ScreenshotAction());
                      }
                 } else {
                        mItems.add(new ScreenshotAction());
-                }
+                }*/
                 /* @} */
             } else if (GLOBAL_ACTION_KEY_LOGOUT.equals(actionKey)) {
                 if (mDevicePolicyManager.isLogoutEnabled()
@@ -423,9 +423,9 @@ public class GlobalActionsDialog implements DialogInterface.OnDismissListener,
                     mHasLogoutButton = true;
                 }
             } else if (GLOBAL_ACTION_KEY_EMERGENCY.equals(actionKey)) {
-                if (!mEmergencyAffordanceManager.needsEmergencyAffordance()) {
+                /*if (!mEmergencyAffordanceManager.needsEmergencyAffordance()) {
                     mItems.add(new EmergencyDialerAction());
-                }
+                }*/
             } else {
                 Log.e(TAG, "Invalid global action key " + actionKey);
             }
@@ -433,9 +433,9 @@ public class GlobalActionsDialog implements DialogInterface.OnDismissListener,
             addedKeys.add(actionKey);
         }
 
-        if (mEmergencyAffordanceManager.needsEmergencyAffordance()) {
+        /*if (mEmergencyAffordanceManager.needsEmergencyAffordance()) {
             mItems.add(new EmergencyAffordanceAction());
-        }
+        }*/
 
         mAdapter = new MyAdapter();

```