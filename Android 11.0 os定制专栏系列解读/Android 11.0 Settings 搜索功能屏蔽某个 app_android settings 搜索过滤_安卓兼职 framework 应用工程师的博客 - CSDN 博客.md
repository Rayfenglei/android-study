> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124790108)

在 11.0 开发 Settings 中由于要屏蔽到某些 app 不让搜索出这个 app，所以就要从搜索流程中来去掉查询到这个 app, 而搜索流程都是在 SettingsIntelligence 中完成的

搜索流程:  
Settings 中点击搜索框, 跳转至 SettingsIntelligence 中的搜索页面, 即 SearchActivity

而 SearchActivity 又切换到了 SearchFragment.

2.SearchFragment 中, SearchFeatureProviderImpl 配合 loaderManager, 获取到数据库中的数据, 然后返回给 Adapter, 并绑定到 [RecycleView](https://so.csdn.net/so/search?q=RecycleView&spm=1001.2101.3001.7020) 中显示

3. 在 onBindViewHolder 时通过 onBind 实现对应点击事件的跳转

SearchFragment 创建时进行了一些对象创建, 如, 也包括数据的初始化, 如 mSavedQueryController,SearchFeatureProviderImpl 等等, mSearchFeatureProvider 调用 updateIndexAsync 开启数据库的初始化

```
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    long startTime = System.currentTimeMillis();
    setHasOptionsMenu(true);

    final LoaderManager loaderManager = getLoaderManager();
    mSearchAdapter = new SearchResultsAdapter(this /* fragment */);
    mSavedQueryController = new SavedQueryController(
            getContext(), loaderManager, mSearchAdapter);
    mSearchFeatureProvider.initFeedbackButton();

    if (savedInstanceState != null) {
        mQuery = savedInstanceState.getString(STATE_QUERY);
        mNeverEnteredQuery = savedInstanceState.getBoolean(STATE_NEVER_ENTERED_QUERY);
        mShowingSavedQuery = savedInstanceState.getBoolean(STATE_SHOWING_SAVED_QUERY);
    } else {
        mShowingSavedQuery = true;
    }
    mSearchFeatureProvider.updateIndexAsync(getContext(), this /* indexingCallback */);
    if (SearchFeatureProvider.DEBUG) {
        Log.d(TAG, "onCreate spent " + (System.currentTimeMillis() - startTime) + " ms");
    }
}

```

监听输入框, 通过 restartLoaders 调用 loaderManager 开启加载数据流程

```
@Override
    public boolean onQueryTextChange(String query) {
        if (TextUtils.equals(query, mQuery)) {
            return true;
        }
        mEnterQueryTimestampMs = System.currentTimeMillis();
        final boolean isEmptyQuery = TextUtils.isEmpty(query);

        // Hide no-results-view when the new query is not a super-string of the previous
        if (mQuery != null
                && mNoResultsView.getVisibility() == View.VISIBLE
                && query.length() < mQuery.length()) {
            mNoResultsView.setVisibility(View.GONE);
        }
        // Add for Bug#1113499: Limit the max length of query text
        query = getLimitedQueryText(query);
        mNeverEnteredQuery = false;
        mQuery = query;

        // If indexing is not finished, register the query text, but don't search.
        if (!mSearchFeatureProvider.isIndexingComplete(getActivity())) {
            return true;
        }

        if (isEmptyQuery) {
            final LoaderManager loaderManager = getLoaderManager();
            loaderManager.destroyLoader(SearchLoaderId.SEARCH_RESULT);
            mShowingSavedQuery = true;
            mSavedQueryController.loadSavedQueries();
            mSearchFeatureProvider.hideFeedbackButton(getView());
        } else {
            mMetricsFeatureProvider.logEvent(SettingsIntelligenceEvent.PERFORM_SEARCH);
            restartLoaders();
        }

        return true;
    }

  

loaderManager查询完数据后回调至onCreateLoader

@Override
    public Loader<List<? extends SearchResult>> onCreateLoader(int id, Bundle args) {
        final Activity activity = getActivity();

        switch (id) {
            case SearchLoaderId.SEARCH_RESULT:
                return mSearchFeatureProvider.getSearchResultLoader(activity, mQuery);
            default:
                return null;
        }
    }

src/com/android/settings/intelligence/search/SearchFeatureProvider.java中
调用getSearchResultLoader方法,得到SearchResultLoader对象,SearchResultLoader在子线程中进行数据查找loadInBackground
    @Override
    public SearchResultLoader getSearchResultLoader(Context context, String query) {
        return new SearchResultLoader(context, cleanQuery(query));
    }

    @Override
    public List<SearchQueryTask> getSearchQueryTasks(Context context, String query) {
        final List<SearchQueryTask> tasks = new ArrayList<>();
        final String cleanQuery = cleanQuery(query);
        tasks.add(DatabaseResultTask.newTask(context, getSiteMapManager(), cleanQuery));
        tasks.add(InstalledAppResultTask.newTask(context, getSiteMapManager(), cleanQuery));
        tasks.add(AccessibilityServiceResultTask.newTask(context, getSiteMapManager(), cleanQuery));
        tasks.add(InputDeviceResultTask.newTask(context, getSiteMapManager(), cleanQuery));
        return tasks;
    }

```

在 getSearchQueryTasks（）中通过 query 方法来查询数据  
SearchFeatureProviderImpl 中构建各类获取数据的 task, 包括 DatabaseResultTask,InstalledAppResultTask,AccessibilityServiceResultTask 等等.

且每个 task 都是继承自 SearchQueryTask, 而 SearchQueryTask 又是继承自 [FutureTask](https://so.csdn.net/so/search?q=FutureTask&spm=1001.2101.3001.7020), 因为 FutureTask 可以在线程运行结束后将结果返回. SearchQueryTask 内封装了 call 方法的回调, 调用抽象方法 query 完成数据的返回

在查询所以 app 中是通过 InstalledAppResultTask.java 的 query 来查询的 所以在 query 方法中屏蔽掉就好了

```
@Override
    protected List<? extends SearchResult> query() {
        final List<AppSearchResult> results = new ArrayList<>();

        List<ApplicationInfo> appsInfo = mPackageManager.getInstalledApplications(
                PackageManager.MATCH_DISABLED_COMPONENTS
                        | PackageManager.MATCH_DISABLED_UNTIL_USED_COMPONENTS
                        | PackageManager.MATCH_INSTANT);

        for (ApplicationInfo info : appsInfo) {
            if (!info.enabled
                    && mPackageManager.getApplicationEnabledSetting(info.packageName)
                    != PackageManager.COMPONENT_ENABLED_STATE_DISABLED_USER) {
                // Disabled by something other than user, skip.
                continue;
            }


           // add code start
            if (info.packageName!=null&&info.packageName.equals("包名")){
                   continue;
              }
          // add code end



            final CharSequence label = info.loadLabel(mPackageManager);
            final int wordDiff = SearchQueryUtils.getWordDifference(label.toString(), mQuery);
            if (wordDiff == SearchQueryUtils.NAME_NO_MATCH) {
                continue;
            }

          
            final Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
                    .setAction(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
                    .setData(
                            Uri.fromParts(INTENT_SCHEME, info.packageName, null /* fragment */))
                    .putExtra(DatabaseIndexingUtils.EXTRA_SOURCE_METRICS_CATEGORY,
                            DatabaseIndexingUtils.DASHBOARD_SEARCH_RESULTS);

            final AppSearchResult.Builder builder = new AppSearchResult.Builder();
            builder.setAppInfo(info)
                    .setDataKey(info.packageName)
                    .setTitle(info.loadLabel(mPackageManager))
                    .setRank(getRank(wordDiff))
                    .addBreadcrumbs(getBreadCrumb())
                    .setPayload(new ResultPayload(intent));
            results.add(builder.build());
        }

        Collections.sort(results);
        return results;
    }

```

所以根据 package 包名屏蔽掉就好了