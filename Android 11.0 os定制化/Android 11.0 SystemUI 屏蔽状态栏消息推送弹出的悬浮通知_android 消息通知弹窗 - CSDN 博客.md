> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125753910)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 屏蔽状态栏消息推送弹出的悬浮通知核心代码](#t1)

[3.SystemUI 屏蔽状态栏消息推送弹出的悬浮通知核心代码分析](#t2)

[3.1NotificationRowContentBinder.java 关于各种通知类型分析](#t3)

 [3.2 NotificationContentInflater.java 中构建通知的相关代码分析](#t4)

[3.3 NotificationMediaManager.java 相关悬浮通知的代码分析](#t5)

[4.NotificationMediaManager.java 屏蔽悬浮通知的功能实现](#t6)

1. 概述
-----

在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的各种定制化也是系统上层常用的功能，状态栏推送消息弹出悬浮通知这个也是在截图闹钟连接特殊 wifi 等功能上  
有的功能，会在截图后 闹钟的时候 和连接一些特殊的 wifi 弹出悬浮通知, 根据开发需求要求屏蔽掉这些悬浮通知，这首先要分析  
这些悬浮通知是怎么弹出来的然后屏蔽掉就可以了

2.SystemUI 屏蔽状态栏[消息推送](https://so.csdn.net/so/search?q=%E6%B6%88%E6%81%AF%E6%8E%A8%E9%80%81&spm=1001.2101.3001.7020)弹出的悬浮通知核心代码
-------------------------------------------------------------------------------------------------------------------------------

```
主要代码:
 frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/NotificationRowContentBinder.java
 frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/NotificationContentInflater.java
 frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationMediaManager.java
```

3.SystemUI 屏蔽状态栏消息推送弹出的悬浮通知核心代码分析
---------------------------------

#### 3.1NotificationRowContentBinder.java 关于各种通知类型分析

```
 public interface NotificationRowContentBinder {
 
/**
* Inflate notification content views and bind to the row.
*
* @param entry notification
* @param row notification row to bind views to
* @param contentToBind content views that should be inflated and bound
* @param bindParams parameters for binding content views
* @param forceInflate true to force reinflation even if views are cached
* @param callback callback after inflation is finished
*/
void bindContent(
@NonNull NotificationEntry entry,
@NonNull ExpandableNotificationRow row,
@InflationFlag int contentToBind,
BindParams bindParams,
boolean forceInflate,
@Nullable InflationCallback callback);
 
/**
* Cancel any on-going bind operation.
*
* @param entry notification
* @param row notification row to cancel bind on
*/
void cancelBind(
@NonNull NotificationEntry entry,
@NonNull ExpandableNotificationRow row);
 
/**
* Unbind content views from the row.
*
* @param entry notification
* @param row notification row to unbind content views from
* @param contentToUnbind content views that should be unbound
*/
void unbindContent(
@NonNull NotificationEntry entry,
@NonNull ExpandableNotificationRow row,
@InflationFlag int contentToUnbind);
 
@Retention(RetentionPolicy.SOURCE)
@IntDef(flag = true,
prefix = {"FLAG_CONTENT_VIEW_"},
value = {
FLAG_CONTENT_VIEW_CONTRACTED,
FLAG_CONTENT_VIEW_EXPANDED,
FLAG_CONTENT_VIEW_HEADS_UP,
FLAG_CONTENT_VIEW_PUBLIC,
FLAG_CONTENT_VIEW_ALL})
@interface InflationFlag {}
/**
* The default, contracted view.  Seen when the shade is pulled down and in the lock screen
* if there is no worry about content sensitivity.
*/
int FLAG_CONTENT_VIEW_CONTRACTED = 1;
/**
* The expanded view.  Seen when the user expands a notification.
*/
int FLAG_CONTENT_VIEW_EXPANDED = 1 << 1;
/**
* The heads up view.  Seen when a high priority notification peeks in from the top.
*/
int FLAG_CONTENT_VIEW_HEADS_UP = 1 << 2;
/**
* The public view.  This is a version of the contracted view that hides sensitive
* information and is used on the lock screen if we determine that the notification's
* content should be hidden.
*/
int FLAG_CONTENT_VIEW_PUBLIC = 1 << 3;
int FLAG_CONTENT_VIEW_ALL = (1 << 4) - 1;
/**
* Parameters for content view binding
*/
class BindParams {
/**
* Bind a low priority version of the content views.
*/
public boolean isLowPriority;
/**
* Use increased height when binding contracted view.
*/
public boolean usesIncreasedHeight;
/**
* Use increased height when binding heads up views.
*/
public boolean usesIncreasedHeadsUpHeight;
}
/**
* Callback for inflation finishing
*/
interface InflationCallback {
/**
* Callback for when there is an inflation exception
*
* @param entry notification which failed to inflate content
* @param e exception
*/
void handleInflationException(NotificationEntry entry, Exception e);
/**
* Callback for after the content views finish inflating.
*
* @param entry the entry with the content views set
*/
void onAsyncInflationFinished(NotificationEntry entry);
}
}
在NotificationContentInflater.java中根据通知类型构建各种各样的通知
```

####  3.2 NotificationContentInflater.java 中构建通知的相关代码分析

```
@Singleton
@VisibleForTesting(visibility = PACKAGE)
public class NotificationContentInflater implements NotificationRowContentBinder {
 
public static final String TAG = "NotifContentInflater";
 
private boolean mInflateSynchronously = false;
private final boolean mIsMediaInQS;
private final NotificationRemoteInputManager mRemoteInputManager;
private final NotifRemoteViewCache mRemoteViewCache;
private final Lazy<SmartReplyConstants> mSmartReplyConstants;
private final Lazy<SmartReplyController> mSmartReplyController;
private final ConversationNotificationProcessor mConversationProcessor;
private final Executor mBgExecutor;
 
@Inject
NotificationContentInflater(
NotifRemoteViewCache remoteViewCache,
NotificationRemoteInputManager remoteInputManager,
Lazy<SmartReplyConstants> smartReplyConstants,
Lazy<SmartReplyController> smartReplyController,
ConversationNotificationProcessor conversationProcessor,
MediaFeatureFlag mediaFeatureFlag,
@Background Executor bgExecutor) {
mRemoteViewCache = remoteViewCache;
mRemoteInputManager = remoteInputManager;
mSmartReplyConstants = smartReplyConstants;
mSmartReplyController = smartReplyController;
mConversationProcessor = conversationProcessor;
mIsMediaInQS = mediaFeatureFlag.getEnabled();
mBgExecutor = bgExecutor;
}
 
@Override
public void bindContent(
NotificationEntry entry,
ExpandableNotificationRow row,
@InflationFlag int contentToBind,
BindParams bindParams,
boolean forceInflate,
@Nullable InflationCallback callback) {
if (row.isRemoved()) {
// We don't want to reinflate anything for removed notifications. Otherwise views might
// be readded to the stack, leading to leaks. This may happen with low-priority groups
// where the removal of already removed children can lead to a reinflation.
return;
}
StatusBarNotification sbn = entry.getSbn();
// To check if the notification has inline image and preload inline image if necessary.
row.getImageResolver().preloadImages(sbn.getNotification());
if (forceInflate) {
mRemoteViewCache.clearCache(entry);
}
// Cancel any pending frees on any view we're trying to bind since we should be bound after.
cancelContentViewFrees(row, contentToBind);
 
AsyncInflationTask task = new AsyncInflationTask(
mBgExecutor,
mInflateSynchronously,
contentToBind,
mRemoteViewCache,
entry,
mSmartReplyConstants.get(),
mSmartReplyController.get(),
mConversationProcessor,
row,
bindParams.isLowPriority,
bindParams.usesIncreasedHeight,
bindParams.usesIncreasedHeadsUpHeight,
callback,
mRemoteInputManager.getRemoteViewsOnClickHandler(),
mIsMediaInQS);
if (mInflateSynchronously) {
task.onPostExecute(task.doInBackground());
} else {
task.executeOnExecutor(mBgExecutor);
}
}
 
@VisibleForTesting
InflationProgress inflateNotificationViews(
NotificationEntry entry,
ExpandableNotificationRow row,
BindParams bindParams,
boolean inflateSynchronously,
@InflationFlag int reInflateFlags,
Notification.Builder builder,
Context packageContext) {
InflationProgress result = createRemoteViews(reInflateFlags,
builder,
bindParams.isLowPriority,
bindParams.usesIncreasedHeight,
bindParams.usesIncreasedHeadsUpHeight,
packageContext);
result = inflateSmartReplyViews(result, reInflateFlags, entry,
row.getContext(), packageContext, row.getHeadsUpManager(),
mSmartReplyConstants.get(), mSmartReplyController.get(),
row.getExistingSmartRepliesAndActions());
 
apply(
mBgExecutor,
inflateSynchronously,
result,
reInflateFlags,
mRemoteViewCache,
entry,
row,
mRemoteInputManager.getRemoteViewsOnClickHandler(),
null);
return result;
}
 
@Override
public void cancelBind(
@NonNull NotificationEntry entry,
@NonNull ExpandableNotificationRow row) {
entry.abortTask();
}
 
@Override
public void unbindContent(
@NonNull NotificationEntry entry,
@NonNull ExpandableNotificationRow row,
@InflationFlag int contentToUnbind) {
int curFlag = 1;
while (contentToUnbind != 0) {
if ((contentToUnbind & curFlag) != 0) {
freeNotificationView(entry, row, curFlag);
}
contentToUnbind &= ~curFlag;
curFlag = curFlag << 1;
}
}
 
/**
* Frees the content view associated with the inflation flag as soon as the view is not showing.
*
* @param inflateFlag the flag corresponding to the content view which should be freed
*/
private void freeNotificationView(
NotificationEntry entry,
ExpandableNotificationRow row,
@InflationFlag int inflateFlag) {
switch (inflateFlag) {
case FLAG_CONTENT_VIEW_CONTRACTED:
row.getPrivateLayout().performWhenContentInactive(VISIBLE_TYPE_CONTRACTED, () -> {
row.getPrivateLayout().setContractedChild(null);
mRemoteViewCache.removeCachedView(entry, FLAG_CONTENT_VIEW_CONTRACTED);
});
break;
case FLAG_CONTENT_VIEW_EXPANDED:
row.getPrivateLayout().performWhenContentInactive(VISIBLE_TYPE_EXPANDED, () -> {
row.getPrivateLayout().setExpandedChild(null);
mRemoteViewCache.removeCachedView(entry, FLAG_CONTENT_VIEW_EXPANDED);
});
break;
case FLAG_CONTENT_VIEW_HEADS_UP:
row.getPrivateLayout().performWhenContentInactive(VISIBLE_TYPE_HEADSUP, () -> {
row.getPrivateLayout().setHeadsUpChild(null);
mRemoteViewCache.removeCachedView(entry, FLAG_CONTENT_VIEW_HEADS_UP);
row.getPrivateLayout().setHeadsUpInflatedSmartReplies(null);
});
break;
case FLAG_CONTENT_VIEW_PUBLIC:
row.getPublicLayout().performWhenContentInactive(VISIBLE_TYPE_CONTRACTED, () -> {
row.getPublicLayout().setContractedChild(null);
mRemoteViewCache.removeCachedView(entry, FLAG_CONTENT_VIEW_PUBLIC);
});
break;
default:
break;
}
}
 
/**
* Cancel any pending content view frees from {@link #freeNotificationView} for the provided
* content views.
*
* @param row top level notification row containing the content views
* @param contentViews content views to cancel pending frees on
*/
private void cancelContentViewFrees(
ExpandableNotificationRow row,
@InflationFlag int contentViews) {
if ((contentViews & FLAG_CONTENT_VIEW_CONTRACTED) != 0) {
row.getPrivateLayout().removeContentInactiveRunnable(VISIBLE_TYPE_CONTRACTED);
}
if ((contentViews & FLAG_CONTENT_VIEW_EXPANDED) != 0) {
row.getPrivateLayout().removeContentInactiveRunnable(VISIBLE_TYPE_EXPANDED);
}
if ((contentViews & FLAG_CONTENT_VIEW_HEADS_UP) != 0) {
row.getPrivateLayout().removeContentInactiveRunnable(VISIBLE_TYPE_HEADSUP);
}
if ((contentViews & FLAG_CONTENT_VIEW_PUBLIC) != 0) {
row.getPublicLayout().removeContentInactiveRunnable(VISIBLE_TYPE_CONTRACTED);
}
}
 
private static InflationProgress inflateSmartReplyViews(InflationProgress result,
@InflationFlag int reInflateFlags, NotificationEntry entry, Context context,
Context packageContext, HeadsUpManager headsUpManager,
SmartReplyConstants smartReplyConstants, SmartReplyController smartReplyController,
SmartRepliesAndActions previousSmartRepliesAndActions) {
if ((reInflateFlags & FLAG_CONTENT_VIEW_EXPANDED) != 0 && result.newExpandedView != null) {
result.expandedInflatedSmartReplies =
InflatedSmartReplies.inflate(
context, packageContext, entry, smartReplyConstants,
smartReplyController, headsUpManager, previousSmartRepliesAndActions);
}
if ((reInflateFlags & FLAG_CONTENT_VIEW_HEADS_UP) != 0 && result.newHeadsUpView != null) {
result.headsUpInflatedSmartReplies =
InflatedSmartReplies.inflate(
context, packageContext, entry, smartReplyConstants,
smartReplyController, headsUpManager, previousSmartRepliesAndActions);
}
return result;
}
 
private static InflationProgress createRemoteViews(@InflationFlag int reInflateFlags,
Notification.Builder builder, boolean isLowPriority, boolean usesIncreasedHeight,
boolean usesIncreasedHeadsUpHeight, Context packageContext) {
InflationProgress result = new InflationProgress();
 
if ((reInflateFlags & FLAG_CONTENT_VIEW_CONTRACTED) != 0) {
result.newContentView = createContentView(builder, isLowPriority, usesIncreasedHeight);
}
 
if ((reInflateFlags & FLAG_CONTENT_VIEW_EXPANDED) != 0) {
result.newExpandedView = createExpandedView(builder, isLowPriority);
}
 
if ((reInflateFlags & FLAG_CONTENT_VIEW_HEADS_UP) != 0) {
result.newHeadsUpView = builder.createHeadsUpContentView(usesIncreasedHeadsUpHeight);
}
 
if ((reInflateFlags & FLAG_CONTENT_VIEW_PUBLIC) != 0) {
result.newPublicView = builder.makePublicContentView(isLowPriority);
}
 
result.packageContext = packageContext;
result.headsUpStatusBarText = builder.getHeadsUpStatusBarText(false /* showingPublic */);
result.headsUpStatusBarTextPublic = builder.getHeadsUpStatusBarText(
true /* showingPublic */);
return result;
}
 
private static CancellationSignal apply(
Executor bgExecutor,
boolean inflateSynchronously,
InflationProgress result,
@InflationFlag int reInflateFlags,
NotifRemoteViewCache remoteViewCache,
NotificationEntry entry,
ExpandableNotificationRow row,
RemoteViews.OnClickHandler remoteViewClickHandler,
@Nullable InflationCallback callback) {
NotificationContentView privateLayout = row.getPrivateLayout();
NotificationContentView publicLayout = row.getPublicLayout();
final HashMap<Integer, CancellationSignal> runningInflations = new HashMap<>();
 
int flag = FLAG_CONTENT_VIEW_CONTRACTED;
if ((reInflateFlags & flag) != 0) {
boolean isNewView =
!canReapplyRemoteView(result.newContentView,
remoteViewCache.getCachedView(entry, FLAG_CONTENT_VIEW_CONTRACTED));
ApplyCallback applyCallback = new ApplyCallback() {
@Override
public void setResultView(View v) {
result.inflatedContentView = v;
}
 
@Override
public RemoteViews getRemoteView() {
return result.newContentView;
}
};
applyRemoteView(bgExecutor, inflateSynchronously, result, reInflateFlags, flag,
remoteViewCache, entry, row, isNewView, remoteViewClickHandler, callback,
privateLayout,  privateLayout.getContractedChild(),
privateLayout.getVisibleWrapper(
NotificationContentView.VISIBLE_TYPE_CONTRACTED),
runningInflations, applyCallback);
}
 
flag = FLAG_CONTENT_VIEW_EXPANDED;
if ((reInflateFlags & flag) != 0) {
if (result.newExpandedView != null) {
boolean isNewView =
!canReapplyRemoteView(result.newExpandedView,
remoteViewCache.getCachedView(entry, FLAG_CONTENT_VIEW_EXPANDED));
ApplyCallback applyCallback = new ApplyCallback() {
@Override
public void setResultView(View v) {
result.inflatedExpandedView = v;
}
 
@Override
public RemoteViews getRemoteView() {
return result.newExpandedView;
}
};
applyRemoteView(bgExecutor, inflateSynchronously, result, reInflateFlags, flag,
remoteViewCache, entry, row, isNewView, remoteViewClickHandler,
callback, privateLayout, privateLayout.getExpandedChild(),
privateLayout.getVisibleWrapper(
NotificationContentView.VISIBLE_TYPE_EXPANDED), runningInflations,
applyCallback);
}
}
 
flag = FLAG_CONTENT_VIEW_HEADS_UP;
if ((reInflateFlags & flag) != 0) {
if (result.newHeadsUpView != null) {
boolean isNewView =
!canReapplyRemoteView(result.newHeadsUpView,
remoteViewCache.getCachedView(entry, FLAG_CONTENT_VIEW_HEADS_UP));
ApplyCallback applyCallback = new ApplyCallback() {
@Override
public void setResultView(View v) {
result.inflatedHeadsUpView = v;
}
 
@Override
public RemoteViews getRemoteView() {
return result.newHeadsUpView;
}
};
applyRemoteView(bgExecutor, inflateSynchronously, result, reInflateFlags, flag,
remoteViewCache, entry, row, isNewView, remoteViewClickHandler,
callback, privateLayout, privateLayout.getHeadsUpChild(),
privateLayout.getVisibleWrapper(
VISIBLE_TYPE_HEADSUP), runningInflations,
applyCallback);
}
}
 
flag = FLAG_CONTENT_VIEW_PUBLIC;
if ((reInflateFlags & flag) != 0) {
boolean isNewView =
!canReapplyRemoteView(result.newPublicView,
remoteViewCache.getCachedView(entry, FLAG_CONTENT_VIEW_PUBLIC));
ApplyCallback applyCallback = new ApplyCallback() {
@Override
public void setResultView(View v) {
result.inflatedPublicView = v;
}
 
@Override
public RemoteViews getRemoteView() {
return result.newPublicView;
}
};
applyRemoteView(bgExecutor, inflateSynchronously, result, reInflateFlags, flag,
remoteViewCache, entry, row, isNewView, remoteViewClickHandler, callback,
publicLayout, publicLayout.getContractedChild(),
publicLayout.getVisibleWrapper(NotificationContentView.VISIBLE_TYPE_CONTRACTED),
runningInflations, applyCallback);
}
 
// Let's try to finish, maybe nobody is even inflating anything
finishIfDone(result, reInflateFlags, remoteViewCache, runningInflations, callback, entry,
row);
CancellationSignal cancellationSignal = new CancellationSignal();
cancellationSignal.setOnCancelListener(
() -> runningInflations.values().forEach(CancellationSignal::cancel));
return cancellationSignal;
}
@VisibleForTesting
static void applyRemoteView(
Executor bgExecutor,
boolean inflateSynchronously,
final InflationProgress result,
final @InflationFlag int reInflateFlags,
@InflationFlag int inflationId,
final NotifRemoteViewCache remoteViewCache,
final NotificationEntry entry,
final ExpandableNotificationRow row,
boolean isNewView,
RemoteViews.OnClickHandler remoteViewClickHandler,
@Nullable final InflationCallback callback,
NotificationContentView parentLayout,
View existingView,
NotificationViewWrapper existingWrapper,
final HashMap<Integer, CancellationSignal> runningInflations,
ApplyCallback applyCallback) {
RemoteViews newContentView = applyCallback.getRemoteView();
if (inflateSynchronously) {
try {
if (isNewView) {
View v = newContentView.apply(
result.packageContext,
parentLayout,
remoteViewClickHandler);
v.setIsRootNamespace(true);
applyCallback.setResultView(v);
} else {
newContentView.reapply(
result.packageContext,
existingView,
remoteViewClickHandler);
existingWrapper.onReinflated();
}
} catch (Exception e) {
handleInflationError(runningInflations, e, row.getEntry(), callback);
// Add a running inflation to make sure we don't trigger callbacks.
// Safe to do because only happens in tests.
runningInflations.put(inflationId, new CancellationSignal());
}
return;
}
RemoteViews.OnViewAppliedListener listener = new RemoteViews.OnViewAppliedListener() {
 
@Override
public void onViewInflated(View v) {
if (v instanceof ImageMessageConsumer) {
((ImageMessageConsumer) v).setImageResolver(row.getImageResolver());
}
}
 
@Override
public void onViewApplied(View v) {
if (isNewView) {
v.setIsRootNamespace(true);
applyCallback.setResultView(v);
} else if (existingWrapper != null) {
existingWrapper.onReinflated();
}
runningInflations.remove(inflationId);
finishIfDone(result, reInflateFlags, remoteViewCache, runningInflations,
callback, entry, row);
}
 
@Override
public void onError(Exception e) {
// Uh oh the async inflation failed. Due to some bugs (see b/38190555), this could
// actually also be a system issue, so let's try on the UI thread again to be safe.
try {
View newView = existingView;
if (isNewView) {
newView = newContentView.apply(
result.packageContext,
parentLayout,
remoteViewClickHandler);
} else {
newContentView.reapply(
result.packageContext,
existingView,
remoteViewClickHandler);
}
Log.wtf(TAG, "Async Inflation failed but normal inflation finished normally.",
e);
onViewApplied(newView);
} catch (Exception anotherException) {
runningInflations.remove(inflationId);
handleInflationError(runningInflations, e, row.getEntry(),
callback);
}
}
};
CancellationSignal cancellationSignal;
if (isNewView) {
cancellationSignal = newContentView.applyAsync(
result.packageContext,
parentLayout,
bgExecutor,
listener,
remoteViewClickHandler);
} else {
cancellationSignal = newContentView.reapplyAsync(
result.packageContext,
existingView,
bgExecutor,
listener,
remoteViewClickHandler);
}
runningInflations.put(inflationId, cancellationSignal);
}
.....
}
    在createRemoteViews(@InflationFlag int reInflateFlags,
Notification.Builder builder, boolean isLowPriority, boolean usesIncreasedHeight,
boolean usesIncreasedHeadsUpHeight, Context packageContext) 
中根据不同通知类型构建通知
```

#### 3.3 NotificationMediaManager.java 相关悬浮通知的代码分析

```
NotificationMediaManager.java构造函数中给NotificationEntryManager 加 NotificationEntryListener ，以便在通知视图首次填充(onEntryInflated) 时 感知并弹出悬浮通知。
NotificationMediaManager.java 负责管理弹出悬浮通知
 
/**
* Injected constructor. See {@link StatusBarModule}.
*/
public NotificationMediaManager(
Context context,
Lazy<StatusBar> statusBarLazy,
Lazy<NotificationShadeWindowController> notificationShadeWindowController,
NotificationEntryManager notificationEntryManager,
MediaArtworkProcessor mediaArtworkProcessor,
KeyguardBypassController keyguardBypassController,
@Main DelayableExecutor mainExecutor,
DeviceConfigProxy deviceConfig,
MediaDataManager mediaDataManager) {
mContext = context;
mMediaArtworkProcessor = mediaArtworkProcessor;
mKeyguardBypassController = keyguardBypassController;
mMediaListeners = new ArrayList<>();
// TODO: use MediaSessionManager.SessionListener to hook us up to future updates
// in session state
mMediaSessionManager = (MediaSessionManager) mContext.getSystemService(
Context.MEDIA_SESSION_SERVICE);
// TODO: use KeyguardStateController#isOccluded to remove this dependency
mStatusBarLazy = statusBarLazy;
mNotificationShadeWindowController = notificationShadeWindowController;
mEntryManager = notificationEntryManager;
mMainExecutor = mainExecutor;
mMediaDataManager = mediaDataManager;
 
notificationEntryManager.addNotificationEntryListener(new NotificationEntryListener() {
 
@Override
public void onPendingEntryAdded(NotificationEntry entry) {
mediaDataManager.onNotificationAdded(entry.getKey(), entry.getSbn());
}
 
@Override
public void onPreEntryUpdated(NotificationEntry entry) {
mediaDataManager.onNotificationAdded(entry.getKey(), entry.getSbn());
}
 
@Override
public void onEntryInflated(NotificationEntry entry) {
findAndUpdateMediaNotifications();
}
 
@Override
public void onEntryReinflated(NotificationEntry entry) {
findAndUpdateMediaNotifications();
}
 
@Override
public void onEntryRemoved(
NotificationEntry entry,
NotificationVisibility visibility,
boolean removedByUser,
int reason) {
removeEntry(entry);
}
});
 
// Pending entries are never inflated, and will never generate a call to onEntryRemoved().
// This can happen when notifications are added and canceled before inflation. Add this
// separate listener for cleanup, since media inflation occurs onPendingEntryAdded().
notificationEntryManager.addCollectionListener(new NotifCollectionListener() {
@Override
public void onEntryCleanUp(@NonNull NotificationEntry entry) {
removeEntry(entry);
}
});
 
mShowCompactMediaSeekbar = "true".equals(
DeviceConfig.getProperty(DeviceConfig.NAMESPACE_SYSTEMUI,
SystemUiDeviceConfigFlags.COMPACT_MEDIA_SEEKBAR_ENABLED));
 
deviceConfig.addOnPropertiesChangedListener(DeviceConfig.NAMESPACE_SYSTEMUI,
mContext.getMainExecutor(),
mPropertiesChangedListener);
}
 
onEntryInflated(NotificationEntry entry)负责构建通知
所以屏蔽通知就是注释掉这里即可
```

4.NotificationMediaManager.java 屏蔽悬浮通知的功能实现
-------------------------------------------

```
public NotificationMediaManager(
Context context,
Lazy<StatusBar> statusBarLazy,
Lazy<NotificationShadeWindowController> notificationShadeWindowController,
NotificationEntryManager notificationEntryManager,
MediaArtworkProcessor mediaArtworkProcessor,
KeyguardBypassController keyguardBypassController,
@Main DelayableExecutor mainExecutor,
DeviceConfigProxy deviceConfig,
MediaDataManager mediaDataManager) {
mContext = context;
mMediaArtworkProcessor = mediaArtworkProcessor;
mKeyguardBypassController = keyguardBypassController;
mMediaListeners = new ArrayList<>();
// TODO: use MediaSessionManager.SessionListener to hook us up to future updates
// in session state
mMediaSessionManager = (MediaSessionManager) mContext.getSystemService(
Context.MEDIA_SESSION_SERVICE);
// TODO: use KeyguardStateController#isOccluded to remove this dependency
mStatusBarLazy = statusBarLazy;
mNotificationShadeWindowController = notificationShadeWindowController;
mEntryManager = notificationEntryManager;
mMainExecutor = mainExecutor;
mMediaDataManager = mediaDataManager;
 
notificationEntryManager.addNotificationEntryListener(new NotificationEntryListener() {
 
@Override
public void onPendingEntryAdded(NotificationEntry entry) {
mediaDataManager.onNotificationAdded(entry.getKey(), entry.getSbn());
}
 
@Override
public void onPreEntryUpdated(NotificationEntry entry) {
mediaDataManager.onNotificationAdded(entry.getKey(), entry.getSbn());
}
 
@Override
public void onEntryInflated(NotificationEntry entry) {
findAndUpdateMediaNotifications();
}
 
@Override
public void onEntryReinflated(NotificationEntry entry) {
//findAndUpdateMediaNotifications();
}
 
@Override
public void onEntryRemoved(
NotificationEntry entry,
NotificationVisibility visibility,
boolean removedByUser,
int reason) {
removeEntry(entry);
}
});
 
// Pending entries are never inflated, and will never generate a call to onEntryRemoved().
// This can happen when notifications are added and canceled before inflation. Add this
// separate listener for cleanup, since media inflation occurs onPendingEntryAdded().
notificationEntryManager.addCollectionListener(new NotifCollectionListener() {
@Override
public void onEntryCleanUp(@NonNull NotificationEntry entry) {
removeEntry(entry);
}
});
 
mShowCompactMediaSeekbar = "true".equals(
DeviceConfig.getProperty(DeviceConfig.NAMESPACE_SYSTEMUI,
SystemUiDeviceConfigFlags.COMPACT_MEDIA_SEEKBAR_ENABLED));
 
deviceConfig.addOnPropertiesChangedListener(DeviceConfig.NAMESPACE_SYSTEMUI,
mContext.getMainExecutor(),
mPropertiesChangedListener);
}
```