> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124855261)

### 1. 概述

在 11.0 12.0 定制化开发中，对于 wifi 的功能开发需求也是比较多的，比如有功能需求在开机以后要连接指定 wifi，这就需要在系统启动完毕后，连接到指定 wifi 就可以了，系统模块有好多，[AMS](https://so.csdn.net/so/search?q=AMS&spm=1001.2101.3001.7020) PMS WMSd 等等，所以在 AMS 开机完成后，进行连接指定的 wifi 也是同样可以完成功能的

### 2. 开机连接指定 wifi 核心类

```
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```

### 3. 开机连接指定 wifi 的功能分析和实现

在系统开机完成后连接指定 wifi, 在 AMS 中系统完成后会调用 systemReady, 而在 finishBooting()  
同样也是在系统开机完成后调用的方法, 所以也可以在这里实现功能  
首选看 AMS 中 开机完成的源码

```
final void finishBooting() {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "FinishBooting");

        synchronized (this) {
            if (!mBootAnimationComplete) {
                mCallFinishBooting = true;
                return;
            }
            mCallFinishBooting = false;
        }

        ArraySet<String> completedIsas = new ArraySet<String>();
        for (String abi : Build.SUPPORTED_ABIS) {
            ZYGOTE_PROCESS.establishZygoteConnectionForAbi(abi);
            final String instructionSet = VMRuntime.getInstructionSet(abi);
            if (!completedIsas.contains(instructionSet)) {
                try {
                    mInstaller.markBootComplete(VMRuntime.getInstructionSet(abi));
                } catch (InstallerException e) {
                    if (!VMRuntime.didPruneDalvikCache()) {
                        // This is technically not the right filter, as different zygotes may
                        // have made different pruning decisions. But the log is best effort,
                        // anyways.
                        Slog.w(TAG, "Unable to mark boot complete for abi: " + abi + " (" +
                                e.getMessage() +")");
                    }
                }
                completedIsas.add(instructionSet);
            }
        }

        IntentFilter pkgFilter = new IntentFilter();
        pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        pkgFilter.addDataScheme("package");
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String[] pkgs = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
                if (pkgs != null) {
                    for (String pkg : pkgs) {
                        synchronized (ActivityManagerService.this) {
                            if (forceStopPackageLocked(pkg, -1, false, false, false, false, false,
                                    0, "query restart")) {
                                setResultCode(Activity.RESULT_OK);
                                return;
                            }
                        }
                    }
                }
            }
        }, pkgFilter);

        IntentFilter dumpheapFilter = new IntentFilter();
        dumpheapFilter.addAction(DumpHeapActivity.ACTION_DELETE_DUMPHEAP);
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                final long delay = intent.getBooleanExtra(
                        DumpHeapActivity.EXTRA_DELAY_DELETE, false) ? 5 * 60 * 1000 : 0;
                mHandler.sendEmptyMessageDelayed(DELETE_DUMPHEAP_MSG, delay);
            }
        }, dumpheapFilter);

        // Inform checkpointing systems of success
        try {
            // This line is needed to CTS test for the correct exception handling
            // See b/138952436#comment36 for context
            Slog.i(TAG, "About to commit checkpoint");
            IStorageManager storageManager = PackageHelper.getStorageManager();
            storageManager.commitChanges();
        } catch (Exception e) {
            PowerManager pm = (PowerManager)
                     mContext.getSystemService(Context.POWER_SERVICE);
            pm.reboot("Checkpoint commit failed");
        }

        // Let system services know.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);

        synchronized (this) {
            // Ensure that any processes we had put on hold are now started
            // up.
            final int NP = mProcessesOnHold.size();
            if (NP > 0) {
                ArrayList<ProcessRecord> procs =
                    new ArrayList<ProcessRecord>(mProcessesOnHold);
                for (int ip=0; ip<NP; ip++) {
                    if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "Starting process on hold: "
                            + procs.get(ip));
                    mProcessList.startProcessLocked(procs.get(ip), new HostingRecord("on-hold"));
                }
            }
            if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                return;
            }
            // Start looking for apps that are abusing wake locks.
            Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_POWER_USE_MSG);
            mHandler.sendMessageDelayed(nmsg, mConstants.POWER_CHECK_INTERVAL);
            // Tell anyone interested that we are done booting!
            SystemProperties.set("sys.boot_completed", "1");

            // And trigger dev.bootcomplete if we are not showing encryption progress
            if (!"trigger_restart_min_framework".equals(VoldProperties.decrypt().orElse(""))
                    || "".equals(VoldProperties.encrypt_progress().orElse(""))) {
                SystemProperties.set("dev.bootcomplete", "1");
            }
            mUserController.sendBootCompleted(
                    new IIntentReceiver.Stub() {
                        @Override
                        public void performReceive(Intent intent, int resultCode,
                                String data, Bundle extras, boolean ordered,
                                boolean sticky, int sendingUser) {
                            synchronized (ActivityManagerService.this) {
                                mOomAdjuster.mAppCompact.compactAllSystem();
                                requestPssAllProcsLocked(SystemClock.uptimeMillis(), true, false);
                            }
                        }
                    });
            mUserController.scheduleStartProfiles();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }

```

在 finishBooting() 开机完成后会调用系统属性设置  
SystemProperties.set(“sys.boot_completed”, “1”); 系统属性标志开机完成，来结束开机动画  
从源码中可以看出 这里开机完成后 肯定后执行这里  
所以就在 这里添加 连接指定 wifi 的代码如下 ：  
接下来实现连接 wifi 的相关功能方法

```
private WifiManager mWifiManager = null;
final void finishBooting() {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "FinishBooting");

    synchronized (this) {
        if (!mBootAnimationComplete) {
            mCallFinishBooting = true;
            return;
    }
        mCallFinishBooting = false;
    }

    ArraySet<String> completedIsas = new ArraySet<String>();
    for (String abi : Build.SUPPORTED_ABIS) {
        ZYGOTE_PROCESS.establishZygoteConnectionForAbi(abi);
        final String instructionSet = VMRuntime.getInstructionSet(abi);
        if (!completedIsas.contains(instructionSet)) {
            try {
                mInstaller.markBootComplete(VMRuntime.getInstructionSet(abi));
            } catch (InstallerException e) {
                if (!VMRuntime.didPruneDalvikCache()) {
                    // This is technically not the right filter, as different zygotes may
                    // have made different pruning decisions. But the log is best effort,
                    // anyways.
                    Slog.w(TAG, "Unable to mark boot complete for abi: " + abi + " (" +
                            e.getMessage() +")");
                }
            }
            completedIsas.add(instructionSet);
        }
    }

    IntentFilter pkgFilter = new IntentFilter();
    pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
    pkgFilter.addDataScheme("package");
    mContext.registerReceiver(new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String[] pkgs = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
            if (pkgs != null) {
                for (String pkg : pkgs) {
                    synchronized (ActivityManagerService.this) {
                        if (forceStopPackageLocked(pkg, -1, false, false, false, false, false,
                                0, "query restart")) {
                            setResultCode(Activity.RESULT_OK);
                            return;
                        }
                    }
                }
            }
        }
    }, pkgFilter);

    IntentFilter dumpheapFilter = new IntentFilter();
    dumpheapFilter.addAction(DumpHeapActivity.ACTION_DELETE_DUMPHEAP);
    mContext.registerReceiver(new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final long delay = intent.getBooleanExtra(
                    DumpHeapActivity.EXTRA_DELAY_DELETE, false) ? 5 * 60 * 1000 : 0;
            mHandler.sendEmptyMessageDelayed(DELETE_DUMPHEAP_MSG, delay);
        }
    }, dumpheapFilter);

    // Inform checkpointing systems of success
    try {
        // This line is needed to CTS test for the correct exception handling
        // See b/138952436#comment36 for context
        Slog.i(TAG, "About to commit checkpoint");
        IStorageManager storageManager = PackageHelper.getStorageManager();
        storageManager.commitChanges();
    } catch (Exception e) {
        PowerManager pm = (PowerManager)
                 mContext.getSystemService(Context.POWER_SERVICE);
        pm.reboot("Checkpoint commit failed");
    }

    // Let system services know.
    mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);

    synchronized (this) {
        // Ensure that any processes we had put on hold are now started
        // up.
        final int NP = mProcessesOnHold.size();
        if (NP > 0) {
            ArrayList<ProcessRecord> procs =
                new ArrayList<ProcessRecord>(mProcessesOnHold);
            for (int ip=0; ip<NP; ip++) {
                if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "Starting process on hold: "
                        + procs.get(ip));
                mProcessList.startProcessLocked(procs.get(ip), new HostingRecord("on-hold"));
            }
        }
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            return;
        }
        // Start looking for apps that are abusing wake locks.
        Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_POWER_USE_MSG);
        mHandler.sendMessageDelayed(nmsg, mConstants.POWER_CHECK_INTERVAL);
        // Tell anyone interested that we are done booting!
        SystemProperties.set("sys.boot_completed", "1");
		
	// 添加指定连接wifi代码开始

       mWifiManager = (WifiManager) mContext.getApplicationContext().getSystemService(Context.WIFI_SERVICE);
        if (!mWifiManager.isWifiEnabled()) mWifiManager.setWifiEnabled(true);
        mHandler.postDelayed(new Runnable() {
			@Override
			public void run() {
                                  addNetwork(createWifiInfo("TP-LINK_5","84660299",3));
			}
		},5000);
     // 添加指定连接wifi代码结束
           
        // And trigger dev.bootcomplete if we are not showing encryption progress
        if (!"trigger_restart_min_framework".equals(VoldProperties.decrypt().orElse(""))
                || "".equals(VoldProperties.encrypt_progress().orElse(""))) {
            SystemProperties.set("dev.bootcomplete", "1");
        }
        mUserController.sendBootCompleted(
                new IIntentReceiver.Stub() {
                    @Override
                    public void performReceive(Intent intent, int resultCode,
                            String data, Bundle extras, boolean ordered,
                            boolean sticky, int sendingUser) {
                        synchronized (ActivityManagerService.this) {
                            mOomAdjuster.mAppCompact.compactAllSystem();
                            requestPssAllProcsLocked(SystemClock.uptimeMillis(), true, false);
                        }
                    }
                });
        mUserController.scheduleStartProfiles();
    }

    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
}


 

// 添加wifi连接的代码

    public void addNetwork(WifiConfiguration wcg) {
        int wcgID = mWifiManager.addNetwork(wcg);
        boolean b = mWifiManager.enableNetwork(wcgID, true);
    }

    public WifiConfiguration createWifiInfo(String SSID, String Password, int Type) {
        WifiConfiguration config = new WifiConfiguration();
        config.allowedAuthAlgorithms.clear();
        config.allowedGroupCiphers.clear();
        config.allowedKeyManagement.clear();
        config.allowedPairwiseCiphers.clear();
        config.allowedProtocols.clear();
        config.SSID = "\"" + SSID + "\"";

        WifiConfiguration tempConfig = this.IsExsits(SSID);
        if (tempConfig != null) {

            mWifiManager.removeNetwork(tempConfig.networkId);

        }

        if (Type == 1) //WIFICIPHER_NOPASS
        {
//            config.wepKeys[0] = "";
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE);
            config.wepTxKeyIndex = 0;
        }
        if (Type == 2) //WIFICIPHER_WEP
        {
            config.hiddenSSID = true;
            config.wepKeys[0] = "\"" + Password + "\"";
            config.allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.SHARED);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.WEP40);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.WEP104);
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE);
            config.wepTxKeyIndex = 0;
        }
        if (Type == 3) //WIFICIPHER_WPA
        {
            config.preSharedKey = "\"" + Password + "\"";
            config.hiddenSSID = true;
            config.allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.OPEN);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP);
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.WPA_PSK);
            config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.TKIP);
            //config.allowedProtocols.set(WifiConfiguration.Protocol.WPA);
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP);
            config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.CCMP);
            config.status = WifiConfiguration.Status.ENABLED;
        }
        return config;
    }
    private WifiConfiguration IsExsits(String SSID) {
        List<WifiConfiguration> existingConfigs = mWifiManager.getConfiguredNetworks();
        if (null == existingConfigs) {
            Log.i("xgr", "existingConfigs null ");
            return null;
        }
        for (WifiConfiguration existingConfig : existingConfigs) {
            if (existingConfig.SSID.equals("\"" + SSID + "\"")) {
                return existingConfig;
            }
        }
        return null;
    }

```

在上述代码中通过 添加 wifi 连接的代码，就可以实现连接 wifi 的功能，然后调用连接 wifi 的 addNetwork(createWifiInfo(“TP-LINK_5”,“82660399”,3)); 实现了 wifi 的连接工作  
就这样实现了 开机连接指定 wifi 的代码