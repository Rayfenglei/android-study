> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889294)

### 1. 概述

在 11.0 12.0 定制化开发中，对于设置默认 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 功能，也是在产品开发中常有的功能，有客户需要提出进入 launcher 后，设置为默认 laucher，而在系统自定义服务中，增加设置系统默认 Launcher 的接口，调用设置 laucher 的方法后重启进入设置的 launcher 功能

### 2.Settings 中通过代码设置默认 laucher 动态切换的相关代码

```
frameworks\base\core\java\android\os\ILgyManager.aidl
frameworks\base\core\java\android\os\LgyManager.java
frameworks\base\services\core\java\com\android\server\lgy\LgyManagerService.java
frameworks/base/core/java/com/android/internal/app/ResolverActivity.java

```

### 3.Settings 中通过代码设置默认 laucher 动态切换核心功能分析

关于添加自定义服务 请到这里

[android 11.0 添加自定义系统服务接口给 app 调用](https://blog.csdn.net/baidu_41666295/article/details/124955049)  
解决思路:  
1. 在启动 laucher 后设置为默认 launcher  
2. 调用设置 launcher 接口，设置 launcher 后默认 launcher  
3. 调用 home 键返回到 launcher 桌面

### 3.1 增加设置默认 launcher 的方法

setDefaultLauncher(“com.[hc](https://so.csdn.net/so/search?q=hc&spm=1001.2101.3001.7020).musicplayer”); 设置 launcher 包名

```
    private List<ResolveInfo> getResolveInfoList() {
        PackageManager pm = mContext.getPackageManager();
        Intent intent = new Intent(Intent.ACTION_MAIN, null);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        return pm.queryIntentActivities(intent, 0);
    }

    private ResolveInfo getCurrentLauncher() {
        PackageManager pm = mContext.getPackageManager();
        Intent intent = new Intent(Intent.ACTION_MAIN, null);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        return pm.resolveActivity(intent, 0);
    }

    private void setDefaultLauncher(String packageName) {
        PackageManager pm = mContext.getPackageManager();
        ResolveInfo currentLauncher = getCurrentLauncher();

        List<ResolveInfo> packageInfos = getResolveInfoList();

        ResolveInfo futureLauncher = null;

        for (ResolveInfo ri : packageInfos) {
            if (!TextUtils.isEmpty(ri.activityInfo.packageName) && !TextUtils.isEmpty(packageName)
                    && TextUtils.equals(ri.activityInfo.packageName, packageName)) {
                futureLauncher = ri;
            }
        }
        if (futureLauncher == null) {
            return;
        }

        pm.clearPackagePreferredActivities(currentLauncher.activityInfo.packageName);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_MAIN);
        intentFilter.addCategory(Intent.CATEGORY_HOME);
        intentFilter.addCategory(Intent.CATEGORY_DEFAULT);
        ComponentName componentName = new ComponentName(futureLauncher.activityInfo.packageName,
                futureLauncher.activityInfo.name);
        ComponentName[] componentNames = new ComponentName[packageInfos.size()];
        int defaultMatch = 0;
        for (int i = 0; i < packageInfos.size(); i++) {
            ResolveInfo resolveInfo = packageInfos.get(i);
            componentNames[i] = new ComponentName(resolveInfo.activityInfo.packageName, resolveInfo.activityInfo.name);
            if (defaultMatch < resolveInfo.match) {
                defaultMatch = resolveInfo.match;
            }
        }
        pm.clearPackagePreferredActivities(currentLauncher.activityInfo.packageName);
        pm.addPreferredActivity(intentFilter, defaultMatch, componentNames, componentName);
    }
}

```

关于开机设置默认 Launcher 分析

```
@UiThread
  public class ResolverActivity extends Activity implements
          ResolverListAdapter.ResolverListCommunicator {
@Override
      protected void onCreate(Bundle savedInstanceState) {
          // Use a specialized prompt when we're handling the 'Home' app startActivity()
          final Intent intent = makeMyIntent();
          final Set<String> categories = intent.getCategories();
          if (Intent.ACTION_MAIN.equals(intent.getAction())
                  && categories != null
                  && categories.size() == 1
                  && categories.contains(Intent.CATEGORY_HOME)) {
              // Note: this field is not set to true in the compatibility version.
              mResolvingHome = true;
          }
  
          setSafeForwardingMode(true);
  
          onCreate(savedInstanceState, intent, null, 0, null, null, true);
      }
  
      /**
       * Compatibility version for other bundled services that use this overload without
       * a default title resource
       */
      @UnsupportedAppUsage
      protected void onCreate(Bundle savedInstanceState, Intent intent,
              CharSequence title, Intent[] initialIntents,
              List<ResolveInfo> rList, boolean supportsAlwaysUseOption) {
          onCreate(savedInstanceState, intent, title, 0, initialIntents, rList,
                  supportsAlwaysUseOption);
      }
  
      protected void onCreate(Bundle savedInstanceState, Intent intent,
              CharSequence title, int defaultTitleRes, Intent[] initialIntents,
              List<ResolveInfo> rList, boolean supportsAlwaysUseOption) {
          setTheme(appliedThemeResId());
          super.onCreate(savedInstanceState);
  
          // Determine whether we should show that intent is forwarded
          // from managed profile to owner or other way around.
          setProfileSwitchMessageId(intent.getContentUserHint());
  
          try {
              mLaunchedFromUid = ActivityTaskManager.getService().getLaunchedFromUid(
                      getActivityToken());
          } catch (RemoteException e) {
              mLaunchedFromUid = -1;
          }
  
          if (mLaunchedFromUid < 0 || UserHandle.isIsolated(mLaunchedFromUid)) {
              // Gulp!
              finish();
              return;
          }
  
          mPm = getPackageManager();
  
          mReferrerPackage = getReferrerPackageName();
  
          // Add our initial intent as the first item, regardless of what else has already been added.
          mIntents.add(0, new Intent(intent));
          mTitle = title;
          mDefaultTitleResId = defaultTitleRes;
  
          mSupportsAlwaysUseOption = supportsAlwaysUseOption;
          mWorkProfileUserHandle = fetchWorkProfileUserProfile();
  
          // The last argument of createResolverListAdapter is whether to do special handling
          // of the last used choice to highlight it in the list.  We need to always
          // turn this off when running under voice interaction, since it results in
          // a more complicated UI that the current voice interaction flow is not able
          // to handle. We also turn it off when the work tab is shown to simplify the UX.
          boolean filterLastUsed = mSupportsAlwaysUseOption && !isVoiceInteraction()
                  && !shouldShowTabs();
          mMultiProfilePagerAdapter = createMultiProfilePagerAdapter(initialIntents, rList, filterLastUsed);
          if (configureContentView()) {
              return;
          }
  
          mPersonalPackageMonitor = createPackageMonitor(
                  mMultiProfilePagerAdapter.getPersonalListAdapter());
          mPersonalPackageMonitor.register(
                  this, getMainLooper(), getPersonalProfileUserHandle(), false);
          if (shouldShowTabs()) {
              mWorkPackageMonitor = createPackageMonitor(
                      mMultiProfilePagerAdapter.getWorkListAdapter());
              mWorkPackageMonitor.register(this, getMainLooper(), getWorkProfileUserHandle(), false);
          }
  
          mRegistered = true;
  
          final ResolverDrawerLayout rdl = findViewById(R.id.contentPanel);
          if (rdl != null) {
              rdl.setOnDismissedListener(new ResolverDrawerLayout.OnDismissedListener() {
                  @Override
                  public void onDismissed() {
                      finish();
                  }
              });
  
              boolean hasTouchScreen = getPackageManager()
                      .hasSystemFeature(PackageManager.FEATURE_TOUCHSCREEN);
  
              if (isVoiceInteraction() || !hasTouchScreen) {
                  rdl.setCollapsed(false);
              }
  
              rdl.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                      | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
              rdl.setOnApplyWindowInsetsListener(this::onApplyWindowInsets);
  
              mResolverDrawerLayout = rdl;
          }
  
          mProfileView = findViewById(R.id.profile_button);
          if (mProfileView != null) {
              mProfileView.setOnClickListener(this::onProfileClick);
              updateProfileViewButton();
          }
  
          final Set<String> categories = intent.getCategories();
          MetricsLogger.action(this, mMultiProfilePagerAdapter.getActiveListAdapter().hasFilteredItem()
                  ? MetricsProto.MetricsEvent.ACTION_SHOW_APP_DISAMBIG_APP_FEATURED
                  : MetricsProto.MetricsEvent.ACTION_SHOW_APP_DISAMBIG_NONE_FEATURED,
                  intent.getAction() + ":" + intent.getType() + ":"
                          + (categories != null ? Arrays.toString(categories.toArray()) : ""));
      }

```

可以在 OnCreate 部分设置系统默认开机 Launcher 这里不做讲解了  
具体请看 [android 9.0 10.0 多个 launcher 启动指定 launcher](https://blog.csdn.net/baidu_41666295/article/details/117700388)

### 3.2 在自定义服务中调用设置默认 Launcher 接口

[aidl](https://so.csdn.net/so/search?q=aidl&spm=1001.2101.3001.7020) 添加接口

```
frameworks\base\core\java\android\os\ILgyManager.aidl

package android.os;
/** @hide */

interface ILgyManager
{
setDefaultLauncher(String packageName);
}


```

2.LgyManagerService.java 增加设置默认 launcher 的接口

```
package com.android.server.lgy;

import com.android.server.SystemService;
import android.content.Context;
import android.util.Log;
import java.util.HashMap;
import android.os.ILgyManager;


public final class LgyManagerService extends ILgyManager.Stub{

private static final String TAG = "LgyManagerService";
final Context mContext;
public LgyManagerService(Context context) {
mContext = context;

}
@Override
public  String getValue(){

try{
Log.d("lgy_bubug", "GetFromJni  ");
return "GetFromJni  ";
}catch(Exception e){
Log.d("lgy_bubug", "nativeReadPwd Exception msg = " + e.getMessage());
return " read nothings!!!";
}
}

}

@Override
public void setDefaultLauncher(String packageName){

try{
setDefaultLauncher("com.hc.musicplayer");
}catch(Exception e){
 e.printStackTrace();
}
}

}

```

这样就在自定义服务中调用设置默认 Launcher 的方法设置 Launcher 从而实现功能