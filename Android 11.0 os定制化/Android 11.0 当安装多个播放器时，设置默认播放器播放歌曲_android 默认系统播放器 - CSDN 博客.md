> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127541150)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 当安装多个播放器时，设置默认播放器播放歌曲的核心类](#t1)

[3. 当安装多个播放器时，设置默认播放器播放歌曲的核心功能分析和实现](#t2)

[3.1 ResolverActivity.java 中关于弹出多个播放器的功能分析](#t3)

[3.2 AbstractMultiProfilePagerAdapter.java 相关代码分析](#t4)

[3.3 ResolverListAdapter.java 的相关 app 列表的代码分析](#t5)

1. 概述
-----

 在 11.0 的系统产品定制化开发中，在有些设备安装多个[视频播放器](https://so.csdn.net/so/search?q=%E8%A7%86%E9%A2%91%E6%92%AD%E6%94%BE%E5%99%A8&spm=1001.2101.3001.7020)的时候，这时如果点击一个视频播放，会弹出选择播放器播放的列表，让用户选择播放器播放，而产品需求要求当多个播放器的时候，设置默认播放器，具体分析弹出多个播放器的页面，然后解决问题

2. 当安装多个播放器时，设置默认播放器播放歌曲的核心类
----------------------------

```
/frameworks/base/core/java/com/android/internal/app/ResolverListAdapter.java
/frameworks/base/core/java/com/android/internal/app/ResolverActivity.java
 /frameworks/base/core/java/com/android/internal/app/AbstractMultiProfilePagerAdapter.java
```

3. 当安装多个播放器时，设置默认播放器播放歌曲的核心功能分析和实现
----------------------------------

### 3.1 ResolverActivity.java 中关于弹出多个播放器的功能分析

```
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

在 ResolverActivity.java 中的 onCreate() 方法中，初始化相关参数，然后查询同类型的 app 列表  
比如说，浏览器 播放器等等，然后弹出播放器列表让用户选择进入哪个播放器播放，  
具体是  
mMultiProfilePagerAdapter = createMultiProfilePagerAdapter(initialIntents, rList, filterLastUsed);  
          if (configureContentView()) {  
              return;  
          }  
这段代码来构建播放器适配器列表, 主要是在 configureContentView() 中处理的，接下来看下  
configureContentView() 的获取 app 列表的代码

```
    /**
       * Sets up the content view.
       * @return <code>true</code> if the activity is finishing and creation should halt.
       */
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

在 configureContentView() 的方法中，可以根据 rebuildCompleted 的值来判断是否获取到 app 列表，而  
它取决于 mMultiProfilePagerAdapter.rebuildActiveTab(true) 的值，所以需要看 mMultiProfilePagerAdapter.rebuildActiveTab(true)  
的相关代码

### 3.2 AbstractMultiProfilePagerAdapter.java 相关代码分析

```
 boolean rebuildActiveTab(boolean doPostProcessing) {
          return rebuildTab(getActiveListAdapter(), doPostProcessing);
      }
 private boolean rebuildTab(ResolverListAdapter activeListAdapter, boolean doPostProcessing) {
          if (shouldShowNoCrossProfileIntentsEmptyState(activeListAdapter)) {
              activeListAdapter.postListReadyRunnable(doPostProcessing, /* rebuildCompleted */ true);
              return false;
          }
          return activeListAdapter.rebuildList(doPostProcessing);
      }
```

  从上述的代码中可以看出在 rebuildActiveTab（）中调用的是 rebuildTab(）执行相关功能，而在 rebuildTab(）  
最终调用的是 activeListAdapter.rebuildList(doPostProcessing) 方法，所以需要分析 ResolverListAdapter.java  
的相关方法

### 3.3 ResolverListAdapter.java 的相关 app 列表的代码分析

```
 protected boolean rebuildList(boolean doPostProcessing) {
          List<ResolvedComponentInfo> currentResolveList = null;
          // Clear the value of mOtherProfile from previous call.
          mOtherProfile = null;
          mLastChosen = null;
          mLastChosenPosition = -1;
          mDisplayList.clear();
          mIsTabLoaded = false;
  
          if (mBaseResolveList != null) {
              currentResolveList = mUnfilteredResolveList = new ArrayList<>();
              mResolverListController.addResolveListDedupe(currentResolveList,
                      mResolverListCommunicator.getTargetIntent(),
                      mBaseResolveList);
          } else {
              currentResolveList = mUnfilteredResolveList =
                      mResolverListController.getResolversForIntent(
                              /* shouldGetResolvedFilter= */ true,
                              mResolverListCommunicator.shouldGetActivityMetadata(),
                              mIntents);
              if (currentResolveList == null) {
                  processSortedList(currentResolveList, doPostProcessing);
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
				List<ResolvedComponentInfo> defaultList= new ArrayList();
				for(ResolvedComponentInfo rci:currentResolveList){
					ResolveInfo resolveInfo = rci.getResolveInfoAt(0);
					ActivityInfo activityinfo = resolveInfo.activityInfo;
					String packagename = activityinfo.packageName;
					android.util.Log.e("ResolverActivity","currentResolveList--"+"---resolveInfo:"+resolveInfo+"--packagename:"+packagename+"--name:"+activityinfo.name);
					if(action_filter!=null&&action_filter.equals("android.intent.action.VIEW")){
                                                //com.android.music 即为设置为默认播放器包名
						if(packagename.equals("com.android.music")){
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
              ResolveInfo resolveInfo = info.getResolveInfoAt(0);
              if (resolveInfo.targetUserId != UserHandle.USER_CURRENT) {
                  Intent pOrigIntent = mResolverListCommunicator.getReplacementIntent(
                          resolveInfo.activityInfo,
                          info.getIntentAt(0));
                  Intent replacementIntent = mResolverListCommunicator.getReplacementIntent(
                          resolveInfo.activityInfo,
                          mResolverListCommunicator.getTargetIntent());
                  mOtherProfile = new DisplayResolveInfo(info.getIntentAt(0),
                          resolveInfo,
                          resolveInfo.loadLabel(mPm),
                          resolveInfo.loadLabel(mPm),
                          pOrigIntent != null ? pOrigIntent : replacementIntent,
                          makePresentationGetter(resolveInfo));
                  currentResolveList.remove(info);
                  break;
              }
          }
  
          if (mOtherProfile == null) {
              try {
                  mLastChosen = mResolverListController.getLastChosen();
              } catch (RemoteException re) {
                  Log.d(TAG, "Error calling getLastChosenActivity\n" + re);
              }
          }
  
          setPlaceholderCount(0);
          int n;
          if ((currentResolveList != null) && ((n = currentResolveList.size()) > 0)) {
              // We only care about fixing the unfilteredList if the current resolve list and
              // current resolve list are currently the same.
              List<ResolvedComponentInfo> originalList =
                      mResolverListController.filterLowPriority(currentResolveList,
                              mUnfilteredResolveList == currentResolveList);
              if (originalList != null) {
                  mUnfilteredResolveList = originalList;
              }
  
              if (currentResolveList.size() > 1) {
                  int placeholderCount = currentResolveList.size();
                  if (mResolverListCommunicator.useLayoutWithDefault()) {
                      --placeholderCount;
                  }
                  setPlaceholderCount(placeholderCount);
                  createSortingTask(doPostProcessing).execute(currentResolveList);
                  postListReadyRunnable(doPostProcessing, /* rebuildCompleted */ false);
                  return false;
              } else {
                  processSortedList(currentResolveList, doPostProcessing);
                  return true;
              }
          } else {
              processSortedList(currentResolveList, doPostProcessing);
              return true;
          }
      }
```

在 ResolverListAdapter.java 的 rebuildList 的方法中通过遍历当前 currentResolveList 的值, 来根据包名移除掉除默认播放器的 app 只保留默认播放器一个 app 的相关信息就可以了，这样就只有一个 app 了，所以就会默认选择这个 app 作为默认播放器了 就实现了功能