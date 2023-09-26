> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127604256)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 系统设置蓝牙配对时去掉配对框实现直接配对功能的核心类](#t1)

[3. 系统设置蓝牙配对时去掉配对框实现直接配对功能的核心功能分析和实现](#t2)

[3.1 BluetoothPairingRequest 关于发出配对请求的相关分析](#t3)

[3.2 BluetoothPairingDialog.java 的相关配对框的源码分析](#t4)

[3.3 BluetoothPairingDialogFragment 相关配对 ui 布局分析](#t5)

[3.4 BluetoothPairingController.java 的相关配对源码分析](#t6)

1. 概述
-----

  在系统产品开发中，对于系统设置的蓝牙连接配对功能，在实现连接蓝牙设备的时候，在点击配对的时候，会首选弹出配对窗口然后点击确定，才开始配对，为了简化配对流程，产品需求要求去掉配对弹窗，直接配对，这就需要分析配对流程，然后去掉配对弹窗，实现直接配对功能

2. 系统设置蓝牙配对时去掉配对框实现直接配对功能的核心类
-----------------------------

```
    packages/apps/Settings/src/com/android/settings/bluetooth/BluetoothPairingRequest.java
  packages\apps\Settings\src\com\android\settings\bluetooth\BluetoothPairingDialog.java
  packages\apps\Settings\src\com\android\settings\bluetooth\BluetoothPairingDialogFragment.java
  packages\apps\Settings\src\com\android\settings\bluetooth\BluetoothPairingController.java
```

3. 系统设置蓝牙配对时去掉配对框实现直接配对功能的核心功能分析和实现
-----------------------------------

### 3.1 BluetoothPairingRequest 关于发出配对请求的[相关分析](https://so.csdn.net/so/search?q=%E7%9B%B8%E5%85%B3%E5%88%86%E6%9E%90&spm=1001.2101.3001.7020)

```
public final class BluetoothPairingRequest extends BroadcastReceiver {
  
      @Override
      public void onReceive(Context context, Intent intent) {
          String action = intent.getAction();
          if (action == null || !action.equals(BluetoothDevice.ACTION_PAIRING_REQUEST)) {
              return;
          }
  
          PowerManager powerManager = context.getSystemService(PowerManager.class);
          BluetoothDevice device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
          int pairingVariant = intent.getIntExtra(BluetoothDevice.EXTRA_PAIRING_VARIANT,
                  BluetoothDevice.ERROR);
          String deviceAddress = device != null ? device.getAddress() : null;
          String deviceName = device != null ? device.getName() : null;
          boolean shouldShowDialog = LocalBluetoothPreferences.shouldShowDialogInForeground(
                  context, deviceAddress, deviceName);
  
          // Skips consent pairing dialog if the device was recently associated with CDM
          if (pairingVariant == BluetoothDevice.PAIRING_VARIANT_CONSENT
                  && device.canBondWithoutDialog()) {
              device.setPairingConfirmation(true);
          } else if (powerManager.isInteractive() && shouldShowDialog) {
              // Since the screen is on and the BT-related activity is in the foreground,
              // just open the dialog
              // convert broadcast intent into activity intent (same action string)
              Intent pairingIntent = BluetoothPairingService.getPairingDialogIntent(context, intent,
                      BluetoothDevice.EXTRA_PAIRING_INITIATOR_FOREGROUND);
  
              context.startActivityAsUser(pairingIntent, UserHandle.CURRENT);
          } else {
              // Put up a notification that leads to the dialog
              intent.setClass(context, BluetoothPairingService.class);
              intent.setAction(BluetoothDevice.ACTION_PAIRING_REQUEST);
              context.startServiceAsUser(intent, UserHandle.CURRENT);
          }
      }
  }
```

在 BluetoothPairingRequest 的相关源码中分析得知，在点击配对发出配对请求时，会调用  
device 的相关方法获取 deviceAddress deviceName 获取地址名称，然后调用 LocalBluetoothPreferences.shouldShowDialogInForeground(  
                  context, deviceAddress, deviceName); 来判断是否弹出配对请求框  
这其实就是在配对框的处理是在 BluetoothPairingDialog.java 中进行，接下来分析下  
BluetoothPairingDialog.java 的相关源码

### 3.2 BluetoothPairingDialog.java 的相关配对框的源码分析

```
      @Override
     protected void onCreate(@Nullable Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
 
         getWindow().addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);
         Intent intent = getIntent();
         if (intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE) == null) {
             // Error handler for the case that dialog is started from adb command.
             finish();
             return;
         }
         mBluetoothPairingController = new BluetoothPairingController(intent, this);
         // build the dialog fragment
         boolean fragmentFound = true;
         // check if the fragment has been preloaded
         BluetoothPairingDialogFragment bluetoothFragment =
             (BluetoothPairingDialogFragment) getSupportFragmentManager().
                     findFragmentByTag(FRAGMENT_TAG);
         // dismiss the fragment if it is already used
         if (bluetoothFragment != null && (bluetoothFragment.isPairingControllerSet()
             || bluetoothFragment.isPairingDialogActivitySet())) {
             bluetoothFragment.dismiss();
             bluetoothFragment = null;
         }
         // build a new fragment if it is null
         if (bluetoothFragment == null) {
             fragmentFound = false;
             bluetoothFragment = new BluetoothPairingDialogFragment();
         }
         bluetoothFragment.setPairingController(mBluetoothPairingController);
         bluetoothFragment.setPairingDialogActivity(this);
         // pass the fragment to the manager when it is created from scratch
         if (!fragmentFound) {
             bluetoothFragment.show(getSupportFragmentManager(), FRAGMENT_TAG);
          }
          /*
           * Leave this registered through pause/resume since we still want to
           * finish the activity in the background if pairing is canceled.
           */
          registerReceiver(mReceiver, new IntentFilter(BluetoothDevice.ACTION_PAIRING_CANCEL));
          registerReceiver(mReceiver, new IntentFilter(BluetoothDevice.ACTION_BOND_STATE_CHANGED));
          mReceiverRegistered = true;
      }
```

  在 BluetoothPairingDialog.java 的 onCreate(@[Nullable](https://so.csdn.net/so/search?q=Nullable&spm=1001.2101.3001.7020) Bundle savedInstanceState) 的布局构造方法中可以看出在这里的布局时间上都是由 BluetoothPairingDialogFragment() 来负责具体的配对 UI 加载布局的，所以接下来分析下 BluetoothPairingDialogFragment() 中的相关配对 UI 布局

### 3.3 BluetoothPairingDialogFragment 相关配对 ui 布局分析

```
        @Override
     public Dialog onCreateDialog(Bundle savedInstanceState) {
         if (!isPairingControllerSet()) {
             throw new IllegalStateException(
                 "Must call setPairingController() before showing dialog");
         }
         if (!isPairingDialogActivitySet()) {
             throw new IllegalStateException(
                 "Must call setPairingDialogActivity() before showing dialog");
         }
         mBuilder = new AlertDialog.Builder(getActivity());
         mDialog = setupDialog();
         mDialog.setCanceledOnTouchOutside(false);
         return mDialog;
     }
 
     @Override
     public void beforeTextChanged(CharSequence s, int start, int count, int after) {
     }
 
     @Override
     public void onTextChanged(CharSequence s, int start, int before, int count) {
     }
 
     @Override
     public void afterTextChanged(Editable s) {
         // enable the positive button when we detect potentially valid input
         Button positiveButton = mDialog.getButton(DialogInterface.BUTTON_POSITIVE);
         if (positiveButton != null) {
             positiveButton.setEnabled(mPairingController.isPasskeyValid(s));
         }
          // notify the controller about user input
          mPairingController.updateUserInput(s.toString());
      }
  
      @Override
      public void onClick(DialogInterface dialog, int which) {
          if (which == DialogInterface.BUTTON_POSITIVE) {
              mPairingController.onDialogPositiveClick(this);
          } else if (which == DialogInterface.BUTTON_NEGATIVE) {
              mPairingController.onDialogNegativeClick(this);
          }
          mPairingDialogActivity.dismiss();
      }
  
      @Override
      public int getMetricsCategory() {
          return SettingsEnums.BLUETOOTH_DIALOG_FRAGMENT;
      }
  
      /**
       * Used in testing to get a reference to the dialog.
       * @return - The fragments current dialog
       */
      protected AlertDialog getmDialog() {
          return mDialog;
      }
  
      /**
       * Sets the controller that the fragment should use. this method MUST be called
       * before you try to show the dialog or an error will be thrown. An implementation
       * of a pairing controller can be found at {@link BluetoothPairingController}. A
       * controller may not be substituted once it is assigned. Forcibly switching a
       * controller for a new one will lead to undefined behavior.
       */
      void setPairingController(BluetoothPairingController pairingController) {
          if (isPairingControllerSet()) {
              throw new IllegalStateException("The controller can only be set once. "
                      + "Forcibly replacing it will lead to undefined behavior");
          }
          mPairingController = pairingController;
      }
  
      /**
       * Checks whether mPairingController is set
       * @return True when mPairingController is set, False otherwise
       */
      boolean isPairingControllerSet() {
          return mPairingController != null;
      }
  
      /**
       * Sets the BluetoothPairingDialog activity that started this fragment
       * @param pairingDialogActivity The pairing dialog activty that started this fragment
       */
      void setPairingDialogActivity(BluetoothPairingDialog pairingDialogActivity) {
          if (isPairingDialogActivitySet()) {
              throw new IllegalStateException("The pairing dialog activity can only be set once");
          }
          mPairingDialogActivity = pairingDialogActivity;
      }
  
      /**
       * Checks whether mPairingDialogActivity is set
       * @return True when mPairingDialogActivity is set, False otherwise
       */
      boolean isPairingDialogActivitySet() {
          return mPairingDialogActivity != null;
      }
  
      /**
       * Creates the appropriate type of dialog and returns it.
       */
      private AlertDialog setupDialog() {
          AlertDialog dialog;
          switch (mPairingController.getDialogType()) {
              case BluetoothPairingController.USER_ENTRY_DIALOG:
                  dialog = createUserEntryDialog();
                  break;
              case BluetoothPairingController.CONFIRMATION_DIALOG:
                  dialog = createConsentDialog();
                  break;
              case BluetoothPairingController.DISPLAY_PASSKEY_DIALOG:
                  dialog = createDisplayPasskeyOrPinDialog();
                  break;
              default:
                  dialog = null;
                  Log.e(TAG, "Incorrect pairing type received, not showing any dialog");
          }
          return dialog;
      }
  
      /**
       * Helper method to return the text of the pin entry field - this exists primarily to help us
       * simulate having existing text when the dialog is recreated, for example after a screen
       * rotation.
       */
      @VisibleForTesting
      CharSequence getPairingViewText() {
          if (mPairingView != null) {
              return mPairingView.getText();
          }
          return null;
      }
  
      /**
       * Returns a dialog with UI elements that allow a user to provide input.
       */
      private AlertDialog createUserEntryDialog() {
          mBuilder.setTitle(getString(R.string.bluetooth_pairing_request,
                  mPairingController.getDeviceName()));
          mBuilder.setView(createPinEntryView());
          mBuilder.setPositiveButton(getString(android.R.string.ok), this);
          mBuilder.setNegativeButton(getString(android.R.string.cancel), this);
          AlertDialog dialog = mBuilder.create();
          dialog.setOnShowListener(d -> {
              if (TextUtils.isEmpty(getPairingViewText())) {
                  mDialog.getButton(Dialog.BUTTON_POSITIVE).setEnabled(false);
              }
              if (mPairingView != null && mPairingView.requestFocus()) {
                  InputMethodManager imm = (InputMethodManager)
                          getContext().getSystemService(Context.INPUT_METHOD_SERVICE);
                  if (imm != null) {
                      imm.showSoftInput(mPairingView, InputMethodManager.SHOW_IMPLICIT);
                  }
              }
          });
          return dialog;
      }
  
      /**
       * Creates the custom view with UI elements for user input.
       */
      private View createPinEntryView() {
          View view = getActivity().getLayoutInflater().inflate(R.layout.bluetooth_pin_entry, null);
          TextView messageViewCaptionHint = (TextView) view.findViewById(R.id.pin_values_hint);
          TextView messageView2 = (TextView) view.findViewById(R.id.message_below_pin);
          CheckBox alphanumericPin = (CheckBox) view.findViewById(R.id.alphanumeric_pin);
          CheckBox contactSharing = (CheckBox) view.findViewById(
                  R.id.phonebook_sharing_message_entry_pin);
          contactSharing.setText(getString(R.string.bluetooth_pairing_shares_phonebook,
                  mPairingController.getDeviceName()));
          EditText pairingView = (EditText) view.findViewById(R.id.text);
  
          contactSharing.setVisibility(mPairingController.isProfileReady()
                  ? View.GONE : View.VISIBLE);
          mPairingController.setContactSharingState();
          contactSharing.setOnCheckedChangeListener(mPairingController);
          contactSharing.setChecked(mPairingController.getContactSharingState());
  
          mPairingView = pairingView;
  
          pairingView.setInputType(InputType.TYPE_CLASS_NUMBER);
          pairingView.addTextChangedListener(this);
          alphanumericPin.setOnCheckedChangeListener((buttonView, isChecked) -> {
              // change input type for soft keyboard to numeric or alphanumeric
              if (isChecked) {
                  mPairingView.setInputType(InputType.TYPE_CLASS_TEXT);
              } else {
                  mPairingView.setInputType(InputType.TYPE_CLASS_NUMBER);
              }
          });
  
          int messageId = mPairingController.getDeviceVariantMessageId();
          int messageIdHint = mPairingController.getDeviceVariantMessageHintId();
          int maxLength = mPairingController.getDeviceMaxPasskeyLength();
          alphanumericPin.setVisibility(mPairingController.pairingCodeIsAlphanumeric()
                  ? View.VISIBLE : View.GONE);
          if (messageId != BluetoothPairingController.INVALID_DIALOG_TYPE) {
              messageView2.setText(messageId);
          } else {
              messageView2.setVisibility(View.GONE);
          }
          if (messageIdHint != BluetoothPairingController.INVALID_DIALOG_TYPE) {
              messageViewCaptionHint.setText(messageIdHint);
          } else {
              messageViewCaptionHint.setVisibility(View.GONE);
          }
          pairingView.setFilters(new InputFilter[]{
                  new LengthFilter(maxLength)});
  
          return view;
      }
  
      /**
       * Creates a dialog with UI elements that allow the user to confirm a pairing request.
       */
      private AlertDialog createConfirmationDialog() {
          mBuilder.setTitle(getString(R.string.bluetooth_pairing_request,
                  mPairingController.getDeviceName()));
          mBuilder.setView(createView());
          mBuilder.setPositiveButton(getString(R.string.bluetooth_pairing_accept), this);
          mBuilder.setNegativeButton(getString(R.string.bluetooth_pairing_decline), this);
          AlertDialog dialog = mBuilder.create();
          return dialog;
      }
  
      /**
       * Creates a dialog with UI elements that allow the user to consent to a pairing request.
       */
      private AlertDialog createConsentDialog() {
          return createConfirmationDialog();
      }
  
      /**
       * Creates a dialog that informs users of a pairing request and shows them the passkey/pin
       * of the device.
       */
      private AlertDialog createDisplayPasskeyOrPinDialog() {
          mBuilder.setTitle(getString(R.string.bluetooth_pairing_request,
                  mPairingController.getDeviceName()));
          mBuilder.setView(createView());
          mBuilder.setNegativeButton(getString(android.R.string.cancel), this);
          AlertDialog dialog = mBuilder.create();
  
          // Tell the controller the dialog has been created.
          mPairingController.notifyDialogDisplayed();
  
          return dialog;
      }
  
      /**
       * Creates a custom view for dialogs which need to show users additional information but do
       * not require user input.
       */
      private View createView() {
          View view = getActivity().getLayoutInflater().inflate(R.layout.bluetooth_pin_confirm, null);
          TextView pairingViewCaption = (TextView) view.findViewById(R.id.pairing_caption);
          TextView pairingViewContent = (TextView) view.findViewById(R.id.pairing_subhead);
          TextView messagePairing = (TextView) view.findViewById(R.id.pairing_code_message);
          CheckBox contactSharing = (CheckBox) view.findViewById(
                  R.id.phonebook_sharing_message_confirm_pin);
          contactSharing.setText(getString(R.string.bluetooth_pairing_shares_phonebook,
                  mPairingController.getDeviceName()));
  
          contactSharing.setVisibility(
                  mPairingController.isProfileReady() ? View.GONE : View.VISIBLE);
          mPairingController.setContactSharingState();
          contactSharing.setChecked(mPairingController.getContactSharingState());
          contactSharing.setOnCheckedChangeListener(mPairingController);
  
          messagePairing.setVisibility(mPairingController.isDisplayPairingKeyVariant()
                  ? View.VISIBLE : View.GONE);
          if (mPairingController.hasPairingContent()) {
              pairingViewCaption.setVisibility(View.VISIBLE);
              pairingViewContent.setVisibility(View.VISIBLE);
              pairingViewContent.setText(mPairingController.getPairingContent());
          }
          return view;
      }
```

在 BluetoothPairingDialogFragment 的相关方法发现，在 createView() 负责构建配对 UI 的相关布局  
而在 createDisplayPasskeyOrPinDialog() 就是配对码弹窗布局，而在 onClick(DialogInterface dialog, int which) 就是配对弹窗 ui 确定取消的  
按键事件处理，mPairingController.onDialogPositiveClick(this); 就是点击配对以后的相关执行流程  
所以接下来看下 BluetoothPairingController.java 的相关源码分析

###   
3.4 BluetoothPairingController.java 的相关配对源码分析

```
       @Override
      public void onDialogPositiveClick(BluetoothPairingDialogFragment dialog) {
          if (mPbapAllowed) {
              mDevice.setPhonebookAccessPermission(BluetoothDevice.ACCESS_ALLOWED);
          } else {
              mDevice.setPhonebookAccessPermission(BluetoothDevice.ACCESS_REJECTED);
          }
  
          if (getDialogType() == USER_ENTRY_DIALOG) {
              onPair(mUserInput);
          } else {
              onPair(null);
          }
      }
  
      @Override
      public void onDialogNegativeClick(BluetoothPairingDialogFragment dialog) {
          mDevice.setPhonebookAccessPermission(BluetoothDevice.ACCESS_REJECTED);
          onCancel();
      }
 
     private void onPair(String passkey) {
          Log.d(TAG, "Pairing dialog accepted");
          switch (mType) {
              case BluetoothDevice.PAIRING_VARIANT_PIN:
              case BluetoothDevice.PAIRING_VARIANT_PIN_16_DIGITS:
                  mDevice.setPin(passkey);
                  break;
  
  
              case BluetoothDevice.PAIRING_VARIANT_PASSKEY_CONFIRMATION:
              case BluetoothDevice.PAIRING_VARIANT_CONSENT:
                  mDevice.setPairingConfirmation(true);
                  break;
  
              case BluetoothDevice.PAIRING_VARIANT_DISPLAY_PASSKEY:
              case BluetoothDevice.PAIRING_VARIANT_DISPLAY_PIN:
              case BluetoothDevice.PAIRING_VARIANT_OOB_CONSENT:
              case BluetoothDevice.PAIRING_VARIANT_PASSKEY:
                  // Do nothing.
                  break;
  
              default:
                  Log.e(TAG, "Incorrect pairing type received");
          }
      }
```

从 BluetoothPairingController.java.onDialogPositiveClick(this) 中可以看出点击配对后调用的就是  
onPair(String passkey) 来执行相关的配对功能，实现无配对弹窗执行配对就需要在配对窗口直接调用这个配对方法  
所以具体修改如下:

```
1.BluetoothPairingDialog.java中修改
      @Override
     protected void onCreate(@Nullable Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
 
         getWindow().addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);
         Intent intent = getIntent();
         if (intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE) == null) {
             // Error handler for the case that dialog is started from adb command.
             finish();
             return;
         }
         mBluetoothPairingController = new BluetoothPairingController(intent, this);
         // build the dialog fragment
       //add core start 
       + mBluetoothPairingController.onPair(null);
       + finish();
       //add core end
......
      }
在BluetoothPairingDialog.java中的onCreate中直接调用mBluetoothPairingController.onPair(null);然后
finish掉窗口，然后把BluetoothPairingController的onPair(null)改为public就可以实现功能了
```