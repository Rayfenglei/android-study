> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828561)

### 1. 概述

在 11.0 12.0 的产品开发中，对于调用 pm 的系统 api 实现静默安装已经受限，并且在 8.0 9.0 以后由于系统对于[权限控制](https://so.csdn.net/so/search?q=%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6&spm=1001.2101.3001.7020)越来越严格 所以说通过 adb shell 来安装卸载 app 都受到了限制但是又不想通过调用系统接口 弹出对话框 让用户同意后在安装 就只能使用静默安装了

### 2. 静默安装 app 和静默卸载 app 功能实现的核心类

```
frameworks/base/core/java/android/content/pm/PackageInstaller.java

```

### 3. 静默安装 app 和静默卸载 app 功能实现的功能实现和分析

在而系统 api 中 PackageInstaller.java 刚好提供了 关于安装 app 的相关方法来实现安装，来实现静默安装功能

### 3.1 PackageInstaller 相关安装 app 的 api 分析

```
public class PackageInstaller {
      private static final String TAG = "PackageInstaller";
  public PackageInstaller(IPackageInstaller installer,
              String installerPackageName, int userId) {
          mInstaller = installer;
          mInstallerPackageName = installerPackageName;
          mUserId = userId;
      }
    public @NonNull Session openSession(int sessionId) throws IOException {
          try {
              try {
                  return new Session(mInstaller.openSession(sessionId));
              } catch (RemoteException e) {
                  throw e.rethrowFromSystemServer();
              }
          } catch (RuntimeException e) {
              ExceptionUtils.maybeUnwrapIOException(e);
              throw e;
          }
      }
         public int createSession(@NonNull SessionParams params) throws IOException {
          try {
              return mInstaller.createSession(params, mInstallerPackageName, mUserId);
          } catch (RuntimeException e) {
              ExceptionUtils.maybeUnwrapIOException(e);
              throw e;
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }

```

通过 PackageInstaller.java 的源码发现，重要的方法 createSession(@NonNull SessionParams params) 和 Session openSession(int sessionId)  
正是在调用系统相关来实现安装 app 功能  
调用 PackageInstaller 安装 app 的案例

```
PackageInstaller packageInstaller = mContext.getPackageManager().getPackageInstaller();
PackageInstaller.SessionParams sessionParams
= new PackageInstaller.SessionParams(PackageInstaller
.SessionParams.MODE_FULL_INSTALL);

```

使用 PackageInstaller 根据相关 api 根据 apk 的包名来进行安装  
具体安装案例如下 ：

```
import android.app.Instrumentation;
import android.app.PendingIntent;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.pm.PackageInfo;
import android.content.pm.PackageInstaller;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.graphics.Bitmap;
import android.net.wifi.ScanResult;
import android.net.wifi.WifiConfiguration;
import android.net.wifi.WifiManager;
import android.net.wifi.WifiScanner;
import android.os.PowerManager;
import android.provider.Settings;
import android.text.TextUtils;
import android.util.Log;
import android.os.SystemProperties;
import java.io.BufferedOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.PrintWriter;
import java.sql.Time;
import java.util.ArrayList;
import java.util.List;
public class Utils {
private static final String TAG = "Utils";
private Context mContext;
private int mSessionId = -1;
private PackageInstaller.SessionCallback mSessionCallback;
public Utils(Context context) {
    mContext = context;

    gpsUtil = new GPSUtil(mContext);
    mSessionCallback = new InstallSessionCallback();
    context.getPackageManager().getPackageInstaller().registerSessionCallback(mSessionCallback);
}



private class InstallSessionCallback extends PackageInstaller.SessionCallback {
    @Override
    public void onCreated(int sessionId) {
        // empty
        Log.d(TAG, "onCreated()" + sessionId);
    }

    @Override
    public void onBadgingChanged(int sessionId) {
        // empty
        Log.d(TAG, "onBadgingChanged()" + sessionId + "active");
    }

    @Override
    public void onActiveChanged(int sessionId, boolean active) {
        // empty
        Log.d(TAG, "onActiveChanged()" + sessionId + "active" + active);
    }

    @Override
    public void onProgressChanged(int sessionId, float progress) {
        Log.d(TAG, "onProgressChanged()" + sessionId);
        if (sessionId == mSessionId) {
            int progres = (int) (Integer.MAX_VALUE * progress);
            Log.d(TAG, "onProgressChanged" + progres);
        }
    }

    @Override
    public void onFinished(int sessionId, boolean success) {
        // empty, finish is handled by InstallResultReceiver
        Log.d(TAG, "onFinished()" + sessionId + "success" + success);
        if (mSessionId == sessionId) {
            if (success) {
                Log.d(TAG, "onFinished() 安装成功");
            } else {
                Log.d(TAG, "onFinished() 安装失败");
            }

        }
    }
}

// 安装app
public void installApp(String apkFilePath) {
    Log.d(TAG, "installApp()------->" + apkFilePath);
    File apkFile = new File(apkFilePath);
    if (!apkFile.exists()) {
        Log.d(TAG, "文件不存在");
    }

    PackageInfo packageInfo = mContext.getPackageManager().getPackageArchiveInfo(apkFilePath, PackageManager.GET_ACTIVITIES | PackageManager.GET_SERVICES);
    if (packageInfo != null) {
        String packageName = packageInfo.packageName;
        int versionCode = packageInfo.versionCode;
        String versionName = packageInfo.versionName;
        Log.d(TAG, "package + versionName);
    }

    PackageInstaller packageInstaller = mContext.getPackageManager().getPackageInstaller();
    PackageInstaller.SessionParams sessionParams
            = new PackageInstaller.SessionParams(PackageInstaller
            .SessionParams.MODE_FULL_INSTALL);
    Log.d(TAG, "apkFile length" + apkFile.length());
    sessionParams.setSize(apkFile.length());

    try {
        mSessionId = packageInstaller.createSession(sessionParams);
    } catch (Exception e) {
        e.printStackTrace();
    }

    Log.d(TAG, "sessionId---->" + mSessionId);
    if (mSessionId != -1) {
        boolean copySuccess = onTransfesApkFile(apkFilePath);
        Log.d(TAG, "copySuccess---->" + copySuccess);
        if (copySuccess) {
            execInstallAPP();
        }
    }
}

/**
 * 通过文件流传输apk
 *
 * @param apkFilePath
 * @return
 */
private boolean onTransfesApkFile(String apkFilePath) {
    Log.d(TAG, "---------->onTransfesApkFile()<---------------------");
    InputStream in = null;
    OutputStream out = null;
    PackageInstaller.Session session = null;
    boolean success = false;
    try {
        File apkFile = new File(apkFilePath);
        session = mContext.getPackageManager().getPackageInstaller().openSession(mSessionId);
        out = session.openWrite("base.apk", 0, apkFile.length());
        in = new FileInputStream(apkFile);
        int total = 0, c;
        byte[] buffer = new byte[1024 * 1024];
        while ((c = in.read(buffer)) != -1) {
            total += c;
            out.write(buffer, 0, c);
        }
        session.fsync(out);
        Log.d(TAG, "streamed " + total + " bytes");
        success = true;
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (null != session) {
            session.close();
        }
        try {
            if (null != out) {
                out.close();
            }
            if (null != in) {
                in.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return success;
}

/**
 * 执行安装并通知安装结果
 */
private void execInstallAPP() {
    Log.d(TAG, "--------------------->execInstallAPP()<------------------");
    PackageInstaller.Session session = null;
    try {
        session = mContext.getPackageManager().getPackageInstaller().openSession(mSessionId);
        Intent intent = new Intent(mContext, InstallResultReceiver.class);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(mContext,
                1, intent,
                PendingIntent.FLAG_UPDATE_CURRENT);
        session.commit(pendingIntent.getIntentSender());
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (null != session) {
            session.close();
        }
    }
}

// 卸载app
public void uninstall(String packageName) {

    Intent broadcastIntent = new Intent(mContext, InstallResultReceiver.class);
    PendingIntent pendingIntent = PendingIntent.getBroadcast(mContext, 1,
            broadcastIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    PackageInstaller packageInstaller = mContext.getPackageManager().getPackageInstaller();
    packageInstaller.uninstall(packageName, pendingIntent.getIntentSender());
}

public void saveCapSceen(Bitmap bitmap) {
    File file = new File("/sdcard/HaiEr/screentshot");
    if (!file.exists()) {
        file.mkdirs();
    }
    File file1 = new File(file, System.currentTimeMillis() + ".jpg");
    try {
        BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(file1));
        bitmap.compress(Bitmap.CompressFormat.JPEG, 100, bos);
        bos.flush();
        bos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

}
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageInstaller;
import android.util.Log;
public class InstallResultReceiver extends BroadcastReceiver {
private static final String TAG = "InstallResultReceiver";
@Override
public void onReceive(Context context, Intent intent) {
    if (intent != null) {
        final int status = intent.getIntExtra(PackageInstaller.EXTRA_STATUS,
                PackageInstaller.STATUS_FAILURE);
        if (status == PackageInstaller.STATUS_SUCCESS) {
            // success
        } else {
            String msg = intent.getStringExtra(PackageInstaller.EXTRA_STATUS_MESSAGE);
            Log.e(TAG, "Install FAILURE status_massage" + msg);
        }
    }
}

}

```

通过上面的代码  
installApp() 传入 apk 路径  
uninstall 传入包名就可实现下载  
在 11.0 上面 普通 app 调用安装 apk 受到系统限制 安装不了  
需要添加到系统 sevice 的安装接口 然后调用安装 apk 的方法即可