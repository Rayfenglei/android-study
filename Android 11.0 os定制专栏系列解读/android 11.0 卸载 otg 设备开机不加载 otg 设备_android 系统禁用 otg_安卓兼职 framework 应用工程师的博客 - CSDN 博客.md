> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889308)

1. 概述
-----

在 11.0 定制化开发过程中, 客户有功能需求，通过系统属性值控制是否加载挂载 [otg](https://so.csdn.net/so/search?q=otg&spm=1001.2101.3001.7020) 设备，当设置为卸载模式时，要求不能挂载 otg 设备，开机也不能挂载 otg 设备

2. 卸载 otg 设备开机不加载 otg 设备的核心代码
-----------------------------

```
frameworks/base/services/core/java/com/android/server/StorageManagerService.java

```

3. 卸载 otg 设备开机不加载 otg 设备的功能分析和实现
--------------------------------

实现思路:  
1.StorageManager 根据路径来卸载 otg 设备  
2. 开机时在 StorageManagerService 中 不挂载 otg 设备

3.1 分析 StorageManagerService.java 挂载 otg 相关方法
---------------------------------------------

```
class StorageManagerService extends IStorageManager.Stub
         implements Watchdog.Monitor, ScreenObserver {
private void start() {
          connectStoraged();
          connectVold();
      }
private final IVoldListener mListener = new IVoldListener.Stub() {
          @Override
          public void onDiskCreated(String diskId, int flags) {
              synchronized (mLock) {
                  final String value = SystemProperties.get(StorageManager.PROP_ADOPTABLE);
                  switch (value) {
                      case "force_on":
                          flags |= DiskInfo.FLAG_ADOPTABLE;
                          break;
                      case "force_off":
                          flags &= ~DiskInfo.FLAG_ADOPTABLE;
                          break;
                  }
                  mDisks.put(diskId, new DiskInfo(diskId, flags));
              }
          }
  
          @Override
          public void onDiskScanned(String diskId) {
              synchronized (mLock) {
                  final DiskInfo disk = mDisks.get(diskId);
                  if (disk != null) {
                      onDiskScannedLocked(disk);
                  }
              }
          }
  
          @Override
          public void onDiskMetadataChanged(String diskId, long sizeBytes, String label,
                  String sysPath) {
              synchronized (mLock) {
                  final DiskInfo disk = mDisks.get(diskId);
                  if (disk != null) {
                      disk.size = sizeBytes;
                      disk.label = label;
                      disk.sysPath = sysPath;
                  }
              }
          }
  
          @Override
          public void onDiskDestroyed(String diskId) {
              synchronized (mLock) {
                  final DiskInfo disk = mDisks.remove(diskId);
                  if (disk != null) {
                      mCallbacks.notifyDiskDestroyed(disk);
                  }
              }
          }
  
          @Override
          public void onVolumeCreated(String volId, int type, String diskId, String partGuid,
                  int userId) {
              synchronized (mLock) {
                  final DiskInfo disk = mDisks.get(diskId);
                  final VolumeInfo vol = new VolumeInfo(volId, type, disk, partGuid);
                  vol.mountUserId = userId;
                  mVolumes.put(volId, vol);
                  onVolumeCreatedLocked(vol);
              }
          }
  
          @Override
          public void onVolumeStateChanged(String volId, int state) {
              synchronized (mLock) {
                  final VolumeInfo vol = mVolumes.get(volId);
                  if (vol != null) {
                      final int oldState = vol.state;
                      final int newState = state;
                      vol.state = newState;
                      final SomeArgs args = SomeArgs.obtain();
                      args.arg1 = vol;
                      args.arg2 = oldState;
                      args.arg3 = newState;
                      mHandler.obtainMessage(H_VOLUME_STATE_CHANGED, args).sendToTarget();
                      onVolumeStateChangedLocked(vol, oldState, newState);
                  }
              }
          }
  
          @Override
          public void onVolumeMetadataChanged(String volId, String fsType, String fsUuid,
                  String fsLabel) {
              synchronized (mLock) {
                  final VolumeInfo vol = mVolumes.get(volId);
                  if (vol != null) {
                      vol.fsType = fsType;
                      vol.fsUuid = fsUuid;
                      vol.fsLabel = fsLabel;
                  }
              }
          }
  
          @Override
          public void onVolumePathChanged(String volId, String path) {
              synchronized (mLock) {
                  final VolumeInfo vol = mVolumes.get(volId);
                  if (vol != null) {
                      vol.path = path;
                  }
              }
          }
  
          @Override
          public void onVolumeInternalPathChanged(String volId, String internalPath) {
              synchronized (mLock) {
                  final VolumeInfo vol = mVolumes.get(volId);
                  if (vol != null) {
                      vol.internalPath = internalPath;
                  }
              }
          }
  
          @Override
          public void onVolumeDestroyed(String volId) {
              VolumeInfo vol = null;
              synchronized (mLock) {
                  vol = mVolumes.remove(volId);
              }
  
              if (vol != null) {
                  mStorageSessionController.onVolumeRemove(vol);
                  try {
                      if (vol.type == VolumeInfo.TYPE_PRIVATE) {
                          mInstaller.onPrivateVolumeRemoved(vol.getFsUuid());
                      }
                  } catch (Installer.InstallerException e) {
                      Slog.i(TAG, "Failed when private volume unmounted " + vol, e);
                  }
              }
          }
      };

```

在 StorageManagerService.java 服务启动的过程中会调用 start() 而在 connectVold(); 会注册监听 IListener  
这个 vold 的挂载 otg 通讯 最后调用 isMountDisallowed(VolumeInfo vol) 来判断是否挂载  
实现步骤：  
开机状态下 根据挂载模式来判断是否需要挂载

```
--- a/frameworks/base/services/core/java/com/android/server/StorageManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/StorageManagerService.java
@@ -126,7 +126,7 @@ import android.util.Xml;
 /* SPRD: add some log for debug @{ */
 import android.util.Log;
 /* @} */
-
+import android.provider.Settings;
 import com.android.internal.annotations.GuardedBy;
 import com.android.internal.app.IAppOpsCallback;
 import com.android.internal.app.IAppOpsService;
@@ -1527,6 +1527,17 @@ class StorageManagerService extends StorageManagerServiceEx
      * Decide if volume is mountable per device policies.
      */
     private boolean isMountDisallowed(VolumeInfo vol) {
+               boolean agpsEnabled = (Settings.Global.getInt(mContext.getContentResolver(),
+               "roco_disable_sdcard", 0) == 1);
+               boolean otg=(Settings.Global.getInt(mContext.getContentResolver(),"roco_disable_otg", 0) == 1);
+               DiskInfo disk=vol.disk;
+               Log.d("StorageManager","isMountDisallowed:"+agpsEnabled+",disk:"+(vol.disk != null && vol.disk.isUsb())+",otg:"+otg);
+                File file = vol.getPath();
+           if(file!=null){

+               if(agpsEnabled && !"/storage/emulated/0".equals(file.getAbsolutePath()) && !(vol.disk != null && vol.disk.isUsb())){
+                       return true;   
+               }
+               if(otg && !"/storage/emulated/0".equals(file.getAbsolutePath()) && (vol.disk != null && vol.disk.isUsb())){
+                       return true;   
+               }
+           }
+               if(vol.getPath()==null){
+                       if(agpsEnabled && vol.disk != null){
+                               return true;
+                       }
+                       if(otg&&(vol.disk != null&&vol.disk.isUsb())){
+                               return true;
                        }
                }
         UserManager userManager = mContext.getSystemService(UserManager.class);
 
         boolean isUsbRestricted = false;

```

在 isMountDisallowed() 中，判断是否需要阻止挂载

在系统自定义服务中，实现 otg 挂载卸载方法的调用  
实现挂载，卸载功能

```
public void setSysUSBOtgDisabled(boolean disabled){
        Settings.Global.putInt(mContext.getContentResolver(),
                "enable_otg", disabled ? 1 : 0);
        StorageManager storageManager = (StorageManager) mContext.getSystemService(Context.STORAGE_SERVICE);
            StorageVolume[] volumeList = storageManager.getVolumeList();
            for (StorageVolume vol : volumeList) {
                Log.d("StorageManager", "path:" + vol.getPath());
				if (disabled) {
					if (!"/storage/emulated/0".equals(vol.getPath()) && "mounted".equals(vol.getState())) {
						storageManager.unmount(vol.getId());
					}
                }else{
                    if (!"/storage/emulated/0".equals(vol.getPath()) && "unmounted".equals(vol.getState())) {
						storageManager.mount(vol.getId());
					}
                }
            }
}

```