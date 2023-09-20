> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125140749)

1. 概述
-----

在 11.0 定制化开发中，在系统原生设置中主页的搜索框是要求去掉的，不需要[搜索功能](https://so.csdn.net/so/search?q=%E6%90%9C%E7%B4%A2%E5%8A%9F%E8%83%BD&spm=1001.2101.3001.7020)，所以首选看下布局文件  
看下搜索框是哪个布局，然后隐藏到布局，达到实现功能的目的

2. 系统 Settings 主页去掉搜索框的主要代码
---------------------------

```
 /packages/apps/Settings/src/com/android/settings/homepage/SettingsHomepageActivity.java
 packages\apps\Settings\AndroidManifest.xml

```

3. 系统 Settings 主页去掉搜索框的主要代码分析和功能实现
----------------------------------

3.1 AndroidManifest.xml 中获取首页代码
-------------------------------

```
      <activity android:
                   android:label="@string/settings_label_launcher"
                   android:theme="@style/Theme.Settings.Home"
                   android:taskAffinity="com.android.settings.root"
                   android:launchMode="singleTask"
                   android:configChanges="keyboard|keyboardHidden">
             <intent-filter android:priority="1">
                 <action android: />
                 <category android: />
             </intent-filter>
             <meta-data android:
                        android:value="true" />
         </activity>

```

从上述代码可以看出就是 Settings 启动主页面的核心页面，  
通过代码发现是在 SettingsHomepageActivity.java 中进行处理的

3.2SettingsHomepageActivity.java 中关于搜索框的分析
------------------------------------------

路径为: /packages/apps/Settings/src/com/android/settings/homepage/SettingsHomepageActivity.java  
接下来看下相关代码

```
public class SettingsHomepageActivity extends FragmentActivity {

@Override
protected void onCreate(Bundle savedInstanceState) {
super.onCreate(savedInstanceState);

setContentView(R.layout.settings_homepage_container);
final View root = findViewById(R.id.settings_homepage_container);
root.setSystemUiVisibility(
View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

setHomepageContainerPaddingTop();

final Toolbar toolbar = findViewById(R.id.search_action_bar);
FeatureFactory.getFactory(this).getSearchFeatureProvider()
initSearchToolbar(this /* activity */, toolbar, SettingsEnums.SETTINGS_HOMEPAGE);

final ImageView avatarView = findViewById(R.id.account_avatar);
getLifecycle().addObserver(new AvatarViewMixin(this, avatarView));
getLifecycle().addObserver(new HideNonSystemOverlayMixin(this));

if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
// Only allow contextual feature on high ram devices.
showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
}
showFragment(new TopLevelSettings(), R.id.main_content);
((FrameLayout) findViewById(R.id.main_content))
getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
}

private void showFragment(Fragment fragment, int id) {
final FragmentManager fragmentManager = getSupportFragmentManager();
final FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
final Fragment showFragment = fragmentManager.findFragmentById(id);

if (showFragment == null) {
fragmentTransaction.add(id, fragment);
} else {
fragmentTransaction.show(showFragment);
}
fragmentTransaction.commit();
}

@VisibleForTesting
void setHomepageContainerPaddingTop() {
final View view = this.findViewById(R.id.homepage_container);

final int searchBarHeight = getResources().getDimensionPixelSize(R.dimen.search_bar_height);
final int searchBarMargin = getResources().getDimensionPixelSize(R.dimen.search_bar_margin);

// The top padding is the height of action bar(48dp) + top/bottom margins(16dp)
final int paddingTop = searchBarHeight + searchBarMargin * 2;
view.setPadding(0 /* left */, paddingTop, 0 /* right */, 0 /* bottom */);

// Prevent inner RecyclerView gets focus and invokes scrolling.
view.setFocusableInTouchMode(true);
view.requestFocus();
}
}

```

在 OnCreate 加载代码的过程中  
通过 settings_homepage_container.xml 发现 search_bar 就是顶部搜索框  
需要在 onCreate() 隐藏掉就行了  
在 setHomepageContainerPaddingTop() 设置置顶高度时去掉 searchBarHeight 就可以了  
所以具体修改如下:

```
--- a/packages/apps/Settings/src/com/android/settings/homepage/SettingsHomepageActivity.java
+++ b/packages/apps/Settings/src/com/android/settings/homepage/SettingsHomepageActivity.java
@@ -40,9 +40,9 @@ import com.android.settings.accounts.AvatarViewMixin;
 import com.android.settings.core.HideNonSystemOverlayMixin;
 import com.android.settings.homepage.contextualcards.ContextualCardsFragment;
 import com.android.settings.overlay.FeatureFactory;
-
+import android.provider.Settings;
 public class SettingsHomepageActivity extends FragmentActivity {

     @Override
     protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
@@ -51,7 +51,7 @@ public class SettingsHomepageActivity extends FragmentActivity {
         final View root = findViewById(R.id.settings_homepage_container);
         root.setSystemUiVisibility(
                 View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

         setHomepageContainerPaddingTop();
 
         final Toolbar toolbar = findViewById(R.id.search_action_bar);
@@ -64,11 +64,13 @@ public class SettingsHomepageActivity extends FragmentActivity {
 
         if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
             // Only allow contextual feature on high ram devices.
-            showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
+            //showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
         }
         showFragment(new TopLevelSettings(), R.id.main_content);
         ((FrameLayout) findViewById(R.id.main_content))
                 .getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
          //隐藏搜索框       
+        findViewById(R.id.search_bar).setVisibility( View.GONE);
     }
 
     private void showFragment(Fragment fragment, int id) {
@@ -93,7 +95,8 @@ public class SettingsHomepageActivity extends FragmentActivity {
 
         // The top padding is the height of action bar(48dp) + top/bottom margins(16dp)
         final int paddingTop = searchBarHeight + searchBarMargin * 2;
-        view.setPadding(0 /* left */, paddingTop, 0 /* right */, 0 /* bottom */);
-          //设置置顶距离
+        view.setPadding(0 /* left */, searchBarMargin, 0 /* right */, 0 /* bottom */);
     }

```

编译验证发现主页搜索框去掉了实现了功能