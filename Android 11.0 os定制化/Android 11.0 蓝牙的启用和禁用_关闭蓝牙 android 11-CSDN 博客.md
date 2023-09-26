> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125580779)

1. 概述
-----

在 11.0 的定制化中，对于蓝牙功能的启用和禁用功能，在一些定制化的平板中，是需要这种功能的 接下来就来从 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) Settings 和 framwork 层来增加接口来实现开启和禁用蓝牙

2. 开启和禁用蓝牙的核心代码
---------------

```
主要核心代码：
frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java
  frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/BluetoothControllerImpl.java
  frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tiles/BluetoothTile.java
  frameworks/base/packages/SettingsLib/src/com/android/settingslib/bluetooth/LocalBluetoothAdapter.java
  packages/apps/Settings/src/com/android/settings/connecteddevice/BluetoothDashboardFragment.java
  packages/apps/Settings/src/com/android/settings/bluetooth/BluetoothEnabler.java
```

3. 开启和禁用蓝牙的核心代码功能分析和实现功能
------------------------

#### 3.1 BluetoothManagerService.java 蓝牙管理服务中开启蓝牙的管控

```
frameworks/base/services/core/java/com/android/server/BluetoothManagerService.java
public boolean enable(String packageName) throws RemoteException {
if (!checkBluetoothPermissions(packageName, true)) {
if (DBG) {
Slog.d(TAG, "enable(): not enabling - bluetooth disallowed");
}
return false;
}
 
final int callingUid = Binder.getCallingUid();
final boolean callerSystem = UserHandle.getAppId(callingUid) == Process.SYSTEM_UID;
if (!callerSystem && !isEnabled() && mWirelessConsentRequired
&& startConsentUiIfNeeded(packageName,
callingUid, BluetoothAdapter.ACTION_REQUEST_ENABLE)) {
return false;
}
 
if (DBG) {
Slog.d(TAG, "enable(" + packageName + "):  mBluetooth =" + mBluetooth + " mBinding = "
+ mBinding + " mState = " + BluetoothAdapter.nameForState(mState));
}
 
synchronized (mReceiver) {
mQuietEnableExternal = false;
mEnableExternal = true;
// waive WRITE_SECURE_SETTINGS permission check
sendEnableMsg(false,
BluetoothProtoEnums.ENABLE_DISABLE_REASON_APPLICATION_REQUEST, packageName);
}
if (DBG) {
Slog.d(TAG, "enable returning");
}
return true;
}
 
修改为:
public Boolean enable(String packageName) throws RemoteException {
	if (!checkBluetoothPermissions(packageName, true)) {
		if (DBG) {
			Slog.d(TAG, "enable(): not enabling - bluetooth disallowed");
		}
		return false;
	}
	+               //add code start
	+               if(SystemProperties.get("persist.sys.disableBT", "false").equals("true")){
	+                       return false;
	+}
	+               //add code end
	final int callingUid = Binder.getCallingUid();
	final Boolean callerSystem = UserHandle.getAppId(callingUid) == Process.SYSTEM_UID;
	if (!callerSystem && !isEnabled() && mWirelessConsentRequired
	&& startConsentUiIfNeeded(packageName,
	callingUid, BluetoothAdapter.ACTION_REQUEST_ENABLE)) {
		return false;
	}
	if (DBG) {
		Slog.d(TAG, "enable(" + packageName + "):  mBluetooth =" + mBluetooth + " mBinding = "
		+ mBinding + " mState = " + BluetoothAdapter.nameForState(mState));
	}
	synchronized (mReceiver) {
		mQuietEnableExternal = false;
		mEnableExternal = true;
		// waive WRITE_SECURE_SETTINGS permission check
		sendEnableMsg(false,
		BluetoothProtoEnums.ENABLE_DISABLE_REASON_APPLICATION_REQUEST, packageName);
	}
	if (DBG) {
		Slog.d(TAG, "enable returning");
	}
	return true;
}
```

#### 3.2 BluetoothControllerImpl.java 关于开启蓝牙的管控

