> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870069)

### 1. 概述

在 11.0 的产品开发中，在对产品进行网络模块开发中，有功能需要要求设置某些 app 可以上网，某些 app 不可以上网，就是所谓的网络应用白名单功能

### 2. 设置上网应用白名单 (上网 app 白名单) 核心代码

```
frameworks/base/core/java/android/os/INetworkManagementService.aidl
frameworks/base/services/core/java/com/android/server/NetworkManagementService.java

```

### 3. 设置上网应用白名单 (上网 app 白名单) 的功能分析和实现

在系统中整个网络模块都是由 [NMS](https://so.csdn.net/so/search?q=NMS&spm=1001.2101.3001.7020) 服务负责通讯的  
接下来先看下 NetworkManagementService.java

### 3.1NetworkManagementService.java 中上网 app 相关代码分析

```
@Override
public void setFirewallEnabled(boolean enabled) {
    enforceSystemUid();
    try {
        mNetdService.firewallSetFirewallType(
                enabled ? INetd.FIREWALL_WHITELIST : INetd.FIREWALL_BLACKLIST);
        mFirewallEnabled = enabled;
    } catch (RemoteException | ServiceSpecificException e) {
        throw new IllegalStateException(e);
    }
}

```

通过读代码可以发现这是个设置防火墙的方法

```
@Override
public void setFirewallUidRules(int chain, int[] uids, int[] rules) {
    enforceSystemUid();
    synchronized (mQuotaLock) {
        synchronized (mRulesLock) {
            SparseIntArray uidFirewallRules = getUidFirewallRulesLR(chain);
            SparseIntArray newRules = new SparseIntArray();
            // apply new set of rules
            for (int index = uids.length - 1; index >= 0; --index) {
                int uid = uids[index];
                int rule = rules[index];
                updateFirewallUidRuleLocked(chain, uid, rule);
                newRules.put(uid, rule);
            }
            // collect the rules to remove.
            SparseIntArray rulesToRemove = new SparseIntArray();
            for (int index = uidFirewallRules.size() - 1; index >= 0; --index) {
                int uid = uidFirewallRules.keyAt(index);
                if (newRules.indexOfKey(uid) < 0) {
                    rulesToRemove.put(uid, FIREWALL_RULE_DEFAULT);
                }
            }
            // remove dead rules
            for (int index = rulesToRemove.size() - 1; index >= 0; --index) {
                int uid = rulesToRemove.keyAt(index);
                updateFirewallUidRuleLocked(chain, uid, FIREWALL_RULE_DEFAULT);
            }
        }
        try {
            switch (chain) {
                case FIREWALL_CHAIN_DOZABLE:
                    mNetdService.firewallReplaceUidChain("fw_dozable", true, uids);
                    break;
                case FIREWALL_CHAIN_STANDBY:
                    mNetdService.firewallReplaceUidChain("fw_standby", false, uids);
                    break;
                case FIREWALL_CHAIN_POWERSAVE:
                    mNetdService.firewallReplaceUidChain("fw_powersave", true, uids);
                    break;
                case FIREWALL_CHAIN_NONE:
                default:
                    Slog.d(TAG, "setFirewallUidRules() called on invalid chain: " + chain);
            }
        } catch (RemoteException e) {
            Slog.w(TAG, "Error flushing firewall chain " + chain, e);
        }
    }
}

```

在 setFirewallUidRules(int chain, int[] uids, int[] rules) 中根据 chain 链 uids 和规则等参数来设置  
上网的规则

通过代码可以了解 这个方法是根据 app 的 uid 设置访问网络  
所以在系统中可以通过 NMS 就可以实现上网应用白名单的功能，只是需要添加接口就好了

### 3.2 INetworkManagementService.aidl 添加接口

具体实现如下:

```
diff --git a/frameworks/base/core/java/android/os/INetworkManagementService.aidl b/frameworks/base/core/java/android/os/INetworkManagementService.aidl
old mode 100644
new mode 100755
index 0ba77be03d..a947881dd7
--- a/frameworks/base/core/java/android/os/INetworkManagementService.aidl
+++ b/frameworks/base/core/java/android/os/INetworkManagementService.aidl
@@ -406,4 +406,6 @@ interface INetworkManagementService
      * destory Sockets by Uid
      */
     void destorySocketByUid(int uid);
+       void setWhiteNetApp(in List<String> packageNames);
+       List<String> getWhiteNetApp();
 }

```

### 3.2 NetworkManagementService.java 实现这些接口 然后设置上网 app

```
diff --git a/frameworks/base/services/core/java/com/android/server/NetworkManagementService.java b/frameworks/base/services/core/java/com/android/server/NetworkManagementService.java
old mode 100644
new mode 100755
index 606dab13e4..a8e2e8509a
--- a/frameworks/base/services/core/java/com/android/server/NetworkManagementService.java
+++ b/frameworks/base/services/core/java/com/android/server/NetworkManagementService.java
@@ -40,7 +40,8 @@ import static android.net.NetworkStats.UID_ALL;
 import static android.net.TrafficStats.UID_TETHERING;
 
 import static com.android.server.NetworkManagementSocketTagger.PROP_QTAGUID_ENABLED;
-
+import android.content.pm.PackageManager;
+import android.content.pm.ApplicationInfo;
 import android.annotation.NonNull;
 import android.app.ActivityManager;
 import android.content.Context;
@@ -85,7 +86,7 @@ import android.util.Slog;
 import android.util.SparseBooleanArray;
 import android.util.SparseIntArray;
 import android.util.StatsLog;
-
+import android.provider.Settings;
 import com.android.internal.annotations.GuardedBy;
 import com.android.internal.annotations.VisibleForTesting;
 import com.android.internal.app.IBatteryStats;
@@ -1637,7 +1638,8 @@ public class NetworkManagementService extends INetworkManagementService.Stub {
 
     @Override
     public void setFirewallEnabled(boolean enabled) {
-        enforceSystemUid();
+        //enforceSystemUid();
         try {
             mNetdService.firewallSetFirewallType(
                     enabled ? INetd.FIREWALL_WHITELIST : INetd.FIREWALL_BLACKLIST);
@@ -1649,7 +1651,8 @@ public class NetworkManagementService extends INetworkManagementService.Stub {
 
     @Override
     public boolean isFirewallEnabled() {
-        enforceSystemUid();
+        //enforceSystemUid();
         return mFirewallEnabled;
     }
 
@@ -1664,7 +1667,95 @@ public class NetworkManagementService extends INetworkManagementService.Stub {
             throw new IllegalStateException(e);
         }
     }
-
+       private List<String> blackpackageNames;
+       private List<Integer> blackpackageNames_uid;
+       private ArrayList<String> urlexceptionblackpackageNames;
+       private ArrayList<Integer> urlexceptionblackpackageNames_uid;
+       @Override
+       public void setWhiteNetApp( List<String> packageNames){
+               PackageManager pm = mContext.getPackageManager();
+               List<String> packageNames_old= this.blackpackageNames;
+               this.blackpackageNames=packageNames;
+               this.blackpackageNames_uid=null;
+
+               if(packageNames_old!=null || packageNames_old.size()>0){
+                           for (String pkgName : packageNames_old) {
+                                        int uid = getUidFromPackageName(pm, pkgName);
+                if (uid > 0) {
+                    setFirewallUidRule( FIREWALL_CHAIN_DOZABLE, uid, 0);
+                }
+            }
+               }
+               if(packageNames==null || packageNames.size()==0){
+                       return ;
+               }
+               blackpackageNames_uid=  new ArrayList<>();
+            for (String pkgName : packageNames) {
+                int uid = getUidFromPackageName(pm, pkgName); 
+                               blackpackageNames_uid.add(uid);
+                if (uid > 0) {
+                    setFirewallUidRule( FIREWALL_CHAIN_DOZABLE, uid, 0); 
+                    setFirewallUidRule( FIREWALL_CHAIN_DOZABLE, uid, 1); 
+                }
+            }
+       }
+       private int getUidFromPackageName(PackageManager pm, String pkgName) {
+        try {
+            ApplicationInfo appInfo = pm.getApplicationInfo(pkgName, PackageManager.MATCH_ALL);
+            if (appInfo != null) {
+                return appInfo.uid;
+            }
+        } catch (PackageManager.NameNotFoundException e) {
+            e.printStackTrace();
+        }
+        return -1;
+    }
+       @Override
+       public List<String> getWhiteNetApp(){
+               return this.blackpackageNames;
+       }
@@ -1763,7 +1870,7 @@ public class NetworkManagementService extends INetworkManagementService.Stub {
 
     @Override
     public void setFirewallChainEnabled(int chain, boolean enable) {
-        enforceSystemUid();
+        //enforceSystemUid();
         synchronized (mQuotaLock) {
             synchronized (mRulesLock) {
                 if (getFirewallChainState(chain) == enable) {
@@ -1879,6 +1986,7 @@ public class NetworkManagementService extends INetworkManagementService.Stub {
     }
 
     private void setFirewallUidRuleLocked(int chain, int uid, int rule) {
         if (updateFirewallUidRuleLocked(chain, uid, rule)) {
             final int ruleType = getFirewallRuleType(chain, rule);
             try {

```

### 3.3 通过自定义服务接口设置白名单 app 列表

```
public void setWhiteNetApp(List<String> packageNames) throws RemoteException {
		INetworkManagementService networkService = INetworkManagementService.Stub.asInterface(ServiceManager.getService(Context.NETWORKMANAGEMENT_SERVICE));
		boolean firewallEnabled = networkService.isFirewallEnabled();
		if (packageNames==null || packageNames.size()==0){
			Settings.System.putString(mContext.getContentResolver(), "whitenetApp", "");
			try {
				networkService.setWhiteNetApp(packageNames);
			} catch (Exception e) {
				e.printStackTrace();
			}
			setNetFirwall(false);
			return;
		}else {
			setNetFirwall(true);
		}
		String s="";
		for (int i =0;i<packageNames.size();i++){
			String str = packageNames.get(i);
			if(i<packageNames.size()-1)s=s+str+",";
			else s=s+str;
		}
		Settings.System.putString(mContext.getContentResolver(), "whitenetApp", s);
		try {
			    networkService.setWhiteNetApp(packageNames);
			
		} catch (Exception e) {
			e.printStackTrace();
		}
}

public void setNetFirewall(boolean on) throws RemoteException {
        try {
             INetworkManagementService networkService = INetworkManagementService.Stub.asInterface(ServiceManager.getService(Context.NETWORKMANAGEMENT_SERVICE));
            networkService.setFirewallChainEnabled(1, on);
              boolean firewallEnabled = networkService.isFirewallEnabled();
         } catch (Exception e){
             e.printStackTrace();
         }
     }

```

通过上述相关代码 实现了设置上网应用 app 功能，