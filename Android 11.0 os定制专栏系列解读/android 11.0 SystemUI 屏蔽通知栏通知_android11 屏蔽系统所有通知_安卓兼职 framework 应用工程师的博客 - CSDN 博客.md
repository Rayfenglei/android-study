> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124908303)

在 11.0 的定制化 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 开发中，所有的通知显示都是由 Notication.java 负责的，所以我们来先看  
Notification 的监听加载流程，回到 statusBar 的 start() 中注册 NotificationListenerWithPlugins 作为系统 service 监听通知消息

```
try {
mNotificationListener.registerAsSystemService(mContext,
new ComponentName(mContext.getPackageName(), getClass().getCanonicalName()),
UserHandle.USER_ALL);
} catch (RemoteException e) {
Log.e(TAG, "Unable to register notification listener", e);

}

private final NotificationListenerWithPlugins mNotificationListener =
new NotificationListenerWithPlugins() {
@Override
public void onListenerConnected() {
services成功启动，获取当前处于活动状态的通知(没被移除的通知)，添加到通知栏，此处应该是重启后重新加载
}

@Override
public void onNotificationPosted(final StatusBarNotification sbn,
final RankingMap rankingMap) {
收到通知消息，添加或者修改
if (isUpdate) {
updateNotification(sbn, rankingMap);
} else {
addNotification(sbn, rankingMap);
}
}

@Override
public void onNotificationRemoved(StatusBarNotification sbn,
final RankingMap rankingMap) {
移除通知消息
if (sbn != null && !onPluginNotificationRemoved(sbn, rankingMap)) {
final String key = sbn.getKey();
mHandler.post(() -> removeNotification(key, rankingMap));
}
}

@Override
public void onNotificationRankingUpdate(final RankingMap rankingMap) {
通知的排序优先级改变，修改通知位置
if (rankingMap != null) {
RankingMap r = onPluginRankingUpdate(rankingMap);
mHandler.post(() -> updateNotificationRanking(r));
}
}

};

```

接下来看下 addNotification() 方法

```
public void addNotification(StatusBarNotification notification, RankingMap ranking)
throws InflationException {
String key = notification.getKey();
if (true/**DEBUG*/) Log.d(TAG, "addNotification key=" + key);
mNotificationData.updateRanking(ranking);
Entry shadeEntry = createNotificationViews(notification);

}

```

可以看到是通过 createNotificationViews() 来创建通知 View 对象，内部继续调用 inflateViews()

```
protected NotificationData.Entry createNotificationViews(StatusBarNotification sbn)
throws InflationException {
if (DEBUG) {
Log.d(TAG, "createNotificationViews(notification=" + sbn);
}
NotificationData.Entry entry = new NotificationData.Entry(sbn);
Dependency.get(LeakDetector.class).trackInstance(entry);
entry.createIcons(mContext, sbn);
// Construct the expanded view.
inflateViews(entry, mStackScroller);
return entry;
}

protected void inflateViews(Entry entry, ViewGroup parent) {
PackageManager pmUser = getPackageManagerForUser(mContext,
entry.notification.getUser().getIdentifier());

final StatusBarNotification sbn = entry.notification;
if (entry.row != null) {
entry.reset();
updateNotification(entry, pmUser, sbn, entry.row);
} else {
new RowInflaterTask().inflate(mContext, parent, entry,
row -> {
bindRow(entry, pmUser, sbn, row);
updateNotification(entry, pmUser, sbn, row);
});
}

}

```

看到上面的方法中，entry 在 createNotificationViews 中创建，只赋值了 icons， entry.row 为 null，进入 RowInflaterTask 中

```
SystemUI\src\com\android\systemui\statusbar\notification\RowInflaterTask.java
public void inflate(Context context, ViewGroup parent, NotificationData.Entry entry,
RowInflationFinishedListener listener) {
mListener = listener;
AsyncLayoutInflater inflater = new AsyncLayoutInflater(context);
mEntry = entry;
entry.setInflationTask(this);
inflater.inflate(R.layout.status_bar_notification_row, parent, this);
}

```

