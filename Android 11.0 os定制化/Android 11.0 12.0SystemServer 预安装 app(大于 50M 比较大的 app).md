> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828274)

### 1. 概述

在 11.0 12.0 定制化开发中预置应用宝到系统中，  
1. 如果直接按照常规方案预置应用宝到 system/app 下的话，会报好多 Selinux 错误，导致应用闪退  
而应用宝又申请了好多并不需要的权限例如 su  
2. 如果预安装比较大的 app 比如 360 浏览器 会发现很难编译过 也只能通过系统服务静默安装  
本次解决方案在系统服务 systemserver 里面启动安装  
预安装应用宝到系统中，

### 2.SystemServer 预安装 app(大于 50M 比较大的 app) 的核心类

```
device\sprd\sharkl5Pro\ums512_1h10\ums512_1h10_Base.mk
frameworks/base/services/java/com/android/server/SystemServer.java

```

### 3.SystemServer 预安装 app(大于 50M 比较大的 app) 的核心功能分析和实现

第一步 系统编译 apk 到 data/app 中  
在 device\sprd\sharkl5Pro\ums512_1h10\ums512_1h10_Base.mk 中

```
PRODUCT_COPY_FILES += 
$(BOARDDIR)/rootdir/root/init.$(TARGET_BOARD).rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/hw/init.$(TARGET_BOARD).rc 
$(ROOTDIR)/prodnv/PCBA.conf:$(TARGET_COPY_OUT_VENDOR)/etc/PCBA.conf 
$(ROOTDIR)/prodnv/BBAT.conf:$(TARGET_COPY_OUT_VENDOR)/etc/BBAT.conf 
$(ROOTDIR)/system/etc/audio_params/audio_config.xml:$(TARGET_COPY_OUT_VENDOR)/etc/audio_config.xml 
$(ROOTDIR)/system/etc/audio_params/audio_pcm.xml:$(TARGET_COPY_OUT_VENDOR)/etc/audio_pcm.xml 
$(PLATCOMM)/rootdir/root/ueventd.common.rc:$(TARGET_COPY_OUT_VENDOR)/ueventd.rc 
$(PLATCOMM)/rootdir/root/init.common.usb.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/hw/init.$(TARGET_BOARD).usb.rc 
frameworks/native/data/etc/android.hardware.camera.flash-autofocus.xml:$(TARGET_COPY_OUT_VENDOR)/etc/permissions/android.hardware.camera.flash-autofocus.xml 
frameworks/native/data/etc/android.hardware.camera.front.xml:$(TARGET_COPY_OUT_VENDOR)/etc/permissions/android.hardware.camera.front.xml 
frameworks/native/data/etc/android.hardware.opengles.aep.xml:$(TARGET_COPY_OUT_VENDOR)/etc/permissions/android.hardware.opengles.aep.xml 
$(BOARDDIR)/log_conf/slog_modem_$(TARGET_BUILD_VARIANT).conf:vendor/etc/slog_modem.conf 
$(BOARDDIR)/log_conf/slog_modem_cali.conf:vendor/etc/slog_modem_cali.conf 
$(BOARDDIR)/log_conf/slog_modem_factory.conf:vendor/etc/slog_modem_factory.conf 
$(BOARDDIR)/log_conf/slog_modem_autotest.conf:vendor/etc/slog_modem_autotest.conf 
$(BOARDDIR)/log_conf/mlogservice_$(TARGET_BUILD_VARIANT).conf:vendor/etc/mlogservice.conf 
$(BOARDDIR)/features/otpdata/otpgoldendata.txt:system/etc/otpdata/otpgoldendata.txt 
$(BOARDDIR)/features/otpdata/input_parameters_values.txt:system/etc/otpdata/input_parameters_values.txt 
$(BOARDDIR)/features/otpdata/obj_disc.txt:system/etc/otpdata/obj_disc.txt 
$(BOARDDIR)/features/otpdata/sell_aft_cali.txt:system/etc/otpdata/sell_aft_cali.txt 
$(BOARDDIR)/features/otpdata/sale_after_input_parameters_values.txt:system/etc/otpdata/sale_after_input_parameters_values.txt 
$(BOARDDIR)/features/otpdata/spw_input_parameters_values.txt:system/etc/otpdata/spw_input_parameters_values.txt 
$(BOARDDIR)/features/otpdata/oz1_input_parameters_values.txt:system/etc/otpdata/oz1_input_parameters_values.txt 
$(BOARDDIR)/features/otpdata/oz2_input_parameters_values.txt:system/etc/otpdata/oz2_input_parameters_values.txt \
device/sprd/sharkl5Pro/ums512_1h10/Yingyongbao.apk:$(PRODUCT_OUT)/etc/app/Yingyongbao.apk

```

