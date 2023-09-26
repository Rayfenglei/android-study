> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760196)

在华为手机上经常看到在锁屏充电时，会出现一个充电动画，几秒钟后消失了，觉得这个动画挺好看的，所以就来在 11.0 的产品上自定义模仿华为动画来实现充电动画  
![](https://img-blog.csdnimg.cn/63a3f693d18d45d18950a78a652451a4.png#pic_center)

1. 首选定义动画[实体类](https://so.csdn.net/so/search?q=%E5%AE%9E%E4%BD%93%E7%B1%BB&spm=1001.2101.3001.7020)  
BubbleEntry.java

```
package com.android.systemui.charging;
 
public class BubbleEntry {
 
    private float random_y = 3;
    private float location_x;
    private float location_y;
    private int bubble_index;
 
    public BubbleEntry(float location_x, float location_y, float random_y, int bubble_index) {
        this.location_x = location_x;
        this.location_y = location_y;
        this.random_y = random_y;
        this.bubble_index = bubble_index;
    }
 
    public void set(float x, float y, float randomY, int index) {
        this.location_x = x;
        this.location_y = y;
        this.random_y = randomY;
        this.bubble_index = index;
    }
 
    public void setMove(int screenHeight, int maxDistance) {
        if (location_y - maxDistance < 110) {
            this.location_y -= 2;
            return;
        }
 
        if (maxDistance <= location_y && screenHeight - location_y > 110) {
            this.location_y -= random_y;
        } else {
            this.location_y -= 0.6;
        }
 
        if (bubble_index == 0) {
            this.location_x -= 0.4;
        } else if (bubble_index == 2) {
            this.location_x += 0.4;
        }
    }
 
    public int getBubble_index() {
        return bubble_index;
    }
 
    public float getLocation_x() {
        return location_x;
    }
 
    public void setLocation_x(float location_x) {
        this.location_x = location_x;
    }
 
    public float getLocation_y() {
        return location_y;
    }
 
    public void setLocation_y(float location_y) {
        this.location_y = location_y;
    }
}

```

动画布局文件  
battery_charging_layout.xml

```
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/shcy_charge_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#99000000">
 
    <com.android.systemui.charging.BubbleDrawable
        android:id="@+id/battery_bubble_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</FrameLayout>

```

绘制动画实体类

```
BubbleDrawable.java

package com.android.systemui.charging;
 
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
 
import android.annotation.SuppressLint;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Path;
import android.graphics.PixelFormat;
import android.graphics.PointF;
import android.graphics.PorterDuff;
import android.graphics.PorterDuffXfermode;
import android.graphics.Rect;
import android.util.AttributeSet;
import android.util.DisplayMetrics;
import android.util.Log;
import android.util.TypedValue;
import android.view.SurfaceHolder;
import android.view.SurfaceView;
 
public class BubbleDrawable extends SurfaceView implements
        SurfaceHolder.Callback, Runnable {
    private static ScheduledExecutorService mScheduledThreadPool;
    private Context mContext;
    private String mPaintColor = "#25DA29";
    private String mCenterColor = "#00000000";
    private String minCentreColor = "#9025DA29";
    private int mScreenHeight;
    private int mScreenWidth;
 
    private float mLastRadius;
    private float mRate = 0.32f;
    private float mOtherRate = 0.45f;
    private PointF mLastCurveStrat = new PointF();
    private PointF mLastCurveEnd = new PointF();
    private PointF mCenterCirclePoint = new PointF();
    private float mCenterRadius;
    private float mBubbleRadius;
 
    private PointF[] mArcPointStrat = new PointF[8];
    private PointF[] mArcPointEnd = new PointF[8];
    private PointF[] mControl = new PointF[8];
    private PointF mArcStrat = new PointF();
    private PointF mArcEnd = new PointF();
    private PointF mControlP = new PointF();
 
    List<PointF> mBubbleList = new ArrayList<>();
    List<BubbleEntry> mBubbleEntries = new ArrayList<>();
 
    private int mRotateAngle = 0;
    private float mControlrate = 1.66f;
    private float mControlrateS = 1.3f;
    private int i = 0;
    private SurfaceHolder mHolder;
    private float mScale = 0;
 
    private Paint mArcPaint;
    private Paint minCenterPaint;
    private Paint mBubblePaint;
    private Paint mCenterPaint;
    private Paint mLastPaint;
    private Path mLastPath;
    private Random mRandom;
    private Paint mTextPaint;
    private String mText = "50 %";
    private Rect mRect;
 
    public BubbleDrawable(Context mContext) {
        this(mContext, null);
    }
 
    public BubbleDrawable(Context mContext, AttributeSet attrs) {
        this(mContext, attrs, 0);
    }
 
    public BubbleDrawable(Context mContext, AttributeSet attrs,
                          int defStyleAttr) {
        super(mContext, attrs, defStyleAttr);
        this.mContext = mContext;
        initTool();
    }
 
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        mScreenHeight = getMeasuredHeight();
        mScreenWidth = getMeasuredWidth();
    }
 
    private void initTool() {
        mRect = new Rect();
        mHolder = getHolder();
        mHolder.addCallback(this);
        setFocusable(true);
        mHolder.setFormat(PixelFormat.TRANSPARENT);
        setZOrderOnTop(true);
        mLastRadius = dpDimension(40f, mContext);
        mCenterRadius = dpDimension(100f, mContext);
        mBubbleRadius = dpDimension(15f, mContext);
        mRandom = new Random();
        mLastPaint = new Paint();
        mLastPaint.setAntiAlias(true);
        mLastPaint.setStyle(Paint.Style.FILL);
        mLastPaint.setColor(Color.parseColor(mPaintColor));
        mLastPaint.setStrokeWidth(2);
 
        mLastPath = new Path();
 
        mCenterPaint = new Paint();
        mCenterPaint.setAntiAlias(true);
        mCenterPaint.setStyle(Paint.Style.FILL);
        mCenterPaint.setStrokeWidth(2);
        mCenterPaint
                .setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_OUT));
        mCenterPaint.setColor(Color.parseColor(mCenterColor));
        mArcPaint = new Paint();
        mArcPaint.setAntiAlias(true);
        mArcPaint.setStyle(Paint.Style.FILL);
        mArcPaint.setColor(Color.parseColor(mPaintColor));
        mArcPaint.setStrokeWidth(2);
        minCenterPaint = new Paint();
        minCenterPaint.setAntiAlias(true);
        minCenterPaint.setStyle(Paint.Style.FILL);
        minCenterPaint.setColor(Color.parseColor(minCentreColor));
        minCenterPaint.setStrokeWidth(2);
        mBubblePaint = new Paint();
        mBubblePaint.setAntiAlias(true);
        mBubblePaint.setStyle(Paint.Style.FILL);
        mBubblePaint.setColor(Color.parseColor(minCentreColor));
        mBubblePaint.setStrokeWidth(2);
        mTextPaint = new Paint();
        mTextPaint.setAntiAlias(true);
        mTextPaint.setStyle(Paint.Style.FILL);
        mTextPaint.setColor(Color.parseColor("#FFFFFF"));
        mTextPaint.setStrokeWidth(2);
        mTextPaint.setTextSize(dpDimension(40f, mContext));
 
    }
 
    private void onBubbleDraw() {
        Canvas canvas = mHolder.lockCanvas();
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
        bubbleDraw(canvas);
        lastCircleDraw(canvas);
        centerCircleDraw(canvas);
        mTextPaint.getTextBounds(mText, 0, mText.length(), mRect);
        canvas.drawText(mText, mCenterCirclePoint.x - mRect.width() / 2,
                mCenterCirclePoint.y + mRect.height() / 2, mTextPaint);
        mHolder.unlockCanvasAndPost(canvas);
    }
 
    public void setBatteryLevel(String level){
        this.mText =level+"%";
        postInvalidate();
    }
    private void centerCircleDraw(Canvas canvas) {
        mCenterCirclePoint.set(mScreenWidth / 2, mScreenHeight / 2);
        circleInCoordinateDraw(canvas);
        canvas.drawCircle(mCenterCirclePoint.x, mCenterCirclePoint.y,
                mCenterRadius, mCenterPaint);
 
    }
 
    private void lastCircleDraw(Canvas canvas) {
        mLastCurveStrat.set(mScreenWidth / 2 - mLastRadius, mScreenHeight);
        mLastCurveEnd.set((mScreenWidth / 2), mScreenHeight);
 
        float k = (mLastRadius / 2) / mLastRadius;
 
        float aX = mLastRadius - mLastRadius * mOtherRate;
        float aY = mLastCurveStrat.y - aX * k;
        float bX = mLastRadius - mLastRadius * mRate;
        float bY = mLastCurveEnd.y - bX * k;
 
        mLastPath.rewind();
        mLastPath.moveTo(mLastCurveStrat.x, mLastCurveStrat.y);
        mLastPath.cubicTo(mLastCurveStrat.x + aX, aY, mLastCurveEnd.x - bX, bY,
                mLastCurveEnd.x, mLastCurveEnd.y - mLastRadius / 2);
        mLastPath.cubicTo(mLastCurveEnd.x + bX, bY, mLastCurveEnd.x + mLastRadius
                - aX, aY, mLastCurveEnd.x + mLastRadius, mLastCurveEnd.y);
 
        mLastPath.lineTo(mLastCurveStrat.x, mLastCurveStrat.y);
        canvas.drawPath(mLastPath, mLastPaint);
 
    }
 
    private int bubbleIndex = 0;
 
    private void bubbleDraw(Canvas canvas) {
         
        for (int i = 0; i < mBubbleEntries.size(); i++) {
            if (mBubbleEntries.get(i).getLocation_y() <= (int) (mScreenHeight / 2 + mCenterRadius)) {
                mBubblePaint.setAlpha(000);
                canvas.drawCircle(mBubbleEntries.get(i).getLocation_x(), mBubbleEntries.get(i)
                        .getLocation_y(), mBubbleRadius, mBubblePaint);
            } else {
                mBubblePaint.setAlpha(150);
                canvas.drawCircle(mBubbleEntries.get(i).getLocation_x(), mBubbleEntries.get(i)
                        .getLocation_y(), mBubbleRadius, mBubblePaint);
            }
        }
 
    }
 
    /**
     * @param dip
     * @param context
     * @return
     */
    public float dpDimension(float dip, Context context) {
        DisplayMetrics displayMetrics = context.getResources()
                .getDisplayMetrics();
        return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dip,
                displayMetrics);
    }
 
    /**
     * @param canvas
     */
    public void circleInCoordinateDraw(Canvas canvas) {
        int angle;
        for (int i = 0; i < mArcPointStrat.length; i++) {
            if (i > 3 && i < 6) {
                if (i == 4) {
                    angle = mRotateAngle + i * 60;
 
                } else {
                    angle = mRotateAngle + i * 64;
                }
            } else if (i > 5) {
                if (i == 6) {
                    angle = mRotateAngle + i * 25;
                } else {
                    angle = mRotateAngle + i * 48;
                }
 
            } else {
                angle = mRotateAngle + i * 90;
            }
 
            float radian = (float) Math.toRadians(angle);
            float adjacent = (float) Math.cos(radian) * mCenterRadius;
            float right = (float) Math.sin(radian) * mCenterRadius;
            float radianControl = (float) Math.toRadians(90 - (45 + angle));
            float xStrat = (float) Math.cos(radianControl) * mCenterRadius;
            float yEnd = (float) Math.sin(radianControl) * mCenterRadius;
            if (i == 0 || i == 1) {
                if (i == 1) {
                    mArcStrat.set(mCenterCirclePoint.x + adjacent - mScale,
                            mCenterCirclePoint.y + right + mScale);
                    mArcEnd.set(mCenterCirclePoint.x - right, mCenterCirclePoint.y
                            + adjacent);
 
                } else {
                    mArcStrat.set(mCenterCirclePoint.x + adjacent,
                            mCenterCirclePoint.y + right);
                    mArcEnd.set(mCenterCirclePoint.x - right - mScale,
                            mCenterCirclePoint.y + adjacent + mScale);
 
                }
                mControlP.set(mCenterCirclePoint.x + yEnd * mControlrate,
                        mCenterCirclePoint.y + xStrat * mControlrate);
            } else {
                mArcStrat.set(mCenterCirclePoint.x + adjacent,
                        mCenterCirclePoint.y + right);
                mArcEnd.set(mCenterCirclePoint.x - right, mCenterCirclePoint.y
                        + adjacent);
                if (i > 5) {
                    mControlP.set(mCenterCirclePoint.x + yEnd * mControlrateS,
                            mCenterCirclePoint.y + xStrat * mControlrateS);
                } else {
                    mControlP.set(mCenterCirclePoint.x + yEnd * mControlrate,
                            mCenterCirclePoint.y + xStrat * mControlrate);
                }
            }
            mArcPointStrat[i] = mArcStrat;
            mArcPointEnd[i] = mArcEnd;
            mControl[i] = mControlP;
 
            mLastPath.rewind();
            mLastPath.moveTo(mArcPointStrat[i].x, mArcPointStrat[i].y);
            mLastPath.quadTo(mControl[i].x, mControl[i].y, mArcPointEnd[i].x,
                    mArcPointEnd[i].y);
 
            if (i > 3 && i < 6) {
                canvas.drawPath(mLastPath, minCenterPaint);
            } else {
                canvas.drawPath(mLastPath, mArcPaint);
            }
            mLastPath.rewind();
        }
    }
 
    private void setAnimation() {
        setScheduleWithFixedDelay(this, 0, 5);
        setScheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                if (bubbleIndex > 2)
                    bubbleIndex = 0;
                if (mBubbleEntries.size() < 8) {
                    mBubbleEntries.add(new BubbleEntry(
                            mBubbleList.get(bubbleIndex).x, mBubbleList
                                    .get(bubbleIndex).y, mRandom.nextInt(4) + 2,
                            bubbleIndex));
                } else {
                    for (int i = 0; i < mBubbleEntries.size(); i++) {
                        if (mBubbleEntries.get(i).getLocation_y() <= (int) (mScreenHeight / 2 + mCenterRadius)) {
                            mBubbleEntries.get(i).set(
                                    mBubbleList.get(bubbleIndex).x,
                                    mBubbleList.get(bubbleIndex).y,
                                    mRandom.nextInt(4) + 2, bubbleIndex);
                            if (mRandom.nextInt(mBubbleEntries.size()) + 3 == 3 ? true
                                    : false) {
                            } else {
                                break;
                            }
                        }
                    }
                }
                bubbleIndex++;
            }
        }, 0, 300);
    }
 
    private static ScheduledExecutorService getInstence() {
        if (mScheduledThreadPool == null) {
            synchronized (BubbleDrawable.class) {
                if (mScheduledThreadPool == null) {
                    mScheduledThreadPool = Executors
                            .newSingleThreadScheduledExecutor();
                }
            }
        }
        return mScheduledThreadPool;
    }
 
    private static void setScheduleWithFixedDelay(Runnable var1, long var2,
            long var4) {
        getInstence().scheduleWithFixedDelay(var1, var2, var4,
                TimeUnit.MILLISECONDS);
 
    }
 
    public static void onDestroyThread() {
        getInstence().shutdownNow();
        if (mScheduledThreadPool != null) {
            mScheduledThreadPool = null;
        }
    }
 
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        mBubbleList.clear();
        setBubbleList();
        startBubbleRunnable();
        setAnimation();
    }
 
    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width,
            int height) {
 
    }
 
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        onDestroyThread();
    }
 
    @Override
    public void run() {
        i++;
        mRotateAngle = i;
        if (i > 90 && i < 180) {
            mScale += 0.25;
            if (mControlrateS < 1.66)
                mControlrateS += 0.005;
        } else if (i >= 180) {
            mScale -= 0.12;
            if (i > 300)
                mControlrateS -= 0.01;
        }
        onBubbleDraw();
        if (i == 360) {
            i = 0;
            mRotateAngle = 0;
            mControlrate = 1.66f;
            mControlrateS = 1.3f;
            mScale = 0;
        }
 
    }
 
    public void setBubbleList() {
        float radian = (float) Math.toRadians(35);
        float adjacent = (float) Math.cos(radian) * mLastRadius / 3;
        float right = (float) Math.sin(radian) * mLastRadius / 3;
        if (!mBubbleList.isEmpty())
            return;
        mBubbleList.add(new PointF(mScreenWidth / 2 - adjacent, mScreenHeight
                - right));
        mBubbleList.add(new PointF(mScreenWidth / 2, mScreenHeight - mLastRadius
                / 4));
        mBubbleList.add(new PointF(mScreenWidth / 2 + adjacent, mScreenHeight
                - right));
        startBubbleRunnable();
    }
     
    public void startBubbleRunnable(){
        setScheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < mBubbleEntries.size(); i++) {
                    mBubbleEntries.get(i).setMove(mScreenHeight,
                            (int) (mScreenHeight / 2 + mCenterRadius));
                }
            }
        }, 0, 4);
    }
}

```

充电动画类  
BatteryChargingAnimation.java

```
package com.android.systemui.charging;
 
import android.annotation.NonNull;
import android.annotation.Nullable;
import android.content.Context;
import android.graphics.PixelFormat;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.util.Log;
import android.util.Slog;
import android.view.Gravity;
import android.view.View;
import android.view.WindowManager;
import android.view.LayoutInflater;
import com.android.systemui.R;
 
public class BatteryChargingAnimation {
 
    private static final long DURATION = 3000;
    private static final String TAG = "BatteryChargingAnimation";
    private static final boolean DEBUG = true || Log.isLoggable(TAG, Log.DEBUG);
 
    private final BatteryChargingView mCurrentWirelessChargingView;
    private static BatteryChargingView mPreviousWirelessChargingView;
    private static boolean mShowingWiredChargingAnimation;
 
    public static boolean isShowingWiredChargingAnimation(){
        return mShowingWiredChargingAnimation;
    }
 
    public BatteryChargingAnimation(@NonNull Context context, @Nullable Looper looper, int
            batteryLevel, boolean isDozing) {
        mCurrentWirelessChargingView = new BatteryChargingView(context, looper,
                batteryLevel, isDozing);
    }
 
    public static BatteryChargingAnimation makeWiredChargingAnimation(@NonNull Context context,
                                                                      @Nullable Looper looper, int batteryLevel, boolean isDozing) {
        mShowingWiredChargingAnimation = true;
        Log.d(TAG,"makeWiredChargingAnimation batteryLevel="+batteryLevel);
        return new BatteryChargingAnimation(context, looper, batteryLevel, isDozing);
    }
 
    /**
     * Show the view for the specified duration.
     */
    public void show() {
        if (mCurrentWirelessChargingView == null ||
                mCurrentWirelessChargingView.mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }
 
        /*if (mPreviousWirelessChargingView != null) {
            mPreviousWirelessChargingView.hide(0);
        }*/
 
        mPreviousWirelessChargingView = mCurrentWirelessChargingView;
        mCurrentWirelessChargingView.show();
        mCurrentWirelessChargingView.hide(DURATION);
    }
 
    private static class BatteryChargingView {
        private static final int SHOW = 0;
        private static final int HIDE = 1;
 
        private final WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
        private final Handler mHandler;
 
        private int mGravity;
        private View mView;
        private View mNextView;
        private WindowManager mWM;
 
        public BatteryChargingView(Context context, @Nullable Looper looper, int batteryLevel, boolean isDozing) {
            //mNextView = new WirelessChargingLayout(context, batteryLevel, isDozing);
            mNextView = LayoutInflater.from(context).inflate(R.layout.battery_charging_layout, null, false);
            BubbleDrawable shcyBubbleDrawable = mNextView.findViewById(R.id.battery_bubble_view);
            shcyBubbleDrawable.setBatteryLevel(batteryLevel+"");
 
            mGravity = Gravity.CENTER_HORIZONTAL | Gravity.CENTER;
 
            final WindowManager.LayoutParams params = mParams;
            params.height = WindowManager.LayoutParams.MATCH_PARENT;
            params.width = WindowManager.LayoutParams.MATCH_PARENT;
            params.format = PixelFormat.TRANSLUCENT;
 
            params.type = WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG;
            params.setTitle("Charging Animation");
            params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                    | WindowManager.LayoutParams.FLAG_DIM_BEHIND;
 
            params.dimAmount = 0.3f;
 
            if (looper == null) {
                // Use Looper.myLooper() if looper is not specified.
                looper = Looper.myLooper();
                if (looper == null) {
                    throw new RuntimeException(
                            "Can't display wireless animation on a thread that has not called "
                                    + "Looper.prepare()");
                }
            }
 
            mHandler = new Handler(looper, null) {
                @Override
                public void handleMessage(Message msg) {
                    switch (msg.what) {
                        case SHOW: {
                            handleShow();
                            break;
                        }
                        case HIDE: {
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            mShowingWiredChargingAnimation = false;
                            break;
                        }
                    }
                }
            };
        }
 
        public void show() {
            mHandler.obtainMessage(SHOW).sendToTarget();
        }
 
        public void hide(long duration) {
            mHandler.removeMessages(HIDE);
            mHandler.sendMessageDelayed(Message.obtain(mHandler, HIDE), duration);
        }
 
        private void handleShow() {
 
            if (mView != mNextView) {
                // remove the old view if necessary
                handleHide();
                mView = mNextView;
                Context context = mView.getContext().getApplicationContext();
                String packageName = mView.getContext().getOpPackageName();
                if (context == null) {
                    context = mView.getContext();
                }
                mWM = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
                mParams.packageName = packageName;
                mParams.hideTimeoutMilliseconds = DURATION;
 
                if (mView.getParent() != null) {
                    mWM.removeView(mView);
                }
 
                try {
                    mWM.addView(mView, mParams);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
 
        private void handleHide() {
            if (mView != null) {
                if (mView.getParent() != null) {
                    mWM.removeViewImmediate(mView);
                }
 
                mView = null;
            }
        }
    }
}

```

以上都是充电动画所需要的内容，接下来 在电源变化的时候 启动动画  
在 StatusBar.java 中添加

```
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\phone\StatusBar.java
 @Override
            public void onBatteryLevelChanged(int level, boolean pluggedIn, boolean charging) {
                // 电量变化的回调 锁屏充电就会调到这里
                // start code
                boolean isShowing = BatteryChargingAnimation.isShowingWiredChargingAnimation();
                android.util.Log.d("BatteryChargingAnimation","level="+level+" charging="+charging+" isShowing="+isShowing+"  mState="+mState);
                if (!isShowing && charging && mState == StatusBarState.KEYGUARD) {
                    BatteryChargingAnimation.makeWiredChargingAnimation(mContext, null,
                            level, false).show();
                }
                //end code
            }

```

onBatteryLevelChanged() 锁屏充电的时候，会调到这里 所以添加这里就可以调出动画