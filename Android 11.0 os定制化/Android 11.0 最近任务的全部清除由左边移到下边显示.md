> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126064671)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 最近任务全部清除按钮左边移到下边显示的核心代码](#t1)

[3. 最近任务全部清除按钮左边移到下边显示的核心代码功能分析](#t2)

[3.1 TaskOverlayFactory.java 关于全部清除相关代码分析](#t3)

[3.2 OverviewActionsView.java 中增加下边全部清除按键的相关调用方法](#t4)

[3.3 RecentsView.java 关于下边全部清除按键的相关调用方法](#t5)

[3.4 overview_actions_container.xml 中添加下边全部清除 btn](#t6)

1. 概述
-----

在最近 11.0 的产品开发中，发现点击 recents 最近任务键后 显示的全部清除按键在左边 由于是横屏的产品显示在左边不太合理 所以要求显示在下边比较合理，所以要从 Launcher3 的显示流程来解决这个问题

2. 最近任务全部清除按钮左边移到下边显示的核心代码
--------------------------

```
packages/apps/Launcher3/quickstep/recents_ui_overrides/src/com/android/quickstep/TaskOverlayFactory.java
  packages/apps/Launcher3/quickstep/recents_ui_overrides/src/com/android/quickstep/views/OverviewActionsView.java
  packages/apps/Launcher3/quickstep/recents_ui_overrides/src/com/android/quickstep/views/RecentsView.java
  packages/apps/Launcher3/quickstep/res/layout/overview_actions_container.xml
```

3. 最近任务全部清除按钮左边移到下边显示的核心代码功能分析
------------------------------

#### 3.1 TaskOverlayFactory.java 关于全部清除相关代码分析

// 添加全部清除方法

```
 
  /**
* Factory class to create and add an overlays on the TaskView
*/
public class TaskOverlayFactory implements ResourceBasedOverride {
 
public static List<SystemShortcut> getEnabledShortcuts(TaskView taskView) {
final ArrayList<SystemShortcut> shortcuts = new ArrayList<>();
final BaseDraggingActivity activity = BaseActivity.fromContext(taskView.getContext());
for (TaskShortcutFactory menuOption : MENU_OPTIONS) {
SystemShortcut shortcut = menuOption.getShortcut(activity, taskView);
if (shortcut != null) {
shortcuts.add(shortcut);
}
}
RecentsOrientedState orientedState = taskView.getRecentsView().getPagedViewOrientedState();
boolean canLauncherRotate = orientedState.canRecentsActivityRotate();
boolean isInLandscape = orientedState.getTouchRotation() != ROTATION_0;
 
// Add overview actions to the menu when in in-place rotate landscape mode.
if (!canLauncherRotate && isInLandscape) {
// Add screenshot action to task menu.
SystemShortcut screenshotShortcut = TaskShortcutFactory.SCREENSHOT
getShortcut(activity, taskView);
if (screenshotShortcut != null) {
shortcuts.add(screenshotShortcut);
}
 
// Add modal action only if display orientation is the same as the device orientation.
if (orientedState.getDisplayRotation() == ROTATION_0) {
SystemShortcut modalShortcut = TaskShortcutFactory.MODAL
getShortcut(activity, taskView);
if (modalShortcut != null) {
shortcuts.add(modalShortcut);
}
}
}
return shortcuts;
}
 
public TaskOverlay createOverlay(TaskThumbnailView thumbnailView) {
return new TaskOverlay(thumbnailView);
}
 
public void initOverlay(Task task, ThumbnailData thumbnail, Matrix matrix,
boolean rotated) {
getActionsView().updateDisabledFlags(DISABLED_NO_THUMBNAIL, thumbnail == null);
 
if (thumbnail != null) {
getActionsView().updateDisabledFlags(DISABLED_ROTATED, rotated);
final boolean isAllowedByPolicy = thumbnail.isRealSnapshot;
 
getActionsView().setCallbacks(new OverlayUICallbacks() {
@Override
public void onShare() {
if (isAllowedByPolicy) {
mImageApi.startShareActivity();
} else {
showBlockedByPolicyMessage();
}
}
 
@SuppressLint("NewApi")
@Override
public void onScreenshot() {
saveScreenshot(task);
}
 
+ @Override
+ public void clearAllRecentTask() {
+    mThumbnailView.getTaskView().getRecentsView().dismissAllTasks(mThumbnailView.getTaskView().getRecentsView());
+ }
 
});
}
}
 
public interface OverlayUICallbacks {
/** User has indicated they want to share the current task. */
void onShare();
 
/** User has indicated they want to screenshot the current task. */
void onScreenshot();
 
 
+ void clearAllRecentTask();
}
```

#### 3.2 OverviewActionsView.java 中增加下边全部清除按键的相关调用方法

```
public class OverviewActionsView<T extends OverlayUICallbacks> extends FrameLayout
implements OnClickListener, Insettable {
 
private final Rect mInsets = new Rect();
 
@IntDef(flag = true, value = {
HIDDEN_UNSUPPORTED_NAVIGATION,
HIDDEN_DISABLED_FEATURE,
HIDDEN_NON_ZERO_ROTATION,
HIDDEN_NO_TASKS,
HIDDEN_GESTURE_RUNNING,
HIDDEN_NO_RECENTS})
@Retention(RetentionPolicy.SOURCE)
public @interface ActionsHiddenFlags { }
 
public static final int HIDDEN_UNSUPPORTED_NAVIGATION = 1 << 0;
public static final int HIDDEN_DISABLED_FEATURE = 1 << 1;
public static final int HIDDEN_NON_ZERO_ROTATION = 1 << 2;
public static final int HIDDEN_NO_TASKS = 1 << 3;
public static final int HIDDEN_GESTURE_RUNNING = 1 << 4;
public static final int HIDDEN_NO_RECENTS = 1 << 5;
 
@IntDef(flag = true, value = {
DISABLED_SCROLLING,
DISABLED_ROTATED,
DISABLED_NO_THUMBNAIL})
@Retention(RetentionPolicy.SOURCE)
public @interface ActionsDisabledFlags { }
 
public static final int DISABLED_SCROLLING = 1 << 0;
public static final int DISABLED_ROTATED = 1 << 1;
public static final int DISABLED_NO_THUMBNAIL = 1 << 2;
 
private static final int INDEX_CONTENT_ALPHA = 0;
private static final int INDEX_VISIBILITY_ALPHA = 1;
private static final int INDEX_FULLSCREEN_ALPHA = 2;
private static final int INDEX_HIDDEN_FLAGS_ALPHA = 3;
 
private final MultiValueAlpha mMultiValueAlpha;
 
@ActionsHiddenFlags
private int mHiddenFlags;
 
@ActionsDisabledFlags
protected int mDisabledFlags;
 
protected T mCallbacks;
 
public OverviewActionsView(Context context) {
this(context, null);
}
 
public OverviewActionsView(Context context, @Nullable AttributeSet attrs) {
this(context, attrs, 0);
}
 
public OverviewActionsView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
super(context, attrs, defStyleAttr, 0);
mMultiValueAlpha = new MultiValueAlpha(this, 4);
mMultiValueAlpha.setUpdateVisibility(true);
}
 
@Override
protected void onFinishInflate() {
super.onFinishInflate();
View share = findViewById(R.id.action_share);
share.setOnClickListener(this);
findViewById(R.id.action_screenshot).setOnClickListener(this);
+        findViewById(R.id.recents_clear_all_btn).setOnClickListener(this);
if (ENABLE_OVERVIEW_SHARE.get()) {
share.setVisibility(VISIBLE);
findViewById(R.id.share_space).setVisibility(VISIBLE);
}
}
 
/**
* Set listener for callbacks on action button taps.
*
* @param callbacks for callbacks, or {@code null} to clear the listener.
*/
public void setCallbacks(T callbacks) {
mCallbacks = callbacks;
}
 
@Override
public void onClick(View view) {
if (mCallbacks == null) {
return;
}
int id = view.getId();
if (id == R.id.action_share) {
mCallbacks.onShare();
} else if (id == R.id.action_screenshot) {
mCallbacks.onScreenshot();
}
+        } else if (id == R.id.recents_clear_all_btn) {
+            mCallbacks.clearAllRecentTask();
         }
}
...
}
 
public void updateDisabledFlags(@ActionsDisabledFlags int disabledFlags, boolean enable) {
if (enable) {
mDisabledFlags |= disabledFlags;
} else {
mDisabledFlags &= ~disabledFlags;
}
//
boolean isEnabled = (mDisabledFlags & ~DISABLED_ROTATED) == 0;
- LayoutUtils.setViewEnabled(this, isEnabled);
+ LayoutUtils.setViewEnabled(this, true);
}
 
```

#### 在 onFinishInflate() 和 onClick(View view) 添加对应的方法

#### 3.3 RecentsView.java 关于下边全部清除按键的相关调用方法

```
@TargetApi(Build.VERSION_CODES.R)
public abstract class RecentsView<T extends StatefulActivity> extends PagedView implements
Insettable, TaskThumbnailCache.HighResLoadingState.HighResLoadingStateChangedCallback,
InvariantDeviceProfile.OnIDPChangeListener, TaskVisualsChangeListener,
SplitScreenBounds.OnChangeListener {
 
private static final String TAG = RecentsView.class.getSimpleName();
 
public static final FloatProperty<RecentsView> CONTENT_ALPHA =
new FloatProperty<RecentsView>("contentAlpha") {
@Override
public void setValue(RecentsView view, float v) {
view.setContentAlpha(v);
}
 
@Override
public Float get(RecentsView view) {
return view.getContentAlpha();
}
};
 
public RecentsView(Context context, AttributeSet attrs, int defStyleAttr,
BaseActivityInterface sizeStrategy) {
super(context, attrs, defStyleAttr);
setPageSpacing(getResources().getDimensionPixelSize(R.dimen.recents_page_spacing));
setEnableFreeScroll(true);
mSizeStrategy = sizeStrategy;
mActivity = BaseActivity.fromContext(context);
mOrientationState = new RecentsOrientedState(
context, mSizeStrategy, this::animateRecentsRotationInPlace);
mOrientationState.setRecentsRotation(mActivity.getDisplay().getRotation());
 
mFastFlingVelocity = getResources()
getDimensionPixelSize(R.dimen.recents_fast_fling_velocity);
mModel = RecentsModel.INSTANCE.get(context);
mIdp = InvariantDeviceProfile.INSTANCE.get(context);
 
mClearAllButton = (ClearAllButton) LayoutInflater.from(context)
inflate(R.layout.overview_clear_all_button, this, false);
mClearAllButton.setOnClickListener(this::dismissAllTasks);
+        mClearAllButton.setVisibility(View.GONE);   //隐藏左侧的“全部清除”按钮
 
 
mTaskViewPool = new ViewPool<>(context, this, R.layout.task, 20 /* max size */,
/* initial size */);
 
mIsRtl = mOrientationHandler.getRecentsRtlSetting(getResources());
setLayoutDirection(mIsRtl ? View.LAYOUT_DIRECTION_RTL : View.LAYOUT_DIRECTION_LTR);
mTaskTopMargin = getResources()
getDimensionPixelSize(R.dimen.task_thumbnail_top_margin);
mSquaredTouchSlop = squaredTouchSlop(context);
 
mEmptyIcon = context.getDrawable(R.drawable.ic_empty_recents);
mEmptyIcon.setCallback(this);
mEmptyMessage = context.getText(R.string.recents_empty_message);
mEmptyMessagePaint = new TextPaint();
mEmptyMessagePaint.setColor(Themes.getAttrColor(context, android.R.attr.textColorPrimary));
mEmptyMessagePaint.setTextSize(getResources()
getDimension(R.dimen.recents_empty_message_text_size));
mEmptyMessagePaint.setTypeface(Typeface.create(Themes.getDefaultBodyFont(context),
Typeface.NORMAL));
mEmptyMessagePadding = getResources()
getDimensionPixelSize(R.dimen.recents_empty_message_text_padding);
setWillNotDraw(false);
updateEmptyMessage();
mOrientationHandler = mOrientationState.getOrientationHandler();
 
 
 
mTaskOverlayFactory = Overrides.getObject(
TaskOverlayFactory.class,
context.getApplicationContext(),
R.string.task_overlay_factory_class);
 
// Initialize quickstep specific cache params here, as this is constructed only once
mActivity.getViewCache().setCacheSize(R.layout.digital_wellbeing_toast, 5);
}
 
    public boolean isClearAllHidden() {
         - return mClearAllButton.getAlpha() != 1f;
         + return true;
      }
    - private void dismissAllTasks(View view) {
    + public void dismissAllTasks(View view) {
          runDismissAnimation(createAllTasksDismissAnimation(DISMISS_TASK_DURATION));
          mActivity.getUserEventDispatcher().logActionOnControl(TAP, CLEAR_ALL_BUTTON);
      }
```

#### 3.4 overview_actions_container.xml 中添加下边全部清除 btn

```
--- a/alps/packages/apps/Launcher3/quickstep/res/layout/overview_actions_container.xml
+++ b/alps/packages/apps/Launcher3/quickstep/res/layout/overview_actions_container.xml
@@ -42,6 +42,14 @@
             android:text="@string/action_screenshot"
+            android:visibility="gone"
             android:theme="@style/ThemeControlHighlightWorkspaceColor" />
 
+        <Button
+            android:id="@+id/recents_clear_all_btn"
+            style="@style/OverviewActionButton"
+            android:layout_width="wrap_content"
+            android:layout_height="wrap_content"
+            android:text="@string/recents_clear_all"
+            android:theme="@style/ThemeControlHighlightWorkspaceColor" />
+
         <Space
             android:layout_width="0dp"
             android:layout_height="1dp"
```