这里我们得到了 Notification 对应的 layout 为 status_bar_notification_row.xml，  
回调方法中将 row 和 entry 绑定，继续再调用 updateNotification()，注意这个方法是四个参数的，该类中还有重载方法是两个参数的。

```
private void updateNotification(Entry entry, PackageManager pmUser,
StatusBarNotification sbn, ExpandableNotificationRow row) {


entry.row = row;
entry.row.setOnActivatedListener(this);

boolean useIncreasedCollapsedHeight = mMessagingUtil.isImportantMessaging(sbn,
mNotificationData.getImportance(sbn.getKey()));
boolean useIncreasedHeadsUp = useIncreasedCollapsedHeight && mPanelExpanded;
row.setUseIncreasedCollapsedHeight(useIncreasedCollapsedHeight);
row.setUseIncreasedHeadsUpHeight(useIncreasedHeadsUp);
row.updateNotification(entry);
}

```

紧接着调用了 ExpandableNotificationRow 的 updateNotification()，内部继续调用 NotificationInflater.inflateNotificationViews()

```
SystemUI\src\com\android\systemui\statusbar\notification\NotificationInflater.java
@VisibleForTesting
void inflateNotificationViews(int reInflateFlags) {
if (mRow.isRemoved()) {
// We don't want to reinflate anything for removed notifications. Otherwise views might
// be readded to the stack, leading to leaks. This may happen with low-priority groups
// where the removal of already removed children can lead to a reinflation.
return;
}
StatusBarNotification sbn = mRow.getEntry().notification;
new AsyncInflationTask(sbn, reInflateFlags, mRow, mIsLowPriority,
mIsChildInGroup, mUsesIncreasedHeight, mUsesIncreasedHeadsUpHeight, mRedactAmbient,
mCallback, mRemoteViewClickHandler).execute();
}


new AsyncInflationTask().execute();
@Override
protected InflationProgress doInBackground(Void... params) {
try {
final Notification.Builder recoveredBuilder
= Notification.Builder.recoverBuilder(mContext,
mSbn.getNotification());
Context packageContext = mSbn.getPackageContext(mContext);
Notification notification = mSbn.getNotification();
if (mIsLowPriority) {
int backgroundColor = mContext.getColor(
R.color.notification_material_background_low_priority_color);
recoveredBuilder.setBackgroundColorHint(backgroundColor);
}
if (notification.isMediaNotification()) {
MediaNotificationProcessor processor = new MediaNotificationProcessor(mContext,
packageContext);
processor.setIsLowPriority(mIsLowPriority);
processor.processNotification(notification, recoveredBuilder);
}
return createRemoteViews(mReInflateFlags,
recoveredBuilder, mIsLowPriority, mIsChildInGroup,
mUsesIncreasedHeight, mUsesIncreasedHeadsUpHeight, mRedactAmbient,
packageContext);
} catch (Exception e) {
mError = e;
return null;
}
}

@Override
protected void onPostExecute(InflationProgress result) {
if (mError == null) {
mCancellationSignal = apply(result, mReInflateFlags, mRow, mRedactAmbient,
mRemoteViewClickHandler, this);
} else {
handleError(mError);
}
}

```

从 msbn 中获取 notifaction，判断是否是媒体类型的通知，进行对应的主题背景色修改，通过传递的优先级设置通知背景色，继续看核心方法

