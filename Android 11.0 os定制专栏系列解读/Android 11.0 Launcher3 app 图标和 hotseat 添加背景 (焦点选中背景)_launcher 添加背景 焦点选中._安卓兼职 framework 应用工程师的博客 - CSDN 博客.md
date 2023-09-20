> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872495)

1. 概述
-----

在 11.0 进行 Tv 设备定制化开发中，由于需要使用遥控器来移动来控制点击功能，所以需要给 app 的 Icon 和 hotseat 添加背景来显示选中状态, 所以要分析在布局中添加背景就可以了

2.app 图标和 hotseat 添加背景 (焦点选中背景) 的核心代码
-------------------------------------

```
packages/apps/Launcher3/src/com/android/launcher3/ShortcutAndWidgetContainer.java

```

3.app 图标和 hotseat 添加背景 (焦点选中背景) 的功能分析和实现
----------------------------------------

功能分析  
由于需求要求给 app 和 hotseat 的 item 都要添加背景，所以就需要找到 Item 添加默认背景  
在 Launcher3 中布局是由 Workspace 构造的 每一个页面由一个 CellLayout 组成, CellLayout 还不是真正容纳图标的 ViewGroup，每个 CellLayout 会包含一个 ShortcutAndWidgetContainer，这才是真正容纳图标和 Widget 的 ViewGroup. 所以给 ShortcutAndWidgetContainer 添加默认背景就可以了。

接下来具体看 ShortcutAndWidgetContainer.java 的源码分析问题

3.1ShortcutAndWidgetContainer 关于背景的代码分析
---------------------------------------

路径: /packages/apps/Launcher3/src/com/android/launcher3/ShortcutAndWidgetContainer.java

```
public class ShortcutAndWidgetContainer extends ViewGroup {
    static final String TAG = "ShortcutAndWidgetContainer";

    // These are temporary variables to prevent having to allocate a new object just to
    // return an (x, y) value from helper functions. Do NOT use them to maintain other state.
    private final int[] mTmpCellXY = new int[2];

    @ContainerType private final int mContainerType;
    private final WallpaperManager mWallpaperManager;

    private int mCellWidth;
    private int mCellHeight;

    private int mCountX;

    private ActivityContext mActivity;
    private boolean mInvertIfRtl = false;

    public ShortcutAndWidgetContainer(Context context, @ContainerType int containerType) {
        super(context);
        mActivity = ActivityContext.lookupContext(context);
        mWallpaperManager = WallpaperManager.getInstance(context);
        mContainerType = containerType;
    }

    public void setCellDimensions(int cellWidth, int cellHeight, int countX, int countY) {
        mCellWidth = cellWidth;
        mCellHeight = cellHeight;
        mCountX = countX;
    }

    public View getChildAt(int x, int y) {
        final int count = getChildCount();
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            CellLayout.LayoutParams lp = (CellLayout.LayoutParams) child.getLayoutParams();

            if ((lp.cellX <= x) && (x < lp.cellX + lp.cellHSpan) &&
                    (lp.cellY <= y) && (y < lp.cellY + lp.cellVSpan)) {
                return child;
            }
        }
        return null;
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecSize =  MeasureSpec.getSize(heightMeasureSpec);
        setMeasuredDimension(widthSpecSize, heightSpecSize);

        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                measureChild(child);
            }
        }
    }

    public void setupLp(View child) {
        CellLayout.LayoutParams lp = (CellLayout.LayoutParams) child.getLayoutParams();
        if (child instanceof LauncherAppWidgetHostView) {
            DeviceProfile profile = mActivity.getWallpaperDeviceProfile();
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX,
                    profile.appWidgetScale.x, profile.appWidgetScale.y);
        } else {
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX);
        }
    }

    // Set whether or not to invert the layout horizontally if the layout is in RTL mode.
    public void setInvertIfRtl(boolean invert) {
        mInvertIfRtl = invert;
    }

    public int getCellContentHeight() {
        return Math.min(getMeasuredHeight(),
                mActivity.getWallpaperDeviceProfile().getCellHeight(mContainerType));
    }

    public void measureChild(View child) {
        CellLayout.LayoutParams lp = (CellLayout.LayoutParams) child.getLayoutParams();
        final DeviceProfile profile = mActivity.getWallpaperDeviceProfile();

        if (child instanceof LauncherAppWidgetHostView) {
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX,
                    profile.appWidgetScale.x, profile.appWidgetScale.y);
            // Widgets have their own padding
        } else {
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX);
            // Center the icon/folder
            int cHeight = getCellContentHeight();
            int cellPaddingY = (int) Math.max(0, ((lp.height - cHeight) / 2f));
            int cellPaddingX = mContainerType == CellLayout.WORKSPACE
                    ? profile.workspaceCellPaddingXPx
                    : (int) (profile.edgeMarginPx / 2f);
            child.setPadding(cellPaddingX, cellPaddingY, cellPaddingX, 0);
        }
        int childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(lp.width, MeasureSpec.EXACTLY);
        int childheightMeasureSpec = MeasureSpec.makeMeasureSpec(lp.height, MeasureSpec.EXACTLY);
        child.measure(childWidthMeasureSpec, childheightMeasureSpec);
    }

    public boolean invertLayoutHorizontally() {
        return mInvertIfRtl && Utilities.isRtl(getResources());
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int count = getChildCount();
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                CellLayout.LayoutParams lp = (CellLayout.LayoutParams) child.getLayoutParams();

                if (child instanceof LauncherAppWidgetHostView) {
                    LauncherAppWidgetHostView lahv = (LauncherAppWidgetHostView) child;

                    // Scale and center the widget to fit within its cells.
                    DeviceProfile profile = mActivity.getDeviceProfile();
                    float scaleX = profile.appWidgetScale.x;
                    float scaleY = profile.appWidgetScale.y;

                    lahv.setScaleToFit(Math.min(scaleX, scaleY));
                    lahv.setTranslationForCentering(-(lp.width - (lp.width * scaleX)) / 2.0f,
                            -(lp.height - (lp.height * scaleY)) / 2.0f);
                }

                int childLeft = lp.x;
                int childTop = lp.y;
                child.layout(childLeft, childTop, childLeft + lp.width, childTop + lp.height);

                if (lp.dropped) {
                    lp.dropped = false;

                    final int[] cellXY = mTmpCellXY;
                    getLocationOnScreen(cellXY);
                    mWallpaperManager.sendWallpaperCommand(getWindowToken(),
                            WallpaperManager.COMMAND_DROP,
                            cellXY[0] + childLeft + lp.width / 2,
                            cellXY[1] + childTop + lp.height / 2, 0, null);
                }
            }
        }
    }

....
}

```

