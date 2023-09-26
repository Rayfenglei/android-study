> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132792525)

1. 概述
-----

在 11.0 定制化开发中，由于[产品开发](https://so.csdn.net/so/search?q=%E4%BA%A7%E5%93%81%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)需要要求系统内置两个 Launcher, 一个是 Launcher3, 一个是自己开发的 Launcher, 当系统启动 Launcher 时，  
不要弹出 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 选择列表 选择哪个 Launcher 要求默认选择自己开发的 Launcher 作为默认 Launcher，关于选择 Launcher 列表  
其实都是在 ResolverActivity.java 中处理的具体看下代码分析解决问题，从而实现  
当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能

2. 当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的核心代码
---------------------------------------------------------------------

```
frameworks\base\core\java\com\android\internal\app\ResolverActivity.java
frameworks/base/core/java/com/android/internal/app/ResolverListAdapter.java
```

3. 当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能分析
---------------------------------------------------------------------

在实现当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能时，  
在 framework 中，关于系统内置多个同类型的 app 时，在[系统启动](https://so.csdn.net/so/search?q=%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8&spm=1001.2101.3001.7020)时，主要是在 ResolverActivity.java 来弹出选择启动列表，让用户选择启动  
ResolverActivity.java 中相关代码分析，在这个类里面主要是让用户选择启动哪个同类型的 app,

此类中有一个内部类 ResolveListAdapter 该类继承自 BaseAdapter，该类是 Home app 选择界面的数据适配器。  
ResolveListAdapter 会在 ResolverActivity 的 onCreate 方法中被初始化并会传入一个 ResolveInfo 类型的 List，ResolveListAdapter 根据会传入的 List 初始化一个 List mList ，用户的点击事件都会在 ResolveListAdapter 获取数据。  
用户点击”ALWAYS” 的事件发生在 ResolverActivity 的 onButtonClick 方法中，此方法会获取选中的 Item 的 position、或者获取用户上一次启动的 Home app 的，mAlwaysUseOption 代表用户选中的是否为历史选择，并调用 startSelected, 作为下次启动的默认 app，所以需要看下如何设置默认 Launcher

3.1ResolverActivity.java 中关于构建 Launcher 的列表相关源码分析
-------------------------------------------------

```
      protected void onCreate(Bundle savedInstanceState, Intent intent,
                    CharSequence title, int defaultTitleRes, Intent[] initialIntents,
                    List<ResolveInfo> rList, boolean supportsAlwaysUseOption) {
                setTheme(R.style.Theme_DeviceDefault_Resolver);
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
         
                mPackageMonitor.register(this, getMainLooper(), false);
                mRegistered = true;
                mReferrerPackage = getReferrerPackageName();
         
                final ActivityManager am = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
                mIconDpi = am.getLauncherLargeIconDensity();
         
                // Add our initial intent as the first item, regardless of what else has already been added.
                mIntents.add(0, new Intent(intent));
                mTitle = title;
                mDefaultTitleResId = defaultTitleRes;
         
                mUseLayoutForBrowsables = getTargetIntent() == null
                        ? false
                        : isHttpSchemeAndViewAction(getTargetIntent());
         
                mSupportsAlwaysUseOption = supportsAlwaysUseOption;
         
                if (configureContentView(mIntents, initialIntents, rList)) {
                    return;
                }
         
               ....
            }
         
         
     private boolean configureContentView() {
          if (mMultiProfilePagerAdapter.getActiveListAdapter() == null) {
              throw new IllegalStateException("mMultiProfilePagerAdapter.getCurrentListAdapter() "
                      + "cannot be null.");
          }
          // We partially rebuild the inactive adapter to determine if we should auto launch
          // isTabLoaded will be true here if the empty state screen is shown instead of the list.
          boolean rebuildCompleted = mMultiProfilePagerAdapter.rebuildActiveTab(true)
                  || mMultiProfilePagerAdapter.getActiveListAdapter().isTabLoaded();
          if (shouldShowTabs()) {
              boolean rebuildInactiveCompleted = mMultiProfilePagerAdapter.rebuildInactiveTab(false)
                      || mMultiProfilePagerAdapter.getInactiveListAdapter().isTabLoaded();
              rebuildCompleted = rebuildCompleted && rebuildInactiveCompleted;
          }
  
          if (useLayoutWithDefault()) {
              mLayoutId = R.layout.resolver_list_with_default;
          } else {
              mLayoutId = getLayoutResource();
          }
          setContentView(mLayoutId);
          mMultiProfilePagerAdapter.setupViewPager(findViewById(R.id.profile_pager));
          return postRebuildList(rebuildCompleted);
      }
```

  在实现当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能时，在上述 ResolverActivity.java  
中的上述源码中分析得知，在系统发现有两个 Launcher 的时候，在启动 Launcher 的列表的时候，会在 configureContentView()  
中会启动 Launcher 列表让用户选择启动哪个作为默认 Launcher, 这时候根据用户的选择，然后保存在系统的数据中，作为系统  
当前默认的 Launcher, 接下来分析下相关的 Launcher 列表的构建过程，接下来看下 ResolverListAdapter.java 中关于  
rebuildList() 的相关源码分析

 3.2 ResolverListAdapter.java 中关于 Launcher 列表构建的相关源码分析
------------------------------------------------------

在实现当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能时，在上述 ResolverActivity.java 的分析得知，  
在 ResolverListAdapter.java 中负责构建 Launcher 列表的适配器，在这里就来具体分析下关于构建列表的核心方法  
rebuildList()，看下是如何具体分析构建的

```
              protected boolean rebuildList() {
                    List<ResolvedComponentInfo> currentResolveList = null;
                    // Clear the value of mOtherProfile from previous call.
                    mOtherProfile = null;
                    mLastChosen = null;
                    mLastChosenPosition = -1;
                    mAllTargetsAreBrowsers = false;
                    mDisplayList.clear();
                    if (mBaseResolveList != null) {
                        currentResolveList = mUnfilteredResolveList = new ArrayList<>();
                        mResolverListController.addResolveListDedupe(currentResolveList,
                                getTargetIntent(),
                                mBaseResolveList);
                    } else {
                        currentResolveList = mUnfilteredResolveList =
                                mResolverListController.getResolversForIntent(shouldGetResolvedFilter(),
                                        shouldGetActivityMetadata(),
                                        mIntents);
                        if (currentResolveList == null) {
                            processSortedList(currentResolveList);
                            return true;
                        }
                        List<ResolvedComponentInfo> originalList =
                                mResolverListController.filterIneligibleActivities(currentResolveList,
                                        true);
                        if (originalList != null) {
                            mUnfilteredResolveList = originalList;
                        }
                    }
         
                    // So far we only support a single other profile at a time.
                    // The first one we see gets special treatment.
                    for (ResolvedComponentInfo info : currentResolveList) {
                        if (info.getResolveInfoAt(0).targetUserId != UserHandle.USER_CURRENT) {
                            mOtherProfile = new DisplayResolveInfo(info.getIntentAt(0),
                                    info.getResolveInfoAt(0),
                                    info.getResolveInfoAt(0).loadLabel(mPm),
                                    info.getResolveInfoAt(0).loadLabel(mPm),
                                    getReplacementIntent(info.getResolveInfoAt(0).activityInfo,
                                            info.getIntentAt(0)));
                            currentResolveList.remove(info);
                            break;
                        }
                    }
         
                    ....
                }
```

  在实现当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能时，  
在上述的 ResolverActivity.java 中的相关源码中，在 onCreate(Bundle savedInstanceState, Intent intent,  
                CharSequence title, int defaultTitleRes, Intent[] initialIntents,  
                List<ResolveInfo> rList, boolean supportsAlwaysUseOption) 中的  
configureContentView(mIntents, initialIntents, rList) 负责构建多个 Launcher 列表  
源码中的  
mAdapter = createAdapter(this, payloadIntents, initialIntents, rList,  
                mLaunchedFromUid, false/*mSupportsAlwaysUseOption && !isVoiceInteraction()*/);  
        boolean rebuildCompleted = mAdapter.rebuildList();  
负责查询相关的 Launcher 然后展示为列表 让用户选择  
当 rList 列表为空时 mAdapter.rebuildList(); 重新构建 Launcher 列表  
currentResolveList 就是查询到的 Launcher 列表 其实解决思路就是在这个列表保留默认 Launcher 就可以了  
具体修改如下:

```
      protected boolean rebuildList() {
                    List<ResolvedComponentInfo> currentResolveList = null;
                    // Clear the value of mOtherProfile from previous call.
                    mOtherProfile = null;
                    mLastChosen = null;
                    mLastChosenPosition = -1;
                    mAllTargetsAreBrowsers = false;
                    mDisplayList.clear();
                    if (mBaseResolveList != null) {
                        currentResolveList = mUnfilteredResolveList = new ArrayList<>();
                        mResolverListController.addResolveListDedupe(currentResolveList,
                                getTargetIntent(),
                                mBaseResolveList);
                    } else {
                        currentResolveList = mUnfilteredResolveList =
                                mResolverListController.getResolversForIntent(shouldGetResolvedFilter(),
                                        shouldGetActivityMetadata(),
                                        mIntents);
                        if (currentResolveList == null) {
                            processSortedList(currentResolveList);
                            return true;
                        }
                        List<ResolvedComponentInfo> originalList =
                                mResolverListController.filterIneligibleActivities(currentResolveList,
                                        true);
                        if (originalList != null) {
                            mUnfilteredResolveList = originalList;
                        }
                    }
         
      // add core start 
    	        if(currentResolveList!=null){
    				String action_filter = mIntents.get(0).getAction();
    				String type = mIntents.get(0).getType();
    				List<ResolvedComponentInfo> defaultList= new ArrayList();
    				for(ResolvedComponentInfo rci:currentResolveList){
    					ResolveInfo resolveInfo = rci.getResolveInfoAt(0);
    					ActivityInfo activityinfo = resolveInfo.activityInfo;
    					String packagename = activityinfo.packageName;
    					android.util.Log.e("ResolverActivity","type:"+type+"--packagename:"+packagename+"--intent:"+mIntents.get(0));
    					if(action_filter!=null&&action_filter.equals("android.intent.action.MAIN")){
    						if(currentResolveList.size()>=2&&!packagename.equals("com.android.launcher3")){
    							defaultList.add(rci);
    						}
    					}
    				}
    				if(defaultList.size()>0){
    					currentResolveList.clear();
    					currentResolveList.addAll(defaultList);
    				}
    			}
    // add core end
         
                    // So far we only support a single other profile at a time.
                    // The first one we see gets special treatment.
                    for (ResolvedComponentInfo info : currentResolveList) {
                        if (info.getResolveInfoAt(0).targetUserId != UserHandle.USER_CURRENT) {
                            mOtherProfile = new DisplayResolveInfo(info.getIntentAt(0),
                                    info.getResolveInfoAt(0),
                                    info.getResolveInfoAt(0).loadLabel(mPm),
                                    info.getResolveInfoAt(0).loadLabel(mPm),
                                    getReplacementIntent(info.getResolveInfoAt(0).activityInfo,
                                            info.getIntentAt(0)));
                            currentResolveList.remove(info);
                            break;
                        }
                    }
    .....
```

在实现当系统内置两个 Launcher 时默认设置 Launcher3 以外的那个 Launcher 为默认 Launcher 的功能时，  
在上述的 rebuildList() 中来通过 currentResolveList 存储当前的两个 Launcher, 所以在这里可以遍历  
currentResolveList 的这个集合，然后通过获取当前集合的每个成员，在根据 action 的 android.intent.action.MAIN  
属性来判断是 launcher，然后就可以把另外一个 launcher 重新填充到 currentResolveList 中，这样就  
过滤掉了 Launcher3, 就可以默认把自己的 Launcher 设置为默认 launcher 了，就实现了功能