```
createRemoteViews()

private static InflationProgress createRemoteViews(int reInflateFlags,
Notification.Builder builder, boolean isLowPriority, boolean isChildInGroup,
boolean usesIncreasedHeight, boolean usesIncreasedHeadsUpHeight, boolean redactAmbient,
Context packageContext) {
InflationProgress result = new InflationProgress();
isLowPriority = isLowPriority && !isChildInGroup;
if ((reInflateFlags & FLAG_REINFLATE_CONTENT_VIEW) != 0) {
result.newContentView = createContentView(builder, isLowPriority, usesIncreasedHeight);
}

if ((reInflateFlags & FLAG_REINFLATE_EXPANDED_VIEW) != 0) {
result.newExpandedView = createExpandedView(builder, isLowPriority);
}

if ((reInflateFlags & FLAG_REINFLATE_HEADS_UP_VIEW) != 0) {
result.newHeadsUpView = builder.createHeadsUpContentView(usesIncreasedHeadsUpHeight);
}

if ((reInflateFlags & FLAG_REINFLATE_PUBLIC_VIEW) != 0) {
result.newPublicView = builder.makePublicContentView();
}

if ((reInflateFlags & FLAG_REINFLATE_AMBIENT_VIEW) != 0) {
result.newAmbientView = redactAmbient ? builder.makePublicAmbientNotification()
: builder.makeAmbientNotification();
}
result.packageContext = packageContext;
return result;
}

```

这里就是创建各种布局 CONTENT_VIEW、EXPANDED_VIEW、HEADS_UP_VIEW、PUBLIC_VIEW、AMBIENT_VIEW，  
然后回到 AsyncInflationTask 的 onPostExecute() 中执行 apply(), 代码太多就不贴了, SystemUI 部分的通知流程分析技术

所有的通知调用的都是 NotificationManager.notify(int id，Notification notification)。最后走到 SystemUI 的时候首先调用 BaseStatusBar 中的成员变量 mNotificationListener 的 onNotificationPosted(final StatusBarNotification sbn,final RankingMap rankingMap) 方法。

所以解决方案:|

SystemUI 的通知流程是通过 NotificationListener 的 registerAsSystemService(）来实现监听通知然后显示通知, 所以用一个[标志位](https://so.csdn.net/so/search?q=%E6%A0%87%E5%BF%97%E4%BD%8D&spm=1001.2101.3001.7020)就可以达到控制通知显示的目的

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationListener.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationListener.java

@@ -29,7 +29,7 @@ import android.os.RemoteException;

import android.os.UserHandle;

import android.service.notification.StatusBarNotification;

import android.util.Log;

-

+import android.provider.Settings;

import com.android.systemui.Dependency;

import com.android.systemui.statusbar.notification.NotificationEntryManager;

import com.android.systemui.statusbar.phone.NotificationGroupManager;

@@ -112,6 +112,8 @@ public class NotificationListener extends NotificationListenerWithPlugins {

@Override
public void onNotificationPosted(final StatusBarNotification sbn,
final RankingMap rankingMap) {
if (DEBUG) Log.d(TAG, "onNotificationPosted: " + sbn);
if (sbn != null && !onPluginNotificationPosted(sbn, rankingMap)) {
Dependency.get(Dependency.MAIN_HANDLER).post(() -> {
processForRemoteInput(sbn.getNotification(), mContext);
String key = sbn.getKey();
boolean isUpdate =
mEntryManager.getNotificationData().get(key) != null;
// In case we don't allow child notifications, we ignore children of
// notifications that have a summary, since` we're not going to show them
// anyway. This is true also when the summary is canceled,
// because children are automatically canceled by NoMan in that case.
if (!ENABLE_CHILD_NOTIFICATIONS
&& mGroupManager.isChildInGroupWithSummary(sbn)) {
if (DEBUG) {
Log.d(TAG, "Ignoring group child due to existing summary: " + sbn);
}

// Remove existing notification to avoid stale data.
if (isUpdate) {
mEntryManager.removeNotification(key, rankingMap, UNDEFINED_DISMISS_REASON);
} else {
mEntryManager.getNotificationData()
updateRanking(rankingMap);
}
return;
}

//接收通知并处理

+                int expand_panel = Settings.Global.getInt(mContext.getContentResolver(),"expand_panel",1);

+                               if(expand_panel==0)return;

if (isUpdate) {

mEntryManager.updateNotification(sbn, rankingMap);

} else {
mEntryManager.addNotification(sbn, rankingMap);
}
});
}
}

```