> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893929)

### 1. 概述

在 11.0 定制化开发中，最近项目需要开发需求是屏蔽来电功能, 需要根据[标志位](https://so.csdn.net/so/search?q=%E6%A0%87%E5%BF%97%E4%BD%8D&spm=1001.2101.3001.7020) 屏蔽一切来电功能  
就是去掉通话功能，这就需要从通话流程进行分析，然后实现功能  
, 而我们知道所有的来电去掉都是 CallManager.java 来负责监听管理的。

### 2. 屏蔽所有电话来电功能的核心代码

```
packages\services\Telecomm\src\com\android\server\telecom\CallsManager.java
packages/services/Telecomm/src/com/android/server/telecom/callfiltering/IncomingCallFilter.java

```

### 3. 屏蔽所有电话来电功能核心部分分析和功能实现

### 3.1CallManager.java 关于通话管理的分析

在整个系统应用的通话部分都是由 CallManager 来管理的  
所以来看下 CallManager.java 源码

路径位于：packages\services\Telecomm\src\com\android\server\telecom\CallsManager.java

```
@VisibleForTesting
  public class CallsManager extends Call.ListenerBase
          implements VideoProviderProxy.Listener, CallFilterResultCallback, CurrentUserProxy {
  
      // TODO: Consider renaming this CallsManagerPlugin.
      @VisibleForTesting
      public interface CallsManagerListener {
          void onCallAdded(Call call);
          void onCallRemoved(Call call);
          void onCallStateChanged(Call call, int oldState, int newState);
          void onConnectionServiceChanged(
                  Call call,
                  ConnectionServiceWrapper oldService,
                  ConnectionServiceWrapper newService);
          void onIncomingCallAnswered(Call call);
          void onIncomingCallRejected(Call call, boolean rejectWithMessage, String textMessage);
          void onCallAudioStateChanged(CallAudioState oldAudioState, CallAudioState newAudioState);
          void onRingbackRequested(Call call, boolean ringback);
          void onIsConferencedChanged(Call call);
          void onIsVoipAudioModeChanged(Call call);
          void onVideoStateChanged(Call call, int previousVideoState, int newVideoState);
          void onCanAddCallChanged(boolean canAddCall);
          void onSessionModifyRequestReceived(Call call, VideoProfile videoProfile);
          void onHoldToneRequested(Call call);
          void onExternalCallChanged(Call call, boolean isExternalCall);
          void onDisconnectedTonePlaying(boolean isTonePlaying);
          void onConnectionTimeChanged(Call call);
          void onConferenceStateChanged(Call call, boolean isConference);
          void onCdmaConferenceSwap(Call call);
          void onSetCamera(Call call, String cameraId);
      }
 @Override
    public void onSuccessfulIncomingCall(Call incomingCall) {
        Log.d(this, "onSuccessfulIncomingCall");
        PhoneAccount phoneAccount = mPhoneAccountRegistrar.getPhoneAccountUnchecked(
                incomingCall.getTargetPhoneAccount());
        Bundle extras =
            phoneAccount == null || phoneAccount.getExtras() == null
                ? new Bundle()
                : phoneAccount.getExtras();
        if (incomingCall.hasProperty(Connection.PROPERTY_EMERGENCY_CALLBACK_MODE) ||
                incomingCall.isSelfManaged() ||
                extras.getBoolean(PhoneAccount.EXTRA_SKIP_CALL_FILTERING)) {
            Log.i(this, "Skipping call filtering for %s (ecm=%b, selfMgd=%b, skipExtra=%b)",
                    incomingCall.getId(),
                    incomingCall.hasProperty(Connection.PROPERTY_EMERGENCY_CALLBACK_MODE),
                    incomingCall.isSelfManaged(),
                    extras.getBoolean(PhoneAccount.EXTRA_SKIP_CALL_FILTERING));
            onCallFilteringComplete(incomingCall, new CallFilteringResult(true, false, true, true));
            incomingCall.setIsUsingCallFiltering(false);
            return;
        }
 
        incomingCall.setIsUsingCallFiltering(true);
        List<IncomingCallFilter.CallFilter> filters = new ArrayList<>();
        filters.add(new DirectToVoicemailCallFilter(mCallerInfoLookupHelper));
        filters.add(new AsyncBlockCheckFilter(mContext, new BlockCheckerAdapter(),
                mCallerInfoLookupHelper, null));
        filters.add(new CallScreeningServiceController(mContext, this, mPhoneAccountRegistrar,
                new ParcelableCallUtils.Converter(), mLock,
                new TelecomServiceImpl.SettingsSecureAdapterImpl(), mCallerInfoLookupHelper,
                new CallScreeningServiceHelper.AppLabelProxy() {
                    @Override
                    public CharSequence getAppLabel(String packageName) {
                        PackageManager pm = mContext.getPackageManager();
                        try {
                            ApplicationInfo info = pm.getApplicationInfo(packageName, 0);
                            return pm.getApplicationLabel(info);
                        } catch (PackageManager.NameNotFoundException nnfe) {
                            Log.w(this, "Could not determine package name.");
                        }
 
                        return null;
                    }
                }));
        new IncomingCallFilter(mContext, this, incomingCall, mLock,
                mTimeoutsAdapter, filters).performFiltering();
    }

```

通过读源码发现 当来电时 都会调用 onSuccessfulIncomingCall（）方法, 而 它, 都会通过 IncomingCallFilter 的 performFiltering 来进行过滤看是不是黑名单号码, 所有可以把标志位加在这里，实现屏蔽所有来电功能

### 3.2 IncomingCallFilter 关于过滤黑名单的相关方法分析

接下来看下的 IncomingCallFilter 的 performFiltering 方法看过滤号码的相关方法

```
public class IncomingCallFilter implements CallFilterResultCallback {
  
      public static class Factory {
          public IncomingCallFilter create(Context context, CallFilterResultCallback listener,
                  Call call, TelecomSystem.SyncRoot lock, Timeouts.Adapter timeoutsAdapter,
                  List<CallFilter> filters) {
              return new IncomingCallFilter(context, listener, call, lock, timeoutsAdapter, filters,
                      new Handler(Looper.getMainLooper()));
          }
      }
 public void performFiltering() {
        Log.addEvent(mCall, LogUtils.Events.FILTERING_INITIATED);
        for (CallFilter filter : mFilters) {
            filter.startFilterLookup(mCall, this);
        }
        // synchronized to prevent a race on mResult and to enter into Telecom.
        mHandler.postDelayed(new Runnable("ICF.pFTO", mTelecomLock) { // performFiltering time-out
            @Override
            public void loggedRun() {
                if (mIsPending) {
                    Log.i(IncomingCallFilter.this, "Call filtering has timed out.");
                    Log.addEvent(mCall, LogUtils.Events.FILTERING_TIMED_OUT);
                    mListener.onCallFilteringComplete(mCall, mResult);
                    mIsPending = false;
                }
            }
        }.prepare(), mTimeoutsAdapter.getCallScreeningTimeoutMillis(mContext.getContentResolver()));
    }

    public void onCallFilteringComplete(Call call, CallFilteringResult result) {
         synchronized (mTelecomLock) { // synchronizing to prevent race on mResult
             mNumPendingFilters--;
             mResult = result.combine(mResult);
             if (mNumPendingFilters == 0) {
                 // synchronized on mTelecomLock to enter into Telecom.
                 mHandler.post(new Runnable("ICF.oCFC", mTelecomLock) {
                     @Override
                     public void loggedRun() {
                         if (mIsPending) {
                             Log.addEvent(mCall, LogUtils.Events.FILTERING_COMPLETED, mResult);
                             mListener.onCallFilteringComplete(mCall, mResult);
                             mIsPending = false;
                         }
                     }
                 }.prepare());
             }
         }
     }

```

通过相关代码分析得知在通话过程中会通过调用 performFiltering() 看是否需要过滤黑名单，如果过滤，就不执行通话过程，所以可以在这里增加系统属性来控制是否需要过滤就实现了屏蔽来电功能  
实现方法如下:

```
a/packages/services/Telecomm/src/com/android/server/telecom/callfiltering/IncomingCallFilter.java
+++ b/packages/services/Telecomm/src/com/android/server/telecom/callfiltering/IncomingCallFilter.java
@@ -27,7 +27,7 @@ import com.android.server.telecom.Call;
 import com.android.server.telecom.LogUtils;
 import com.android.server.telecom.TelecomSystem;
 import com.android.server.telecom.Timeouts;
-
+import android.provider.Settings;
 import java.util.List;
 
 public class IncomingCallFilter implements CallFilterResultCallback {
@@ -67,6 +67,9 @@ public class IncomingCallFilter implements CallFilterResultCallback {
     }
 
     public void performFiltering() {
+        int roincoming_disable = Settings.Global.getInt(mContext.getContentResolver(), "roincoming_dia_disable", 0);
+        android.util.Log.e("SuperPower","roincoming_disable:"+roincoming_disable);
+               if(roincoming_disable==1)return;
         Log.addEvent(mCall, LogUtils.Events.FILTERING_INITIATED);
         for (CallFilter filter : mFilters) {
             filter.startFilterLookup(mCall, this);

```