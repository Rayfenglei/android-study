> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760600)

### 1. 概述

在 11.0 的系统产品开发中，对于蓝牙的管控也是常有的功能，比如禁止连接蓝牙，禁止蓝牙传输文件等功能，最近有产品功能需求，要求禁止蓝牙传输文件，这就要从蓝牙文件传输流程分析，然后禁用传输功能就可以了

### 2. 蓝牙去掉传输文件的功能的核心类

```
packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppLauncherActivity.java
/packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppReceiver.java
packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppManager.java
packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java

```

### 3. 蓝牙去掉传输文件的功能的核心功能分析和实现

### 3.1 关于 BluetoothOppLauncherActivity.java 处理蓝牙传输请求的分析

关于蓝牙相关的功能都是在 Bluetooth 这个 app 中的在蓝牙文件传输过程中，开始传输的时候 由 packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppLauncherActivity.java 来处理  
接下来看相关源码

```
public class BluetoothOppLauncherActivity extends Activity {
    private static final String TAG = "BluetoothLauncherActivity";
    private static final boolean D = Constants.DEBUG;
    private static final boolean V = Constants.VERBOSE;

    // Regex that matches characters that have special meaning in HTML. '<', '>', '&' and
    // multiple continuous spaces.
    private static final Pattern PLAIN_TEXT_TO_ESCAPE = Pattern.compile("[<>&]| {2,}|\r?\n");

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(new ProgressBar(this));

        Intent intent = getIntent();
        String action = intent.getAction();
        if (action == null) {
            Log.w(TAG, " Received " + intent + " with null action");
            finish();
            return;
        }

        if (action == null) {
            Log.w(TAG, "action is null");
            finish();
            return;
        }

        if (action.equals(Intent.ACTION_SEND) || action.equals(Intent.ACTION_SEND_MULTIPLE)) {
            //Check if Bluetooth is available in the beginning instead of at the end
            if (!isBluetoothAllowed()) {
                Intent in = new Intent(this, BluetoothOppBtErrorActivity.class);
                in.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                in.putExtra("title", this.getString(R.string.airplane_error_title));
                in.putExtra("content", this.getString(R.string.airplane_error_msg));
                startActivity(in);
                finish();
                return;
            }

            /*
             * Other application is trying to share a file via Bluetooth,
             * probably Pictures, videos, or vCards. The Intent should contain
             * an EXTRA_STREAM with the data to attach.
             */
            if (action.equals(Intent.ACTION_SEND)) {
                // TODO: handle type == null case
                final String type = intent.getType();
                final Uri stream = (Uri) intent.getParcelableExtra(Intent.EXTRA_STREAM);
                CharSequence extraText = intent.getCharSequenceExtra(Intent.EXTRA_TEXT);
                // If we get ACTION_SEND intent with EXTRA_STREAM, we'll use the
                // uri data;
                // If we get ACTION_SEND intent without EXTRA_STREAM, but with
                // EXTRA_TEXT, we will try send this TEXT out; Currently in
                // Browser, share one link goes to this case;
                if (stream != null && type != null) {
                    if (V) {
                        Log.v(TAG,
                                "Get ACTION_SEND intent: Uri = " + stream + "; mimetype = " + type);
                    }
                    // Save type/stream, will be used when adding transfer
                    // session to DB.
                    Thread t = new Thread(new Runnable() {
                        @Override
                        public void run() {
                            sendFileInfo(type, stream.toString(), false /* isHandover */, true /*
                             fromExternal */);
                        }
                    });
                    t.start();
                    return;
                } else if (extraText != null && type != null) {
                    if (V) {
                        Log.v(TAG,
                                "Get ACTION_SEND intent with Extra_text = " + extraText.toString()
                                        + "; mimetype = " + type);
                    }
                    final Uri fileUri = creatFileForSharedContent(
                            this.createCredentialProtectedStorageContext(), extraText);
                    if (fileUri != null) {
                        Thread t = new Thread(new Runnable() {
                            @Override
                            public void run() {
                                sendFileInfo(type, fileUri.toString(), false /* isHandover */,
                                        false /* fromExternal */);
                            }
                        });
                        t.start();
                        return;
                    } else {
                        Log.w(TAG, "Error trying to do set text...File not created!");
                        finish();
                        return;
                    }
                } else {
                    Log.e(TAG, "type is null; or sending file URI is null");
                    finish();
                    return;
                }
            } else if (action.equals(Intent.ACTION_SEND_MULTIPLE)) {
                final String mimeType = intent.getType();
                final ArrayList<Uri> uris = intent.getParcelableArrayListExtra(Intent.EXTRA_STREAM);
                if (mimeType != null && uris != null) {
                    if (V) {
                        Log.v(TAG, "Get ACTION_SHARE_MULTIPLE intent: uris " + uris + "\n Type= "
                                + mimeType);
                    }
                    Thread t = new Thread(new Runnable() {
                        @Override
                        public void run() {
                            try {
                                BluetoothOppManager.getInstance(BluetoothOppLauncherActivity.this)
                                        .saveSendingFileInfo(mimeType, uris, false /* isHandover */,
                                                true /* fromExternal */);
                                //Done getting file info..Launch device picker
                                //and finish this activity
                                launchDevicePicker();
                                finish();
                            } catch (IllegalArgumentException exception) {
                                showToast(exception.getMessage());
                                finish();
                            }
                        }
                    });
                    t.start();
                    return;
                } else {
                    Log.e(TAG, "type is null; or sending files URIs are null");
                    finish();
                    return;
                }
            }
 .....
    }

```

