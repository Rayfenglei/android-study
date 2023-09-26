> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130493763)

1. 概述
-----

在 11.0 的系统 rom 开发过程中，在进行[以太网](https://so.csdn.net/so/search?q=%E4%BB%A5%E5%A4%AA%E7%BD%91&spm=1001.2101.3001.7020)产品开发的过程中，有功能要求设置默认静态 ip 地址的功能，不使用动态 ip,  
方便 ip 地址管理所以就需要熟悉以太网的 ip 设置流程，然后设置对应的 ip 地址就可以了

2. 以太网设置默认[静态 ip](https://so.csdn.net/so/search?q=%E9%9D%99%E6%80%81ip&spm=1001.2101.3001.7020) 地址的相关代码
-------------------------------------------------------------------------------------------------------

```
     frameworks\opt\net\ethernet\java\com\android\server\ethernet\EthernetServiceImpl.java
      frameworks\opt\net\ethernet\java\com\android\server\ethernet\EthernetService.java
      frameworks\opt\net\ethernet\java\com\android\server\ethernet\EthernetConfigStore.java
```

3. 以太网设置默认静态 ip 地址的功能分析和实现  
3.1 EthernetService.java 以太网服务的功能分析
----------------------------------------------------------------

```
    public final class EthernetService extends SystemService {
     
        private static final String TAG = "EthernetService";
        final EthernetServiceImpl mImpl;
     
        public EthernetService(Context context) {
            super(context);
            mImpl = new EthernetServiceImpl(context);
        }
     
        @Override
        public void onStart() {
            Log.i(TAG, "Registering service " + Context.ETHERNET_SERVICE);
            publishBinderService(Context.ETHERNET_SERVICE, mImpl);
        }
     
        @Override
        public void onBootPhase(int phase) {
            if (phase == SystemService.PHASE_SYSTEM_SERVICES_READY) {
                mImpl.start();
            }
        }
    }
```

在上述 EthernetService.java 中的代码中，可以看出在 EthernetService 以太网系统服务会在 systemserver 进程启动的时候 启动这个以太网服务  
而核心实现是在 EthernetServiceImpl 实现的通过 start 启动相关服务功能，接下来分析下 EthernetServiceImpl 的相关服务方法

3.2 EthernetServiceImpl 的相关方法分析
-------------------------------

```
      public class EthernetServiceImpl extends IEthernetManager.Stub {
...
     
        public EthernetServiceImpl(Context context) {
            mContext = context;
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
     
        @Override
        public String[] getAvailableInterfaces() throws RemoteException {
            return mTracker.getInterfaces(checkUseRestrictedNetworksPermission());
        }
     
        /**
         * Get Ethernet configuration
         * @return the Ethernet Configuration, contained in {@link IpConfiguration}.
         */
        @Override
        public IpConfiguration getConfiguration(String iface) {
            enforceAccessPermission();
     
            if (mTracker.isRestrictedInterface(iface)) {
                enforceUseRestrictedNetworksPermission();
            }
     
            return new IpConfiguration(mTracker.getIpConfiguration(iface));
        }
    ...
    }
```

在 EthernetServiceImpl.java 中的上述源码中，可以看出，它作为 EthernetService 的具体实现类, 主要实现以太网服务的相关功能，所以  
可以在这里设置静态 ip 的功能，增加设置静态 ip 功能  
设置如下:

```
    +import java.net.InetAddress;
    +import java.net.Inet4Address;
    +import android.net.LinkAddress;
    +import android.net.NetworkUtils;
    +import android.net.StaticIpConfiguration;
     
    // add core start
        private final EthernetConfigStore mEthernetConfigStore;
        private final StaticIpConfiguration mStaicIpConfig;
        private IpConfiguration mIpConfiguration;
    // add core end
     
        public EthernetServiceImpl(Context context) {
            mContext = context;
    // add core start
            mEthernetConfigStore = new EthernetConfigStore();
            mStaicIpConfig = new StaticIpConfiguration();
            String ipaddress = "192.168.1.125";
            int networkmask = 24;
            String gateway = "192.168.1.1";
            String dns1 = "10.10.10.1";
            String dns2 = "8.8.8.8";
     
            Inet4Address inetAddr = getIpAddress(ipaddress);
            int prefixLength = networkmask;
            InetAddress gatewayAddr =getIpAddress(gateway); 
            InetAddress dnsAddr = getIpAddress(dns1);
                    
            mStaicIpConfig.ipAddress = new LinkAddress(inetAddr, prefixLength);
            mStaicIpConfig.gateway=gatewayAddr;
            mStaicIpConfig.dnsServers.add(dnsAddr);
            mStaicIpConfig.dnsServers.add(getIpAddress(dns2));
            mIpConfiguration = new IpConfiguration(IpAssignment.STATIC, ProxySettings.NONE,mStaicIpConfig,null);
            mEthernetConfigStore.writeIpAndProxyConfigurations(mIpConfiguration);
    // add core end
        }
    // 增加获取Inet4Address方法
       private Inet4Address getIpAddress(String text) {
            try {
                return (Inet4Address) NetworkUtils.numericToInetAddress(text);
            } catch (Exception e) {
                return null;
            }
       }
```

而 10.0 以后在 EthernetConfigStore 的有些 api 有些调整 没有 writeIpAndProxyConfigurations 了所以需要新增 writeIpAndProxyConfigurations 方法

3.3 EthernetConfigStore 相关 api 的修改
----------------------------------

通过注释发现此类 EthernetConfigStore.java，它是提供了一个 API 来存储和管理以太网[网络配置](https://so.csdn.net/so/search?q=%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE&spm=1001.2101.3001.7020)

```
     /**
     * This class provides an API to store and manage Ethernet network configuration.
     */
    public class EthernetConfigStore {
        private static final String ipConfigFile = Environment.getDataDirectory() +
                "/misc/ethernet/ipconfig.txt";
     
        private IpConfigStore mStore = new IpConfigStore();
        private ArrayMap<String, IpConfiguration> mIpConfigurations;
        private IpConfiguration mIpConfigurationForDefaultInterface;
        private final Object mSync = new Object();
     
        public EthernetConfigStore() {
            mIpConfigurations = new ArrayMap<>(0);
        }
     //add core start
       增加writeIpAndProxyConfigurations设置静态ip方法
        public void writeIpAndProxyConfigurations(IpConfiguration config) {
              ArrayMap<IpConfiguration> networks = new ArrayMap<IpConfiguration>();
              networks.put(0, config);
              mStore.writeIpConfigurations(ipConfigFile, networks);
        }
     //add core end
```