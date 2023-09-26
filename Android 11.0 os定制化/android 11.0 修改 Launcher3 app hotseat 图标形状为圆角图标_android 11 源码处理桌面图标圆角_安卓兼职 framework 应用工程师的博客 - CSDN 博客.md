> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124855271)

### 1. 概述

在对 11.0 产品开发中，对于 Launcher3 做各种定制化开发，也是常见的，最近有功能需求要求，对于修改图标的形状为圆角图标，而在 Launcher3 中，所有的 app 和 hotseat 都是由 BubbleTextView 负责构建的，所以对于图标的修改也是要从 BubbleTextView.java 修改的

在这里插入图片描述

### 2. 修改 Launcher3 app hotseat 图标形状为圆角图标的相关类

```
/package/app/Launcher3/src/com/android/launcher3/BubbleTextView.java
/packages/apps/Launcher3/src/com/android/launcher3/FastBitmapDrawable.java

```

### 3. 修改 Launcher3 app hotseat 图标形状为圆角图标的相关功能分析和实现

### 3.1 看 BubbleTextView.java [源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)关于图标的绑定

```
/package/app/Launcher3/src/com/android/launcher3/BubbleTextView.java
public class BubbleTextView extends TextView implements ItemInfoUpdateReceiver, OnResumeCallback,
         IconLabelDotView, DraggableView, Reorderable {
 
     private static final int DISPLAY_WORKSPACE = 0;
     private static final int DISPLAY_ALL_APPS = 1;
     private static final int DISPLAY_FOLDER = 2;
 
     private static final int[] STATE_PRESSED = new int[] {android.R.attr.state_pressed};

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

在上述代码中发现调用 applyFromApplicationInfo((AppInfo) info) 进行图标数据的绑定加载，而在 applyFromApplicationInfo((AppInfo) info) 中 applyIconAndLabel(info) 是绑定图标和文字的  
applyIconAndLabel 是对图标做处理显示的  
所以就从这里解决问题

进入 FastBitmapDrawable .newIcon();

```
public class FastBitmapDrawable extends Drawable {
  
      private static final float PRESSED_SCALE = 1.1f;
  
      private static final float DISABLED_DESATURATION = 1f;
      private static final float DISABLED_BRIGHTNESS = 0.5f;
  
      public static final int CLICK_FEEDBACK_DURATION = 200;
  
      private static ColorFilter sDisabledFColorFilter;
  
      protected final Paint mPaint = new Paint(Paint.FILTER_BITMAP_FLAG | Paint.ANTI_ALIAS_FLAG);
      protected Bitmap mBitmap;
      protected final int mIconColor;
  
      private boolean mIsPressed;
      private boolean mIsDisabled;
      private float mDisabledAlpha = 1f;
  
      // Animator and properties for the fast bitmap drawable's scale
      private static final Property<FastBitmapDrawable, Float> SCALE
              = new Property<FastBitmapDrawable, Float>(Float.TYPE, "scale") {
          @Override
          public Float get(FastBitmapDrawable fastBitmapDrawable) {
              return fastBitmapDrawable.mScale;
          }
  
          @Override
          public void set(FastBitmapDrawable fastBitmapDrawable, Float value) {
              fastBitmapDrawable.mScale = value;
              fastBitmapDrawable.invalidateSelf();
          }
      };
 /**
       * Returns a FastBitmapDrawable with the icon.
       */
      public static FastBitmapDrawable newIcon(Context context, ItemInfoWithIcon info) {
          FastBitmapDrawable drawable = newIcon(context, info.bitmap);
          drawable.setIsDisabled(info.isDisabled());
          return drawable;
      }
  
      /**
       * Creates a drawable for the provided BitmapInfo
       */
      public static FastBitmapDrawable newIcon(Context context, BitmapInfo info) {
          final FastBitmapDrawable drawable;
          if (info instanceof Factory) {
              drawable = ((Factory) info).newDrawable();
          } else if (info.isLowRes()) {
              drawable = new PlaceHolderIconDrawable(info, context);
          } else {
              drawable = new FastBitmapDrawable(info);
          }
          drawable.mDisabledAlpha = Themes.getFloat(context, R.attr.disabledIconAlpha, 1f);
          return drawable;
      }
  }

```

通过调用 FastBitmapDrawable .newIcon() 等相关方法，发现最后  
发现实际上是有 FastBitmapDrawable 来处理图标而在 FastBitmapDrawable 实际上也是 Drawable 的资子类，最后得用 draw(Canvas canvas) 来绘制图标  
继续跟踪 FastBitmapDrawable

```
public FastBitmapDrawable(ItemInfoWithIcon info) {
        this(info.iconBitmap, info.iconColor);
    }

    protected FastBitmapDrawable(Bitmap b, int iconColor) {
        this(b, iconColor, false);
    }

    protected FastBitmapDrawable(Bitmap b, int iconColor, boolean isDisabled) {
        mBitmap = b;
        mIconColor = iconColor;
        setFilterBitmap(true);
        setIsDisabled(isDisabled);
    }

    @Override
    public final void draw(Canvas canvas) {
        if (mScale != 1f) {
            int count = canvas.save();
            Rect bounds = getBounds();
            canvas.scale(mScale, mScale, bounds.exactCenterX(), bounds.exactCenterY());
            drawInternal(canvas, bounds);
            canvas.restoreToCount(count);
        } else {
            drawInternal(canvas, getBounds());
        }
    }

    protected void drawInternal(Canvas canvas, Rect bounds) {
        canvas.drawBitmap(mBitmap, null, bounds, mPaint);
    }

```

在 draw(Canvas canvas) 绘制图标后，最终调用 drawInternal(Canvas canvas, Rect bounds) 负责绘制图标，从以上代码可以看出 具体处理图标的地方是在 drawInternal 来实现对图标的绘制  
所以就在这里对图标做圆角处理 具体如下:

```
     protected void drawInternal(Canvas canvas, Rect bounds) {
-        canvas.drawBitmap(mBitmap, null, bounds, mPaint);
+        // 初始化绘制纹理图
+        BitmapShader bitmapShader = new BitmapShader(mBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
+               Paint paint = new Paint();
+               paint.setAntiAlias(true);
+               paint.setDither(true);
+        paint.setShader(bitmapShader);
+               paint.setStrokeWidth(16);
+        // 利用画笔将纹理图绘制到画布上面
+               int mWidth = Math.min(mBitmap.getWidth(), mBitmap.getHeight());
+        canvas.drawRoundRect(new RectF(8, 8, mWidth-8, mWidth-8), 40, 40, paint);
+               //canvas.drawCircle(mWidth / 2, mWidth / 2, mWidth / 2, paint);
+        //canvas.drawBitmap(mBitmap, null, bounds, mPaint);
     }

```