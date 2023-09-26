> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/133230252)

1. 前言
-----

  在 11.0 的系统开发中，在做系统定制化开发中，在对系统的静态壁纸做定制的时候，需要增加几种静态壁纸可以让用户自己设置壁纸，所以可以在壁纸的系统应用中  
添加几种静态壁纸图片，然后配置好 就可以在选择壁纸的时候，作为静态壁纸，接下来看如何具体实现这个功能

2. 增加多张图片作为系统静态壁纸的功能实现的核心类
--------------------------

```
   packages\apps\WallpaperPicker\src\com\android\wallpaperpicker\WallpaperPickerActivity.java
    packages\apps\WallpaperPicker\res\values-nodpi\wallpapers.xml
```

3. 增加多张图片作为系统静态壁纸的功能实现的核心功能分析和实现
--------------------------------

  
在增加多张图片作为系统静态壁纸的功能实现的核心功能实现中，  
在系统中壁纸也是一个核心功能，包括默认壁纸的更换，设置静态壁纸[动态壁纸](https://so.csdn.net/so/search?q=%E5%8A%A8%E6%80%81%E5%A3%81%E7%BA%B8&spm=1001.2101.3001.7020)等等，这些都是 WallpaperManagerService 来负责管理的，二 WallpaperManager 类是一个系统服务（System Service），  
其实现位于 frameworks/base/core/java/[android](https://so.csdn.net/so/search?q=android&spm=1001.2101.3001.7020)/app/WallpaperManager.java 文件中。该类继承自 ContextWrapper 类，实现了 IWallpaperManager 接口，成员变量包括壁纸服务的代理（IWallpaperManager 对象）和上下文对象  
而在系统源码中，WallpaperPicker 就是壁纸选择器 Android 壁纸选择器的一个端口，用于启动器本身。而在 WallpaperPickerActivity.java 中就是具体负责加载默认壁纸的，所以接下来分析下  
WallpaperPickerActivity.java 的相关源码，分析下相关设置默认静态壁纸的相关功能

3.1 WallpaperPickerActivity.java 中相关加载静态壁纸相关源码分析
------------------------------------------------

  
在增加多张图片作为系统静态壁纸的功能实现的核心功能实现中，在壁纸选择器默认静态壁纸的相关源码中，在这个 app 中在管理壁纸类中  
WallpaperPickerActivity.java 的相关源码中，在这个类中负责加载静态壁纸，然后在静态壁纸的列表中，接下来分析下这个壁纸加载相关功能

```
   public class WallpaperPickerActivity extends WallpaperCropActivity
            implements OnClickListener, OnLongClickListener, ActionMode.Callback {
        static final String TAG = "WallpaperPickerActivity";
     
        protected void init() {
            setContentView(R.layout.wallpaper_picker);
     
            mCropView = (CropView) findViewById(R.id.cropView);
            mCropView.setVisibility(View.INVISIBLE);
     
            mProgressView = findViewById(R.id.loading);
            mWallpaperScrollContainer = (HorizontalScrollView) findViewById(R.id.wallpaper_scroll_container);
            mWallpaperStrip = findViewById(R.id.wallpaper_strip);
            mCropView.setTouchCallback(new ToggleOnTapCallback(mWallpaperStrip));
     
            mWallpaperParallaxOffset = getIntent().getFloatExtra(
                    WallpaperUtils.EXTRA_WALLPAPER_OFFSET, 0);
     
            mWallpapersView = (LinearLayout) findViewById(R.id.wallpaper_list);
            // Populate the saved wallpapers
            mSavedImages = new SavedWallpaperImages(this);
            populateWallpapers(mWallpapersView, mSavedImages.loadThumbnailsAndImageIdList(), true);
     
            // Populate the built-in wallpapers
            ArrayList<WallpaperTileInfo> wallpapers = findBundledWallpapers();
            populateWallpapers(mWallpapersView, wallpapers, false);
     
            // Load live wallpapers asynchronously
            new LiveWallpaperInfo.LoaderTask(this) {
     
                @Override
                protected void onPostExecute(List<LiveWallpaperInfo> result) {
                    populateWallpapers((LinearLayout) findViewById(R.id.live_wallpaper_list),
                            result, false);
                    initializeScrollForRtl();
                    updateTileIndices();
                }
            }.execute();
     
            // Add a tile for the Gallery
            LinearLayout masterWallpaperList = (LinearLayout) findViewById(R.id.master_wallpaper_list);
            masterWallpaperList.addView(
                    createTileView(masterWallpaperList, new PickImageInfo(), false), 0);
     
            // Select the first item; wait for a layout pass so that we initialize the dimensions of
            // cropView or the defaultWallpaperView first
            mCropView.addOnLayoutChangeListener(new OnLayoutChangeListener() {
                @Override
                public void onLayoutChange(View v, int left, int top, int right, int bottom,
                        int oldLeft, int oldTop, int oldRight, int oldBottom) {
                    if ((right - left) > 0 && (bottom - top) > 0) {
                        if (mSelectedIndex >= 0 && mSelectedIndex < mWallpapersView.getChildCount()) {
                            onClick(mWallpapersView.getChildAt(mSelectedIndex));
                            setSystemWallpaperVisiblity(false);
                        }
                        v.removeOnLayoutChangeListener(this);
                    }
                }
            });
     
            updateTileIndices();
     
            // Update the scroll for RTL
            initializeScrollForRtl();
    ......
    }
     
        public ArrayList<WallpaperTileInfo> findBundledWallpapers() {
            final ArrayList<WallpaperTileInfo> bundled = new ArrayList<WallpaperTileInfo>(24);
            Pair<ApplicationInfo, Integer> r = getWallpaperArrayResourceId();
            if (r != null) {
                try {
                    Resources wallpaperRes = getPackageManager().getResourcesForApplication(r.first);
                    addWallpapers(bundled, wallpaperRes, r.first.packageName, r.second);
                } catch (PackageManager.NameNotFoundException e) {
                }
            }
     
            // Add an entry for the default wallpaper (stored in system resources)
            WallpaperTileInfo defaultWallpaperInfo = DefaultWallpaperInfo.get(this);
            if (defaultWallpaperInfo != null) {
                bundled.add(0, defaultWallpaperInfo);
            }
            return bundled;
        }
     
        public Pair<ApplicationInfo, Integer> getWallpaperArrayResourceId() {
            return new Pair<>(getApplicationInfo(), R.array.wallpapers);
        }
```

在增加多张图片作为系统静态壁纸的功能实现的核心功能实现中，在壁纸选择器默认静态壁纸的相关源码中，  
在 WallpaperPickerActivity 的上述源码中分析得知，在 init() 中来负责加载系统的动态壁纸和静态壁纸，所以  
可以在 ArrayList<WallpaperTileInfo> wallpapers = findBundledWallpapers(); 中来负责查询系统中的  
静态壁纸，而在 findBundledWallpapers() 中，主要是调用 getWallpaperArrayResourceId() 来构建  
系统壁纸的集合，就是从 R.array.wallpapers 中获取静态壁纸资源集合，获取到静态资源添加到集合中，然后  
在静态壁纸列表中显示处理，接下来具体分析相关功能

3.2 wallpapers.xml 中关于 wallpapers 中静态壁纸的相关配置
--------------------------------------------

  
在增加多张图片作为系统静态壁纸的功能实现的核心功能实现中，在壁纸选择器默认静态壁纸的相关源码中，  
在通过上述的分析后，得知需要在 WallpaperPicker 这个 app 的资源配置文件中的 R.array.wallpapers 中获取静态壁纸资源集合来增加相关的静态图片的资源，这样来添加  
静态图片作为静态壁纸的资源来让客户可以设置自己喜欢的静态壁纸，接下来就来具体分析下  
R.array.wallpapers 中的相关静态壁纸资源来添加壁纸图片

```
    <resources>
        <string-array >
        +    <item>wallpaper_00</item>
        +        <item>wallpaper_01</item>
        +        <item>wallpaper_02</item>
         +       <item>wallpaper_03</item>
        +        <item>wallpaper_04</item>
         +       <item>wallpaper_05</item>
         +       <item>wallpaper_06</item>
         +       <item>wallpaper_07</item>
        +        <item>wallpaper_08</item>
         +       <item>wallpaper_09</item>
         +       <item>wallpaper_10</item>
        </string-array>
    </resources>
```

在增加多张图片作为系统静态壁纸的功能实现的核心功能实现中，在壁纸选择器默认静态壁纸的相关源码中，  
从上述的 wallpapers.xml 中的相关源码中分析得知，  
在上述的 wallpapers.xml 中的 wallpapers 的 string-array 的资源配置中，  
添加静态壁纸的名称作为 item，然后在 res/drawable-nodpi 目录下  
添加 wallpaper_00.jpg，wallpaper_00_small.jpg 等一组静态图标作为  
系统静态壁纸，通过这样添加对应的静态壁纸的图片，按照顺序添加，这样就实现了系统中添加静态壁纸的功能