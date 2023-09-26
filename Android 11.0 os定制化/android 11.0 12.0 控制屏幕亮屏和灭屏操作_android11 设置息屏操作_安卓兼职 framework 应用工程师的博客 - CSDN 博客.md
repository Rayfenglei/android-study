> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124955206)

在 11.0 12.0 的产品开发中， 需要提供亮屏和灭屏的接口在 8.0 以后系统对于屏幕亮灭屏做了限制，直接调用亮屏和灭屏的方法就调不到了，  
接下来就来看 PowerManage.java 类 这个是一个电源管理的服务类

PowerManager 的几个实用方法

boolean PowerManager::isScreenOn ()

判断屏幕是否亮着（不管是暗的 dimed 还是正常亮度），在 API20 被弃用，推荐 boolean PowerManager::isInteractive ()

void PowerManager::goToSleep(long time)

time 是时间戳，一般是 System.currentTimeMillis()+timeDelay。强制系统立刻休眠，需要 Manifest 中添加权限 "android.permission.DEVICE_POWER"。按下电源键锁屏时调用的就是这个方法。

void PowerManager::wakeUp(long time)

与上面对应。参数含义，所需权限与上同。按下电源键解锁屏幕时调用的就是这个方法。

void PowerManager::reboot()

重启手机，reason 是要传给 [linux 内核](https://so.csdn.net/so/search?q=linux%E5%86%85%E6%A0%B8&spm=1001.2101.3001.7020)的参数，比如 “recovery” 重启进 recovery 模式，“fastboot”重启进 fastboot 模式。需要权限 "android.permission.REBOOT"。

通过上面的方法可以看到还是可以亮屏和灭屏的 但是现在方法被隐藏了 直接调用调不到了，  
但是通过 powerManager 反射还是可以实现亮灭屏操作的 goToSleep 实现灭屏 通过 wakeup 实现亮屏

灭屏

```
/**
* @see
*/
public void turnOffScreen(){
//WindowManagerGlobal.getWindowManagerService().lockNow(null /* options */);
PowerManager powerManager= (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
try {
powerManager.getClass().getMethod("goToSleep", new Class[]{long.class}).invoke(powerManager, SystemClock.uptimeMillis());
} catch (Exception e) {
e.printStackTrace();
}
}

```

亮屏:

```
/**
* @see
*/
public void turnOnScreen(){
try {
PowerManager powerManager= (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
try {
powerManager.getClass().getMethod("wakeUp", new Class[]{long.class}).invoke(powerManager, SystemClock.uptimeMillis());
} catch (Exception e) {
e.printStackTrace();
}
} catch (Exception e) {
e.printStackTrace();
}
}

```