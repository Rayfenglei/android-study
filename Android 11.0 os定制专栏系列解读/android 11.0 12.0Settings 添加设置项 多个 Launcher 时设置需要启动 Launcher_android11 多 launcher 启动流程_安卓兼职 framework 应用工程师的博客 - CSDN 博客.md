> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889336)

1. 概述
-----

在 11.0 12.0 产品开发中最近客户有需求，要求在系统设置 TvSettings 中，当多个 [launcher](https://so.csdn.net/so/search?q=launcher&spm=1001.2101.3001.7020) 时, 选择自己需要启动 launcher。这需要在增加自定义的 Fragment 然后通过选择 Launcher 设置为默认 Launcher

2.Settings 添加设置项 多个 Launcher 时设置需要启动 Launcher 相关代码
--------------------------------------------------

```
packages\apps\Settings\src\com\android\settings\DefaultLauncherFragment.java

```

3.Settings 添加设置项 多个 Launcher 时设置需要启动 Launcher 功能分析和实现
-----------------------------------------------------

实现思路:  
1. 获取系统中所有 launcher, 然后 RadioPreference 动态显示出来  
2. 设置所选 launcher 作为启动 launcher

具体实现案例:

```
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.os.Bundle;
import android.support.v7.preference.Preference;
import android.support.v7.preference.PreferenceScreen;
import android.text.TextUtils;
import android.util.Log;
import android.provider.Settings;
import com.android.internal.logging.nano.MetricsProto;
import com.android.tv.settings.R;
import com.android.tv.settings.RadioPreference;
import com.android.tv.settings.SettingsPreferenceFragment;

import java.util.Iterator;
import java.util.List;

/**
 * The location settings screen in TV settings.
 */
public class DefaultLauncherFragment extends SettingsPreferenceFragment {

    private static final String TAG = "DefaultLauncherFragment";

    private PackageManager packageManger;

    public static DefaultLauncherFragment newInstance() {
        return new DefaultLauncherFragment();
    }

    @Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        Log.e(TAG, "onCreatePreferences");
        final Context themedContext = getPreferenceManager().getContext();
        final PreferenceScreen screen = getPreferenceManager().createPreferenceScreen(themedContext);
        screen.setTitle(R.string.default_launcher);

        packageManger = getActivity().getPackageManager();
        PackageManager pm = packageManger;
        //获取系统中所有的launcher
        List<ResolveInfo> packageInfos = getResolveInfoList();
//当前launcher
        ResolveInfo currentRI = getCurrentLauncher();

        Iterator<ResolveInfo> it = packageInfos.iterator();
        while (it.hasNext()) {
            ResolveInfo ri = it.next();
            if (ri != null && ri.activityInfo.applicationInfo.isDirectBootAware()) {
                it.remove();
            }
        }

        if (packageInfos != null && packageInfos.size() > 0) {
            Preference activePref = null;
            //动态添加RadioPreference
            for (ResolveInfo resolveInfo : packageInfos) {
                final RadioPreference radioPreference = new RadioPreference(themedContext);
                radioPreference.setKey(resolveInfo.activityInfo.packageName);
                radioPreference.setPersistent(false);
                radioPreference.setTitle(resolveInfo.loadLabel(pm).toString());
                radioPreference.setIcon(resolveInfo.loadIcon(pm));
                radioPreference.setLayoutResource(R.layout.preference_reversed_widget);

                if (!TextUtils.isEmpty(currentRI.activityInfo.packageName)
                        && !TextUtils.isEmpty(resolveInfo.activityInfo.packageName)
                        && TextUtils.equals(currentRI.activityInfo.packageName, resolveInfo.activityInfo.packageName)) {
                    radioPreference.setChecked(true);
                    activePref = radioPreference;
                }

                screen.addPreference(radioPreference);
            }
           // 选中当前launcher
            if (activePref != null && savedInstanceState == null) {
                scrollToPreference(activePref);
            }
        }
        setPreferenceScreen(screen);
    }

    @Override
    public boolean onPreferenceTreeClick(Preference preference) {
        Log.e(TAG, "onPreferenceTreeClick");
        if (preference instanceof RadioPreference) {
            final RadioPreference radioPreference = (RadioPreference) preference;
            radioPreference.clearOtherRadioPreferences(getPreferenceScreen());
            if (radioPreference.isChecked()) {
                String packageName = radioPreference.getKey();
                setDefaultLauncher(packageName);
            } else {
                radioPreference.setChecked(true);
            }
        }
        return super.onPreferenceTreeClick(preference);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
    }

    @Override
    public int getMetricsCategory() {
        return MetricsProto.MetricsEvent.MANAGE_APPLICATIONS;
    }
}

```

在 onCreatePreferences 中通过布局一个 RadioPreference 然后查询当前 Launcher 有几个 添加到这个控件里面  
然后当选择 launcher 时通过获取包名 在 setDefaultLauncher(String packageName) 调用系统 api 设置为默认 Launcher

3.2 设置默认 Launcher 的相关代码

```
    private List<ResolveInfo> getResolveInfoList() {
        PackageManager pm = packageManger;
        Intent intent = new Intent(Intent.ACTION_MAIN, null);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        return pm.queryIntentActivities(intent, 0);
    }

    private ResolveInfo getCurrentLauncher() {
        PackageManager pm = packageManger;
        Intent intent = new Intent(Intent.ACTION_MAIN, null);
        intent.addCategory(Intent.CATEGORY_HOME);
        intent.addCategory(Intent.CATEGORY_DEFAULT);
        return pm.resolveActivity(intent, 0);
    }

    private void setDefaultLauncher(String packageName) {
        PackageManager pm = packageManger;
        ResolveInfo currentLauncher = getCurrentLauncher();

        List<ResolveInfo> packageInfos = getResolveInfoList();

        ResolveInfo futureLauncher = null;

        for (ResolveInfo ri : packageInfos) {
            if (!TextUtils.isEmpty(ri.activityInfo.packageName) && !TextUtils.isEmpty(packageName)
                    && TextUtils.equals(ri.activityInfo.packageName, packageName)) {
                futureLauncher = ri;
            }
        }
        Settings.Global.putString(getActivity().getContentResolver(),
                   "default_launcher_packagename",packageName);
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

```

通过 PackageManager 的 api 先清除原来 launcher 代码 pm.clearPackagePreferredActivities(currentLauncher.activityInfo.packageName);  
然后设置现在的默认 Launcher  
pm.addPreferredActivity(intentFilter, defaultMatch, componentNames, componentName);  
从而实现设置默认 Launcher 的功能