在 onCreate() 判断当前蓝牙是否打开，蓝牙打开后通过 lauchDevicePicker（）跳转到蓝牙 DeviceListPreferenceFragment(DevicePickerFragment) 选择设备  
通过 BluetoothDevicePicker.ACTION_DEVICE_SELECTED 查找，会在 / packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppReceiver.java 这个找到对该广播的处理,

### 3.2 BluetoothOppReceiver.java 关于蓝牙广播的处理

```
        if (action.equals(BluetoothDevicePicker.ACTION_DEVICE_SELECTED)) {
            BluetoothOppManager mOppManager = BluetoothOppManager.getInstance(context);

            BluetoothDevice remoteDevice = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);

            if (D) {
                Log.d(TAG, "Received BT device selected intent, bt device: " + remoteDevice);
            }

            if (remoteDevice == null) {
                mOppManager.cleanUpSendingFileInfo();
                return;
            }
            // Insert transfer session record to database
            mOppManager.startTransfer(remoteDevice);

            // Display toast message
            String deviceName = mOppManager.getDeviceName(remoteDevice);
            String toastMsg;
            int batchSize = mOppManager.getBatchSize();
            if (mOppManager.mMultipleFlag) {
                toastMsg = context.getString(R.string.bt_toast_5, Integer.toString(batchSize),
                        deviceName);
            } else {
                toastMsg = context.getString(R.string.bt_toast_4, deviceName);
            }
            Toast.makeText(context, toastMsg, Toast.LENGTH_SHORT).show();
        }

```

在收到蓝牙广播后通过 BluetoothDevicePicker.ACTION_DEVICE_SELECTED 做判断，然后获取蓝牙对象开始传输  
蓝牙文件请求  
mOppManager.startTransfer(remoteDevice)，在 packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppManager.java，里面开启线程执行发送动作

### 3.3BluetoothOppManager.java 在线程中请求传输蓝牙文件

