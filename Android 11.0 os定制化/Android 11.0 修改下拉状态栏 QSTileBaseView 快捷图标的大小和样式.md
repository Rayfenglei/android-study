> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760485)

在对 11.0SystemUI 的下拉状态栏部分做定制时，需要修改快捷功能图标的大小和背景. 所以就要看 QSTileBaseView.java 的实现布局  
通过源码来修改它的样式

```
QSTileBaseView.java的路径为：

frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSTileBaseView.java

public class QSTileBaseView extends com.android.systemui.plugins.qs.QSTileView {

 .....
    private final int[] mLocInScreen = new int[2];
    private final FrameLayout mIconFrame;
    protected QSIconView mIcon;
    protected RippleDrawable mRipple;
    private Drawable mTileBackground;
    private String mAccessibilityClass;
    private boolean mTileState;
    private boolean mCollapsedView;
    private boolean mClicked;
    private boolean mShowRippleEffect = true;

    private final ImageView mBg;
    private final int mColorActive;
    private final int mColorInactive;
    private final int mColorDisabled;
    private int mCircleColor;
    private int mBgSize;
    private ValueAnimator mAnimator; // UNISOC: Modify for bug 1244855

    public QSTileBaseView(Context context, QSIconView icon) {
        this(context, icon, false);
    }

    public QSTileBaseView(Context context, QSIconView icon, boolean collapsedView) {
        super(context);
        // Default to Quick Tile padding, and QSTileView will specify its own padding.
        int padding = context.getResources().getDimensionPixelSize(R.dimen.qs_quick_tile_padding);
        mIconFrame = new FrameLayout(context);
        int size = context.getResources().getDimensionPixelSize(R.dimen.qs_quick_tile_size);
        addView(mIconFrame, new LayoutParams(size, size));
        mBg = new ImageView(getContext());
        Path path = new Path(PathParser.createPathFromPathData(
                context.getResources().getString(ICON_MASK_ID)));
        float pathSize = AdaptiveIconDrawable.MASK_SIZE;
        PathShape p = new PathShape(path, pathSize, pathSize);
        ShapeDrawable d = new ShapeDrawable(p);
        d.setTintList(ColorStateList.valueOf(Color.TRANSPARENT));
        int bgSize = context.getResources().getDimensionPixelSize(R.dimen.qs_tile_background_size);
        d.setIntrinsicHeight(bgSize);
        d.setIntrinsicWidth(bgSize);
        mBg.setImageDrawable(d);
        FrameLayout.LayoutParams lp = new FrameLayout.LayoutParams(bgSize, bgSize, Gravity.CENTER);
        mIconFrame.addView(mBg, lp);
        mBg.setLayoutParams(lp);
        mIcon = icon;
        FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT,
                Gravity.CENTER);
        mIconFrame.addView(mIcon, params);
        mIconFrame.setClipChildren(false);
        mIconFrame.setClipToPadding(false);

        mTileBackground = newTileBackground();
        if (mTileBackground instanceof RippleDrawable) {
            setRipple((RippleDrawable) mTileBackground);
        }
        setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
        setBackground(mTileBackground);

        mColorActive = /*Utils.getColorAttrDefaultColor(context, android.R.attr.colorAccent)*/Color.parseColor("#1E90FF");
        mColorDisabled = /*Utils.getDisabled(context,Utils.getColorAttrDefaultColor(context, android.R.attr.textColorTertiary))*/Color.parseColor("#F5F6F7");
        mColorInactive = Utils.getColorAttrDefaultColor(context, android.R.attr.textColorSecondary);

        setPadding(0, 0, 0, 0);
        setClipChildren(false);
        setClipToPadding(false);
        mCollapsedView = collapsedView;
        setFocusable(true);
    }

```

在构造方法中主要设置相关的参数  
ICON_MASK_ID 为背景 svg xml 相关代码  
修改背景颜色就是这里修改 修改成 svg 样式的就可以了

```
<string </string>

```

背景图片的大小为 res/value/dimens.xml 中的  
qs_tile_background_size

图标的宽高  
qs_quick_tile_size

可以调整自己适合的高度即可  
mColorActive 为快捷键开关打开状态的颜色  
如果想修改颜色 可以这样修改：

```
mColorActive = /*Utils.getColorAttrDefaultColor(context, android.R.attr.colorAccent)*/Color.parseColor("#1E90FF");

```

同理 mColorDisabled 为未选中的颜色  
可以这样修改：

```
 mColorDisabled = /*Utils.getDisabled(context,Utils.getColorAttrDefaultColor(context, android.R.attr.textColorTertiary))*/Color.parseColor("#F5F6F7");

```

然后可以编译 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 验证修改样式就可以了