> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124855283)

### 1. 概述

在 11.0 的产品开发中对 laucher3 中的需求定制也是比较多的，有功能需求要求修改 app 的 [icon 图标](https://so.csdn.net/so/search?q=icon%E5%9B%BE%E6%A0%87&spm=1001.2101.3001.7020)的功能，而 workspace 的 app 列表页中的所有的 app icon 和名称 以及 Hotseat 文件夹都是用 BubbleTextView 类来构造的，所以要修改 app 的图标就要从这个类来分析源码

### 2.Launcher3 修改某个 app icon 的图标的代码

```
/package/app/Launcher3/src/com/android/launcher3/BubbleTextView.java

```

### 3.Launcher3 修改某个 app icon 的图标的具体功能分析和实现

在 WorkSpace.java 中，app 的 icon 文件夹 hotseat 等图标 文字的构造都是由 BubbleTextView 来负责构造的，所以主要的功能还是在这里根据不同类型来构造不同的图标的，所以具体分析问题就从这里来分析相关源码

```
public class BubbleTextView extends TextView implements ItemInfoUpdateReceiver, OnResumeCallback,
         IconLabelDotView, DraggableView, Reorderable {
 
     private static final int DISPLAY_WORKSPACE = 0;
     private static final int DISPLAY_ALL_APPS = 1;
     private static final int DISPLAY_FOLDER = 2;
 
     private static final int[] STATE_PRESSED = new int[] {android.R.attr.state_pressed};
 
     private final PointF mTranslationForReorderBounce = new PointF(0, 0);
     private final PointF mTranslationForReorderPreview = new PointF(0, 0);
 
     private float mScaleForReorderBounce = 1f;
 
     private static final Property<BubbleTextView, Float> DOT_SCALE_PROPERTY
             = new Property<BubbleTextView, Float>(Float.TYPE, "dotScale") {
         @Override
         public Float get(BubbleTextView bubbleTextView) {
             return bubbleTextView.mDotParams.scale;
         }
 
         @Override
         public void set(BubbleTextView bubbleTextView, Float value) {
             bubbleTextView.mDotParams.scale = value;
             bubbleTextView.invalidate();
         }
      };
  
      public static final Property<BubbleTextView, Float> TEXT_ALPHA_PROPERTY
              = new Property<BubbleTextView, Float>(Float.class, "textAlpha") {
          @Override
          public Float get(BubbleTextView bubbleTextView) {
              return bubbleTextView.mTextAlpha;
          }
  
          @Override
          public void set(BubbleTextView bubbleTextView, Float alpha) {
              bubbleTextView.setTextAlpha(alpha);
          }
      };
  
      public BubbleTextView(Context context) {
          this(context, null, 0);
      }
  
      public BubbleTextView(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }
  
      public BubbleTextView(Context context, AttributeSet attrs, int defStyle) {
          super(context, attrs, defStyle);
          mActivity = ActivityContext.lookupContext(context);
  
          TypedArray a = context.obtainStyledAttributes(attrs,
                  R.styleable.BubbleTextView, defStyle, 0);
          mLayoutHorizontal = a.getBoolean(R.styleable.BubbleTextView_layoutHorizontal, false);
          DeviceProfile grid = mActivity.getDeviceProfile();
  
          mDisplay = a.getInteger(R.styleable.BubbleTextView_iconDisplay, DISPLAY_WORKSPACE);
          final int defaultIconSize;
          if (mDisplay == DISPLAY_WORKSPACE) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.iconTextSizePx);
              setCompoundDrawablePadding(grid.iconDrawablePaddingPx);
              defaultIconSize = grid.iconSizePx;
          } else if (mDisplay == DISPLAY_ALL_APPS) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.allAppsIconTextSizePx);
              setCompoundDrawablePadding(grid.allAppsIconDrawablePaddingPx);
              defaultIconSize = grid.allAppsIconSizePx;
          } else if (mDisplay == DISPLAY_FOLDER) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.folderChildTextSizePx);
              setCompoundDrawablePadding(grid.folderChildDrawablePaddingPx);
              defaultIconSize = grid.folderChildIconSizePx;
          } else {
              // widget_selection or shortcut_popup
              defaultIconSize = grid.iconSizePx;
          }
  
          mCenterVertically = a.getBoolean(R.styleable.BubbleTextView_centerVertically, false);
  
          mIconSize = a.getDimensionPixelSize(R.styleable.BubbleTextView_iconSizeOverride,
                  defaultIconSize);
          a.recycle();
  
          mLongPressHelper = new CheckLongPressHelper(this);
  
          mDotParams = new DotRenderer.DrawParams();
  
          setEllipsize(TruncateAt.END);
          setAccessibilityDelegate(mActivity.getAccessibilityDelegate());
          setTextAlpha(1f);
      }
      @Override
      public void reapplyItemInfo(ItemInfoWithIcon info) {
          if (getTag() == info) {
              mIconLoadRequest = null;
              mDisableRelayout = true;
  
              // Optimization: Starting in N, pre-uploads the bitmap to RenderThread.
              info.bitmap.icon.prepareToDraw();
  
              if (info instanceof AppInfo) {
                  applyFromApplicationInfo((AppInfo) info);
              } else if (info instanceof WorkspaceItemInfo) {
                  applyFromWorkspaceItem((WorkspaceItemInfo) info);
                  mActivity.invalidateParent(info);
              } else if (info instanceof PackageItemInfo) {
                  applyFromPackageItemInfo((PackageItemInfo) info);
              }
  
              mDisableRelayout = false;
          }
      }
public void applyFromWorkspaceItem(WorkspaceItemInfo info) {
        applyFromWorkspaceItem(info, false);
    }

    public void applyFromWorkspaceItem(WorkspaceItemInfo info, boolean promiseStateChanged) {
        applyIconAndLabel(info);
        setTag(info);
        if (promiseStateChanged || (info.hasPromiseIconUi())) {
            applyPromiseState(promiseStateChanged);
        }

        applyDotState(info, false /* animate */);
    }

    public void applyFromApplicationInfo(AppInfo info) {
        applyIconAndLabel(info);

        // We don't need to check the info since it's not a WorkspaceItemInfo
        super.setTag(info);

        // Verify high res immediately
        verifyHighRes();

        if (info instanceof PromiseAppInfo) {
            PromiseAppInfo promiseAppInfo = (PromiseAppInfo) info;
            applyProgressLevel(promiseAppInfo.level);
        }
        applyDotState(info, false /* animate */);
    }

    public void applyFromPackageItemInfo(PackageItemInfo info) {
        applyIconAndLabel(info);
        // We don't need to check the info since it's not a WorkspaceItemInfo
        super.setTag(info);

        // Verify high res immediately
        verifyHighRes();
    }

```

在 reapplyItemInfo(ItemInfoWithIcon info) 中通过判断是否是 app 然后调用相关方法，通过跟代码发现  
最终方法  
通过读取以上方法可以得知保存 workspace 的 item 的图标和文字相关信息  
而都调用了 applyIconAndLabel(info);

```
	private void applyIconAndLabel(ItemInfoWithIcon info) {
         FastBitmapDrawable iconDrawable = newIcon(getContext(), info);
         mDotParams.color = IconPalette.getMutedColor(info.bitmap.color, 0.54f);
 
         setIcon(iconDrawable);
         setText(info.title);
         if (info.contentDescription != null) {
             setContentDescription(info.isDisabled()
                     ? getContext().getString(R.string.disabled_app_label, info.contentDescription)
                     : info.contentDescription);
         }
     }

```

最后在代码中通过而这个 applyIconAndLabel 就是保存 icon 信息到 Icon 缓存中，每次刷新页面的时候  
通过读取缓存加快刷新速度

所以修改如下:

```
 private void applyIconAndLabel(ItemInfoWithIcon info) {
              // 修改某个app的icon start
		if (info.getIntent() != null && info.getIntent().getComponent() != null) {
		  String pkg = info.getIntent().getComponent().getPackageName();
		  if ("com.android.deskclock".equals(pkg) || info.itemType == LauncherSettings.Favorites.ITEM_TYPE_DEEP_SHORTCUT) {
                    //设置bitmap图标
		    info.iconBitmap = IconUtil.getDeskClockIcon(getContext());
		  }
		}
       // 修改某个app的icon end
        FastBitmapDrawable iconDrawable = newIcon(getContext(), info);
        mDotParams.color = IconPalette.getMutedColor(info.iconColor, 0.54f);
		setIcon(iconDrawable);
        setText(info.title);
        if (info.contentDescription != null) {
            setContentDescription(info.isDisabled()
                    ? getContext().getString(R.string.disabled_app_label, info.contentDescription)
                    : info.contentDescription);
        }
    }

```

在 applyIconAndLabel(ItemInfoWithIcon info) 中根据包名，修改对应的 app 的 icon 图标，就达到了功能要求实现了功能  
编译验证 可以看到时钟图标已经更换了