> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131233887)

1. 前言
-----

  
  在 11.0 的系统开发中，在定制 [recovery](https://so.csdn.net/so/search?q=recovery&spm=1001.2101.3001.7020) 模块的时候，由于产品开发需要要求禁用 recovery 的相关功能，比如在通过 adb 命令的  
[adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) reboot recovery 的方式进入 recovery 也需要实现禁用，所以就需要了解相关进入 recovery 流程来禁用该功能

2. 禁用 adb reboot recovery 命令实现正常重启功能的核心类
----------------------------------------

```
system\core\adb\daemon\services.cpp

```

3. 禁用 adb reboot recovery 命令实现正常重启功能的核心功能分析和实现
----------------------------------------------

  
 在 11.0 的产品中，在通过 adb reboot recovery 进入 recovery 模式后正常可以进行 recovery 的相关操作，而  
adb 是 pc 端工具，adbd 是服务端，运行在手机 adbd 读取 socket 解析由 adb 传过来的命令串，解析相关的  
命令执行相关功能，所以在 pc 端输入 adb 相关命令 就会在 system\core\adb 模块解析相关命令  
所以说在 services.cpp 中来作为服务端来执行相关功能

```
asocket* daemon_service_to_socket(std::string_view name) {
    if (name == "jdwp") {
        return create_jdwp_service_socket();
    } else if (name == "track-jdwp") {
        return create_jdwp_tracker_service_socket();
    } else if (android::base::ConsumePrefix(&name, "sink:")) {
        uint64_t byte_count = 0;
        if (!ParseUint(&byte_count, name)) {
            return nullptr;
        }
        return new SinkSocket(byte_count);
    } else if (android::base::ConsumePrefix(&name, "source:")) {
        uint64_t byte_count = 0;
        if (!ParseUint(&byte_count, name)) {
            return nullptr;
        }
        return new SourceSocket(byte_count);
    }
 
    return nullptr;
}
 
unique_fd daemon_service_to_fd(std::string_view name, atransport* transport) {
    ADB_LOG(Service) << "transport " << transport->serial_name() << " opening service " << name;
 
#if defined(__ANDROID__) && !defined(__ANDROID_RECOVERY__)
    if (name.starts_with("abb:") || name.starts_with("abb_exec:")) {
        return execute_abb_command(name);
    }
#endif
 
#if defined(__ANDROID__)
    if (name.starts_with("framebuffer:")) {
        return create_service_thread("fb", framebuffer_service);
    } else if (android::base::ConsumePrefix(&name, "remount:")) {
        std::string cmd = "/system/bin/remount ";
        cmd += name;
        return StartSubprocess(cmd, nullptr, SubprocessType::kRaw, SubprocessProtocol::kNone);
    } else if (android::base::ConsumePrefix(&name, "reboot:")) {
        return reboot_device(std::string(name));
    } else if (name.starts_with("root:")) {
        return create_service_thread("root", restart_root_service);
    } else if (name.starts_with("unroot:")) {
        return create_service_thread("unroot", restart_unroot_service);
    } else if (android::base::ConsumePrefix(&name, "backup:")) {
        std::string cmd = "/system/bin/bu backup ";
        cmd += name;
        return StartSubprocess(cmd, nullptr, SubprocessType::kRaw, SubprocessProtocol::kNone);
    } else if (name.starts_with("restore:")) {
        return StartSubprocess("/system/bin/bu restore", nullptr, SubprocessType::kRaw,
                               SubprocessProtocol::kNone);
    } else if (name.starts_with("disable-verity:")) {
        return StartSubprocess("/system/bin/disable-verity", nullptr, SubprocessType::kRaw,
                               SubprocessProtocol::kNone);
    } else if (name.starts_with("enable-verity:")) {
        return StartSubprocess("/system/bin/enable-verity", nullptr, SubprocessType::kRaw,
                               SubprocessProtocol::kNone);
    } else if (android::base::ConsumePrefix(&name, "tcpip:")) {
        std::string str(name);
 
        int port;
        if (sscanf(str.c_str(), "%d", &port) != 1) {
            return unique_fd{};
        }
        return create_service_thread("tcp",
                                     std::bind(restart_tcp_service, std::placeholders::_1, port));
    } else if (name.starts_with("usb:")) {
        return create_service_thread("usb", restart_usb_service);
    }
#endif
 
    if (android::base::ConsumePrefix(&name, "dev:")) {
        return unique_fd{unix_open(name, O_RDWR | O_CLOEXEC)};
    } else if (android::base::ConsumePrefix(&name, "jdwp:")) {
        pid_t pid;
        if (!ParseUint(&pid, name)) {
            return unique_fd{};
        }
        return create_jdwp_connection_fd(pid);
    } else if (android::base::ConsumePrefix(&name, "shell")) {
        return ShellService(name, transport);
    } else if (android::base::ConsumePrefix(&name, "exec:")) {
        return StartSubprocess(std::string(name), nullptr, SubprocessType::kRaw,
                               SubprocessProtocol::kNone);
    } else if (name.starts_with("sync:")) {
        return create_service_thread("sync", file_sync_service);
    } else if (android::base::ConsumePrefix(&name, "reverse:")) {
        return reverse_service(name, transport);
    } else if (name == "reconnect") {
        return create_service_thread(
                "reconnect", std::bind(reconnect_service, std::placeholders::_1, transport));
    } else if (name == "spin") {
        return create_service_thread("spin", spin_service);
    }
 
    return unique_fd{};
}
```

在 services.cpp 中的上述相关源码中分析得知，在 daemon_service_to_fd(std::string_view name, atransport* transport)  
中负责解析 adb reboot adb remount 等相关命令，所以在  
else if (android::base::ConsumePrefix(&name, "reboot:")) {  
        return reboot_device(std::string(name));  
    }  
这段代码中表示需要在 reboot_service 方法中执行 reboot 的相关功能，接下来  
分析下 reboot_service 的相关源码

```
[[maybe_unused]] static unique_fd reboot_device(const std::string& name) {
#if defined(__ANDROID_RECOVERY__)
    if (!__android_log_is_debuggable()) {
        auto reboot_service = [name](unique_fd fd) {
            std::string reboot_string = android::base::StringPrintf("reboot,%s", name.c_str());
            if (!android::base::SetProperty(ANDROID_RB_PROPERTY, reboot_string)) {
                WriteFdFmt(fd.get(), "reboot (%s) failed\n", reboot_string.c_str());
                return;
            }
            while (true) pause();
        };
        return create_service_thread("reboot", reboot_service);
    }
#endif
    // Fall through
    std::string cmd = "/system/bin/reboot ";
 
// core modify start
     if (strcmp("recovery", name.c_str()) != 0){
        cmd += name;
    }
// core modify end
 
    return StartSubprocess(cmd, nullptr, SubprocessType::kRaw, SubprocessProtocol::kNone);
}
```

在 reboot_service 的相关源码分析得知，在 adb reboot 的相关参数 arg 就是相关的具体操作，  
比如 recovery 等，所以可以判断 name.c_str() 是否等于 recovery, 然后如果等于 recovery，就不调用 cmd += name;  
这样就负责重启就可以了，然后就实现相关的功能