```
        @Override
     public boolean isBluetoothSupported() {
 return mLocalBluetoothManager != null;
     }
         关于是否支持蓝牙的管控 添加系统属性的控制
        @Override
     public boolean isBluetoothSupported() {
-        return mLocalBluetoothManager != null;
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               boolean support=!"true".equals(flag);
+               Log.d("dong","isBluetoothSupported():"+flag+".support:"+support);
+        return support;
```

#### 3.3 BluetoothTile.java 中关于下拉状态栏蓝牙快捷功能开启关闭蓝牙的管控

```
public class BluetoothTile extends QSTileImpl<BooleanState> {
private static final Intent BLUETOOTH_SETTINGS = new Intent(Settings.ACTION_BLUETOOTH_SETTINGS);
 
private final BluetoothController mController;
private final BluetoothDetailAdapter mDetailAdapter;
private final ActivityStarter mActivityStarter;
 
@Inje       return mLocalBluetoothManager != null;
     }
 
public BluetoothTile(QSHost host,
BluetoothController bluetoothController,
ActivityStarter activityStarter) {
super(host);
mController = bluetoothController;
mActivityStarter = activityStarter;
mDetailAdapter = (BluetoothDetailAdapter) createDetailAdapter();
mController.observe(getLifecycle(), mCallback);
}
 
@Override
public DetailAdapter getDetailAdapter() {
return mDetailAdapter;
}
 
@Override
public BooleanState newTileState() {
return new BooleanState();
}
 
@Override
protected void handleClick() {
// Secondary clicks are header clicks, just toggle.
final boolean isEnabled = mState.value;
// Immediately enter transient enabling state when turning bluetooth on.
refreshState(isEnabled ? null : ARG_SHOW_TRANSIENT_ENABLING);
mController.setBluetoothEnabled(!isEnabled);
}
 
@Override
public Intent getLongClickIntent() {
return new Intent(Settings.ACTION_BLUETOOTH_SETTINGS);
}
 
@Override
protected void handleSecondaryClick() {
if (!mController.canConfigBluetooth()) {
mActivityStarter.postStartActivityDismissingKeyguard(
new Intent(Settings.ACTION_BLUETOOTH_SETTINGS), 0);
return;
}
showDetail(true);
if (!mState.value) {
mController.setBluetoothEnabled(true);
}
}
 
@Override
public CharSequence getTileLabel() {
return mContext.getString(R.string.quick_settings_bluetooth_label);
}
 
@Override
protected void handleUpdateState(BooleanState state, Object arg) {
final boolean transientEnabling = arg == ARG_SHOW_TRANSIENT_ENABLING;
final boolean enabled = transientEnabling || mController.isBluetoothEnabled();
final boolean connected = mController.isBluetoothConnected();
final boolean connecting = mController.isBluetoothConnecting();
state.isTransient = transientEnabling || connecting ||
mController.getBluetoothState() == BluetoothAdapter.STATE_TURNING_ON;
state.dualTarget = true;
state.value = enabled;
if (state.slash == null) {
state.slash = new SlashState();
}
state.slash.isSlashed = !enabled;
state.label = mContext.getString(R.string.quick_settings_bluetooth_label);
state.secondaryLabel = TextUtils.emptyIfNull(
getSecondaryLabel(enabled, connecting, connected, state.isTransient));
state.contentDescription = state.label;
state.stateDescription = "";
if (enabled) {
if (connected) {
state.icon = new BluetoothConnectedTileIcon();
if (!TextUtils.isEmpty(mController.getConnectedDeviceName())) {
state.label = mController.getConnectedDeviceName();
}
state.stateDescription =
mContext.getString(R.string.accessibility_bluetooth_name, state.label)
+ ", " + state.secondaryLabel;
} else if (state.isTransient) {
state.icon = ResourceIcon.get(
com.android.internal.R.drawable.ic_bluetooth_transient_animation);
state.stateDescription = state.secondaryLabel;
} else {
state.icon =
ResourceIcon.get(com.android.internal.R.drawable.ic_qs_bluetooth);
state.contentDescription = mContext.getString(
R.string.accessibility_quick_settings_bluetooth);
state.stateDescription = mContext.getString(R.string.accessibility_not_connected);
}
state.state = Tile.STATE_ACTIVE;
} else {
state.icon = ResourceIcon.get(com.android.internal.R.drawable.ic_qs_bluetooth);
state.contentDescription = mContext.getString(
R.string.accessibility_quick_settings_bluetooth);
state.state = Tile.STATE_INACTIVE;
}
 
state.dualLabelContentDescription = mContext.getResources().getString(
R.string.accessibility_quick_settings_open_settings, getTileLabel());
state.expandedAccessibilityClassName = Switch.class.getName();
}
 
/**
* Returns the secondary label to use for the given bluetooth connection in the form of the
* battery level or bluetooth profile name. If the bluetooth is disabled, there's no connected
* devices, or we can't map the bluetooth class to a profile, this instead returns {@code null}.
* @param enabled whether bluetooth is enabled
* @param connecting whether bluetooth is connecting to a device
* @param connected whether there's a device connected via bluetooth
* @param isTransient whether bluetooth is currently in a transient state turning on
*/
@Nullable
private String getSecondaryLabel(boolean enabled, boolean connecting, boolean connected,
boolean isTransient) {
if (connecting) {
return mContext.getString(R.string.quick_settings_connecting);
}
if (isTransient) {
return mContext.getString(R.string.quick_settings_bluetooth_secondary_label_transient);
}
List<CachedBluetoothDevice> connectedDevices = mController.getConnectedDevices();
if (enabled && connected && !connectedDevices.isEmpty()) {
if (connectedDevices.size() > 1) {
// TODO(b/76102598): add a new string for "X connected devices" after P
return mContext.getResources().getQuantityString(
R.plurals.quick_settings_hotspot_secondary_label_num_devices,
connectedDevices.size(),
connectedDevices.size());
}
CachedBluetoothDevice lastDevice = connectedDevices.get(0);
final int batteryLevel = lastDevice.getBatteryLevel();
if (batteryLevel > BluetoothDevice.BATTERY_LEVEL_UNKNOWN) {
return mContext.getString(
R.string.quick_settings_bluetooth_secondary_label_battery_level,
Utils.formatPercentage(batteryLevel));
} else {
final BluetoothClass bluetoothClass = lastDevice.getBtClass();
if (bluetoothClass != null) {
if (lastDevice.isHearingAidDevice()) {
return mContext.getString(
R.string.quick_settings_bluetooth_secondary_label_hearing_aids);
} else if (bluetoothClass.doesClassMatch(BluetoothClass.PROFILE_A2DP)) {
return mContext.getString(
R.string.quick_settings_bluetooth_secondary_label_audio);
} else if (bluetoothClass.doesClassMatch(BluetoothClass.PROFILE_HEADSET)) {
return mContext.getString(
R.string.quick_settings_bluetooth_secondary_label_headset);
} else if (bluetoothClass.doesClassMatch(BluetoothClass.PROFILE_HID)) {
return mContext.getString(
R.string.quick_settings_bluetooth_secondary_label_input);
}
}
}
}
return null;
}
@Override
public int getMetricsCategory() {
return MetricsEvent.QS_BLUETOOTH;
}
@Override
protected String composeChangeAnnouncement() {
if (mState.value) {
return mContext.getString(R.string.accessibility_quick_settings_bluetooth_changed_on);
} else {
return mContext.getString(R.string.accessibility_quick_settings_bluetooth_changed_off);
}
}
@Override
public boolean isAvailable() {
return mController.isBluetoothSupported();
}
private final BluetoothController.Callback mCallback = new BluetoothController.Callback() {
@Override
public void onBluetoothStateChange(boolean enabled) {
refreshState();
if (isShowingDetail()) {
mDetailAdapter.updateItems();
fireToggleStateChanged(mDetailAdapter.getToggleState());
}
}
@Override
public void onBluetoothDevicesChanged() {
refreshState();
if (isShowingDetail()) {
mDetailAdapter.updateItems();
}
}
};
@Override
protected DetailAdapter createDetailAdapter() {
return new BluetoothDetailAdapter();
}
/**
* Bluetooth icon wrapper for Quick Settings with a battery indicator that reflects the
* connected device's battery level. This is used instead of
* {@link com.android.systemui.qs.tileimpl.QSTileImpl.DrawableIcon} in order to use a context
* that reflects dark/light theme attributes.
*/
private class BluetoothBatteryTileIcon extends Icon {
private int mBatteryLevel;
private float mIconScale;
 
BluetoothBatteryTileIcon(int batteryLevel, float iconScale) {
mBatteryLevel = batteryLevel;
mIconScale = iconScale;
}
 
@Override
public Drawable getDrawable(Context context) {
// This method returns Pair<Drawable, String> while first value is the drawable
return BluetoothDeviceLayerDrawable.createLayerDrawable(
context,
R.drawable.ic_bluetooth_connected,
mBatteryLevel,
mIconScale);
}
}
 
 
/**
* Bluetooth icon wrapper (when connected with no battery indicator) for Quick Settings. This is
* used instead of {@link com.android.systemui.qs.tileimpl.QSTileImpl.DrawableIcon} in order to
* use a context that reflects dark/light theme attributes.
*/
private class BluetoothConnectedTileIcon extends Icon {
 
BluetoothConnectedTileIcon() {
// Do nothing. Default constructor to limit visibility.
}
 
@Override
public Drawable getDrawable(Context context) {
// This method returns Pair<Drawable, String> - the first value is the drawable.
return context.getDrawable(R.drawable.ic_bluetooth_connected);
}
}
 
protected class BluetoothDetailAdapter implements DetailAdapter, QSDetailItems.Callback {
// We probably won't ever have space in the UI for more than 20 devices, so don't
// get info for them.
private static final int MAX_DEVICES = 20;
private QSDetailItems mItems;
 
@Override
public CharSequence getTitle() {
return mContext.getString(R.string.quick_settings_bluetooth_label);
}
 
@Override
public Boolean getToggleState() {
return mState.value;
}
 
@Override
public boolean getToggleEnabled() {
return mController.getBluetoothState() == BluetoothAdapter.STATE_OFF
|| mController.getBluetoothState() == BluetoothAdapter.STATE_ON;
}
 
@Override
public Intent getSettingsIntent() {
return BLUETOOTH_SETTINGS;
}
 
@Override
public void setToggleState(boolean state) {
MetricsLogger.action(mContext, MetricsEvent.QS_BLUETOOTH_TOGGLE, state);
mController.setBluetoothEnabled(state);
}
 
@Override
public int getMetricsCategory() {
return MetricsEvent.QS_BLUETOOTH_DETAILS;
}
 
@Override
public View createDetailView(Context context, View convertView, ViewGroup parent) {
mItems = QSDetailItems.convertOrInflate(context, convertView, parent);
mItems.setTagSuffix("Bluetooth");
mItems.setCallback(this);
updateItems();
setItemsVisible(mState.value);
return mItems;
}
 
public void setItemsVisible(boolean visible) {
if (mItems == null) return;
mItems.setItemsVisible(visible);
}
 
private void updateItems() {
if (mItems == null) return;
if (mController.isBluetoothEnabled()) {
mItems.setEmptyState(R.drawable.ic_qs_bluetooth_detail_empty,
R.string.quick_settings_bluetooth_detail_empty_text);
} else {
mItems.setEmptyState(R.drawable.ic_qs_bluetooth_detail_empty,
R.string.bt_is_off);
}
ArrayList<Item> items = new ArrayList<Item>();
final Collection<CachedBluetoothDevice> devices = mController.getDevices();
if (devices != null) {
int connectedDevices = 0;
int count = 0;
for (CachedBluetoothDevice device : devices) {
if (mController.getBondState(device) == BluetoothDevice.BOND_NONE) continue;
final Item item = new Item();
item.iconResId = com.android.internal.R.drawable.ic_qs_bluetooth;
item.line1 = device.getName();
item.tag = device;
int state = device.getMaxConnectionState();
if (state == BluetoothProfile.STATE_CONNECTED) {
item.iconResId = R.drawable.ic_bluetooth_connected;
int batteryLevel = device.getBatteryLevel();
if (batteryLevel > BluetoothDevice.BATTERY_LEVEL_UNKNOWN) {
item.icon = new BluetoothBatteryTileIcon(batteryLevel,1 /* iconScale */);
item.line2 = mContext.getString(
R.string.quick_settings_connected_battery_level,
Utils.formatPercentage(batteryLevel));
} else {
item.line2 = mContext.getString(R.string.quick_settings_connected);
}
item.canDisconnect = true;
items.add(connectedDevices, item);
connectedDevices++;
} else if (state == BluetoothProfile.STATE_CONNECTING) {
item.iconResId = R.drawable.ic_qs_bluetooth_connecting;
item.line2 = mContext.getString(R.string.quick_settings_connecting);
items.add(connectedDevices, item);
} else {
items.add(item);
}
if (++count == MAX_DEVICES) {
break;
}
}
}
mItems.setItems(items.toArray(new Item[items.size()]));
}
 
@Override
public void onDetailItemClick(Item item) {
if (item == null || item.tag == null) return;
final CachedBluetoothDevice device = (CachedBluetoothDevice) item.tag;
if (device != null && device.getMaxConnectionState()
== BluetoothProfile.STATE_DISCONNECTED) {
mController.connect(device);
}
}
 
@Override
public void onDetailItemDisconnect(Item item) {
if (item == null || item.tag == null) return;
final CachedBluetoothDevice device = (CachedBluetoothDevice) item.tag;
if (device != null) {
mController.disconnect(device);
}
}
}
}
在 handleClick() 做以下修改
     @Override
     protected void handleClick() {
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               Log.d("dong","enableWIFI:"+flag);
+               if("true".equals(flag)){
+                       return ;
+               }
refreshState(isEnabled ? null : ARG_SHOW_TRANSIENT_ENABLING);
mController.setBluetoothEnabled(!isEnabled);
}
```