编译报错：  
build/make/core/Makefile:29: error: Prebuilt apk found in PRODUCT_COPY_FILES: device/sprd/sharkl5Pro/ums512_1h10/Yingyongbao.apk, use BUILD_PREBUILT instead!.  
ckati failed with: exit status 1  
问题原因：原因是系统编译文件对拷贝 apk 文件做了限制检查，不允许拷贝 apk 文件。  
解决方法：修改 build\core\Makefile, 注释掉 check-product-copy-files 的定义

```
#define check-product-copy-files
#$(if $(filter-out $(TARGET_COPY_OUT_SYSTEM_OTHER)/%,$(2)), \
$(if $(filter %.apk, $(2)),$(error \
Prebuilt apk found in PRODUCT_COPY_FILES: $(1), use BUILD_PREBUILT instead!)))
#endef
第二步：
添加系统服务静默安装应用宝
frameworks/base/services/java/com/android/server/SystemServer.java
traceBeginAndSlog("StartBootPhaseDeviceSpecificServicesReady");
mSystemServiceManager.startBootPhase(SystemService.PHASE_DEVICE_SPECIFIC_SERVICES_READY);
traceEnd();
// add install apk code start ---------------------加载时机不要太靠前,需要等pms systemReady后加载
traceBeginAndSlog("AppInstallService");
mSystemServiceManager.startService(AppInstallService.class);
traceEnd();
// add code end
自定义安装app系统服务
frameworks / base/services/install/java/com/android/server/install/AppInstallService.java
package com.android.server.install;
import android.content.Context;
import com.android.server.SystemService;
import android.content.pm.PackageInfo;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.io.FileOutputStream;
import java.io.FileNotFoundException;
import java.io.BufferedReader;
import java.util.List;
import android.content.pm.IPackageManager;
import android.app.AppGlobals;
import android.os.RemoteException;
import android.os.SystemProperties;
import android.content.pm.PackageInstaller;
import android.app.PendingIntent;
import android.content.Intent;
import android.util.Log;
public class AppInstallService extends SystemService {
private Context mContext;
private String LOG_TAG = "AppInstallService";
private AppInstallUtils mAppInstall;
public AppInstallService(Context context) {
super(context);
mContext = context ;
mAppInstall = new AppInstallUtils(context);
}
@Override
public void onStart() {
    try {
        mAppInstall.installApp("/product/etc/app/Yingyongbao.apk");
    } catch (Exception e) {
        e.printStackTrace();
    }
}

}

```

自定义静默安装工具类

```
package com.android.server.install;
import android.app.PendingIntent;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageInfo;
import android.content.pm.PackageInstaller;
import android.content.pm.PackageManager;
import android.util.Log;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
public class AppInstallUtils {
private static final String TAG = "AppInstallUtils";
private Context mContext;
private int mSessionId = -1;
private PackageInstaller.SessionCallback mSessionCallback;
public AppInstallUtils(Context context) {
mContext = context;
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

public void uninstall(String packageName) {

    Intent broadcastIntent = new Intent(mContext, InstallResultReceiver.class);
    PendingIntent pendingIntent = PendingIntent.getBroadcast(mContext, 1,
            broadcastIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    PackageInstaller packageInstaller = mContext.getPackageManager().getPackageInstaller();
    packageInstaller.uninstall(packageName, pendingIntent.getIntentSender());
}

}

```

静默安装 app 接收结果  
frameworks / base/services/install/java/com/android/server/install/InstallResultReceiver.java

```
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageInstaller;
public class InstallResultReceiver extends BroadcastReceiver {
@Override
public void onReceive(Context context, Intent intent) {
if (intent != null) {
final int status = intent.getIntExtra(PackageInstaller.EXTRA_STATUS,
PackageInstaller.STATUS_FAILURE);
if (status == PackageInstaller.STATUS_SUCCESS) {
// success
} else {
//Log.e(TAG, intent.getStringExtra(PackageInstaller.EXTRA_STATUS_MESSAGE));
}
}
}
}

```