```
    /**
     * Fork a thread to insert share info to db.
     */
    public void startTransfer(BluetoothDevice device) {
        if (V) {
            Log.v(TAG, "Active InsertShareThread number is : " + mInsertShareThreadNum);
        }
        InsertShareInfoThread insertThread;
        synchronized (BluetoothOppManager.this) {
            if (mInsertShareThreadNum > ALLOWED_INSERT_SHARE_THREAD_NUMBER) {
                Log.e(TAG, "Too many shares user triggered concurrently!");

                // Notice user
                Intent in = new Intent(mContext, BluetoothOppBtErrorActivity.class);
                in.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                in.putExtra("title", mContext.getString(R.string.enabling_progress_title));
                in.putExtra("content", mContext.getString(R.string.ErrorTooManyRequests));
                mContext.startActivity(in);

                return;
            }
            insertThread = new InsertShareInfoThread(device, mMultipleFlag, mMimeTypeOfSendingFile,
                    mUriOfSendingFile, mMimeTypeOfSendingFiles, mUrisOfSendingFiles,
                    mIsHandoverInitiated);
            if (mMultipleFlag) {
                mFileNumInBatch = mUrisOfSendingFiles.size();
            }
        }

        insertThread.start();
    }
InsertShareInfoThread的方法如下：
        @Override
        public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            if (mRemoteDevice == null) {
                Log.e(TAG, "Target bt device is null!");
                return;
            }
            if (mIsMultiple) {
                insertMultipleShare();
            } else {
                insertSingleShare();
            }
            synchronized (BluetoothOppManager.this) {
                mInsertShareThreadNum--;
            }
        }

        /**
         * Insert multiple sending sessions to db, only used by Opp application.
         */
        private void insertMultipleShare() {
            int count = mUris.size();
            Long ts = System.currentTimeMillis();
            for (int i = 0; i < count; i++) {
                Uri fileUri = mUris.get(i);
                ContentValues values = new ContentValues();
                values.put(BluetoothShare.URI, fileUri.toString());
                ContentResolver contentResolver = mContext.getContentResolver();
                fileUri = BluetoothOppUtility.originalUri(fileUri);
                String contentType = contentResolver.getType(fileUri);
                if (V) {
                    Log.v(TAG, "Got mimetype: " + contentType + "  Got uri: " + fileUri);
                }
                if (TextUtils.isEmpty(contentType)) {
                    contentType = mTypeOfMultipleFiles;
                }

                values.put(BluetoothShare.MIMETYPE, contentType);
                values.put(BluetoothShare.DESTINATION, mRemoteDevice.getAddress());
                values.put(BluetoothShare.TIMESTAMP, ts);
                if (mIsHandoverInitiated) {
                    values.put(BluetoothShare.USER_CONFIRMATION,
                            BluetoothShare.USER_CONFIRMATION_HANDOVER_CONFIRMED);
                }
                final Uri contentUri =
                        mContext.getContentResolver().insert(BluetoothShare.CONTENT_URI, values);
                if (V) {
                    Log.v(TAG, "Insert contentUri: " + contentUri + "  to device: " + getDeviceName(
                            mRemoteDevice));
                }
            }
        }

        /**
         * Insert single sending session to db, only used by Opp application.
         */
        private void insertSingleShare() {
            ContentValues values = new ContentValues();
            values.put(BluetoothShare.URI, mUri);
            values.put(BluetoothShare.MIMETYPE, mTypeOfSingleFile);
            values.put(BluetoothShare.DESTINATION, mRemoteDevice.getAddress());
            if (mIsHandoverInitiated) {
                values.put(BluetoothShare.USER_CONFIRMATION,
                        BluetoothShare.USER_CONFIRMATION_HANDOVER_CONFIRMED);
            }
            final Uri contentUri =
                    mContext.getContentResolver().insert(BluetoothShare.CONTENT_URI, values);
            if (V) {
                Log.v(TAG, "Insert contentUri: " + contentUri + "  to device: " + getDeviceName(
                        mRemoteDevice));
            }
        }
    }

```

在代码中的 insertSingleShare() 在它的实现会看到 mContext.getContentResolver().insert 要去 provider 里找到 insert() 函数了，  
对应的代码在 BluetoothOppProvider.java  
(bluetooth\src\com\android\bluetooth\opp)，insert 的函数实现如下，里面又拉起 BluetoothOppService，开始还以为只是针对数据库的操作，差点错过了风景。路径 / packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java

### 3.4BluetoothOppService.java 蓝牙传输服务的相关功能分析

```
public Uri insert(Uri uri, ContentValues values) {  
  if (rowID != -1) {  
     context.startService(new Intent(context, BluetoothOppService.class));  
    ret = Uri.parse(BluetoothShare.CONTENT_URI + "/" + rowID);  
     context.getContentResolver().notifyChange(uri, null);  
 } else {  
      if (D) Log.d(TAG, "couldn't insert into btopp database");  
 }   

```

在 BluetoothOppService 的 onStartCommand 方法中会看到 updateFromProvider()，  
所以最终是通过 updateFromProvider() 来负责传输文件的  
具体修改如下:

```
diff --git a/packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java b/packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java
old mode 100644 (file)
new mode 100755 (executable)
index 0b6af53..3a9fd17
--- a/packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java
+++ b/packages/apps/Bluetooth/src/com/android/bluetooth/opp/BluetoothOppService.java
@@ -101,7 +101,7 @@ public class BluetoothOppService extends ProfileService implements IObexConnecti
             if (V) {
                 Log.v(TAG, "ContentObserver received notification");
             }
-            updateFromProvider();
+            //updateFromProvider();//motify for Disable Bluetooth transmission
         }
     }
 
@@ -237,8 +237,8 @@ public class BluetoothOppService extends ProfileService implements IObexConnecti
         getContentResolver().registerContentObserver(BluetoothShare.CONTENT_URI, true, mObserver);
         mNotifier = new BluetoothOppNotification(this);
         mNotifier.mNotificationMgr.cancelAll();
-        mNotifier.updateNotification();
-        updateFromProvider();
+        //mNotifier.updateNotification();//motify Disable Bluetooth transmission
+        //updateFromProvider();//motify for Disable Bluetooth transmission
         setBluetoothOppService(this);
         return true;

```