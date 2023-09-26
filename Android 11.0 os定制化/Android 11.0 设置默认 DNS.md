> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130273874)

1. 前言
-----

  
 在 11.0 的系统 rom 产品定制化开发中，由于是 wifi 产品的定制，需要对 wifi 功能要求比较高，所以在 wifi 需求方面要求设置默认的 dns 功能，这就需要分析网络通讯  
流程，然后在联网之后，设置默认的 dns, 来实现功能要求

2. 设置默认 DNS 的核心类
----------------

```
frameworks\base\core\java\android\net\IConnectivityManager.aidl
frameworks\base\core\java\android\net\ConnectivityManager.java
frameworks/base/services/core/java/com/android/server/ConnectivityService.java
```

3. 设置默认 DNS 的核心功能分析和实现
----------------------

  
 在 android 系统中，在 ConnectivityService 的主要功能就是通过 wifi，mobile data，Tethering，[VPN](https://so.csdn.net/so/search?q=VPN&spm=1001.2101.3001.7020) 等方式来获取路由配置信息。  
无论通过哪种方式，获取到[路由配置](https://so.csdn.net/so/search?q=%E8%B7%AF%E7%94%B1%E9%85%8D%E7%BD%AE&spm=1001.2101.3001.7020)信息后，需要交给 ConnectivityService 来处理，ConnectivityService 通过 ping 网络来检查网络的有效性，  
进而影响到各个数据业务方式的网络通讯速度值，ConnectivityService 通过这些网络通讯速度值来决定以哪个数据业务方式连接网络。决定好数据业务方式后，  
把这些路由配置信息设置到网络物理设备中。这样我们的手机就可以正常上网了

在系统 ConnectivityService 服务中，是通过 ConnectivityManager 管理类来提供接口调用 ConnectivityService 中相关的接口，来实现网络通讯的  
所以需要在 IConnectivityManager.aidl 增加设置默认 dns 的接口 来实现功能

3.1 IConnectivityManager.aidl 中增加设置默认 dns 的接口
---------------------------------------------

```
/**
 * Copyright (c) 2008, The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
 
 
package android.net;
 
import android.app.PendingIntent;
import android.net.ConnectionInfo;
import android.net.LinkProperties;
import android.net.ITetheringEventCallback;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.net.NetworkInfo;
import android.net.NetworkMisc;
import android.net.NetworkQuotaInfo;
import android.net.NetworkRequest;
import android.net.NetworkState;
import android.net.ISocketKeepaliveCallback;
import android.net.ProxyInfo;
import android.os.Bundle;
import android.os.IBinder;
import android.os.Messenger;
import android.os.ParcelFileDescriptor;
import android.os.ResultReceiver;
 
import com.android.internal.net.LegacyVpnInfo;
import com.android.internal.net.VpnConfig;
import com.android.internal.net.VpnInfo;
import com.android.internal.net.VpnProfile;
 
/**
 * Interface that answers queries about, and allows changing, the
 * state of network connectivity.
 */
/** {@hide} */
interface IConnectivityManager
{
    Network getActiveNetwork();
    Network getActiveNetworkForUid(int uid, boolean ignoreBlocked);
    @UnsupportedAppUsage
    NetworkInfo getActiveNetworkInfo();
    NetworkInfo getActiveNetworkInfoForUid(int uid, boolean ignoreBlocked);
    NetworkInfo getNetworkInfo(int networkType);
    NetworkInfo getNetworkInfoForUid(in Network network, int uid, boolean ignoreBlocked);
    @UnsupportedAppUsage
    NetworkInfo[] getAllNetworkInfo();
    Network getNetworkForType(int networkType);
    Network[] getAllNetworks();
    NetworkCapabilities[] getDefaultNetworkCapabilitiesForUser(int userId);
 
    /* SPRD: add for Bug1120111, AndroidQ begin*/
    boolean isWifiInUse();
    /* SPRD: add for Bug1120111, AndroidQ end*/
 
+ void setDefaultDns(in List<String> dnsList);
}
```

在 IConnectivityManager.aidl 中 增加 void setDefaultDns(in List dnsList); 这个设置默认的 dns 接口来实现通过  
[binder](https://so.csdn.net/so/search?q=binder&spm=1001.2101.3001.7020) 通讯的方式来提供给 ConnectivityService.java 来设置默认 dns 功能

3.2 ConnectivityManager.java 中增加设置默认 dns 的接口
--------------------------------------------

```
@SystemService(Context.CONNECTIVITY_SERVICE)
public class ConnectivityManager {
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 130143562)
    private final IConnectivityManager mService;
//add core start
    public void setDefaultDns(List<String> dnsList) {
        try {
            return mService.setDefaultDns(dnsList);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
//add core end
```

在原生系统中 ConnectivityManager 类用于查询网络状态，并且也能被动监听网络状态的变化。  
主要是提供管理网络服务的管理类，通过这个管理类来调用 ConnectivityService.java 的相关接口  
所以可以通过 setDefaultDns(List dnsList) 来增加设置默认 dns 的接口，来调用 ConnectivityService.java 来具体  
实现设置默认 dns 的功能

3.3 ConnectivityService.java 中增加设置默认 dns 的相关功能
----------------------------------------------

```
  public void handleUpdateLinkProperties(NetworkAgentInfo nai, LinkProperties newLp) {
        ensureRunningOnConnectivityServiceThread();
 
        if (getNetworkAgentInfoForNetId(nai.network.netId) != nai) {
            // Ignore updates for disconnected networks
            return;
        }
        // newLp is already a defensive copy.
        newLp.ensureDirectlyConnectedRoutes();
        if (VDBG || DDBG) {
            log("Update of LinkProperties for " + nai.name() +
                    "; created=" + nai.created +
                    "; everConnected=" + nai.everConnected);
        }
        updateLinkProperties(nai, newLp, new LinkProperties(nai.linkProperties));
    }
    private void updateLinkProperties(NetworkAgentInfo networkAgent, LinkProperties newLp,
            LinkProperties oldLp) {
        int netId = networkAgent.network.netId;
 
        // The NetworkAgentInfo does not know whether clatd is running on its network or not, or
        // whether there is a NAT64 prefix. Before we do anything else, make sure its LinkProperties
        // are accurate.
        networkAgent.clatd.fixupLinkProperties(oldLp, newLp);
 
        updateInterfaces(newLp, oldLp, netId, networkAgent.networkCapabilities);
 
        // update filtering rules, need to happen after the interface update so netd knows about the
        // new interface (the interface name -> index map becomes initialized)
        updateVpnFiltering(newLp, oldLp, networkAgent);
 
        updateMtu(newLp, oldLp);
        // TODO - figure out what to do for clat
//        for (LinkProperties lp : newLp.getStackedLinks()) {
//            updateMtu(lp, null);
//        }
        if (isDefaultNetwork(networkAgent)) {
            updateTcpBufferSizes(newLp.getTcpBufferSizes());
        }
 
        updateRoutes(newLp, oldLp, netId);
 
        boolean hasInvalidDnsServer = false;
        for (InetAddress dnsServer : newLp.getDnsServers()) {
            if ((dnsServer instanceof Inet4Address) && dnsServer.equals(Inet4Address.ALL) ) {
                hasInvalidDnsServer = true;
                break;
                //newLp.removeDnsServer(dnsServer);
            }
        }
        if (hasInvalidDnsServer) {
            log("updateLinkProperties: one dns server is 255.255.255.255, remove it ");
            newLp.removeDnsServer(Inet4Address.ALL);
        }
 
        updateDnses(newLp, oldLp, netId);
        // Make sure LinkProperties represents the latest private DNS status.
        // This does not need to be done before updateDnses because the
        // LinkProperties are not the source of the private DNS configuration.
        // updateDnses will fetch the private DNS configuration from DnsManager.
        mDnsManager.updatePrivateDnsStatus(netId, newLp);
....
}
 
    private void updateDnses(LinkProperties newLp, LinkProperties oldLp, int netId) {
          if (oldLp != null && newLp.isIdenticalDnses(oldLp)) {
              return;  // no updating necessary
          }
  
          final NetworkAgentInfo defaultNai = getDefaultNetwork();
          final boolean isDefaultNetwork = (defaultNai != null && defaultNai.network.netId == netId);
  
          if (DBG) {
              final Collection<InetAddress> dnses = newLp.getDnsServers();
              log("Setting DNS servers for network " + netId + " to " + dnses);
          }
          try {
              mDnsManager.noteDnsServersForNetwork(netId, newLp);
              // TODO: netd should listen on [::1]:53 and proxy queries to the current
              // default network, and we should just set net.dns1 to ::1, not least
              // because applications attempting to use net.dns resolvers will bypass
              // the privacy protections of things like DNS-over-TLS.
              if (isDefaultNetwork)                     mDnsManager.setDefaultDnsSystemProperties(newLp.getDnsServers());
              mDnsManager.flushVmDnsCache();
          } catch (Exception e) {
              loge("Exception in setDnsConfigurationForNetwork: " + e);
          }
    }
```

在 ConnectivityService.java 中的上述的方法中，通过源码分析，可以得知 handleUpdateLinkProperties(NetworkAgentInfo nai, LinkProperties newLp)  
收到更新网络链路的时候，调用具体的方法 updateLinkProperties(NetworkAgentInfo networkAgent, LinkProperties newLp,  
            LinkProperties oldLp) 来更新相关网络数据配置，而在 updateDnses(LinkProperties newLp, LinkProperties oldLp, int netId) 中  
负责更新网络的 dns, 而具体更新 dns 的就是 mDnsManager.setDnsConfigurationForNetwork(netId, newLp, isDefaultNetwork);  
这个方法，  
所以需要增加设置默认 dns 的方法为:

```
//add core start
 @Override
    public void setDefaultDns(List<String> dnsList) {
       try{
            NetworkAgentInfo defaultNai = getDefaultNetwork();
			int netId = defaultNai.network.netId;
            LinkProperties newLp = defaultNai.linkProperties;
            List<InetAddress> dnsServers = new ArrayList<InetAddress>();
            for(String str:dnsList){
               dnsServers.add(InetAddress.getByName(str));
            }
            newLp.setDnsServers(dnsServers);
            mDnsManager.noteDnsServersForNetwork(netId, newLp);
            mDnsManager.setDefaultDnsSystemProperties(newLp.getDnsServers());
            mDnsManager.flushVmDnsCache();
        }catch(Exception e){
	e.printStackTrace();
        }
    }
//add core end
```