#### 3.4 LocalBluetoothAdapter.java 关于开启蓝牙的管控

```
public boolean enable() {
          return mAdapter.enable();
      }
 
public boolean setBluetoothEnabled(boolean enabled) {
boolean success = enabled
? mAdapter.enable()
: mAdapter.disable();
 
if (success) {
setBluetoothStateInt(enabled
? BluetoothAdapter.STATE_TURNING_ON
: BluetoothAdapter.STATE_TURNING_OFF);
} else {
if (BluetoothUtils.V) {
Log.v(TAG, "setBluetoothEnabled call, manager didn't return " +
"success for enabled: " + enabled);
}
 
syncBluetoothState();
}
return success;
}
主要是这两个函数来管控蓝牙的开启 
修改如下:
     public boolean enable() {
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return false;
+               }               
         return mAdapter.enable();
     }
 
public boolean setBluetoothEnabled(boolean enabled) {
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return false;
+               }
boolean success = enabled
? mAdapter.enable()
: mAdapter.disable();
 
if (success) {
setBluetoothStateInt(enabled
? BluetoothAdapter.STATE_TURNING_ON
: BluetoothAdapter.STATE_TURNING_OFF);
} else {
if (BluetoothUtils.V) {
Log.v(TAG, "setBluetoothEnabled call, manager didn't return " +
"success for enabled: " + enabled);
}
 
syncBluetoothState();
}
return success;
}
```