在 ShortcutAndWidgetContainer 是 viewgroup 的子类，通过自定义 view 来实现布局  
从源码中可以看出 onMeasure(int widthMeasureSpec, int heightMeasureSpec) 和 onLayout(boolean changed, int l, int t, int r, int b) 负责绘制子 Item

所以要想给 app 和 hotseat 添加背景 就得在这里添加就好了

所以修改如下:  
在 measureChild(View child) 中添加背景

```
public void measureChild(View child) {
        CellLayout.LayoutParams lp = (CellLayout.LayoutParams) child.getLayoutParams();
        final DeviceProfile profile = mActivity.getWallpaperDeviceProfile();

        if (child instanceof LauncherAppWidgetHostView) {
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX,
                    profile.appWidgetScale.x, profile.appWidgetScale.y);
            // Widgets have their own padding
        } else {
            lp.setup(mCellWidth, mCellHeight, invertLayoutHorizontally(), mCountX);
            // Center the icon/folder
            int cHeight = getCellContentHeight();
            int cellPaddingY = (int) Math.max(0, ((lp.height - cHeight) / 2f));
            int cellPaddingX = mContainerType == CellLayout.WORKSPACE
                    ? profile.workspaceCellPaddingXPx
                    : (int) (profile.edgeMarginPx / 2f);
            child.setPadding(cellPaddingX, cellPaddingY, cellPaddingX, 0);
        }
        int childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(lp.width, MeasureSpec.EXACTLY);
        int childheightMeasureSpec = MeasureSpec.makeMeasureSpec(lp.height, MeasureSpec.EXACTLY);
        child.measure(childWidthMeasureSpec, childheightMeasureSpec);
		
        // 添加背景颜色
         child.setBackgroundResource(R.drawable.shape_button);
    }

```

背景布局文件  
shape_button.xml 为:

```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android" >
    <item android:state_pressed="true">
        <shape>
            <solid android:color="#CFCFCF"/>
        </shape>
    </item>
    <item android:state_focused="true">
        <shape>
            <solid android:color="#CFCFCF"/>
        </shape>
    </item>
    <item android:state_selected="true">
        <shape>
            <solid android:color="#E8E8E8"/>
        </shape>
    </item>
    <item android:state_selected="false">
        <shape>
            <solid android:color="#00000000"/>
        </shape>
    </item>
    <item android:state_pressed="false">
        <shape>
            <solid android:color="#00000000"/>
        </shape>
    </item>
    <item android:state_focused="false">
        <shape>
            <solid android:color="#00000000"/>
        </shape>
    </item>

    </selector>

```

编译 launcher3 发现功能已经实现了