> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131691008)

1. 前言
-----

在 11.0 的系统 rom 定制化开发中，对于限制系统上网功能中，在 system 中 netd 网络这块的[产品开发](https://so.csdn.net/so/search?q=%E4%BA%A7%E5%93%81%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，会要求设置屏蔽 ip 地址之内的功能，  
[liunx](https://so.csdn.net/so/search?q=liunx&spm=1001.2101.3001.7020) 中 iptables 命令也是比较重要的，接下来就来在 IOemNetd 这块实现删除创建子链的相关功能

2.  系统限制上网系统之 iptables 用 IOemNetd 实现删除创建子链功能的实现的核心类
---------------------------------------------------

```
   system\netd\server\binder\com\android\internal\net\IOemNetd.aidl
    system\netd\server\OemNetdListener.cpp
    system\netd\server\OemNetdListener.h
```

3. 系统限制上网系统之 iptables 用 IOemNetd 实现删除创建子链功能的实现的核心功能分析和实现
--------------------------------------------------------

在 android 原生系统中，iptables 是在网络过滤包模块非常重要的, Iptabels 是与 Linux 内核集成的包过滤防火墙系统，linux 和 android 都会包含 Iptables 的功能。  
如果 Linux 系统连接到因特网或 LAN、服务器或连接 LAN 和因特网的代理服务器， 则 Iptables 有利于在 Linux 系统上更好地控制 IP 信息包过滤和防火墙配置。  
netfilter/iptables 的另一个重要优点是，它使用户可以完全控制防火墙配置和信息包过滤。您可以定制自己的规则来满足您的特定需求

iptables 常用命令如下:  
命令                                        说明

-L  --list          <链名>  查看 iptables 规则列表  
-A  --append        <链名>  在规则列表的最后增加 1 条规则  
-I  --insert        <链名>  在指定的位置插入 1 条规则  
-D  --delete        <链名>  从规则列表中删除 1 条规则  
-F  --flush         <链名>  删除表中所有规则  
-X  --delete-chain  <链名>  删除自定义链

删除创建子链的命令如下:  
iptables -X 名称  
通过上面的相关 iptables 的关键删除创建的子链的代码分析，得知了如何使用 iptables 来删除创建的子链  
接下来就在 IOemNetd 中增加删除创建子链的接口来实现功能

3.1 IOemNetd.aidl 中增加删除创建子链的接口
------------------------------

```
    /** {@hide} */
      interface IOemNetd {
         /**
          * Returns true if the service is responding.
          */
          boolean isAlive();
      
         /**
          * Register oem unsolicited event listener
          *
          * @param listener oem unsolicited event listener to register
          */
          void registerOemUnsolicitedEventListener(IOemNetdUnsolicitedEventListener listener);
     +     void deleteChildChain();// add core
      }
```

在 system/netd 下的 IOemNetd.aidl 的接口中，同时也是通过 binder 通讯的方式，提供给  
framework 层调用 system/netd 的接口，来实现对系统网络的限制，通过上述的方法可以看出，在这里增加删除创建子链的接口，  
deleteChildChain(); 来实现功能，在这里 framework 可以通过 binder 通讯的方式，实现接口的调用

3.2 OemNetdListener.h 中增加删除创建子链的接口
----------------------------------

```
   namespace com {
      namespace android {
      namespace internal {
      namespace net {
      
      class OemNetdListener : public BnOemNetd {
        public:
          using OemUnsolListenerMap = std::map<const ::android::sp<IOemNetdUnsolicitedEventListener>,
                                               const ::android::sp<::android::IBinder::DeathRecipient>>;
      
          OemNetdListener() = default;
          ~OemNetdListener() = default;
          static ::android::sp<::android::IBinder> getListener();
      
          ::android::binder::Status isAlive(bool* alive) override;
          ::android::binder::Status registerOemUnsolicitedEventListener(
                  const ::android::sp<IOemNetdUnsolicitedEventListener>& listener) override;
     +      ::android::binder::Status deleteChildChain() override;
        private:
          std::mutex mMutex;
          std::mutex mOemUnsolicitedMutex;
      
          ::android::sp<::android::IBinder> mIBinder GUARDED_BY(mMutex);
          OemUnsolListenerMap mOemUnsolListenerMap GUARDED_BY(mOemUnsolicitedMutex);
      
          ::android::sp<::android::IBinder> getIBinder() EXCLUDES(mMutex);
      
          void registerOemUnsolicitedEventListenerInternal(
                  const ::android::sp<IOemNetdUnsolicitedEventListener>& listener)
                  EXCLUDES(mOemUnsolicitedMutex);
          void unregisterOemUnsolicitedEventListenerInternal(
                  const ::android::sp<IOemNetdUnsolicitedEventListener>& listener)
                  EXCLUDES(mOemUnsolicitedMutex);
      };
      
      }  // namespace net
      }  // namespace internal
      }  // namespace android
      }  // namespace com
      
      #endif  // NETD_SERVER_OEM_NETD_LISTENER_H
```

在 system/netd 下的 IOemNetd 的模块中，在 IOemNetd.aidl 的接口中增加对外调用 IOemNetd 的接口，  
而具体在 OemNetdListener.cpp 中具体实现 IOemNetd.aidl 的接口的功能的实现，接下来就需要在  
OemNetdListener.h 中增加::android::binder::Status deleteChildChain() override; 来定义  
接口，然后实现功能

3.3 OemNetdListener.cpp 中实现删除创建子链的接口
------------------------------------

```
    #define LOG_TAG "OemNetd"
      
      #include "OemNetdListener.h"
      
      namespace com {
      namespace android {
      namespace internal {
      namespace net {
      
      ::android::sp<::android::IBinder> OemNetdListener::getListener() {
          static OemNetdListener listener;
          return listener.getIBinder();
      }
      
      ::android::sp<::android::IBinder> OemNetdListener::getIBinder() {
          std::lock_guard lock(mMutex);
          if (mIBinder == nullptr) {
              mIBinder = ::android::IInterface::asBinder(this);
          }
          return mIBinder;
      }
      
      ::android::binder::Status OemNetdListener::isAlive(bool* alive) {
          *alive = true;
          return ::android::binder::Status::ok();
      }
      
      ::android::binder::Status OemNetdListener::registerOemUnsolicitedEventListener(
              const ::android::sp<IOemNetdUnsolicitedEventListener>& listener) {
          registerOemUnsolicitedEventListenerInternal(listener);
          listener->onRegistered();
          return ::android::binder::Status::ok();
      }
     
    //add core start
    ::android::binder::Status OemNetdListener::deleteChildChain() {
         ALOGD("OemNetdListener:deleteChildChain");
        int res = -1;
        IptablesTarget target = V4V6;
        std::string command = "*filter\n";
        ::android::base::StringAppendF(&command, "-F %s\n", "reject_dns_chain");
        ::android::base::StringAppendF(&command, "-X %s\n", "reject_dns_chain");
        ::android::base::StringAppendF(&command, "COMMIT\n");
        
        res = execIptablesRestore(target, command.c_str());
        ALOGD("OemNetdListener:deleteChildChain command=%s,res=%d",command.c_str(),res);
        if(res == 0) {
             return ::android::binder::Status::ok();
        } else {
            return ::android::binder::Status::fromServiceSpecificError(res,
                ::android::String8::format("deleteChildChain error: %d", res));
        }
    }
    //add core end
```

在 system/netd 下的 IOemNetd 的模块中，在 IOemNetd.aidl 的接口中增加对外调用 IOemNetd 的接口，  
而具体在 OemNetdListener.cpp 中具体实现 IOemNetd.aidl 的接口的功能的实现, 所以在 OemNetdListener.cpp 中  
增加::android::binder::Status OemNetdListener::deleteChildChain() 来具体实现删除创建子链的功能，  
所以 & command 拼接 iptables 的命令，然后调用 execIptablesRestore(target, command.c_str()); 来  
执行删除创建子链的功能，然后可以根据返回的 res 值来得知执行命令的结果

3.4 在创建的自定义服务中，来调用执行删除创建子链的功能
-----------------------------

```
    //add core start
    import android.os.INetworkManagementService;
    import com.android.internal.net.IOemNetd;
    import android.net.util.NetdService;
    import android.content.pm.ApplicationInfo;
    import java.net.InetAddress;
    import android.content.pm.PackageManager;
     
        private IOemNetd mIoemNetd;
            try {
                mIoemNetd = IOemNetd.Stub.asInterface(NetdService.getInstance().getOemNetd());
                //Log.d(TAG,"mIoemNetd:"+mIoemNetd);
            } catch (Exception e) {
                e.printStackTrace();
            }
     
    //add core end
```

在自定义服务中，通过 binder 通讯的方式，通过创建 IOemNetd mIoemNetd 的实例，然后在 framework 中的自定义服务中，  
在通过 binder 通讯的方式调用在 IOemNetd 中增加的创建子链的功能，接下来来实现这个功能

```
   //add core start
        private Runnable mThreadRunnable = new Runnable() {
            @Override
            public void run() {
                try {
        mIoemNetd.deleteChildChain();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        };
     
    //add core end
```

在自定义服务中，通过创建 IOemNetd mIoemNetd 的实例后，就可以调用 mIoemNetd.deleteChildChain();  
来实现 IOemNetd 中删除创建子链的功能