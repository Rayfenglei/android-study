> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124830909)

### 1. 概述

在 11.0android 系统产品开发中，UsbManager 调用接口，会 [binder](https://so.csdn.net/so/search?q=binder&spm=1001.2101.3001.7020) 通信到 UsbService。而 UsbService 又有两个实例，一个  
UsbHostManager，一个 UsbDeviceManager。UsbDeviceManager 和  
UsbHostManager 是一个相对的概念，  
UsbHostManager 是手机作为一个 host，比如键盘、鼠标通过 usb 连接手机。而 UsbDeviceManager 是手机与电脑连接  
USB 的连接方式都是在 UsbDeviceManager.java 中处理的

### 2.USB 连接模式默认设为 MTP 的核心类

```
frameworks/base/services/usb/java/com/android/server/usb/UsbDeviceManager.java

```

### 3.USB 连接模式默认设为 MTP 的核心功能实现和分析

在系统中 UsbDeviceManager.java 是对 USB 设备管理的核心类，在 usb 连接以后，弹出对话框来判断当前  
usb 设备以什么样的形式来连接设备，  
路径为：  
接下来看下 frameworks/base/services/usb/java/com/android/server/usb/UsbDeviceManager.java

```
 @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_UPDATE_STATE:
                    mConnected = (msg.arg1 == 1);
                    mConfigured = (msg.arg2 == 1);

                    updateUsbNotification(false);
                    updateAdbNotification(false);
                    if (mBootCompleted) {
                        /*SPRD: add for usb sim activate @{ */
                        if (mConnected && isUsbShouldActived() && !mIsSimChecking) {
                            setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                            clearNotification();
                        }
                        /* @} */
                        updateUsbStateBroadcastIfNeeded(getAppliedFunctions(mCurrentFunctions));
                    }
                    if ((mCurrentFunctions & UsbManager.FUNCTION_ACCESSORY) != 0) {
                        updateCurrentAccessory();
                    }
                    if (mBootCompleted) {
                        if (!mConnected && !hasMessages(MSG_ACCESSORY_MODE_ENTER_TIMEOUT)
                                && !hasMessages(MSG_FUNCTION_SWITCH_TIMEOUT)) {
                            // restore defaults when USB is disconnected
                            if (!mScreenLocked
                                    && mScreenUnlockedFunctions != UsbManager.FUNCTION_NONE) {
                                setScreenUnlockedFunctions();
                            } else {
                                 setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                            }
                        }
                        updateUsbFunctions();
                    } else {
                        mPendingBootBroadcast = true;
                    }
                    break;
                case MSG_UPDATE_PORT_STATE:
                    SomeArgs args = (SomeArgs) msg.obj;
                    boolean prevHostConnected = mHostConnected;
                    UsbPort port = (UsbPort) args.arg1;
                    UsbPortStatus status = (UsbPortStatus) args.arg2;
                    mHostConnected = status.getCurrentDataRole() == DATA_ROLE_HOST;
                    mSourcePower = status.getCurrentPowerRole() == POWER_ROLE_SOURCE;
                    mSinkPower = status.getCurrentPowerRole() == POWER_ROLE_SINK;
                    mAudioAccessoryConnected = (status.getCurrentMode() == MODE_AUDIO_ACCESSORY);
                    mAudioAccessorySupported = port.isModeSupported(MODE_AUDIO_ACCESSORY);
                    // Ideally we want to see if PR_SWAP and DR_SWAP is supported.
                    // But, this should be suffice, since, all four combinations are only supported
                    // when PR_SWAP and DR_SWAP are supported.
                    mSupportsAllCombinations = status.isRoleCombinationSupported(
                            POWER_ROLE_SOURCE, DATA_ROLE_HOST)
                            && status.isRoleCombinationSupported(POWER_ROLE_SINK, DATA_ROLE_HOST)
                            && status.isRoleCombinationSupported(POWER_ROLE_SOURCE,
                            DATA_ROLE_DEVICE)
                            && status.isRoleCombinationSupported(POWER_ROLE_SINK, DATA_ROLE_DEVICE);

                    args.recycle();
                    updateUsbNotification(false);
                    if (mBootCompleted) {
                        if (mHostConnected || prevHostConnected) {
                            updateUsbStateBroadcastIfNeeded(getAppliedFunctions(mCurrentFunctions));
                        }
                    } else {
                        mPendingBootBroadcast = true;
                    }
                    break;
                case MSG_UPDATE_CHARGING_STATE:
                    updateUsbNotification(false);
                    boolean usbCharging = (msg.arg1 == 1);
                    /* add sprd usb feature @{*/
                    if(usbCharging != mUsbCharging) {
                        mUsbCharging = usbCharging;
                        if(!usbCharging) {
                            try{
                                String state = FileUtils.readTextFile(new File(STATE_PATH), 0, null).trim();
                                Slog.d(TAG, "receive power disconnected : usb state " + state);
                                if ("DISCONNECTED".equals(state)) {
                                    updateDisconnectStateAndNotification();
                                }
                            } catch (IOException e) {
                                Slog.e(TAG, "Error read usb_state path", e);
                            }
                        }
                    }
                    /* @} */
                    break;
                case MSG_UPDATE_HOST_STATE:
                    Iterator devices = (Iterator) msg.obj;
                    boolean connected = (msg.arg1 == 1);

                    if (DEBUG) {
                        Slog.i(TAG, "HOST_STATE connected:" + connected);
                    }

                    mHideUsbNotification = false;
                    while (devices.hasNext()) {
                        Map.Entry pair = (Map.Entry) devices.next();
                        if (DEBUG) {
                            Slog.i(TAG, pair.getKey() + " = " + pair.getValue());
                        }
                        UsbDevice device = (UsbDevice) pair.getValue();
                        int configurationCount = device.getConfigurationCount() - 1;
                        while (configurationCount >= 0) {
                            UsbConfiguration config = device.getConfiguration(configurationCount);
                            configurationCount--;
                            int interfaceCount = config.getInterfaceCount() - 1;
                            while (interfaceCount >= 0) {
                                UsbInterface intrface = config.getInterface(interfaceCount);
                                interfaceCount--;
                                if (sBlackListedInterfaces.contains(intrface.getInterfaceClass())) {
                                    mHideUsbNotification = true;
                                    break;
                                }
                            }
                        }
                    }
                    updateUsbNotification(false);
                    break;
                case MSG_ENABLE_ADB:
                    setAdbEnabled(msg.arg1 == 1);
                    break;
                case MSG_SET_CURRENT_FUNCTIONS:
                    long functions = (Long) msg.obj;
                    setEnabledFunctions(functions, false);
                    break;
                case MSG_SET_SCREEN_UNLOCKED_FUNCTIONS:
                    mScreenUnlockedFunctions = (Long) msg.obj;
                    if (mSettings != null) {
                        SharedPreferences.Editor editor = mSettings.edit();
                        editor.putString(String.format(Locale.ENGLISH, UNLOCKED_CONFIG_PREF,
                                mCurrentUser),
                                UsbManager.usbFunctionsToString(mScreenUnlockedFunctions));
                        editor.commit();
                    }
                    if (!mScreenLocked && mScreenUnlockedFunctions != UsbManager.FUNCTION_NONE) {
                        // If the screen is unlocked, also set current functions.
                        setScreenUnlockedFunctions();
                    }
                    break;
                case MSG_UPDATE_SCREEN_LOCK:
                    if (msg.arg1 == 1 == mScreenLocked) {
                        break;
                    }
                    mScreenLocked = msg.arg1 == 1;
                    if (!mBootCompleted) {
                        break;
                    }
                    if (mScreenLocked) {
                        if (!mConnected) {
                            setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                        }
                    } else {
                        if (mScreenUnlockedFunctions != UsbManager.FUNCTION_NONE
                                && mCurrentFunctions == UsbManager.FUNCTION_NONE) {
                            // Set the screen unlocked functions if current function is charging.
                            setScreenUnlockedFunctions();
                        }
                    }
                    break;
                case MSG_UPDATE_USER_RESTRICTIONS:
                    // Restart the USB stack if USB transfer is enabled but no longer allowed.
                    if (isUsbDataTransferActive(mCurrentFunctions) && !isUsbTransferAllowed()) {
                        setEnabledFunctions(UsbManager.FUNCTION_NONE, true);
                    }
                    break;
                case MSG_SYSTEM_READY:
                    mNotificationManager = (NotificationManager)
                            mContext.getSystemService(Context.NOTIFICATION_SERVICE);
                    mConnectivityManager = (ConnectivityManager) mContext.getSystemService(
                            Context.CONNECTIVITY_SERVICE);

                    LocalServices.getService(
                            AdbManagerInternal.class).registerTransport(new AdbTransport(this));

                    // Ensure that the notification channels are set up
                    if (isTv()) {
                        // TV-specific notification channel
                        mNotificationManager.createNotificationChannel(
                                new NotificationChannel(ADB_NOTIFICATION_CHANNEL_ID_TV,
                                        mContext.getString(
                                                com.android.internal.R.string
                                                        .adb_debugging_notification_channel_tv),
                                        NotificationManager.IMPORTANCE_HIGH));
                    }
                    mSystemReady = true;
                    finishBoot();
                    break;
                case MSG_LOCALE_CHANGED:
                    updateAdbNotification(true);
                    updateUsbNotification(true);
                    break;
                case MSG_BOOT_COMPLETED:
                    mBootCompleted = true;
                    finishBoot();
                    /*SPRD: add for usb state activate @{ */
                    if (isUsbShouldActived()) {
                        sendMessageDelayed(Message.obtain(this, MSG_SIM_CHECKING), SIM_CHECKING_TIMEOUT);
                    }
                    /* @} */
                    break;
                case MSG_USER_SWITCHED: {
                    if (mCurrentUser != msg.arg1) {
                        if (DEBUG) {
                            Slog.v(TAG, "Current user switched to " + msg.arg1);
                        }
                        mCurrentUser = msg.arg1;
                        mScreenLocked = true;
                        mScreenUnlockedFunctions = UsbManager.FUNCTION_NONE;
                        if (mSettings != null) {
                            mScreenUnlockedFunctions = UsbManager.usbFunctionsFromString(
                                    mSettings.getString(String.format(Locale.ENGLISH,
                                            UNLOCKED_CONFIG_PREF, mCurrentUser), ""));
                        }
                        setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                    }
                    break;
                }
                case MSG_ACCESSORY_MODE_ENTER_TIMEOUT: {
                    if (DEBUG) {
                        Slog.v(TAG, "Accessory mode enter timeout: " + mConnected);
                    }
                    if (!mConnected || (mCurrentFunctions & UsbManager.FUNCTION_ACCESSORY) == 0) {
                        notifyAccessoryModeExit();
                    }
                    break;
                }

                /*SPRD: add for Usb sim activate @{ */
                case MSG_SIM_CHECKING:
                    mIsSimChecking = false;
                    updateUsbNotification(false);
                    updateAdbNotification(false);
                    if (mConnected && isUsbShouldActived()) {
                        setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                        showWarningDialog();
                    }
                    break;
                case MSG_SWITCH_FOR_USB_ACTIVE_CHANGED:
                    if (msg.arg1 == 1) {
                        mIsSimChecking = false;
                        if (mConnected && isUsbShouldActived()) {
                            setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                            showWarningDialog();
                        }
                        if (isUsbShouldActived() && mIsSimStateRegister == false) {
                            IntentFilter simFilter = new IntentFilter(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
                            mContext.registerReceiver(mSimStateChangeReceiver, simFilter);
                            mIsSimStateRegister = true;
                        }
                    } else {
                        if (mWarningDialog != null) {
                            mWarningDialog.dismiss();
                            if(!mScreenLocked)
                                setScreenUnlockedFunctions();
                            else
                                setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                        }
                        if (mIsSimStateRegister) {
                            mContext.unregisterReceiver(mSimStateChangeReceiver);
                            mIsSimStateRegister = false;
                        }
                    }
                    break;
                /* @} */
            }
        }

```

在 UsbDeviceManager.java 的相关源码中，可以从 handler 事件看出这里是对 usb 设备各种状态的分析和处理  
而在其中的 hanlder 消息中分析得知 MSG_BOOT_COMPLETED  
处理 开机完成的事件 finishBoot();

看下 finishBoot()

```
protected void finishBoot() {
	android.service.oemlock.OemLockManager mOemLockManager = (android.service.oemlock.OemLockManager) mContext.getSystemService(Context.OEM_LOCK_SERVICE);
	mOemLockManager.setOemUnlockAllowedByUser(true);
    if (mBootCompleted && mCurrentUsbFunctionsReceived && mSystemReady) {
        if (mPendingBootBroadcast) {
            updateUsbStateBroadcastIfNeeded(getAppliedFunctions(mCurrentFunctions));
            mPendingBootBroadcast = false;
        }
        if (!mScreenLocked
                && mScreenUnlockedFunctions != UsbManager.FUNCTION_NONE) {
            setScreenUnlockedFunctions();
        } else {
             setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
        }
        if (mCurrentAccessory != null) {
            mUsbDeviceManager.getCurrentSettings().accessoryAttached(mCurrentAccessory);
        }

        updateUsbNotification(false);
        updateAdbNotification(false);
        updateUsbFunctions();
    }
}

```

通过分析 finishBoot() 的事件中得知  
从 setEnabledFunctions(FUNCTION_NONE，false); 设置默认为充电模式

主要修改如下在 handleMessage(Message msg) 中的

MSG_UPDATE_STATE 事件修改 usb 连接模式

```
case MSG_UPDATE_STATE:
                    mConnected = (msg.arg1 == 1);
                    mConfigured = (msg.arg2 == 1);

                    updateUsbNotification(false);
                    updateAdbNotification(false);
                    if (mBootCompleted) {
                        /*SPRD: add for usb sim activate @{ */
                        if (mConnected && isUsbShouldActived() && !mIsSimChecking) {
                            setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                            clearNotification();
                        }
                        /* @} */
                        updateUsbStateBroadcastIfNeeded(getAppliedFunctions(mCurrentFunctions));
                    }
                    if ((mCurrentFunctions & UsbManager.FUNCTION_ACCESSORY) != 0) {
                        updateCurrentAccessory();
                    }
                    if (mBootCompleted) {
                        if (!mConnected && !hasMessages(MSG_ACCESSORY_MODE_ENTER_TIMEOUT)
                                && !hasMessages(MSG_FUNCTION_SWITCH_TIMEOUT)) {
                            // restore defaults when USB is disconnected
                            if (!mScreenLocked
                                    && mScreenUnlockedFunctions != UsbManager.FUNCTION_NONE) {
                                setScreenUnlockedFunctions();
                            } else {
                               - setEnabledFunctions(UsbManager.FUNCTION_NONE, false);
                                + setEnabledFunctions(UsbManager.FUNCTION_MTP, false);
                            }
                        }
                        updateUsbFunctions();
                    } else {
                        mPendingBootBroadcast = true;
                    }

```