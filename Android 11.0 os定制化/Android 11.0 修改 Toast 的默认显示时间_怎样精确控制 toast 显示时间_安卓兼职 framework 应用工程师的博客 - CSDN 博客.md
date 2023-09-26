> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126256918)

1. 概述
-----

  在 android 系统和 [app 开发](https://so.csdn.net/so/search?q=app%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，Toast 提醒功能非常常见，但是由于 Toast 的显示时间是在原生系统不能设置的，所以必须要通过源码修改显示时间才可以由于项目开发需要，要求延长 toast 的显示时间，所以需要找到相关设置显示时间的相关代码来设置对应的时间就可以了

2. 修改 [Toast](https://so.csdn.net/so/search?q=Toast&spm=1001.2101.3001.7020) 的显示时间核心代码
--------------------------------------------------------------------------------------

```
 核心代码:
frameworks/base/core/java/android/widget/Toast.java
   framework/base/services/core/java/com/android/server/notification/NotificationManagerService.java
```

3. 修改 Toast 的显示时间核心代码以及功能分析
---------------------------

####  3.1 Toast.java 关于显示时间的相关代码

首先从 show() 分析相关代码

```
 
public class Toast {
	static final String TAG = "Toast";
	static final Boolean localLOGV = false;
	/** @hide */
	@IntDef(prefix = { "LENGTH_" }, value = {
	LENGTH_sHORT,
	LENGTH_lONG
	})
	@Retention(RetentionPolicy.SOURCE)
	public @interface Duration {
	}
	/**
* Show the view or text notification for a short period of time.  This time
* could be user-definable.  This is the default.
* @see #setDuration
*/
	public static final int LENGTH_sHORT = 0;
	/**
* Show the view or text notification for a long period of time.  This time
* could be user-definable.
* @see #setDuration
*/
	public static final int LENGTH_lONG = 1;
	/**
* Text toasts will be rendered by SystemUI instead of in-app, so apps can't circumvent
* background custom toast restrictions.
*/
	@ChangeId
	@EnabledAfter(targetSdkVersion = Build.VERSION_CODES.Q)
	private static final long CHANGE_TEXT_TOASTS_IN_THE_SYSTEM = 147798919L;
	private final Binder mToken;
	private final Context mContext;
	private final Handler mHandler;
	@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
	final TN mTN;
	@UnsupportedAppUsage
	int mDuration;
	/**
* This is also passed to {@link TN} object, where it's also accessed with itself as its own
* lock.
*/
	@GuardedBy("mCallbacks")
	private final List<Callback> mCallbacks;
	/**
* View to be displayed, in case this is a custom toast (e.g. not created with {@link
* #makeText(Context, int, int)} or its variants).
*/
	@Nullable
	private View mNextView;
	/**
* Text to be shown, in case this is NOT a custom toast (e.g. created with {@link
* #makeText(Context, int, int)} or its variants).
*/
	@Nullable
	private CharSequence mText;
	/**
* Construct an empty Toast object.  You must call {@link #setView} before you
* can call {@link #show}.
*
* @param context  The context to use.  Usually your {@link android.app.Application}
*                 or {@link android.app.Activity} object.
*/
	public Toast(Context context) {
		this(context, null);
	}
	/**
* Constructs an empty Toast object.  If looper is null, Looper.myLooper() is used.
* @hide
*/
	public Toast(@NonNull Context context, @Nullable Looper looper) {
		mContext = context;
		mToken = new Binder();
		looper = getLooper(looper);
		mHandler = new Handler(looper);
		mCallbacks = new ArrayList<>();
		mTN = new TN(context, context.getPackageName(), mToken,
		mCallbacks, looper);
		mTN.mY = context.getResources().getDimensionPixelSize(
		com.android.internal.R.dimen.toast_y_offset);
		mTN.mGravity = context.getResources().getInteger(
		com.android.internal.R.integer.config_toastDefaultGravity);
	}
	private Looper getLooper(@Nullable Looper looper) {
		if (looper != null) {
			return looper;
		}
		return checkNotNull(Looper.myLooper(),
		"Can't toast on a thread that has not called Looper.prepare()");
	}
	/**
* Show the view for the specified duration.
*/
	public void show() {
		if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
			checkState(mNextView != null || mText != null, "You must either set a text or a view");
		} else {
			if (mNextView == null) {
				throw new RuntimeException("setView must have been called");
			}
		}
		INotificationManager service = getService();
		String pkg = mContext.getOpPackageName();
		TN tn = mTN;
		tn.mNextView = mNextView;
		final int displayId = mContext.getDisplayId();
		try {
			if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
				if (mNextView != null) {
					// It's a custom toast
					service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
				} else {
					// It's a text toast
					ITransientNotificationCallback callback =
					new CallbackBinder(mCallbacks, mHandler);
					service.enqueueTextToast(pkg, mToken, mText, mDuration, displayId, callback);
				}
			} else {
				service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
			}
		}
		catch (RemoteException e) {
			// Empty
		}
	}
	/**
* Close the view if it's showing, or don't show it if it isn't showing yet.
* You do not normally have to call this.  Normally view will disappear on its own
* after the appropriate duration.
*/
	public void cancel() {
		if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)
		&& mNextView == null) {
			try {
				getService().cancelToast(mContext.getOpPackageName(), mToken);
			}
			catch (RemoteException e) {
				// Empty
			}
		} else {
			mTN.cancel();
		}
	}
	......
}
```

#### show() 就是调用 Toast 显示的方法  
最终调用 service.enqueueToast(pkg, mToken, tn, mDuration, displayId); 显示 其实就是 NotificationManagerService.java  
来负责处理显示 Toast 的，接下来看 NotificationManagerService.java 的相关代码

#### 3.2 NotificationManagerService.java 负责处理 Toast 的相关代码

```
public class NotificationManagerService extends SystemService {
	public static final String TAG = "NotificationService";
	public static final Boolean DBG = Log.isLoggable(TAG, Log.DEBUG);
	public static final Boolean ENABLE_CHILD_NOTIFICATIONS
	= SystemProperties.getBoolean("debug.child_notifs", true);
	// pullStats report request: undecorated remote view stats
	public static final int REPORT_REMOTE_VIEWS = 0x01;
	static final Boolean DEBUG_INTERRUPTIVENESS = SystemProperties.getBoolean(
	"debug.notification.interruptiveness", false);
	static final int MAX_PACKAGE_NOTIFICATIONS = 50;
	static final float DEFAULT_MAX_NOTIFICATION_ENQUEUE_RATE = 5f;
	// message codes
	static final int MESSAGE_DURATION_REACHED = 2;
	// 3: removed to a different handler
	static final int MESSAGE_SEND_RANKING_UPDATE = 4;
	static final int MESSAGE_LISTENER_HINTS_CHANGED = 5;
	static final int MESSAGE_LISTENER_NOTIFICATION_FILTER_CHANGED = 6;
	static final int MESSAGE_FINISH_TOKEN_TIMEOUT = 7;
	static final int MESSAGE_ON_PACKAGE_CHANGED = 8;
	// ranking thread messages
	private static final int MESSAGE_RECONSIDER_RANKING = 1000;
	private static final int MESSAGE_RANKING_SORT = 1001;
	static final int LONG_DELAY = PhoneWindowManager.TOAST_WINDOW_TIMEOUT;
	static final int SHORT_DELAY = 2000;
	// 2 seconds
	// 1 second past the ANR timeout.
	static final int FINISH_TOKEN_TIMEOUT = 11 * 1000;
	static final long[] DEFAULT_VIBRATE_PATTERN = {0, 250, 250, 250};
	static final long SNOOZE_UNTIL_UNSPECIFIED = -1;
	static final int VIBRATE_PATTERN_MAXLEN = 8 * 2 + 1;
	// up to eight bumps
	static final int INVALID_UID = -1;
	static final String ROOT_PKG = "root";
	static final Boolean ENABLE_BLOCKED_TOASTS = true;
	static final String[] DEFAULT_ALLOWED_ADJUSTMENTS = new String[] {
	Adjustment.KEY_CONTEXTUAL_ACTIONS,
	Adjustment.KEY_TEXT_REPLIES};
	static final String[] NON_BLOCKABLE_DEFAULT_ROLES = new String[] {
	RoleManager.ROLE_DIALER,
	RoleManager.ROLE_EMERGENCY
	};
	// When #matchesCallFilter is called from the ringer, wait at most
	// 3s to resolve the contacts. This timeout is required since
	// ContactsProvider might take a long time to start up.
	//
	// Return STARRED_CONTACT when the timeout is hit in order to avoid
	// missed calls in ZEN mode "Important".
	static final int MATCHES_CALL_FILTER_CONTACTS_TIMEOUT_MS = 3000;
	static final float MATCHES_CALL_FILTER_TIMEOUT_AFFINITY =
	ValidateNotificationPeople.STARRED_CONTACT;
	/** notification_enqueue status value for a newly enqueued notification. */
	private static final int EVENTLOG_ENQUEUE_STATUS_NEW = 0;
	/** notification_enqueue status value for an existing notification. */
	private static final int EVENTLOG_ENQUEUE_STATUS_UPDATE = 1;
	/** notification_enqueue status value for an ignored notification. */
	private static final int EVENTLOG_ENQUEUE_STATUS_IGNORED = 2;
	private static final long MIN_PACKAGE_OVERRATE_LOG_INTERVAL = 5000;
	// milliseconds
	private static final long DELAY_FOR_ASSISTANT_TIME = 200;
	private static final String ACTION_NOTIFICATION_TIMEOUT =
	NotificationManagerService.class.getSimpleName() + ".TIMEOUT";
	private static final int REQUEST_CODE_TIMEOUT = 1;
	private static final String SCHEME_TIMEOUT = "timeout";
	private static final String EXTRA_KEY = "key";
	private static final int NOTIFICATION_INSTANCE_ID_MAX = (1 << 13);
.....
	@VisibleForTesting
	final IBinder mService = new INotificationManager.Stub() {
		// Toasts
		// ============================================================================
		@Override
		public void enqueueTextToast(String pkg, IBinder token, CharSequence text, int duration,
		int displayId, @Nullable ITransientNotificationCallback callback) {
			enqueueToast(pkg, token, text, null, duration, displayId, callback);
		}
		@Override
		public void enqueueToast(String pkg, IBinder token, ITransientNotification callback,
		int duration, int displayId) {
			enqueueToast(pkg, token, null, callback, duration, displayId, null);
		}
		private void enqueueToast(String pkg, IBinder token, @Nullable CharSequence text,
		@Nullable ITransientNotification callback, int duration, int displayId,
		@Nullable ITransientNotificationCallback textCallback) {
			if (DBG) {
				Slog.i(TAG, "enqueueToast pkg=" + pkg + " token=" + token
				+ " duration=" + duration + " displayId=" + displayId);
			}
			if (pkg == null || (text == null && callback == null)
			|| (text != null && callback != null) || token == null) {
				Slog.e(TAG, "Not enqueuing toast. pkg=" + pkg + " text=" + text + " callback="
				+ " token=" + token);
				return;
			}
			final int callingUid = Binder.getCallingUid();
			final UserHandle callingUser = Binder.getCallingUserHandle();
			final Boolean isSystemToast = isCallerSystemOrPhone()
			|| PackageManagerService.PLATFORM_PACKAGE_NAME.equals(pkg);
			final Boolean isPackageSuspended = isPackagePaused(pkg);
			final Boolean notificationsDisabledForPackage = !areNotificationsEnabledForPackage(pkg,
			callingUid);
			final Boolean appIsForeground;
			long callingIdentity = Binder.clearCallingIdentity();
			try {
				appIsForeground = mActivityManager.getUidImportance(callingUid)
				== IMPORTANCE_FOREGROUND;
			}
			finally {
				Binder.restoreCallingIdentity(callingIdentity);
			}
			if (ENABLE_BLOCKED_TOASTS && !isSystemToast && ((notificationsDisabledForPackage
			&& !appIsForeground) || isPackageSuspended)) {
				Slog.e(TAG, "Suppressing toast from package " + pkg
				+ (isPackageSuspended ? " due to package suspended."
				: " by user request."));
				return;
			}
			Boolean isAppRenderedToast = (callback != null);
			if (isAppRenderedToast && !isSystemToast && !isPackageInForegroundForToast(pkg,
			callingUid)) {
				Boolean block;
				long id = Binder.clearCallingIdentity();
				try {
					// CHANGE_BACKGROUND_CUSTOM_TOAST_BLOCK is gated on targetSdk, so block will be
					// false for apps with targetSdk < R. For apps with targetSdk R+, text toasts
					// are not app-rendered, so isAppRenderedToast == true means it's a custom
					// toast.
					block = mPlatformCompat.isChangeEnabledByPackageName(
					CHANGE_BACKGROUND_CUSTOM_TOAST_BLOCK, pkg,
					callingUser.getIdentifier());
				}
				catch (RemoteException e) {
					// Shouldn't happen have since it's a local local
					Slog.e(TAG, "Unexpected exception while checking block background custom toasts"
					+ " change", e);
					block = false;
				}
				finally {
					Binder.restoreCallingIdentity(id);
				}
				if (block) {
					Slog.w(TAG, "Blocking custom toast from package " + pkg
					+ " due to package not in the foreground");
					return;
				}
			}
			synchronized (mToastQueue) {
				int callingPid = Binder.getCallingPid();
				long callingId = Binder.clearCallingIdentity();
				try {
					ToastRecord record;
					int index = indexOfToastLocked(pkg, token);
					// If it's already in the queue, we update it in place, we don't
					// move it to the end of the queue.
					if (index >= 0) {
						record = mToastQueue.get(index);
						record.update(duration);
					} else {
						// Limit the number of toasts that any given package can enqueue.
						// Prevents DOS attacks and deals with leaks.
						int count = 0;
						final int N = mToastQueue.size();
						for (int i = 0; i < N; i++) {
							final ToastRecord r = mToastQueue.get(i);
							if (r.pkg.equals(pkg)) {
								count++;
								if (count >= MAX_PACKAGE_NOTIFICATIONS) {
									Slog.e(TAG, "Package has already posted " + count
									+ " toasts. Not showing more. Package=" + pkg);
									return;
								}
							}
						}
						Binder windowToken = new Binder();
						mWindowManagerInternal.addWindowToken(windowToken, TYPE_TOAST, displayId);
						record = getToastRecord(callingUid, callingPid, pkg, token, text, callback,
						duration, windowToken, displayId, textCallback);
						mToastQueue.add(record);
						index = mToastQueue.size() - 1;
						keepProcessAliveForToastIfNeededLocked(callingPid);
					}
					// If it's at index 0, it's the current toast.  It doesn't matter if it's
					// new or just been updated, show it.
					// If the callback fails, this will remove it from the list, so don't
					// assume that it's valid after this.
					if (index == 0) {
						showNextToastLocked();
					}
				}
				finally {
					Binder.restoreCallingIdentity(callingId);
				}
			}
		}
		....
	}
@GuardedBy("mToastQueue")
private void scheduleDurationReachedLocked(ToastRecord r)
{
	mHandler.removeCallbacksAndMessages(r);
	Message m = Message.obtain(mHandler, MESSAGE_DURATION_REACHED, r);
	int delay = r.getDuration() == Toast.LENGTH_lONG ? LONG_DELAY : SHORT_DELAY;
	// Accessibility users may need longer timeout duration. This api compares original delay
	// with user's preference and return longer one. It returns original delay if there's no
	// preference.
	delay = mAccessibilityManager.getRecommendedTimeoutMillis(delay,
	AccessibilityManager.FLAG_CONTENT_TEXT);
	mHandler.sendMessageDelayed(m, delay);
}
```

最终执行 int delay = r.getDuration() == Toast.LENGTH_lONG ? LONG_DELAY : SHORT_DELAY; 来设置显示时间

所以最后显示时间可以修改  
static final int LONG_DELAY = 5000;//5seconds  
static final int SHORT_DELAY = 3500; // 3.5seconds