#### 3.5 BluetoothEnabler.java 中关于管控蓝牙部分

```
@Override
public boolean onSwitchToggled(boolean isChecked) {
if (maybeEnforceRestrictions()) {
triggerParentPreferenceCallback(isChecked);
return true;
}
 
// Show toast message if Bluetooth is not allowed in airplane mode
if (isChecked &&
!WirelessUtils.isRadioAllowed(mContext, Settings.Global.RADIO_BLUETOOTH)) {
Toast.makeText(mContext, R.string.wifi_in_airplane_mode, Toast.LENGTH_SHORT).show();
// Reset switch to off
mSwitchController.setChecked(false);
triggerParentPreferenceCallback(false);
return false;
}
 
mMetricsFeatureProvider.action(mContext, mMetricsEvent, isChecked);
 
if (mBluetoothAdapter != null) {
boolean status = setBluetoothEnabled(isChecked);
// If we cannot toggle it ON then reset the UI assets:
// a) The switch should be OFF but it should still be togglable (enabled = True)
// b) The switch bar should have OFF text.
if (isChecked && !status) {
mSwitchController.setChecked(false);
mSwitchController.setEnabled(true);
mSwitchController.updateTitle(false);
triggerParentPreferenceCallback(false);
return false;
}
}
mSwitchController.setEnabled(false);
triggerParentPreferenceCallback(isChecked);
return true;
}
 
修改为:
  @Override
public boolean onSwitchToggled(boolean isChecked) {
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               Log.d("dong","BluetoothDashboardFragment:"+flag);
+               if("true".equals(flag)){
+                       Log.d("dong","onSwitchToggled:"+flag);
+                       return false;
+               } 
 
if (maybeEnforceRestrictions()) {
triggerParentPreferenceCallback(isChecked);
return true;
}
 
// Show toast message if Bluetooth is not allowed in airplane mode
if (isChecked &&
!WirelessUtils.isRadioAllowed(mContext, Settings.Global.RADIO_BLUETOOTH)) {
Toast.makeText(mContext, R.string.wifi_in_airplane_mode, Toast.LENGTH_SHORT).show();
// Reset switch to off
mSwitchController.setChecked(false);
triggerParentPreferenceCallback(false);
return false;
}
 
mMetricsFeatureProvider.action(mContext, mMetricsEvent, isChecked);
 
if (mBluetoothAdapter != null) {
boolean status = setBluetoothEnabled(isChecked);
// If we cannot toggle it ON then reset the UI assets:
// a) The switch should be OFF but it should still be togglable (enabled = True)
// b) The switch bar should have OFF text.
if (isChecked && !status) {
mSwitchController.setChecked(false);
mSwitchController.setEnabled(true);
mSwitchController.updateTitle(false);
triggerParentPreferenceCallback(false);
return false;
}
}
mSwitchController.setEnabled(false);
triggerParentPreferenceCallback(isChecked);
return true;
}
```

