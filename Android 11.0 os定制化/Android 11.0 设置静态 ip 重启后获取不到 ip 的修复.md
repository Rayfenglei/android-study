> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130565238)

1. 概述
-----

在 11.0 的系统 rom 定制化开发中，在一些[产品开发](https://so.csdn.net/so/search?q=%E4%BA%A7%E5%93%81%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，在对于设置静态 ip 以后可以正常使用，  
但是遇到一个新问题 就是开机以后，获取不到 [ip](https://so.csdn.net/so/search?q=ip&spm=1001.2101.3001.7020)，地址，这就有点不正常了，  
获取不到 ip 就自然连不上网了，所以要分析问题所在解决问题

2. 设置[静态](https://so.csdn.net/so/search?q=%E9%9D%99%E6%80%81&spm=1001.2101.3001.7020) ip 重启后获取不到 ip 的修复的核心代码
------------------------------------------------------------------------------------------------------------

```
      frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetTracker.java
      frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetServiceImpl.java
```

3. 设置静态 ip 重启后获取不到 ip 的修复的功能分析和实现功能
-----------------------------------

[Android](https://so.csdn.net/so/search?q=Android&spm=1001.2101.3001.7020) 以太网服务启动源码分析中，我们知道在 NetworkManagementService 的 connectNativeService 中注册了  
NetdUnsolicitedListener。所以下面将会进入 framework 层的处理。这里我们最后在分析一下  
getNetdUnsolicitedEventListenerMap 的注册和获取。

在 EthernetManager 中，当 EthernetTracker 中有底层状态信息改变的时候，会回调到 IEthernetServiceListener 的  
onAvailabilityChanged 中。通过 Handler 发送 MSG_AVAILABILITY_CHANGED 消息，并且循环回调每个  
Listener 的 onAvailabilityChanged 中，而 Listener 是通过应用调用 addListener 注册的。

EthernetServiceImpl 这个类主要是通过 NetworkManagementService 监听网络状态的变化  
Android 以太网框架情景分析之 EthernetNetworkFactory 深入分析可知 EthernetNetworkFactory 类是 Ethernet 的核心管理类，  
几乎包含了 Ethernet 所有的网络管理操作，这其中就包括了通过 NetworkManagementService 监听网络状态的变化  
在 EthernetNetworkFactory 的 start 方法中，而在这个方法中 EthernetNetworkFactory 会以观察者的身份注册到  
NetworkManagementService 中从而监听网络状态的变化

  
3.1 EthernetServiceImpl.java 关于以太网服务的管理实现相关方法
------------------------------------------------

```
      public class EthernetServiceImpl extends IEthernetManager.Stub {
        private static final String TAG = "EthernetServiceImpl";
     
        private final Context mContext;
        private final AtomicBoolean mStarted = new AtomicBoolean(false);
     
        private Handler mHandler;
        private EthernetTracker mTracker;
     
        public EthernetServiceImpl(Context context) {
            mContext = context;
     
        }
     
        private void enforceAccessPermission() {
            mContext.enforceCallingOrSelfPermission(
                    android.Manifest.permission.ACCESS_NETWORK_STATE,
                    "EthernetService");
        }
     
        private void enforceConnectivityInternalPermission() {
            mContext.enforceCallingOrSelfPermission(
                    android.Manifest.permission.CONNECTIVITY_INTERNAL,
                    "ConnectivityService");
        }
     
        private void enforceUseRestrictedNetworksPermission() {
            mContext.enforceCallingOrSelfPermission(
                    android.Manifest.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS,
                    "ConnectivityService");
        }
     
        private boolean checkUseRestrictedNetworksPermission() {
            return mContext.checkCallingOrSelfPermission(
                    android.Manifest.permission.CONNECTIVITY_USE_RESTRICTED_NETWORKS)
                    == PackageManager.PERMISSION_GRANTED;
        }
     
        public void start() {
            Log.i(TAG, "Starting Ethernet service");
     
            HandlerThread handlerThread = new HandlerThread("EthernetServiceThread");
            handlerThread.start();
            mHandler = new Handler(handlerThread.getLooper());
     
            mTracker = new EthernetTracker(mContext, mHandler);
            mTracker.start();
     
            mStarted.set(true);
        }
    ..
    }
```

在 EthernetServiceImpl.java 的上述方法中，可以看出在以太网服务中 EthernetService 是在 systemserver 进程启动后启动的关于以太网管理的服务，而正在实现启动服务的在 EthernetServiceImpl 中实现的，  
而在 start() 中启动以太网功能在这里面调用了 EthernetTracker 的 start 来启动相关功能 接下来看 EthernetTracker 的相关代码

3.2 EthernetTracker 的相关代码分析
---------------------------

```
   final class EthernetTracker {
        EthernetTracker(Context context, Handler handler) {
            mHandler = handler;
     
            // The services we use.
            IBinder b = ServiceManager.getService(Context.NETWORKMANAGEMENT_SERVICE);
            mNMService = INetworkManagementService.Stub.asInterface(b);
     
            // Interface match regex.
            mIfaceMatch = context.getResources().getString(
                    com.android.internal.R.string.config_ethernet_iface_regex);
     
            // Read default Ethernet interface configuration from resources
            final String[] interfaceConfigs = context.getResources().getStringArray(
                    com.android.internal.R.array.config_ethernet_interfaces);
            for (String strConfig : interfaceConfigs) {
                parseEthernetConfig(strConfig);
            }
     
            mConfigStore = new EthernetConfigStore();
     
            NetworkCapabilities nc = createNetworkCapabilities(true /* clear default capabilities */);
            mFactory = new EthernetNetworkFactory(handler, context, nc);
            mFactory.register();
        }
     
        void start() {
            mConfigStore.read();
     
            // Default interface is just the first one we want to track.
            mIpConfigForDefaultInterface = mConfigStore.getIpConfigurationForDefaultInterface();
            final ArrayMap<String, IpConfiguration> configs = mConfigStore.getIpConfigurations();
            for (int i = 0; i < configs.size(); i++) {
                mIpConfigurations.put(configs.keyAt(i), configs.valueAt(i));
            }
     
            try {
                mNMService.registerObserver(new InterfaceObserver());
            } catch (RemoteException e) {
                Log.e(TAG, "Could not register InterfaceObserver " + e);
            }
     
            mHandler.post(this::trackAvailableInterfaces);
        }
    private void maybeTrackInterface(String iface) {
            if (DBG) Log.i(TAG, "maybeTrackInterface " + iface);
            // If we don't already track this interface, and if this interface matches
            // our regex, start tracking it.
            if (!iface.matches(mIfaceMatch) || mFactory.hasInterface(iface)) {
                return;
            }
            if (mIpConfigForDefaultInterface != null) {
                updateIpConfiguration(iface, mIpConfigForDefaultInterface);
                mIpConfigForDefaultInterface = null;
            }
            addInterface(iface);
        }
        private void trackAvailableInterfaces() {
            try {
                final String[] ifaces = mNMService.listInterfaces();
                for (String iface : ifaces) {
                    maybeTrackInterface(iface);
                }
            } catch (RemoteException | IllegalStateException e) {
                Log.e(TAG, "Could not get list of interfaces " + e);
            }
        }
    }
```

在 EthernetTracker.java 中的上述方法可以看出，在通过分析问题发现动态 ip 开机可以正常获取到 ip, 而在设置静态  
ip 开机后没有获取到 ip 时，使用 ifconfig 查看，发现网卡设备连接正常。  
而在这时拔插网线，或者使用 ifconfig eth0 down + ifconfig eth0 up 命令来开关一次设备后，就能正常获取到 ip  
在 start 方法中通过 trackAvailableInterfaces() 打开默认以太网网络，所以可以在这里实现用代码插拔网线来实现功能  
首选修改 maybeTrackInterface 为 boolean 类型作为标志是否开启网络，在 trackAvailableInterfaces()  
中遍历网络时用 updateInterfaceState(mIfaceTmp, true); 执行插拔网线动作

具体修改如下:

```
    diff --git a/frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetTracker.java b/frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetTracker.java
    index 308e328..919edc6 100644
    --- a/frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetTracker.java
    +++ b/frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetTracker.java
     
    -    private void maybeTrackInterface(String iface) {
    +    private boolean maybeTrackInterface(String iface) {
             if (DBG) Log.i(TAG, "maybeTrackInterface " + iface);
             // If we don't already track this interface, and if this interface matches
             // our regex, start tracking it.
             if (!iface.matches(mIfaceMatch) || mFactory.hasInterface(iface)) {
    -            return;
    +            return false;
             }
     
            if (mIpConfigForDefaultInterface != null) {
                updateIpConfiguration(iface, mIpConfigForDefaultInterface);
                mIpConfigForDefaultInterface = null;
            }
     
             addInterface(iface);
    +        return true;
         }
     
         private void trackAvailableInterfaces() {
             try {
                 final String[] ifaces = mNMService.listInterfaces();
                 for (String iface : ifaces) {
    -                maybeTrackInterface(iface);
    +               if (maybeTrackInterface(iface)) {
    +                    String mIfaceTmp = iface;
    +                    new Thread(new Runnable() {
    +                        public void run() {
    +                            try {
    +                                Thread.sleep(3000);
    +                            } catch (Exception e) {
    +                                  e.printStackTrace();
    +                            }
    +        if(isEthernetInterfaceActive()){
    +                                IpConfiguration Ipconfig = getIpConfiguration(mIfaceTmp);
    +                                if(config != null && IpAssignment.STATIC == Ipconfig.getIpAssignment())
    +                                {
    +                                    updateInterfaceState(mIfaceTmp, false);
    +                                    updateInterfaceState(mIfaceTmp, true);
    +                                }
    +                            }else{
    +                                 updateInterfaceState(mIfaceTmp, false);
    +                             }
    +                        }
    +                    }).start();
    +                    break;
    +                }
                 }
             } catch (RemoteException | IllegalStateException e) {
                 Log.e(TAG, "Could not get list of interfaces " + e);
```

通过在 EthernetTracker.java 中的上述修改发现，在经过这些修改以后就可以实现在以太网产品  
重新后重新获取到 ip 地址了，这样就实现了功能要求