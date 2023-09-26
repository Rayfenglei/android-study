> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774747)

11.0 产品由于客户需求不需要通知，所以要求去掉所有通知，而通知部分就是在 SystemUI 部分管理的，所以就要从这里入手来去掉关于通知栏的部分  
首选要从两部分入手

第一部分，状态栏显示通知图标的部分  
状态栏布局为 status_bar.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020)  
首选来看 status_bar.xml

```
<com.android.systemui.statusbar.phone.PhoneStatusBarView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res/com.android.systemui"
    android:layout_width="match_parent"
    android:layout_height="@dimen/status_bar_height"
    android:id="@+id/status_bar"
    android:background="@drawable/system_bar_background"
    android:orientation="vertical"
    android:focusable="false"
    android:descendantFocusability="afterDescendants"
    android:accessibilityPaneTitle="@string/status_bar"
    >

    <ImageView
        android:id="@+id/notification_lights_out"
        android:layout_width="@dimen/status_bar_icon_size"
        android:layout_height="match_parent"
        android:paddingStart="@dimen/status_bar_padding_start"
        android:paddingBottom="2dip"
        android:src="@drawable/ic_sysbar_lights_out_dot_small"
        android:scaleType="center"
        android:visibility="gone"
        />

    <LinearLayout android:id="@+id/status_bar_contents"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingStart="@dimen/status_bar_padding_start"
        android:paddingEnd="@dimen/status_bar_padding_end"
        android:paddingTop="@dimen/status_bar_padding_top"
        android:orientation="horizontal"
        >
        <FrameLayout
            android:layout_height="match_parent"
            android:layout_width="0dp"
            android:layout_weight="1">

            <include layout="@layout/heads_up_status_bar_layout" />

            <!-- The alpha of the left side is controlled by PhoneStatusBarTransitions, and the
             individual views are controlled by StatusBarManager disable flags DISABLE_CLOCK and
             DISABLE_NOTIFICATION_ICONS, respectively -->
            <LinearLayout
                android:id="@+id/status_bar_left_side"
                android:layout_height="match_parent"
                android:layout_width="match_parent"
                android:clipChildren="false"
            >
                <ViewStub
                    android:id="@+id/operator_name"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent"
                    android:layout="@layout/operator_name" />

                <com.android.systemui.statusbar.policy.Clock
                    android:id="@+id/clock"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent"
                    android:textAppearance="@style/TextAppearance.StatusBar.Clock"
                    android:singleLine="true"
                    android:paddingStart="@dimen/status_bar_left_clock_starting_padding"
                    android:paddingEnd="@dimen/status_bar_left_clock_end_padding"
                    android:gravity="center_vertical|start"
                />

                <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
                    android:id="@+id/notification_icon_area"
                    android:layout_width="0dp"
                    android:layout_height="match_parent"
                    android:layout_weight="1"
					android:visibility="gone"
                    android:orientation="horizontal"
                    android:clipChildren="false"/>

            </LinearLayout>
        </FrameLayout>

        <!-- Space should cover the notch (if it exists) and let other views lay out around it -->
        <android.widget.Space
            android:id="@+id/cutout_space_view"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:gravity="center_horizontal|center_vertical"
        />

        <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
            android:id="@+id/centered_icon_area"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:orientation="horizontal"
            android:clipChildren="false"
            android:gravity="center_horizontal|center_vertical"/>

        <com.android.keyguard.AlphaOptimizedLinearLayout android:id="@+id/system_icon_area"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:orientation="horizontal"
            android:gravity="center_vertical|end"
            >

            <include layout="@layout/system_icons" />
        </com.android.keyguard.AlphaOptimizedLinearLayout>
    </LinearLayout>

。。。

</com.android.systemui.statusbar.phone.PhoneStatusBarView>

```

而

```
<com.android.systemui.statusbar.AlphaOptimizedFrameLayout
                android:id="@+id/notification_icon_area"
                android:layout_width="0dp"
                android:layout_height="match_parent"
                android:layout_weight="1"
				android:visibility="gone"
                android:orientation="horizontal"
                android:clipChildren="false"/>

```

即为通知栏显示的区域 首选隐藏掉即可

第二部分 就是 下拉状态栏部分关于显示通知栏的区域  
就是在 status_bar_expended.xml 中的

```
<com.android.systemui.statusbar.notification.stack.NotificationStackScrollLayout
    android:id="@+id/notification_stack_scroller"
    android:layout_marginTop="@dimen/notification_panel_margin_top"
    android:layout_width="@dimen/notification_panel_width"
    android:layout_height="match_parent"
    android:layout_gravity="@integer/notification_panel_layout_gravity"
    android:layout_marginBottom="@dimen/close_handle_underlap" />

```

而这部分是根据接收到的通知而显示的

所以如果屏蔽掉接收通知的相关代码就可以保证这里不会再有通知显示  
SystemUI 的通知流程是通过 NotificationListener 的 registerAsSystemService(）来实现监听通知然后显示通知  
而具体通知是由 NotificationListener.java 负责管理的  
路径为: frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationListener.java

```
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
                                .updateRanking(rankingMap);
                    }
                    return;
                }
                int expand_panel = Settings.Global.getInt(mContext.getContentResolver(),"expand_panel",1);
				if(expand_panel==0)return;
                if (isUpdate) {
                    mEntryManager.updateNotification(sbn, rankingMap);
                } else {
                    mEntryManager.addNotification(sbn, rankingMap);
                }
            });
        }
    }

```

onNotificationPosted 负责接收到消息后通过通知栏显示消息

所以修改此处

```
 /*if (isUpdate) {
                    mEntryManager.updateNotification(sbn, rankingMap);
                } else {
                    mEntryManager.addNotification(sbn, rankingMap);
                }*/

```

注释掉显示通知部分代码即可