#### 3.6 BluetoothDashboardFragment.java 关于管控部分

```
@Override
public void onActivityCreated(Bundle savedInstanceState) {
super.onActivityCreated(savedInstanceState);
 
SettingsActivity activity = (SettingsActivity) getActivity();
mSwitchBar = activity.getSwitchBar();
mController = new BluetoothSwitchPreferenceController(activity,
new SwitchBarController(mSwitchBar), mFooterPreference);
Lifecycle lifecycle = getSettingsLifecycle();
if (lifecycle != null) {
lifecycle.addObserver(mController);
}
}
 
修改部分如下：
@Override
public void onActivityCreated(Bundle savedInstanceState) {
super.onActivityCreated(savedInstanceState);
 
SettingsActivity activity = (SettingsActivity) getActivity();
mSwitchBar = activity.getSwitchBar();
mController = new BluetoothSwitchPreferenceController(activity,
new SwitchBarController(mSwitchBar), mFooterPreference);
Lifecycle lifecycle = getSettingsLifecycle();
if (lifecycle != null) {
lifecycle.addObserver(mController);
}
+               String flag = SystemProperties.get("persist.sys.disableBT", "false");
+               Log.d("dong","BluetoothDashboardFragment:"+flag);
+               if("true".equals(flag)){
+                       Log.d("dong","BluetoothDashboardFragment1:"+flag);
+                       mSwitchBar.setEnabled(false);
+               } 
}
```

4. 总结
-----

 这就是管控蓝牙的全部代码 包含 Settings SystemUI SettingsLib 等部分的代码