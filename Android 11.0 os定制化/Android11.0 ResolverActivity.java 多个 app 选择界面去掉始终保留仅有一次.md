> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828133)

### 1. 概述

11.0 定制化开发中，如果在播放音乐时，这时软件中有多个播放器时，点击音乐时，会弹出选择播放音乐 app 的界面 而这个界面  
就是 ResolverActivity.java 来实现处理的

### 2.ResolverActivity.java 多个 app 选择界面去掉始终保留仅有一次的核心类

frameworks/base/core/java/com/android/internal/app/ResolverActivity.java

### 3.ResolverActivity.java 多个 app 选择界面去掉始终保留仅有一次的核心功能实现和分析

经过分析得知在系统中处理多个同类型 app 选择进入哪个 app 的处理都是在  
ResolverActivity.java 的  
路径为: frameworks/base/core/java/com/android/internal/app/ResolverActivity.java

接下来看 ResolverActivity.java 的[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        ActivityDebugConfigs.addConfigChangedListener(mDebugConfigListener);

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

        final ResolverDrawerLayout rdl = findViewById(R.id.contentPanel);
        if (rdl != null) {
            rdl.setOnDismissedListener(new ResolverDrawerLayout.OnDismissedListener() {
                @Override
                public void onDismissed() {
                    finish();
                }
            });
            if (isVoiceInteraction()) {
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
            bindProfileView();
        }

        initSuspendedColorMatrix();

        if (isVoiceInteraction()) {
            onSetupVoiceInteraction();
        }
        final Set<String> categories = intent.getCategories();
        MetricsLogger.action(this, mAdapter.hasFilteredItem()
                ? MetricsProto.MetricsEvent.ACTION_SHOW_APP_DISAMBIG_APP_FEATURED
                : MetricsProto.MetricsEvent.ACTION_SHOW_APP_DISAMBIG_NONE_FEATURED,
                intent.getAction() + ":" + intent.getType() + ":"
                        + (categories != null ? Arrays.toString(categories.toArray()) : ""));
    }

```

从源码中可以看出 configureContentView(mIntents, initialIntents, rList) 中构建选择启动 app 的列表

接下来看 configureContentView(mIntents, initialIntents, rList)

```
public boolean configureContentView(List<Intent> payloadIntents, Intent[] initialIntents,
        List<ResolveInfo> rList) {
    // The last argument of createAdapter is whether to do special handling
    // of the last used choice to highlight it in the list.  We need to always
    // turn this off when running under voice interaction, since it results in
    // a more complicated UI that the current voice interaction flow is not able
    // to handle.
    mAdapter = createAdapter(this, payloadIntents, initialIntents, rList,
            mLaunchedFromUid, mSupportsAlwaysUseOption && !isVoiceInteraction());
    boolean rebuildCompleted = mAdapter.rebuildList();

    if (useLayoutWithDefault()) {
        mLayoutId = R.layout.resolver_list_with_default;
    } else {
        mLayoutId = getLayoutResource();
    }
    setContentView(mLayoutId);

    int count = mAdapter.getUnfilteredCount();
    // We only rebuild asynchronously when we have multiple elements to sort. In the case where
    // we're already done, we can check if we should auto-launch immediately.
    if (rebuildCompleted) {
        if (count == 1 && mAdapter.getOtherProfile() == null) {
            // Only one target, so we're a candidate to auto-launch!
            final TargetInfo target = mAdapter.targetInfoForPosition(0, false);
            if (shouldAutoLaunchSingleChoice(target)) {
                safelyStartActivity(target);
                mPackageMonitor.unregister();
                mRegistered = false;
                finish();
                return true;
            }
        }
    }


    mAdapterView = findViewById(R.id.resolver_list);
    if (count == 0 && mAdapter.mPlaceholderCount == 0) {
        final TextView emptyView = findViewById(R.id.empty);
        emptyView.setVisibility(View.VISIBLE);
        mAdapterView.setVisibility(View.GONE);
    } else {
        mAdapterView.setVisibility(View.VISIBLE);
        onPrepareAdapterView(mAdapterView, mAdapter);
    }
    return false;
}

public void onPrepareAdapterView(AbsListView adapterView, ResolveListAdapter adapter) {
    final boolean useHeader = adapter.hasFilteredItem();
    final ListView listView = adapterView instanceof ListView ? (ListView) adapterView : null;

    adapterView.setAdapter(mAdapter);

    final ItemClickListener listener = new ItemClickListener();
    adapterView.setOnItemClickListener(listener);
    adapterView.setOnItemLongClickListener(listener);

    if (mSupportsAlwaysUseOption || mUseLayoutForBrowsables) {
        listView.setChoiceMode(AbsListView.CHOICE_MODE_SINGLE);
    }

    // In case this method is called again (due to activity recreation), avoid adding a new
    // header if one is already present.
    if (useHeader && listView != null && listView.getHeaderViewsCount() == 0) {
        listView.addHeaderView(LayoutInflater.from(this).inflate(
                R.layout.resolver_different_item_header, listView, false));
    }
}

```

在代码中可以看出 useLayoutWithDefault() 来决定加载 R.layout.resolver_list_with_default 或 R.layout.resolver_list 哪个 layout  
接下来看下 useLayoutWithDefault()

```
  private boolean useLayoutWithDefault() {
        return mSupportsAlwaysUseOption && mAdapter.hasFilteredItem();
    }

```

而 mSupportsAlwaysUseOption 始终为 true  
接下来看 mAdapter.hasFilteredItem();

```
  public boolean hasFilteredItem() {
        return mFilterLastUsed && mLastChosen != null;
    }

```

hasFileteredItem() 由 mFilterLastUsed 决定  
只要它的值为 false 就可以  
接下来看 mFilterLastUsed 的值

查找代码发现

```
   private boolean mFilterLastUsed;

    public ResolveListAdapter(Context context, List<Intent> payloadIntents,
            Intent[] initialIntents, List<ResolveInfo> rList, int launchedFromUid,
            boolean filterLastUsed,
            ResolverListController resolverListController) {
        mIntents = payloadIntents;
        mInitialIntents = initialIntents;
        mBaseResolveList = rList;
        mLaunchedFromUid = launchedFromUid;
        mInflater = LayoutInflater.from(context);
        mDisplayList = new ArrayList<>();
        mFilterLastUsed = filterLastUsed;
        mResolverListController = resolverListController;
    }

```

mFilterLastUsed 的值 是由构造 ResoleListAdapter 传的值  
所以修改 为

```
 /**
     * Returns true if the activity is finishing and creation should halt
     */
    public boolean configureContentView(List<Intent> payloadIntents, Intent[] initialIntents,
            List<ResolveInfo> rList) {
        // The last argument of createAdapter is whether to do special handling
        // of the last used choice to highlight it in the list.  We need to always
        // turn this off when running under voice interaction, since it results in
        // a more complicated UI that the current voice interaction flow is not able
        // to handle.
        mAdapter = createAdapter(this, payloadIntents, initialIntents, rList,
             - mLaunchedFromUid, mSupportsAlwaysUseOption && !isVoiceInteraction());
             +   mLaunchedFromUid, false/*mSupportsAlwaysUseOption && !isVoiceInteraction()*/);
        boolean rebuildCompleted = mAdapter.rebuildList();

        if (useLayoutWithDefault()) {
            mLayoutId = R.layout.resolver_list_with_default;
        } else {
            mLayoutId = getLayoutResource();
        }
        setContentView(mLayoutId);

        int count = mAdapter.getUnfilteredCount();
        // We only rebuild asynchronously when we have multiple elements to sort. In the case where
        // we're already done, we can check if we should auto-launch immediately.
        if (rebuildCompleted) {
            if (count == 1 && mAdapter.getOtherProfile() == null) {
                // Only one target, so we're a candidate to auto-launch!
                final TargetInfo target = mAdapter.targetInfoForPosition(0, false);
                if (shouldAutoLaunchSingleChoice(target)) {
                    safelyStartActivity(target);
                    mPackageMonitor.unregister();
                    mRegistered = false;
                    finish();
                    return true;
                }
            }
        }


        mAdapterView = findViewById(R.id.resolver_list);
		//Log.e(TAG,"count:"+count+"---rebuildCompleted:"+rebuildCompleted+"---mPlaceholderCount:"+mAdapter.mPlaceholderCount);
        if (count == 0 && mAdapter.mPlaceholderCount == 0) {
            final TextView emptyView = findViewById(R.id.empty);
            emptyView.setVisibility(View.VISIBLE);
            mAdapterView.setVisibility(View.GONE);
        } else {
            mAdapterView.setVisibility(View.VISIBLE);
            onPrepareAdapterView(mAdapterView, mAdapter);
        }
        return false;
    }

