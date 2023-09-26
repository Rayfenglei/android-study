> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126237057)

1. 概述
-----

在 11.0 的产品定制化中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的定制也是常用的功能，而在下拉状态栏中的通知栏部分也是极其重要的部分，每条通知实时更新在通知栏部分，由于通知栏高度的限制，每条通知是默认收缩的，功能开发需要要求通知默认展开，所以就要从通知的加载流程分析

如图:

![](https://img-blog.csdnimg.cn/2b16cae79f9a437bb4c18f81c1890ab9.png)

 2. 通知栏通知设置默认展开的相关代码
--------------------

```
  /frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationViewHierarchyManager.java
  /frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBarNotificationPresenter.java
  frameworks\base\packages\SystemUI\res\values\config.xml
```

3. 通知栏通知设置默认展开的相关代码以及功能分析
-------------------------

####  3.1 StatusBarNotificationPresenter.java 关于通知栏通知的相关管理

```
public class StatusBarNotificationPresenter implements NotificationPresenter,
ConfigurationController.ConfigurationListener,
NotificationRowBinderImpl.BindRowCallback,
CommandQueue.Callbacks {
 
private final LockscreenGestureLogger mLockscreenGestureLogger =
Dependency.get(LockscreenGestureLogger.class);
 
private static final String TAG = "StatusBarNotificationPresenter";
 
private final ActivityStarter mActivityStarter = Dependency.get(ActivityStarter.class);
private final KeyguardStateController mKeyguardStateController;
private final NotificationViewHierarchyManager mViewHierarchyManager =
Dependency.get(NotificationViewHierarchyManager.class);
private final NotificationLockscreenUserManager mLockscreenUserManager =
Dependency.get(NotificationLockscreenUserManager.class);
private final SysuiStatusBarStateController mStatusBarStateController =
(SysuiStatusBarStateController) Dependency.get(StatusBarStateController.class);
private final NotificationEntryManager mEntryManager =
Dependency.get(NotificationEntryManager.class);
private final NotificationMediaManager mMediaManager =
Dependency.get(NotificationMediaManager.class);
private final VisualStabilityManager mVisualStabilityManager =
Dependency.get(VisualStabilityManager.class);
private final NotificationGutsManager mGutsManager =
Dependency.get(NotificationGutsManager.class);
 
private final NotificationPanelViewController mNotificationPanel;
private final HeadsUpManagerPhone mHeadsUpManager;
private final AboveShelfObserver mAboveShelfObserver;
private final DozeScrimController mDozeScrimController;
private final ScrimController mScrimController;
private final Context mContext;
private final KeyguardIndicationController mKeyguardIndicationController;
private final StatusBar mStatusBar;
private final ShadeController mShadeController;
private final CommandQueue mCommandQueue;
 
private final AccessibilityManager mAccessibilityManager;
private final KeyguardManager mKeyguardManager;
private final ActivityLaunchAnimator mActivityLaunchAnimator;
private final int mMaxAllowedKeyguardNotifications;
private final IStatusBarService mBarService;
private final DynamicPrivacyController mDynamicPrivacyController;
private boolean mReinflateNotificationsOnUserSwitched;
private boolean mDispatchUiModeChangeOnUserSwitched;
private TextView mNotificationPanelDebugText;
 
protected boolean mVrMode;
private int mMaxKeyguardNotifications;
 
public StatusBarNotificationPresenter(Context context,
NotificationPanelViewController panel,
HeadsUpManagerPhone headsUp,
NotificationShadeWindowView statusBarWindow,
ViewGroup stackScroller,
DozeScrimController dozeScrimController,
ScrimController scrimController,
ActivityLaunchAnimator activityLaunchAnimator,
DynamicPrivacyController dynamicPrivacyController,
KeyguardStateController keyguardStateController,
KeyguardIndicationController keyguardIndicationController,
StatusBar statusBar,
ShadeController shadeController,
CommandQueue commandQueue,
InitController initController,
NotificationInterruptStateProvider notificationInterruptStateProvider) {
mContext = context;
mKeyguardStateController = keyguardStateController;
mNotificationPanel = panel;
mHeadsUpManager = headsUp;
mDynamicPrivacyController = dynamicPrivacyController;
mKeyguardIndicationController = keyguardIndicationController;
// TODO: use KeyguardStateController#isOccluded to remove this dependency
mStatusBar = statusBar;
mShadeController = shadeController;
mCommandQueue = commandQueue;
mAboveShelfObserver = new AboveShelfObserver(stackScroller);
mActivityLaunchAnimator = activityLaunchAnimator;
mAboveShelfObserver.setListener(statusBarWindow.findViewById(
R.id.notification_container_parent));
mAccessibilityManager = context.getSystemService(AccessibilityManager.class);
mDozeScrimController = dozeScrimController;
mScrimController = scrimController;
mKeyguardManager = context.getSystemService(KeyguardManager.class);
mMaxAllowedKeyguardNotifications = context.getResources().getInteger(
R.integer.keyguard_max_notification_count);
mBarService = IStatusBarService.Stub.asInterface(
ServiceManager.getService(Context.STATUS_BAR_SERVICE));
 
if (MULTIUSER_DEBUG) {
mNotificationPanelDebugText = mNotificationPanel.getHeaderDebugInfo();
mNotificationPanelDebugText.setVisibility(View.VISIBLE);
}
 
IVrManager vrManager = IVrManager.Stub.asInterface(ServiceManager.getService(
Context.VR_SERVICE));
if (vrManager != null) {
try {
vrManager.registerListener(mVrStateCallbacks);
} catch (RemoteException e) {
Slog.e(TAG, "Failed to register VR mode state listener: " + e);
}
}
NotificationRemoteInputManager remoteInputManager =
Dependency.get(NotificationRemoteInputManager.class);
remoteInputManager.setUpWithCallback(
Dependency.get(NotificationRemoteInputManager.Callback.class),
mNotificationPanel.createRemoteInputDelegate());
remoteInputManager.getController().addCallback(
Dependency.get(NotificationShadeWindowController.class));
 
NotificationListContainer notifListContainer = (NotificationListContainer) stackScroller;
initController.addPostInitTask(() -> {
NotificationEntryListener notificationEntryListener = new NotificationEntryListener() {
@Override
public void onEntryRemoved(
@Nullable NotificationEntry entry,
NotificationVisibility visibility,
boolean removedByUser,
int reason) {
StatusBarNotificationPresenter.this.onNotificationRemoved(
entry.getKey(), entry.getSbn());
if (removedByUser) {
maybeEndAmbientPulse();
}
}
};
 
mViewHierarchyManager.setUpWithPresenter(this, notifListContainer);
mEntryManager.setUpWithPresenter(this);
mEntryManager.addNotificationEntryListener(notificationEntryListener);
mEntryManager.addNotificationLifetimeExtender(mHeadsUpManager);
mEntryManager.addNotificationLifetimeExtender(mGutsManager);
mEntryManager.addNotificationLifetimeExtenders(
remoteInputManager.getLifetimeExtenders());
notificationInterruptStateProvider.addSuppressor(mInterruptSuppressor);
mLockscreenUserManager.setUpWithPresenter(this);
mMediaManager.setUpWithPresenter(this);
mVisualStabilityManager.setUpWithPresenter(this);
mGutsManager.setUpWithPresenter(this,
notifListContainer, mCheckSaveListener, mOnSettingsClickListener);
// ForegroundServiceNotificationListener adds its listener in its constructor
// but we need to request it here in order for it to be instantiated.
// TODO: figure out how to do this correctly once Dependency.get() is gone.
Dependency.get(ForegroundServiceNotificationListener.class);
 
onUserSwitched(mLockscreenUserManager.getCurrentUserId());
});
Dependency.get(ConfigurationController.class).addCallback(this);
}
 
@Override
public void updateNotificationViews(final String reason) {
// The function updateRowStates depends on both of these being non-null, so check them here.
// We may be called before they are set from DeviceProvisionedController's callback.
if (mScrimController == null) return;
// Do not modify the notifications during collapse.
if (isCollapsing()) {
mShadeController.addPostCollapseAction(() -> updateNotificationViews(reason));
return;
}
mViewHierarchyManager.updateNotificationViews();
mNotificationPanel.updateNotificationViews(reason);
}
```

#### 在更新通知栏时调用 updateNotificationViews(final String reason) 来刷新通知栏

#### 3.2 NotificationViewHierarchyManager.java 通知栏通知的展开与否的相关代码

```
public class NotificationViewHierarchyManager implements DynamicPrivacyController.Listener {
private static final String TAG = "NotificationViewHierarchyManager";
 
private final Handler mHandler;
 
/**
* Re-usable map of top-level notifications to their sorted children if any.
* If the top-level notification doesn't have children, its key will still exist in this map
* with its value explicitly set to null.
*/
private final HashMap<NotificationEntry, List<NotificationEntry>> mTmpChildOrderMap =
new HashMap<>();
 
// Dependencies:
private final DynamicChildBindController mDynamicChildBindController;
protected final NotificationLockscreenUserManager mLockscreenUserManager;
protected final NotificationGroupManager mGroupManager;
protected final VisualStabilityManager mVisualStabilityManager;
private final SysuiStatusBarStateController mStatusBarStateController;
private final NotificationEntryManager mEntryManager;
private final LowPriorityInflationHelper mLowPriorityInflationHelper;
 
/**
* {@code true} if notifications not part of a group should by default be rendered in their
* expanded state. If {@code false}, then only the first notification will be expanded if
* possible.
*/
private final boolean mAlwaysExpandNonGroupedNotification;
private final BubbleController mBubbleController;
private final DynamicPrivacyController mDynamicPrivacyController;
private final KeyguardBypassController mBypassController;
private final ForegroundServiceSectionController mFgsSectionController;
private final Context mContext;
 
private NotificationPresenter mPresenter;
private NotificationListContainer mListContainer;
 
// Used to help track down re-entrant calls to our update methods, which will cause bugs.
private boolean mPerformingUpdate;
// Hack to get around re-entrant call in onDynamicPrivacyChanged() until we can track down
// the problem.
private boolean mIsHandleDynamicPrivacyChangeScheduled;
 
/**
* Injected constructor. See {@link StatusBarModule}.
*/
public NotificationViewHierarchyManager(
Context context,
@Main Handler mainHandler,
NotificationLockscreenUserManager notificationLockscreenUserManager,
NotificationGroupManager groupManager,
VisualStabilityManager visualStabilityManager,
StatusBarStateController statusBarStateController,
NotificationEntryManager notificationEntryManager,
KeyguardBypassController bypassController,
BubbleController bubbleController,
DynamicPrivacyController privacyController,
ForegroundServiceSectionController fgsSectionController,
DynamicChildBindController dynamicChildBindController,
LowPriorityInflationHelper lowPriorityInflationHelper) {
mContext = context;
mHandler = mainHandler;
mLockscreenUserManager = notificationLockscreenUserManager;
mBypassController = bypassController;
mGroupManager = groupManager;
mVisualStabilityManager = visualStabilityManager;
mStatusBarStateController = (SysuiStatusBarStateController) statusBarStateController;
mEntryManager = notificationEntryManager;
mFgsSectionController = fgsSectionController;
Resources res = context.getResources();
mAlwaysExpandNonGroupedNotification =
res.getBoolean(R.bool.config_alwaysExpandNonGroupedNotifications);
mBubbleController = bubbleController;
mDynamicPrivacyController = privacyController;
mDynamicChildBindController = dynamicChildBindController;
mLowPriorityInflationHelper = lowPriorityInflationHelper;
}
 
public void setUpWithPresenter(NotificationPresenter presenter,
NotificationListContainer listContainer) {
mPresenter = presenter;
mListContainer = listContainer;
mDynamicPrivacyController.addListener(this);
}
 
/**
* Updates the visual representation of the notifications.
*/
//TODO: Rewrite this to focus on Entries, or some other data object instead of views
public void updateNotificationViews() {
Assert.isMainThread();
beginUpdate();
 
List<NotificationEntry> activeNotifications = mEntryManager.getVisibleNotifications();
ArrayList<ExpandableNotificationRow> toShow = new ArrayList<>(activeNotifications.size());
final int N = activeNotifications.size();
for (int i = 0; i < N; i++) {
NotificationEntry ent = activeNotifications.get(i);
if (ent.isRowDismissed() || ent.isRowRemoved()
|| mBubbleController.isBubbleNotificationSuppressedFromShade(ent)
|| mFgsSectionController.hasEntry(ent)) {
// we don't want to update removed notifications because they could
// temporarily become children if they were isolated before.
continue;
}
 
int userId = ent.getSbn().getUserId();
 
// Display public version of the notification if we need to redact.
// TODO: This area uses a lot of calls into NotificationLockscreenUserManager.
// We can probably move some of this code there.
int currentUserId = mLockscreenUserManager.getCurrentUserId();
boolean devicePublic = mLockscreenUserManager.isLockscreenPublicMode(currentUserId);
boolean userPublic = devicePublic
|| mLockscreenUserManager.isLockscreenPublicMode(userId);
if (userPublic && mDynamicPrivacyController.isDynamicallyUnlocked()
&& (userId == currentUserId || userId == UserHandle.USER_ALL
|| !mLockscreenUserManager.needsSeparateWorkChallenge(userId))) {
userPublic = false;
}
boolean needsRedaction = mLockscreenUserManager.needsRedaction(ent);
boolean sensitive = userPublic && needsRedaction;
boolean deviceSensitive = devicePublic
&& !mLockscreenUserManager.userAllowsPrivateNotificationsInPublic(
currentUserId);
ent.setSensitive(sensitive, deviceSensitive);
ent.getRow().setNeedsRedaction(needsRedaction);
mLowPriorityInflationHelper.recheckLowPriorityViewAndInflate(ent, ent.getRow());
boolean isChildInGroup = mGroupManager.isChildInGroupWithSummary(ent.getSbn());
 
boolean groupChangesAllowed =
mVisualStabilityManager.areGroupChangesAllowed() // user isn't looking at notifs
|| !ent.hasFinishedInitialization(); // notif recently added
 
NotificationEntry parent = mGroupManager.getGroupSummary(ent.getSbn());
if (!groupChangesAllowed) {
// We don't to change groups while the user is looking at them
boolean wasChildInGroup = ent.isChildInGroup();
if (isChildInGroup && !wasChildInGroup) {
isChildInGroup = wasChildInGroup;
mVisualStabilityManager.addGroupChangesAllowedCallback(mEntryManager,
false /* persistent */);
} else if (!isChildInGroup && wasChildInGroup) {
// We allow grouping changes if the group was collapsed
if (mGroupManager.isLogicalGroupExpanded(ent.getSbn())) {
isChildInGroup = wasChildInGroup;
parent = ent.getRow().getNotificationParent().getEntry();
mVisualStabilityManager.addGroupChangesAllowedCallback(mEntryManager,
false /* persistent */);
}
}
}
 
if (isChildInGroup) {
List<NotificationEntry> orderedChildren = mTmpChildOrderMap.get(parent);
if (orderedChildren == null) {
orderedChildren = new ArrayList<>();
mTmpChildOrderMap.put(parent, orderedChildren);
}
orderedChildren.add(ent);
} else {
// Top-level notif (either a summary or single notification)
 
// A child may have already added its summary to mTmpChildOrderMap with a
// list of children. This can happen since there's no guarantee summaries are
// sorted before its children.
if (!mTmpChildOrderMap.containsKey(ent)) {
// mTmpChildOrderMap's keyset is used to iterate through all entries, so it's
// necessary to add each top-level notif as a key
mTmpChildOrderMap.put(ent, null);
}
toShow.add(ent.getRow());
}
 
}
 
ArrayList<ExpandableNotificationRow> viewsToRemove = new ArrayList<>();
for (int i=0; i< mListContainer.getContainerChildCount(); i++) {
View child = mListContainer.getContainerChildAt(i);
if (!toShow.contains(child) && child instanceof ExpandableNotificationRow) {
ExpandableNotificationRow row = (ExpandableNotificationRow) child;
 
// Blocking helper is effectively a detached view. Don't bother removing it from the
// layout.
if (!row.isBlockingHelperShowing()) {
viewsToRemove.add((ExpandableNotificationRow) child);
}
}
}
 
for (ExpandableNotificationRow viewToRemove : viewsToRemove) {
if (mEntryManager.getPendingOrActiveNotif(viewToRemove.getEntry().getKey()) != null) {
// we are only transferring this notification to its parent, don't generate an
// animation
mListContainer.setChildTransferInProgress(true);
}
if (viewToRemove.isSummaryWithChildren()) {
viewToRemove.removeAllChildren();
}
mListContainer.removeContainerView(viewToRemove);
mListContainer.setChildTransferInProgress(false);
}
 
removeNotificationChildren();
 
for (int i = 0; i < toShow.size(); i++) {
View v = toShow.get(i);
if (v.getParent() == null) {
mVisualStabilityManager.notifyViewAddition(v);
mListContainer.addContainerView(v);
} else if (!mListContainer.containsView(v)) {
// the view is added somewhere else. Let's make sure
// the ordering works properly below, by excluding these
toShow.remove(v);
i--;
}
}
 
addNotificationChildrenAndSort();
 
// So after all this work notifications still aren't sorted correctly.
// Let's do that now by advancing through toShow and mListContainer in
// lock-step, making sure mListContainer matches what we see in toShow.
int j = 0;
for (int i = 0; i < mListContainer.getContainerChildCount(); i++) {
View child = mListContainer.getContainerChildAt(i);
if (!(child instanceof ExpandableNotificationRow)) {
// We don't care about non-notification views.
continue;
}
if (((ExpandableNotificationRow) child).isBlockingHelperShowing()) {
// Don't count/reorder notifications that are showing the blocking helper!
continue;
}
 
ExpandableNotificationRow targetChild = toShow.get(j);
if (child != targetChild) {
// Oops, wrong notification at this position. Put the right one
// here and advance both lists.
if (mVisualStabilityManager.canReorderNotification(targetChild)) {
mListContainer.changeViewPosition(targetChild, i);
} else {
mVisualStabilityManager.addReorderingAllowedCallback(mEntryManager,
false  /* persistent */);
}
}
j++;
 
}
 
mDynamicChildBindController.updateContentViews(mTmpChildOrderMap);
mVisualStabilityManager.onReorderingFinished();
// clear the map again for the next usage
mTmpChildOrderMap.clear();
 
updateRowStatesInternal();
 
mListContainer.onNotificationViewUpdateFinished();
 
endUpdate();
}
 
public void updateRowStates() {
Assert.isMainThread();
beginUpdate();
updateRowStatesInternal();
endUpdate();
}
 
private void updateRowStatesInternal() {
Trace.beginSection("NotificationViewHierarchyManager#updateRowStates");
final int N = mListContainer.getContainerChildCount();
 
int visibleNotifications = 0;
boolean onKeyguard = mStatusBarStateController.getState() == StatusBarState.KEYGUARD;
int maxNotifications = -1;
if (onKeyguard && !mBypassController.getBypassEnabled()) {
maxNotifications = mPresenter.getMaxNotificationsWhileLocked(true /* recompute */);
}
mListContainer.setMaxDisplayedNotifications(maxNotifications);
Stack<ExpandableNotificationRow> stack = new Stack<>();
for (int i = N - 1; i >= 0; i--) {
View child = mListContainer.getContainerChildAt(i);
if (!(child instanceof ExpandableNotificationRow)) {
continue;
}
stack.push((ExpandableNotificationRow) child);
}
while(!stack.isEmpty()) {
ExpandableNotificationRow row = stack.pop();
NotificationEntry entry = row.getEntry();
boolean isChildNotification =
mGroupManager.isChildInGroupWithSummary(entry.getSbn());
 
row.setOnKeyguard(onKeyguard);
 
if (!onKeyguard) {
// If mAlwaysExpandNonGroupedNotification is false, then only expand the
// very first notification and if it's not a child of grouped notifications.
row.setSystemExpanded(mAlwaysExpandNonGroupedNotification
|| (visibleNotifications == 0 && !isChildNotification
&& !row.isLowPriority()));
}
 
int userId = entry.getSbn().getUserId();
boolean suppressedSummary = mGroupManager.isSummaryOfSuppressedGroup(
entry.getSbn()) && !entry.isRowRemoved();
boolean showOnKeyguard = mLockscreenUserManager.shouldShowOnKeyguard(entry);
if (!showOnKeyguard) {
// min priority notifications should show if their summary is showing
if (mGroupManager.isChildInGroupWithSummary(entry.getSbn())) {
NotificationEntry summary = mGroupManager.getLogicalGroupSummary(
entry.getSbn());
if (summary != null && mLockscreenUserManager.shouldShowOnKeyguard(summary)) {
showOnKeyguard = true;
}
}
}
if (suppressedSummary
|| mLockscreenUserManager.shouldHideNotifications(userId)
|| (onKeyguard && !showOnKeyguard)) {
entry.getRow().setVisibility(View.GONE);
} else {
boolean wasGone = entry.getRow().getVisibility() == View.GONE;
if (wasGone) {
entry.getRow().setVisibility(View.VISIBLE);
}
if (!isChildNotification && !entry.getRow().isRemoved()) {
if (wasGone) {
// notify the scroller of a child addition
mListContainer.generateAddAnimation(entry.getRow(),
!showOnKeyguard /* fromMoreCard */);
}
visibleNotifications++;
}
}
if (row.isSummaryWithChildren()) {
List<ExpandableNotificationRow> notificationChildren =
row.getAttachedChildren();
int size = notificationChildren.size();
for (int i = size - 1; i >= 0; i--) {
stack.push(notificationChildren.get(i));
}
}
 
row.showAppOpsIcons(entry.mActiveAppOps);
row.setLastAudiblyAlertedMs(entry.getLastAudiblyAlertedMs());
}
 
Trace.beginSection("NotificationPresenter#onUpdateRowStates");
mPresenter.onUpdateRowStates();
Trace.endSection();
Trace.endSection();
}
...
}
 
 
 
  
```

在 updateNotificationViews() 中是负责构造整个通知栏的相关功能，而在 updateRowStatesInternal();  
负责构造每个通知栏通知的相关布局  
if (!onKeyguard) {  
            // If mAlwaysExpandNonGroupedNotification is false, then only expand the  
            // very first [notification](https://so.csdn.net/so/search?q=notification&spm=1001.2101.3001.7020) and if it's not a child of grouped notifications.  
            row.setSystemExpanded(mAlwaysExpandNonGroupedNotification  
                    || (visibleNotifications == 0 && !isChildNotification  
                    && !row.isLowPriority()));  
        }  
设置通知是否默认展开  
mAlwaysExpandNonGroupedNotification 的值来判断  
mAlwaysExpandNonGroupedNotification =  
        res.getBoolean(R.bool.config_alwaysExpandNonGroupedNotifications);  
所以具体修改为:  
frameworks\base\packages\SystemUI\res\values\config.xml  
<bool >true</bool>