> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126004790)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 进入默认 Launcher 前黑屏 2 秒的解决办法的相关代码](#t1)

[3. 进入默认 Launcher 前黑屏 2 秒的解决办法的相关代码分析和解决方法](#t2)

[3.1 BootAnimation.cpp 的开机动画的相关代码分析](#t3)

[3.2 WindowManagerService.java 关于是否播放完开机动画的相关代码分析](#t4)

[3.3 ActivityRecord.java 来判断是否进入 Launcher, 进入 Launcher 后设置系统属性然后退出开机动画](#t5)

1. 概述
-----

在 11.0 12.0 的产品定制化开发过程中，需要默认设置 [launcher](https://so.csdn.net/so/search?q=launcher&spm=1001.2101.3001.7020) 来替换 laucher3, 结果遇到黑屏两秒才进入默认 Launcher 的情况原本应该是开机动画播放完就进入 launcher 的，所以这就需要设置系统属性值来延时开机动画，然后等进入 Launcher 以后在退出开机动画

2. 进入默认 Launcher 前黑屏 2 秒的解决办法的相关代码
----------------------------------

```
frameworks\base\cmds\bootanimation\BootAnimation.cpp
   frameworks\base\services\core\java\com\android\server\wm\WindowManagerService.java
   frameworks\base\services\core\java\com\android\server\am\ActivityRecord.java
```

3. 进入默认 Launcher 前黑屏 2 秒的解决办法的相关代码分析和解决方法
-----------------------------------------

#### 3.1 BootAnimation.cpp 的开机动画的相关代码分析

```
namespace android {
 
static const char OEM_BOOTANIMATION_FILE[] = "/oem/media/bootanimation.zip";
static const char PRODUCT_BOOTANIMATION_DARK_FILE[] = "/product/media/bootanimation-dark.zip";
static const char PRODUCT_BOOTANIMATION_FILE[] = "/product/media/bootanimation.zip";
static const char SYSTEM_BOOTANIMATION_FILE[] = "/system/media/bootanimation.zip";
static const char APEX_BOOTANIMATION_FILE[] = "/apex/com.android.bootanimation/etc/bootanimation.zip";
static const char PRODUCT_ENCRYPTED_BOOTANIMATION_FILE[] = "/product/media/bootanimation-encrypted.zip";
static const char SYSTEM_ENCRYPTED_BOOTANIMATION_FILE[] = "/system/media/bootanimation-encrypted.zip";
static const char OEM_SHUTDOWNANIMATION_FILE[] = "/oem/media/shutdownanimation.zip";
static const char PRODUCT_SHUTDOWNANIMATION_FILE[] = "/product/media/shutdownanimation.zip";
static const char SYSTEM_SHUTDOWNANIMATION_FILE[] = "/system/media/shutdownanimation.zip";
 
static constexpr const char* PRODUCT_USERSPACE_REBOOT_ANIMATION_FILE = "/product/media/userspace-reboot.zip";
static constexpr const char* OEM_USERSPACE_REBOOT_ANIMATION_FILE = "/oem/media/userspace-reboot.zip";
static constexpr const char* SYSTEM_USERSPACE_REBOOT_ANIMATION_FILE = "/system/media/userspace-reboot.zip";
 
static const char SYSTEM_DATA_DIR_PATH[] = "/data/system";
static const char SYSTEM_TIME_DIR_NAME[] = "time";
static const char SYSTEM_TIME_DIR_PATH[] = "/data/system/time";
static const char CLOCK_FONT_ASSET[] = "images/clock_font.png";
static const char CLOCK_FONT_ZIP_NAME[] = "clock_font.png";
static const char LAST_TIME_CHANGED_FILE_NAME[] = "last_time_change";
static const char LAST_TIME_CHANGED_FILE_PATH[] = "/data/system/time/last_time_change";
static const char ACCURATE_TIME_FLAG_FILE_NAME[] = "time_is_accurate";
static const char ACCURATE_TIME_FLAG_FILE_PATH[] = "/data/system/time/time_is_accurate";
static const char TIME_FORMAT_12_HOUR_FLAG_FILE_PATH[] = "/data/system/time/time_format_12_hour";
// Java timestamp format. Don't show the clock if the date is before 2000-01-01 00:00:00.
static const long long ACCURATE_TIME_EPOCH = 946684800000;
static constexpr char FONT_BEGIN_CHAR = ' ';
static constexpr char FONT_END_CHAR = '~' + 1;
static constexpr size_t FONT_NUM_CHARS = FONT_END_CHAR - FONT_BEGIN_CHAR + 1;
static constexpr size_t FONT_NUM_COLS = 16;
static constexpr size_t FONT_NUM_ROWS = FONT_NUM_CHARS / FONT_NUM_COLS;
static const int TEXT_CENTER_VALUE = INT_MAX;
static const int TEXT_MISSING_VALUE = INT_MIN;
static const char EXIT_PROP_NAME[] = "service.bootanim.exit";
static const char DISPLAYS_PROP_NAME[] = "persist.service.bootanim.displays";
static const int ANIM_ENTRY_NAME_MAX = ANIM_PATH_MAX + 1;
static constexpr size_t TEXT_POS_LEN_MAX = 16;
 
// ---------------------------------------------------------------------------
 
BootAnimation::BootAnimation(sp<Callbacks> callbacks)
: Thread(false), mClockEnabled(true), mTimeIsAccurate(false), mTimeFormat12Hour(false),
mTimeCheckThread(nullptr), mCallbacks(callbacks), mLooper(new Looper(false)) {
mSession = new SurfaceComposerClient();
 
std::string powerCtl = android::base::GetProperty("sys.powerctl", "");
if (powerCtl.empty()) {
mShuttingDown = false;
} else {
mShuttingDown = true;
}
ALOGD("%sAnimationStartTiming start time: %" PRId64 "ms", mShuttingDown ? "Shutdown" : "Boot",
elapsedRealtime());
}
 
BootAnimation::~BootAnimation() {
if (mAnimation != nullptr) {
releaseAnimation(mAnimation);
mAnimation = nullptr;
}
ALOGD("%sAnimationStopTiming start time: %" PRId64 "ms", mShuttingDown ? "Shutdown" : "Boot",
elapsedRealtime());
}
 
void BootAnimation::onFirstRef() {
status_t err = mSession->linkToComposerDeath(this);
SLOGE_IF(err, "linkToComposerDeath failed (%s) ", strerror(-err));
if (err == NO_ERROR) {
// Load the animation content -- this can be slow (eg 200ms)
// called before waitForSurfaceFlinger() in main() to avoid wait
ALOGD("%sAnimationPreloadTiming start time: %" PRId64 "ms",
mShuttingDown ? "Shutdown" : "Boot", elapsedRealtime());
preloadAnimation();
ALOGD("%sAnimationPreloadStopTiming start time: %" PRId64 "ms",
mShuttingDown ? "Shutdown" : "Boot", elapsedRealtime());
}
}
 
status_t BootAnimation::readyToRun() {
mAssets.addDefaultAssets();
 
mDisplayToken = SurfaceComposerClient::getInternalDisplayToken();
if (mDisplayToken == nullptr)
return NAME_NOT_FOUND;
 
DisplayConfig displayConfig;
const status_t error =
SurfaceComposerClient::getActiveDisplayConfig(mDisplayToken, &displayConfig);
if (error != NO_ERROR)
return error;
 
mMaxWidth = android::base::GetIntProperty("ro.surface_flinger.max_graphics_width", 0);
mMaxHeight = android::base::GetIntProperty("ro.surface_flinger.max_graphics_height", 0);
ui::Size resolution = displayConfig.resolution;
resolution = limitSurfaceSize(resolution.width, resolution.height);
// create the native surface
sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
resolution.getWidth(), resolution.getHeight(), PIXEL_FORMAT_RGB_565);
 
SurfaceComposerClient::Transaction t;
 
// this guest property specifies multi-display IDs to show the boot animation
// multiple ids can be set with comma (,) as separator, for example:
// setprop persist.boot.animation.displays 19260422155234049,19261083906282754
Vector<uint64_t> physicalDisplayIds;
char displayValue[PROPERTY_VALUE_MAX] = "";
property_get(DISPLAYS_PROP_NAME, displayValue, "");
bool isValid = displayValue[0] != '\0';
if (isValid) {
char *p = displayValue;
while (*p) {
if (!isdigit(*p) && *p != ',') {
isValid = false;
break;
}
p ++;
}
if (!isValid)
SLOGE("Invalid syntax for the value of system prop: %s", DISPLAYS_PROP_NAME);
}
if (isValid) {
std::istringstream stream(displayValue);
for (PhysicalDisplayId id; stream >> id; ) {
physicalDisplayIds.add(id);
if (stream.peek() == ',')
stream.ignore();
}
 
// In the case of multi-display, boot animation shows on the specified displays
// in addition to the primary display
auto ids = SurfaceComposerClient::getPhysicalDisplayIds();
constexpr uint32_t LAYER_STACK = 0;
for (auto id : physicalDisplayIds) {
if (std::find(ids.begin(), ids.end(), id) != ids.end()) {
sp<IBinder> token = SurfaceComposerClient::getPhysicalDisplayToken(id);
if (token != nullptr)
t.setDisplayLayerStack(token, LAYER_STACK);
}
}
t.setLayerStack(control, LAYER_STACK);
}
 
t.setLayer(control, 0x40000000)
apply();
 
sp<Surface> s = control->getSurface();
 
// initialize opengl and egl
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
eglInitialize(display, nullptr, nullptr);
EGLConfig config = getEglConfig(display);
EGLSurface surface = eglCreateWindowSurface(display, config, s.get(), nullptr);
EGLContext context = eglCreateContext(display, config, nullptr, nullptr);
EGLint w, h;
eglQuerySurface(display, surface, EGL_WIDTH, &w);
eglQuerySurface(display, surface, EGL_HEIGHT, &h);
 
if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)
return NO_INIT;
 
mDisplay = display;
mContext = context;
mSurface = surface;
mWidth = w;
mHeight = h;
mFlingerSurfaceControl = control;
mFlingerSurface = s;
mTargetInset = -1;
 
// Register a display event receiver
mDisplayEventReceiver = std::make_unique<DisplayEventReceiver>();
status_t status = mDisplayEventReceiver->initCheck();
SLOGE_IF(status != NO_ERROR, "Initialization of DisplayEventReceiver failed with status: %d",
status);
mLooper->addFd(mDisplayEventReceiver->getFd(), 0, Looper::EVENT_INPUT,
new DisplayEventCallback(this), nullptr);
 
return NO_ERROR;
}
readyToRun() 负责准备绘制每张开机动画
 
void BootAnimation::checkExit() {
// Allow surface flinger to gracefully request shutdown
char value[PROPERTY_VALUE_MAX];
property_get(EXIT_PROP_NAME, value, "0");
int exitnow = atoi(value);
if (exitnow) {
requestExit();
mCallbacks->shutdown();
}
}
 
checkExit() 在绘制每一帧开机动画时都会调用checkExit()判断是否退出开机动画
所以在此处可以添加系统属性来判断是否进入默认Launcher后在退出开机动画
 
修改如下:
 + static const char EXIT_ANIM_NAME[] = "persist.sys.bootanim.exit";//自定义属性
void BootAnimation::checkExit() {
// Allow surface flinger to gracefully request shutdown
char value[PROPERTY_VALUE_MAX];
+ char jvalue[PROPERTY_VALUE_MAX];
property_get(EXIT_PROP_NAME, value, "0");
 + property_get(EXIT_ANIM_NAME, jvalue, "0");
int exitnow = atoi(value);
+ int jexitnow = atoi(jvalue);
if (exitnow) {
+ if(jexitnow == 0) {
 + return;
}
requestExit();
mCallbacks->shutdown();
}
}
```

#### 3.2 WindowManagerService.java 关于是否播放完开机动画的相关代码分析

```
private void performEnableScreen() {
synchronized (mGlobalLock) {
ProtoLog.i(WM_DEBUG_BOOT, "performEnableScreen: mDisplayEnabled=%b"
+ " mForceDisplayEnabled=%b" + " mShowingBootMessages=%b"
+ " mSystemBooted=%b mOnlyCore=%b. %s", mDisplayEnabled,
mForceDisplayEnabled, mShowingBootMessages, mSystemBooted, mOnlyCore,
new RuntimeException("here").fillInStackTrace());
if (mDisplayEnabled) {
return;
}
if (!mSystemBooted && !mShowingBootMessages) {
return;
}
 
if (!mShowingBootMessages && !mPolicy.canDismissBootAnimation()) {
return;
}
 
// Don't enable the screen until all existing windows have been drawn.
if (!mForceDisplayEnabled) {
for (int i = mRoot.getChildCount() - 1; i >= 0; i--) {
if (mRoot.getChildAt(i).shouldWaitForSystemDecorWindowsOnBoot()) {
return;
}
}
}
 
if (!mBootAnimationStopped) {
Trace.asyncTraceBegin(TRACE_TAG_WINDOW_MANAGER, "Stop bootanim", 0);
// stop boot animation
// formerly we would just kill the process, but we now ask it to exit so it
// can choose where to stop the animation.
SystemProperties.set("service.bootanim.exit", "1");
mBootAnimationStopped = true;
}
 
if (!mForceDisplayEnabled && !checkBootAnimationCompleteLocked()) {
ProtoLog.i(WM_DEBUG_BOOT, "performEnableScreen: Waiting for anim complete");
return;
}
 
try {
IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
if (surfaceFlinger != null) {
ProtoLog.i(WM_ERROR, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
Parcel data = Parcel.obtain();
data.writeInterfaceToken("android.ui.ISurfaceComposer");
surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
data, null, 0);
data.recycle();
}
} catch (RemoteException ex) {
ProtoLog.e(WM_ERROR, "Boot completed: SurfaceFlinger is dead!");
}
 
EventLogTags.writeWmBootAnimationDone(SystemClock.uptimeMillis());
Trace.asyncTraceEnd(TRACE_TAG_WINDOW_MANAGER, "Stop bootanim", 0);
mDisplayEnabled = true;
ProtoLog.i(WM_DEBUG_SCREEN_ON, "******************** ENABLING SCREEN!");
 
// Enable input dispatch.
mInputManagerCallback.setEventDispatchingLw(mEventDispatchingEnabled);
}
 
try {
mActivityManager.bootAnimationComplete();
} catch (RemoteException e) {
}
 
mPolicy.enableScreenAfterBoot();
 
// Make sure the last requested orientation has been applied.
updateRotationUnchecked(false, false);
}
 
private boolean checkBootAnimationCompleteLocked() {
if (SystemService.isRunning(BOOT_ANIMATION_SERVICE)) {
mH.removeMessages(H.CHECK_IF_BOOT_ANIMATION_FINISHED);
mH.sendEmptyMessageDelayed(H.CHECK_IF_BOOT_ANIMATION_FINISHED,
BOOT_ANIMATION_POLL_INTERVAL);
ProtoLog.i(WM_DEBUG_BOOT, "checkBootAnimationComplete: Waiting for anim complete");
return false;
}
ProtoLog.i(WM_DEBUG_BOOT, "checkBootAnimationComplete: Animation complete!");
return true;
}
 
在开机动画播放完毕后 会调用performEnableScreen() 来通知系统退出开机动画，然后发送开机启动广播通知
checkBootAnimationCompleteLocked()就是在检验是否开机动画完成
```

#### 3.3 ActivityRecord.java 来判断是否进入 Launcher, 进入 Launcher 后设置系统属性然后退出开机动画

```
ActivityRecord是Activity在SystemService中的实现。
  ActivityRecord同步着Activity的生命周期，记录了Activity的关键信息。
  ActivityRecord参与了窗口显示、尺寸、图层
 
  final class ActivityRecord extends WindowToken implements WindowManagerService.AppFreezeListener {
private static final String TAG = TAG_WITH_CLASS_NAME ? "ActivityRecord" : TAG_ATM;
private static final String TAG_ADD_REMOVE = TAG + POSTFIX_ADD_REMOVE;
private static final String TAG_APP = TAG + POSTFIX_APP;
private static final String TAG_CONFIGURATION = TAG + POSTFIX_CONFIGURATION;
private static final String TAG_CONTAINERS = TAG + POSTFIX_CONTAINERS;
private static final String TAG_FOCUS = TAG + POSTFIX_FOCUS;
private static final String TAG_PAUSE = TAG + POSTFIX_PAUSE;
private static final String TAG_RESULTS = TAG + POSTFIX_RESULTS;
private static final String TAG_SAVED_STATE = TAG + POSTFIX_SAVED_STATE;
private static final String TAG_STATES = TAG + POSTFIX_STATES;
private static final String TAG_SWITCH = TAG + POSTFIX_SWITCH;
private static final String TAG_TRANSITION = TAG + POSTFIX_TRANSITION;
private static final String TAG_USER_LEAVING = TAG + POSTFIX_USER_LEAVING;
private static final String TAG_VISIBILITY = TAG + POSTFIX_VISIBILITY;
 
private static final String ATTR_ID = "id";
private static final String TAG_INTENT = "intent";
private static final String ATTR_USERID = "user_id";
private static final String TAG_PERSISTABLEBUNDLE = "persistable_bundle";
private static final String ATTR_LAUNCHEDFROMUID = "launched_from_uid";
private static final String ATTR_LAUNCHEDFROMPACKAGE = "launched_from_package";
private static final String ATTR_LAUNCHEDFROMFEATURE = "launched_from_feature";
private static final String ATTR_RESOLVEDTYPE = "resolved_type";
private static final String ATTR_COMPONENTSPECIFIED = "component_specified";
static final String ACTIVITY_ICON_SUFFIX = "_activity_icon_";
 
// How many activities have to be scheduled to stop to force a stop pass.
private static final int MAX_STOPPING_TO_FORCE = 3;
 
private static final int STARTING_WINDOW_TYPE_NONE = 0;
private static final int STARTING_WINDOW_TYPE_SNAPSHOT = 1;
private static final int STARTING_WINDOW_TYPE_SPLASH_SCREEN = 2;
 
/**
* Value to increment the z-layer when boosting a layer during animations. BOOST in l33tsp34k.
*/
@VisibleForTesting static final int Z_BOOST_BASE = 800570000;
static final int INVALID_PID = -1;
 
// How long we wait until giving up on the last activity to pause.  This
// is short because it directly impacts the responsiveness of starting the
// next activity.
private static final int PAUSE_TIMEOUT = 500;
 
// Ticks during which we check progress while waiting for an app to launch.
private static final int LAUNCH_TICK = 500;
 
// How long we wait for the activity to tell us it has stopped before
// giving up.  This is a good amount of time because we really need this
// from the application in order to get its saved state. Once the stop
// is complete we may start destroying client resources triggering
// crashes if the UI thread was hung. We put this timeout one second behind
// the ANR timeout so these situations will generate ANR instead of
// Surface lost or other errors.
private static final int STOP_TIMEOUT = 11 * 1000;
 
// How long we wait until giving up on an activity telling us it has
// finished destroying itself.
private static final int DESTROY_TIMEOUT = 10 * 1000;
 
final ActivityTaskManagerService mAtmService;
final ActivityInfo info; // activity info provided by developer in AndroidManifest
// Non-null only for application tokens.
// TODO: rename to mActivityToken
final ActivityRecord.Token appToken;
// Which user is this running for?
final int mUserId;
// The package implementing intent's component
// TODO: rename to mPackageName
final String packageName;
// the intent component, or target of an alias.
final ComponentName mActivityComponent;
// Has a wallpaper window as a background.
// TODO: Rename to mHasWallpaper and also see if it possible to combine this with the
// mOccludesParent field.
final boolean hasWallpaper;
// Input application handle used by the input dispatcher.
final InputApplicationHandle mInputApplicationHandle;
 
boolean updateDrawnWindowStates(WindowState w) {
w.setDrawnStateEvaluated(true /*evaluated*/);
 
if (DEBUG_STARTING_WINDOW_VERBOSE && w == startingWindow) {
Slog.d(TAG, "updateWindows: starting " + w + " isOnScreen=" + w.isOnScreen()
+ " allDrawn=" + allDrawn + " freezingScreen=" + mFreezingScreen);
}
 
if (allDrawn && !mFreezingScreen) {
return false;
}
 
if (mLastTransactionSequence != mWmService.mTransactionSequence) {
mLastTransactionSequence = mWmService.mTransactionSequence;
mNumDrawnWindows = 0;
startingDisplayed = false;
 
// There is the main base application window, even if it is exiting, wait for it
mNumInterestingWindows = findMainWindow(false /* includeStartingApp */) != null ? 1 : 0;
}
 
final WindowStateAnimator winAnimator = w.mWinAnimator;
 
boolean isInterestingAndDrawn = false;
 
if (!allDrawn && w.mightAffectAllDrawn()) {
if (DEBUG_VISIBILITY || WM_DEBUG_ORIENTATION.isLogToLogcat()) {
Slog.v(TAG, "Eval win " + w + ": isDrawn=" + w.isDrawnLw()
+ ", isAnimationSet=" + isAnimating(TRANSITION | PARENTS));
if (!w.isDrawnLw()) {
Slog.v(TAG, "Not displayed: s=" + winAnimator.mSurfaceController
+ " pv=" + w.isVisibleByPolicy()
+ " mDrawState=" + winAnimator.drawStateToString()
+ " ph=" + w.isParentWindowHidden() + " th=" + mVisibleRequested
+ " a=" + isAnimating(TRANSITION | PARENTS));
}
}
 
if (w != startingWindow) {
if (w.isInteresting()) {
// Add non-main window as interesting since the main app has already been added
if (findMainWindow(false /* includeStartingApp */) != w) {
mNumInterestingWindows++;
}
if (w.isDrawnLw()) {
mNumDrawnWindows++;
 
if (DEBUG_VISIBILITY || WM_DEBUG_ORIENTATION.isLogToLogcat()) {
Slog.v(TAG, "tokenMayBeDrawn: "
+ this + " w=" + w + " numInteresting=" + mNumInterestingWindows
+ " freezingScreen=" + mFreezingScreen
+ " mAppFreezing=" + w.mAppFreezing);
}
 
isInterestingAndDrawn = true;
}
}
} else if (w.isDrawnLw()) {
// The starting window for this container is drawn.
mStackSupervisor.getActivityMetricsLogger().notifyStartingWindowDrawn(this);
startingDisplayed = true;
}
}
 
return isInterestingAndDrawn;
}
 
/** Called when the windows associated app window container are drawn. */
void onWindowsDrawn(boolean drawn, long timestampNs) {
mDrawn = drawn;
if (!drawn) {
return;
}
final TransitionInfoSnapshot info = mStackSupervisor
getActivityMetricsLogger().notifyWindowsDrawn(this, timestampNs);
final boolean validInfo = info != null;
final int windowsDrawnDelayMs = validInfo ? info.windowsDrawnDelayMs : INVALID_DELAY;
final @LaunchState int launchState = validInfo ? info.getLaunchState() : -1;
// The activity may have been requested to be invisible (another activity has been launched)
// so there is no valid info. But if it is the current top activity (e.g. sleeping), the
// invalid state is still reported to make sure the waiting result is notified.
if (validInfo || this == getDisplayArea().topRunningActivity()) {
mStackSupervisor.reportActivityLaunchedLocked(false /* timeout */, this,
windowsDrawnDelayMs, launchState);
mStackSupervisor.stopWaitingForActivityVisible(this, windowsDrawnDelayMs);
}
finishLaunchTickingLocked();
if (task != null) {
task.setHasBeenVisible(true);
}
}
 
在进入到Launcher后会调用updateDrawnWindowStates(WindowState w) 而这个方法
又调用onWindowsDrawn(）
所以在onWindowDrawn（）中来设置系统属性来退出开机动画 
具体修改为:
void onWindowsDrawn(boolean drawn, long timestampNs) {
mDrawn = drawn;
if (!drawn) {
return;
}
final TransitionInfoSnapshot info = mStackSupervisor
getActivityMetricsLogger().notifyWindowsDrawn(this, timestampNs);
final boolean validInfo = info != null;
final int windowsDrawnDelayMs = validInfo ? info.windowsDrawnDelayMs : INVALID_DELAY;
final @LaunchState int launchState = validInfo ? info.getLaunchState() : -1;
// The activity may have been requested to be invisible (another activity has been launched)
// so there is no valid info. But if it is the current top activity (e.g. sleeping), the
// invalid state is still reported to make sure the waiting result is notified.
if (validInfo || this == getDisplayArea().topRunningActivity()) {
mStackSupervisor.reportActivityLaunchedLocked(false /* timeout */, this,
windowsDrawnDelayMs, launchState);
mStackSupervisor.stopWaitingForActivityVisible(this, windowsDrawnDelayMs);
}
finishLaunchTickingLocked();
if (task != null) {
task.setHasBeenVisible(true);
}
 
// add core start
    if(shortComponentName!=null&&shortComponentName.contains("yourlauncher")){//判断是否已经启动你launcher
     SystemProperties.set("persist.sys.bootanim.exit", "1");//设置启动完成标志
   }
     // add code end
}
```