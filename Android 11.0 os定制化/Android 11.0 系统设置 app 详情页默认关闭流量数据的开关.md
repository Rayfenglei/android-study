> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127501958)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 系统设置 app 详情页默认关闭流量数据的开关的核心类](#t1)

[3. 系统设置 app 详情页默认关闭流量数据的开关的核心功能分析和实现](#t2)

 [3.1 AppDataUsage.java 关于流量开关代码分析](#t3)

[3.2 DataSaverBackend 中相关开启关闭流量的源码分析](#t4)

[3.3 NetworkPolicyManager.java 来设置流量开启关闭功能分析](#t5)

[3.4 NetworkPolicyManagerService.java 关于流量开关的相关代码](#t6)

1. 概述
-----

 在 11.0 的系统产品开发中，对于 app 上网流量的管理也是很重要的，有些产品而言对于流量消耗也是在意成本的所以产品需求需要默认关闭 app 数据流量开关，这就要从 app 详情页来分析 app 的流量相关情况

2. 系统设置 app 详情页默认关闭流量数据的开关的核心类
------------------------------

```
   packages/apps/Settings/src/com/android/settings/datausage/AppDataUsage.java
   packages/apps/Settings/src/com/android/settings/datausage/DataSaverBackend.java
   frameworks/base/core/java/android/net/NetworkPolicyManager.java
   frameworks/base/services/core/java/com/android/server/net/NetworkPolicyManagerService.java
```

3. 系统设置 app 详情页默认关闭流量数据的开关的核心功能分析和实现
------------------------------------

 在系统设置中，app 详情页关于处理流量开关的类就是 AppDataUsage.java  
接下来看下 AppDataUsage.java 的相关功能实现

###   3.1 AppDataUsage.java 关于流量开关代码分析

```
   public class AppDataUsage extends DataUsageBaseFragment implements OnPreferenceChangeListener,
         DataSaverBackend.Listener {
      @Override
      public boolean onPreferenceChange(Preference preference, Object newValue) {
          if (preference == mRestrictBackground) {
              mDataSaverBackend.setIsBlacklisted(mAppItem.key, mPackageName, !(Boolean) newValue);
              updatePrefs();
              return true;
          } else if (preference == mUnrestrictedData) {
              mDataSaverBackend.setIsWhitelisted(mAppItem.key, mPackageName, (Boolean) newValue);
              return true;
          }
          return false;
      }
  
      @Override
      public boolean onPreferenceTreeClick(Preference preference) {
          if (preference == mAppSettings) {
              // TODO: target towards entire UID instead of just first package
              getActivity().startActivityAsUser(mAppSettingsIntent, new UserHandle(
                      UserHandle.getUserId(mAppItem.key)));
              return true;
          }
          return super.onPreferenceTreeClick(preference);
      }
...
}
```

通过在 AppDataUsage 中的 onPreferenceChange 方法开启关闭流量开关的方法中可以看到  
mDataSaverBackend.setIsBlacklisted 和 mDataSaverBackend.setIsWhitelisted 来设置开启和  
关闭某个 app 的流量 接下来就来分析下 DataSaverBackend 的相关源码来看怎么开启和关闭流量的

### 3.2 DataSaverBackend 中相关开启关闭流量的源码分析

```
 public class DataSaverBackend {
  
      private static final String TAG = "DataSaverBackend";
  
      private final Context mContext;
      private final MetricsFeatureProvider mMetricsFeatureProvider;
  
      private final NetworkPolicyManager mPolicyManager;
      private final ArrayList<Listener> mListeners = new ArrayList<>();
      private SparseIntArray mUidPolicies = new SparseIntArray();
      private boolean mWhitelistInitialized;
      private boolean mBlacklistInitialized;
  
      // TODO: Staticize into only one.
      public DataSaverBackend(Context context) {
          mContext = context;
          mMetricsFeatureProvider = FeatureFactory.getFactory(context).getMetricsFeatureProvider();
          mPolicyManager = NetworkPolicyManager.from(context);
      }
      public void setIsWhitelisted(int uid, String packageName, boolean whitelisted) {
         final int policy = whitelisted ? POLICY_ALLOW_METERED_BACKGROUND : POLICY_NONE;
         mPolicyManager.setUidPolicy(uid, policy);
         mUidPolicies.put(uid, policy);
         if (whitelisted) {
             mMetricsFeatureProvider.action(
                     mContext, SettingsEnums.ACTION_DATA_SAVER_WHITELIST, packageName);
         }
     }
 
     public boolean isWhitelisted(int uid) {
         loadWhitelist();
         return mUidPolicies.get(uid, POLICY_NONE) == POLICY_ALLOW_METERED_BACKGROUND;
     }
 
     private void loadWhitelist() {
          if (mWhitelistInitialized) {
              return;
          }
  
          for (int uid : mPolicyManager.getUidsWithPolicy(POLICY_ALLOW_METERED_BACKGROUND)) {
              mUidPolicies.put(uid, POLICY_ALLOW_METERED_BACKGROUND);
          }
          mWhitelistInitialized = true;
      }
  
      public void refreshBlacklist() {
          loadBlacklist();
      }
  
      public void setIsBlacklisted(int uid, String packageName, boolean blacklisted) {
          final int policy = blacklisted ? POLICY_REJECT_METERED_BACKGROUND : POLICY_NONE;
          mPolicyManager.setUidPolicy(uid, policy);
          mUidPolicies.put(uid, policy);
          if (blacklisted) {
              mMetricsFeatureProvider.action(
                      mContext, SettingsEnums.ACTION_DATA_SAVER_BLACKLIST, packageName);
          }
      }
  
      public boolean isBlacklisted(int uid) {
          loadBlacklist();
          return mUidPolicies.get(uid, POLICY_NONE) == POLICY_REJECT_METERED_BACKGROUND;
      }
```

在 DataSaverBackend 的 setIsBlacklisted 和 setIsWhitelisted 方法可以看出在具体是否开启流量和关闭流量  
都是通过 policy 值来调用 NetworkPolicyManager.java 的 setUidPolicy 来实现控制的，所以  
需要具体来看 NetworkPolicyManager.java 来设置流量开启关闭的方法

### 3.3 NetworkPolicyManager.java 来设置流量开启关闭功能分析

```
      @UnsupportedAppUsage
      public void setUidPolicy(int uid, int policy) {
          try {
              mService.setUidPolicy(uid, policy);
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
```

### **3.4 NetworkPolicyManagerService.java 关于流量开关的相关代码**

```
         @Override
      public void setUidPolicy(int uid, int policy) {
          mContext.enforceCallingOrSelfPermission(MANAGE_NETWORK_POLICY, TAG);
  
          if (!UserHandle.isApp(uid)) {
              throw new IllegalArgumentException("cannot apply policy to UID " + uid);
          }
          synchronized (mUidRulesFirstLock) {
              final long token = Binder.clearCallingIdentity();
              try {
                  final int oldPolicy = mUidPolicy.get(uid, POLICY_NONE);
                  if (oldPolicy != policy) {
                      setUidPolicyUncheckedUL(uid, oldPolicy, policy, true);
                      mLogger.uidPolicyChanged(uid, oldPolicy, policy);
                  }
              } finally {
                  Binder.restoreCallingIdentity(token);
              }
          }
      }
     @GuardedBy("mUidRulesFirstLock")
      private void setUidPolicyUncheckedUL(int uid, int policy, boolean persist) {
          if (policy == POLICY_NONE) {
              mUidPolicy.delete(uid);
          } else {
              mUidPolicy.put(uid, policy);
          }
  
          // uid policy changed, recompute rules and persist policy.
          updateRulesForDataUsageRestrictionsUL(uid);
          if (persist) {
              synchronized (mNetworkPoliciesSecondLock) {
                  writePolicyAL();
              }
          }
      }
     private vate void updateRulesForDataUsageRestrictionsUL(int uid) {
          if (Trace.isTagEnabled(Trace.TRACE_TAG_NETWORK)) {
              Trace.traceBegin(Trace.TRACE_TAG_NETWORK,
                      "updateRulesForDataUsageRestrictionsUL: " + uid);
          }
          try {
              updateRulesForDataUsageRestrictionsULInner(uid);
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_NETWORK);
          }
      }
  
      private void updateRulesForDataUsageRestrictionsULInner(int uid) {
          if (!isUidValidForWhitelistRulesUL(uid)) {
              if (LOGD) Slog.d(TAG, "no need to update restrict data rules for uid " + uid);
              return;
          }
  
          final int uidPolicy = mUidPolicy.get(uid, POLICY_NONE);
          final int oldUidRules = mUidRules.get(uid, RULE_NONE);
          final boolean isForeground = isUidForegroundOnRestrictBackgroundUL(uid);
          final boolean isRestrictedByAdmin = isRestrictedByAdminUL(uid);
  
          final boolean isBlacklisted = (uidPolicy & POLICY_REJECT_METERED_BACKGROUND) != 0;
          final boolean isWhitelisted = (uidPolicy & POLICY_ALLOW_METERED_BACKGROUND) != 0;
          final int oldRule = oldUidRules & MASK_METERED_NETWORKS;
          int newRule = RULE_NONE;
  
          // First step: define the new rule based on user restrictions and foreground state.
          if (isRestrictedByAdmin) {
              newRule = RULE_REJECT_METERED;
          } else if (isForeground) {
              if (isBlacklisted || (mRestrictBackground && !isWhitelisted)) {
                  newRule = RULE_TEMPORARY_ALLOW_METERED;
              } else if (isWhitelisted) {
                  newRule = RULE_ALLOW_METERED;
              }
          } else {
              if (isBlacklisted) {
                  newRule = RULE_REJECT_METERED;
              } else if (mRestrictBackground && isWhitelisted) {
                  newRule = RULE_ALLOW_METERED;
              }
          }
          final int newUidRules = newRule | (oldUidRules & MASK_ALL_NETWORKS);
  
          if (LOGV) {
              Log.v(TAG, "updateRuleForRestrictBackgroundUL(" + uid + ")"
                      + ": isForeground=" +isForeground
                      + ", isBlacklisted=" + isBlacklisted
                      + ", isWhitelisted=" + isWhitelisted
                      + ", isRestrictedByAdmin=" + isRestrictedByAdmin
                      + ", oldRule=" + uidRulesToString(oldRule)
                      + ", newRule=" + uidRulesToString(newRule)
                      + ", newUidRules=" + uidRulesToString(newUidRules)
                      + ", oldUidRules=" + uidRulesToString(oldUidRules));
          }
  
          if (newUidRules == RULE_NONE) {
              mUidRules.delete(uid);
          } else {
              mUidRules.put(uid, newUidRules);
          }
  
          // Second step: apply bw changes based on change of state.
          if (newRule != oldRule) {
              if (hasRule(newRule, RULE_TEMPORARY_ALLOW_METERED)) {
                  // Temporarily whitelist foreground app, removing from blacklist if necessary
                  // (since bw_penalty_box prevails over bw_happy_box).
  
                  setMeteredNetworkWhitelist(uid, true);
                  // TODO: if statement below is used to avoid an unnecessary call to netd / iptables,
                  // but ideally it should be just:
                  //    setMeteredNetworkBlacklist(uid, isBlacklisted);
                  if (isBlacklisted) {
                      setMeteredNetworkBlacklist(uid, false);
                  }
              } else if (hasRule(oldRule, RULE_TEMPORARY_ALLOW_METERED)) {
                  // Remove temporary whitelist from app that is not on foreground anymore.
  
                  // TODO: if statements below are used to avoid unnecessary calls to netd / iptables,
                  // but ideally they should be just:
                  //    setMeteredNetworkWhitelist(uid, isWhitelisted);
                  //    setMeteredNetworkBlacklist(uid, isBlacklisted);
                  if (!isWhitelisted) {
                      setMeteredNetworkWhitelist(uid, false);
                  }
                  if (isBlacklisted || isRestrictedByAdmin) {
                      setMeteredNetworkBlacklist(uid, true);
                  }
              } else if (hasRule(newRule, RULE_REJECT_METERED)
                      || hasRule(oldRule, RULE_REJECT_METERED)) {
                  // Flip state because app was explicitly added or removed to blacklist.
                  setMeteredNetworkBlacklist(uid, (isBlacklisted || isRestrictedByAdmin));
                  if (hasRule(oldRule, RULE_REJECT_METERED) && isWhitelisted) {
                      // Since blacklist prevails over whitelist, we need to handle the special case
                      // where app is whitelisted and blacklisted at the same time (although such
                      // scenario should be blocked by the UI), then blacklist is removed.
                      setMeteredNetworkWhitelist(uid, isWhitelisted);
                  }
              } else if (hasRule(newRule, RULE_ALLOW_METERED)
                      || hasRule(oldRule, RULE_ALLOW_METERED)) {
                  // Flip state because app was explicitly added or removed to whitelist.
                  setMeteredNetworkWhitelist(uid, isWhitelisted);
              } else {
                  // All scenarios should have been covered above.
                  Log.wtf(TAG, "Unexpected change of metered UID state for " + uid
                          + ": foreground=" + isForeground
                          + ", whitelisted=" + isWhitelisted
                          + ", blacklisted=" + isBlacklisted
                          + ", isRestrictedByAdmin=" + isRestrictedByAdmin
                          + ", newRule=" + uidRulesToString(newUidRules)
                          + ", oldRule=" + uidRulesToString(oldUidRules));
              }
  
              // Dispatch changed rule to existing listeners.
              mHandler.obtainMessage(MSG_RULES_CHANGED, uid, newUidRules).sendToTarget();
          }
      }
```

通过上述 NetworkPolicyManagerService.java 的分析发现在 setUidPolicy 中调用 setUidPolicyUncheckedUL，而  
它继续调用 setUidPolicyUncheckedUL，而它调用 updateRulesForDataUsageRestrictionsUL 最后调用  
updateRulesForDataUsageRestrictionsULInner 来实现设置流量开关功能，经过分析  
得出结论就是根据是不是前台应用, 以及是否要加入黑名单这两个点来更新当前策略组, 其中 RULE_REJECT_METERED 策略表示不允许访问流量.  
除了设置 RULE_REJECT_METERED 这个状态外, 还需将 App 加入到黑名单中, 通过调用函数 setMeteredNetworkBlacklist(uid, true); 来实现, 如果要从黑名单中移除, 则调用 setMeteredNetworkBlacklist(uid, false); 即可.  
要想完整控制某个 APP 是否能使用流量, 我们只需控制 isForeground 和 isBlacklisted 这两个布尔变量的值

具体修改为:

```
1. packages/apps/Settings/src/com/android/settings/datausage/DataSaverBackend.java
    public void setIsBlacklisted(int uid, String packageName, boolean blacklisted) {
       - final int policy = blacklisted ? POLICY_REJECT_METERED_BACKGROUND : POLICY_NONE;
       + final int policy = blacklisted ? RULE_REJECT_METERED: POLICY_NONE;
        mPolicyManager.setUidPolicy(uid, policy);
        mUidPolicies.put(uid, policy);
        if (blacklisted) {
            mMetricsFeatureProvider.action(
                    mContext, SettingsEnums.ACTION_DATA_SAVER_BLACKLIST, packageName);
        }
    }
 
2.private void updateRulesForDataUsageRestrictionsULInner(int uid) {
          if (!isUidValidForWhitelistRulesUL(uid)) {
              if (LOGD) Slog.d(TAG, "no need to update restrict data rules for uid " + uid);
              return;
          }
  
          final int uidPolicy = mUidPolicy.get(uid, POLICY_NONE);
          final int oldUidRules = mUidRules.get(uid, RULE_NONE);
          final boolean isForeground = isUidForegroundOnRestrictBackgroundUL(uid);
          final boolean isRestrictedByAdmin = isRestrictedByAdminUL(uid);
  
          final boolean isBlacklisted = (uidPolicy & POLICY_REJECT_METERED_BACKGROUND) != 0;
          final boolean isWhitelisted = (uidPolicy & POLICY_ALLOW_METERED_BACKGROUND) != 0;
          final int oldRule = oldUidRules & MASK_METERED_NETWORKS;
          int newRule = RULE_NONE;
  
          // First step: define the new rule based on user restrictions and foreground state.
          if (isRestrictedByAdmin) {
              newRule = RULE_REJECT_METERED;
          } else if (isForeground) {
              if (isBlacklisted || (mRestrictBackground && !isWhitelisted)) {
                  newRule = RULE_TEMPORARY_ALLOW_METERED;
              } else if (isWhitelisted) {
                  newRule = RULE_ALLOW_METERED;
              }
          } else {
              if (isBlacklisted) {
                  newRule = RULE_REJECT_METERED;
              } else if (mRestrictBackground && isWhitelisted) {
                  newRule = RULE_ALLOW_METERED;
              }
          }
+        if ((uidPolicy & RULE_REJECT_METERED) != 0) {
+            newRule = RULE_REJECT_METERED;
+            isBlacklisted = true;
+            isWhitelisted = false;
+        }
 
        final int newUidRules = newRule | (oldUidRules & MASK_ALL_NETWORKS);
.....
}
```