```

而 resolver_list.xml 中隐藏始终

<?xml version="1.0" encoding="utf-8"?>

```
<com.android.internal.widget.ResolverDrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:maxWidth="@dimen/resolver_max_width"
    android:maxCollapsedHeight="192dp"
    android:maxCollapsedHeightSmall="56dp"
    android:id="@id/contentPanel">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alwaysShow="true"
        android:elevation="8dp"
        android:background="?attr/colorBackgroundFloating">

        <TextView
            android:id="@+id/profile_button"
            android:layout_width="wrap_content"
            android:layout_height="48dp"
            android:layout_marginEnd="8dp"
            android:paddingStart="8dp"
            android:paddingEnd="8dp"
            android:visibility="gone"
            style="?attr/borderlessButtonStyle"
            android:textAppearance="?attr/textAppearanceButton"
            android:textColor="?attr/colorAccent"
            android:gravity="center_vertical"
            android:layout_alignParentTop="true"
            android:layout_alignParentEnd="true"
            android:singleLine="true" />

        <TextView
            android:id="@+id/title"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:minHeight="56dp"
            android:textAppearance="?attr/textAppearanceMedium"
            android:gravity="start|center_vertical"
            android:paddingStart="?attr/dialogPreferredPadding"
            android:paddingEnd="?attr/dialogPreferredPadding"
            android:paddingTop="8dp"
            android:layout_below="@id/profile_button"
            android:layout_alignParentStart="true"
            android:paddingBottom="8dp" />
    </RelativeLayout>

    <ListView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:id="@+id/resolver_list"
        android:clipToPadding="false"
        android:scrollbarStyle="outsideOverlay"
        android:background="?attr/colorBackgroundFloating"
        android:elevation="8dp"
        android:nestedScrollingEnabled="true"
        android:scrollIndicators="top|bottom"
        android:divider="@null" />

    <TextView android:id="@+id/empty"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:background="?attr/colorBackgroundFloating"
              android:elevation="8dp"
              android:layout_alwaysShow="true"
              android:text="@string/noApplications"
              android:padding="32dp"
              android:gravity="center"
              android:visibility="gone" />

    <LinearLayout
        android:id="@+id/button_bar"
        android:visibility="gone"
        style="?attr/buttonBarStyle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_ignoreOffset="true"
        android:layout_alwaysShow="true"
        android:layout_hasNestedScrollIndicator="true"
        android:gravity="end|center_vertical"
        android:orientation="horizontal"
        android:layoutDirection="locale"
        android:measureWithLargestChild="true"
        android:background="?attr/colorBackgroundFloating"
        android:paddingTop="@dimen/resolver_button_bar_spacing"
        android:paddingBottom="@dimen/resolver_button_bar_spacing"
        android:paddingStart="12dp"
        android:paddingEnd="12dp"
        android:elevation="8dp">

        <Button
            android:id="@+id/button_once"
            android:layout_width="wrap_content"
            android:layout_gravity="center"
            android:maxLines="2"
            style="?attr/buttonBarNegativeButtonStyle"
            android:minHeight="@dimen/alert_dialog_button_bar_height"
            android:layout_height="wrap_content"
            android:enabled="false"
            android:text="@string/activity_resolver_use_once"
            android:onClick="onButtonClick" />

        <Button
            android:id="@+id/button_always"
            android:layout_width="wrap_content"
            android:layout_gravity="end"
            android:maxLines="2"
            android:minHeight="@dimen/alert_dialog_button_bar_height"
            style="?attr/buttonBarPositiveButtonStyle"
            android:layout_height="wrap_content"
           + android:visibility="gone"
            android:enabled="false"
            android:text="@string/activity_resolver_use_always"
            android:onClick="onButtonClick" />
    </LinearLayout>

</com.android.internal.widget.ResolverDrawerLayout>

```

修改完以后 编译 framework 然后替换 system 下的整个 framework 重启验证就可以了