> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132864594)

1. 使用场景
-------

  在进行 11.0 的系统定制化开发中，在一些产品中由于一些开发的功能比较重要，防止技术点外泄在出货产品中，禁用  
adb pull 和 adb push 等命令 来获取系统 system 下的 jar 和 apk 等文件，所以需要禁用这些命令

2. 系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的分析
-------------------------------------------------

系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现中，在  
看了下系统 system 模块源码中的 adb 的代码，adb 的源码在 system/core/adb 下面,  
接下来就来分析下关于 adb 在 system 模块中的核心功能  
（1）adb 的本质，就是 [socket](https://so.csdn.net/so/search?q=socket&spm=1001.2101.3001.7020) 的通信，通过 secket 传送数据及文件，然后通过在设备中监听相关的命令来执行相关的功能  
（2）adb 传送是以每个固定格式的包发送的数据，通过在设备端接收相关的数据，执行相关的指令

ADB（[Android](https://so.csdn.net/so/search?q=Android&spm=1001.2101.3001.7020) Debug Bridge）驱动是用于在计算机和安卓设备之间建立连接和通信的驱动程序。ADB 驱动的主要作用是帮助开发人员和用户在计算机上执行一系列与安卓设备相关的调试、测试和管理操作, 通过 adb 我们可以在 Eclipse 中方便通过 DDMS 来调试 Android 程序，说白了就是 debug 工具。

3. 系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的代码
-------------------------------------------------

```
     system\core\adb\daemon\main.cpp
     system\core\adb\transport.cpp
     system\core\adb\daemon\services.cpp
     system\core\adb\daemon\file_sync_service.cpp
```

3.1 main.cpp 中相关 adb 终端的代码分析
----------------------------

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
在 adb/daemon 下的 main.cpp 主要就是执行 adb 传输过来的相关 adb 命令的，接下来  
分析下 adb/daemon 下的 main.cpp 的相关源码

```
   int main(int argc, char** argv) {
    #if defined(__BIONIC__)
        // Set M_DECAY_TIME so that our allocations aren't immediately purged on free.
        mallopt(M_DECAY_TIME, 1);
    #endif
     
        while (true) {
            static struct option opts[] = {
                    {"root_seclabel", required_argument, nullptr, 's'},
                    {"device_banner", required_argument, nullptr, 'b'},
                    {"version", no_argument, nullptr, 'v'},
                    {"logpostfsdata", no_argument, nullptr, 'l'},
            };
     
            int option_index = 0;
            int c = getopt_long(argc, argv, "", opts, &option_index);
            if (c == -1) {
                break;
            }
     
            switch (c) {
    #if defined(__ANDROID__)
                case 's':
                    root_seclabel = optarg;
                    break;
    #endif
                case 'b':
                    adb_device_banner = optarg;
                    break;
                case 'v':
                    printf("Android Debug Bridge Daemon version %d.%d.%d\n", ADB_VERSION_MAJOR,
                           ADB_VERSION_MINOR, ADB_SERVER_VERSION);
                    return 0;
                case 'l':
                    LOG(ERROR) << "post-fs-data triggered";
                    return 0;
                default:
                    // getopt already prints "adbd: invalid option -- %c" for us.
                    return 1;
            }
        }
     
        close_stdin();
     
        adb_trace_init(argv);
     
        D("Handling main()");
        return adbd_main(DEFAULT_ADB_PORT);
    }
```

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
在上述 main.cpp 中的源码中发现在  
在 main 函数 中主要调用了 adbd_main(DEFAULT_ADB_PORT)  
作为处理终端命令的相关功能 接下来看下 adbd_main(DEFAULT_ADB_PORT)  
的相关源码

```
    int adbd_main(int server_port) {
        umask(0);
     
        signal(SIGPIPE, SIG_IGN);
     
    #if defined(__BIONIC__)
        auto fdsan_level = android_fdsan_get_error_level();
        if (fdsan_level == ANDROID_FDSAN_ERROR_LEVEL_DISABLED) {
            android_fdsan_set_error_level(ANDROID_FDSAN_ERROR_LEVEL_WARN_ONCE);
        }
    #endif
     
        init_transport_registration();
     
        // We need to call this even if auth isn't enabled because the file
        // descriptor will always be open.
        adbd_cloexec_auth_socket();
     
    #if defined(ALLOW_ADBD_NO_AUTH)
        // If ro.adb.secure is unset, default to no authentication required.
        auth_required = android::base::GetBoolProperty("ro.adb.secure", false);
    #elif defined(__ANDROID__)
        if (is_device_unlocked()) {  // allows no authentication when the device is unlocked.
            auth_required = android::base::GetBoolProperty("ro.adb.secure", false);
        }
    #endif
     
        adbd_auth_init();
    ...
    }
```

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
在上述 main.cpp 中的源码中，分析发现  
在 adbd_main 方法中通过 init_transport_registration() 来  
注册 socket 传输功能监听, 当 pc 端和设备端通讯的时候，通过 socket 监听来传输相关的  
adb 命令，来实现 adb 通讯功能  
我们主要看一下 init_transport_registration 方法

3.2 transport.cpp 相关源码分析
------------------------

  
在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
通过上述的 main.cpp 中的源码分析，执行相关 adb 通讯的就是在 init_transport_registration(void)  
中负责执行的，接下来分析下 init_transport_registration(void) 的相关源码

```
    void init_transport_registration(void) {
        int s[2];
     
        if (adb_socketpair(s)) {
            PLOG(FATAL) << "cannot open transport registration socketpair";
        }
        D("socketpair: (%d,%d)", s[0], s[1]);
     
        transport_registration_send = s[0];
        transport_registration_recv = s[1];
     
        transport_registration_fde =
            fdevent_create(transport_registration_recv, transport_registration_func, nullptr);
        fdevent_set(transport_registration_fde, FDE_READ);
    }
```

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
在 transport.cpp 中的调用了 adb_socketpair(s), 开启 socket 通讯，当有数据的时候，后会调用 transport_registration_func 函数  
来进行 socket 通讯，而在 transport_registration_func 中先调用了 transport_read_action 来读取 transport_registration_recv 的数据，  
也就是之前 transport_registration_send 发来的 adb 节点的数据。然后  
通过 connection() 连接获得对传输的引用，并通过 SetReadCallback 回调 对读写数据进行处理，  
在主线程上排队（fdevent_run_on_main_thread）执行 handle_packet(packet, t) 对从驱动器读取到的命令处理 看  
 到在 OPEN 指令中（传输数据） 调用 create_local_service_socket 方法  
最终通过在 services.cpp 中的 daemon_service_to_fd 方法中的 create_service_thread("sync", file_sync_service)  
来最终由 file_sync_service.cpp 来处理 adb push 和 adb pull 传输文件功能

```
    unique_fd daemon_service_to_fd(std::string_view name, atransport* transport) {
    ....
        if (ConsumePrefix(&name, "dev:")) {
            return unique_fd{unix_open(name, O_RDWR | O_CLOEXEC)};
        } else if (ConsumePrefix(&name, "jdwp:")) {
            pid_t pid;
            if (!ParseUint(&pid, name)) {
                return unique_fd{};
            }
            return create_jdwp_connection_fd(pid);
        } else if (ConsumePrefix(&name, "shell")) {
            return ShellService(name, transport);
        } else if (ConsumePrefix(&name, "exec:")) {
            return StartSubprocess(std::string(name), nullptr, SubprocessType::kRaw,
                                   SubprocessProtocol::kNone);
        } else if (name.starts_with("sync:")) {
            return create_service_thread("sync", file_sync_service);
        } else if (ConsumePrefix(&name, "reverse:")) {
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

3.3 file_sync_service.cpp 中相关源码分析
---------------------------------

```
   static bool handle_sync_command(int fd, std::vector<char>& buffer) {
        D("sync: waiting for request");
     
        ATRACE_CALL();
        SyncRequest request;
        if (!ReadFdExactly(fd, &request, sizeof(request))) {
            SendSyncFail(fd, "command read failure");
            return false;
        }
        size_t path_length = request.path_length;
        if (path_length > 1024) {
            SendSyncFail(fd, "path too long");
            return false;
        }
        char name[1025];
        if (!ReadFdExactly(fd, name, path_length)) {
            SendSyncFail(fd, "filename read failure");
            return false;
        }
        name[path_length] = 0;
     
        std::string id_name = sync_id_to_name(request.id);
        std::string trace_name = StringPrintf("%s(%s)", id_name.c_str(), name);
        ATRACE_NAME(trace_name.c_str());
     
        D("sync: %s('%s')", id_name.c_str(), name);
        switch (request.id) {
            case ID_LSTAT_V1:
                if (!do_lstat_v1(fd, name)) return false;
                break;
            case ID_LSTAT_V2:
            case ID_STAT_V2:
                if (!do_stat_v2(fd, request.id, name)) return false;
                break;
            case ID_LIST:
                if (!do_list(fd, name)) return false;
                break;
            case ID_SEND:
                if (!do_send(fd, name, buffer)) return false;
                break;
            case ID_RECV:
                if (!do_recv(fd, name, buffer)) return false;
                break;
            case ID_QUIT:
                return false;
            default:
                SendSyncFail(fd, StringPrintf("unknown command %08x", request.id));
                return false;
        }
     
        return true;
    }
```

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
在 file_sync_service.cpp 中相关源码中的源码中，由 handle_sync_command(int fd, std::vector<char>& buffer)  
来处理 adb 命令，而在 case ID_SEND 就表示处理 adb push 命令 所以具体实现在 do_send(fd, name, buffer)  
而 在 case ID_RECV 就表示处理 adb pull 命令 所以具体实现在 do_recv(fd, name, buffer) 中

```
   + static const char EXIT_ADB_ENABLE[] = "persist.sys.adb.enable";//自定义属性
    static bool do_send(int s, const std::string& spec, std::vector<char>& buffer) {
    + char jvalue[256];
    + property_get(EXIT_ADB_ENABLE, jvalue, "0");
    + int jexitnow = atoi(jvalue);
    + if(jexitnow == 0) {
    + retrun false;
    + }
     
        // 'spec' is of the form "/some/path,0755". Break it up.
        size_t comma = spec.find_last_of(',');
        if (comma == std::string::npos) {
            SendSyncFail(s, "missing , in ID_SEND");
            return false;
        }
     
        std::string path = spec.substr(0, comma);
     
        errno = 0;
        mode_t mode = strtoul(spec.substr(comma + 1).c_str(), nullptr, 0);
        if (errno != 0) {
            SendSyncFail(s, "bad mode");
            return false;
        }
     
        // Don't delete files before copying if they are not "regular" or symlinks.
        struct stat st;
        bool do_unlink = (lstat(path.c_str(), &st) == -1) || S_ISREG(st.st_mode) ||
                         (S_ISLNK(st.st_mode) && !S_ISLNK(mode));
        if (do_unlink) {
            adb_unlink(path.c_str());
        }
    ....
    }
    static bool do_recv(int s, const char* path, std::vector<char>& buffer) {
    // add core start
    + char jvalue[256];
    + property_get(EXIT_ADB_ENABLE, jvalue, "0");
    + int jexitnow = atoi(jvalue);
    + if(jexitnow == 0) {
    + retrun false;
    + }
    //add core end
        __android_log_security_bswrite(SEC_TAG_ADB_RECV_FILE, path);
        unique_fd fd(adb_open(path, O_RDONLY | O_CLOEXEC));
        if (fd < 0) {
            SendSyncFailErrno(s, "open failed");
            return false;
        }
        if (posix_fadvise(fd.get(), 0, 0, POSIX_FADV_SEQUENTIAL | POSIX_FADV_NOREUSE) < 0) {
            D("[ Failed to fadvise: %d ]", errno);
        }
    ....
    }
```

在系统 system 模块开启禁用 adb push 和 adb pull 传输文件功能的实现功能中，  
通过上述在 file_sync_service.cpp 中的相关源码中，处理 adb pull 和 adb push 的 do_send(int s, const std::string& spec, std::vector<char>& buffer)  
和 do_recv(int s, const char* path, std::vector<char>& buffer) 添加判断系统属性 persist.sys.adb.enable  
是否为 0, 当为 0 时就代表禁用 adb push 和 pull 功能