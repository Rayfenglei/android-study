> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893890)

### 1. 概述

在 11.0 进行定制化开发，会根据需要去掉 [recovery](https://so.csdn.net/so/search?q=recovery&spm=1001.2101.3001.7020) 模式的一些选项 就是在 device.cpp 去掉一些选项就可以了

### 2. 去掉 recovery 模式 UI 页面的选项核心代码

```
bootable/recovery/recovery_ui/device.cpp
bootable/recovery/recovery_main.cpp

```

### 3. 去掉 recovery 模式 UI 页面的选项的核心功能分析和实现

在 device.cpp 中 g_menu_actions 就是 recovery 选项集合，对应的事件处理

```
diff --git a/bootable/recovery/recovery_ui/device.cpp b/bootable/recovery/recovery_ui/device.cpp
old mode 100644
new mode 100755
index e7ae1a3e19..567610a55d
--- a/bootable/recovery/recovery_ui/device.cpp
+++ b/bootable/recovery/recovery_ui/device.cpp
@@ -27,17 +27,7 @@
 
 static std::vector<std::pair<std::string, Device::BuiltinAction>> g_menu_actions{
   { "Reboot system now", Device::REBOOT },
-  { "Reboot to bootloader", Device::REBOOT_BOOTLOADER },
-  { "Enter fastboot", Device::ENTER_FASTBOOT },
-  { "Apply update from ADB", Device::APPLY_ADB_SIDELOAD },
-  { "Apply update from SD card", Device::APPLY_SDCARD },
   { "Wipe data/factory reset", Device::WIPE_DATA },
-  { "Wipe cache partition", Device::WIPE_CACHE },
-  { "Mount /system", Device::MOUNT_SYSTEM },
-  { "View recovery logs", Device::VIEW_RECOVERY_LOGS },
-  { "Run graphics test", Device::RUN_GRAPHICS_TEST },
-  { "Run locale test", Device::RUN_LOCALE_TEST },
-  { "Enter rescue", Device::ENTER_RESCUE },
   { "Power off", Device::SHUTDOWN },
 };

```

g_menu_actions 对应 UI 的每个选项和相应的按键 action 事件，在这里去掉不需要的选项就可以实现某些功能了但是在实际项目开发测试中发现 user 模式情况下会存在 bug 问题，最后跟代码发现是在 recovery_main.cpp 中的，main() 方法中在 user 模式下会移除到一些选项造成 recovery 失败一直  
卡住出现黑屏问题

### 3.2 recovery_main.cpp 中关于调用 recovery 选项的分析

在 main 方法中也会通过调用 device->RemoveMenuItemForAction(来实现移除某些 recovery 选项  
来达到去掉选项的目的

```
    int main(int argc, char** argv) {
      // We don't have logcat yet under recovery; so we'll print error on screen and log to stdout
      // (which is redirected to recovery.log) as we used to do.
      android::base::InitLogging(argv, &UiLogger);
     
      // Take last pmsg contents and rewrite it to the current pmsg session.
      static constexpr const char filter[] = "recovery/";
      // Do we need to rotate?
      bool do_rotate = false;
     
      __android_log_pmsg_file_read(LOG_ID_SYSTEM, ANDROID_LOG_INFO, filter, logbasename, &do_rotate);
      // Take action to refresh pmsg contents
      __android_log_pmsg_file_read(LOG_ID_SYSTEM, ANDROID_LOG_INFO, filter, logrotate, &do_rotate);
     
      time_t start = time(nullptr);
     
      // redirect_stdio should be called only in non-sideload mode. Otherwise we may have two logger
      // instances with different timestamps.
      redirect_stdio(Paths::Get().temporary_log_file().c_str());
     
      load_volume_table();
      has_cache = volume_for_mount_point(CACHE_ROOT) != nullptr;
     
      std::vector<std::string> args = get_args(argc, argv);
      auto args_to_parse = StringVectorToNullTerminatedArray(args);
     
      static constexpr struct option OPTIONS[] = {
        { "fastboot", no_argument, nullptr, 0 },
        { "locale", required_argument, nullptr, 0 },
        { "show_text", no_argument, nullptr, 't' },
        { nullptr, 0, nullptr, 0 },
      };
     
      bool show_text = false;
      bool fastboot = false;
      std::string locale;
     
      int arg;
      int option_index;
      while ((arg = getopt_long(args_to_parse.size() - 1, args_to_parse.data(), "", OPTIONS,
                                &option_index)) != -1) {
        switch (arg) {
          case 't':
            show_text = true;
            break;
          case 0: {
            std::string option = OPTIONS[option_index].name;
            if (option == "locale") {
              locale = optarg;
            } else if (option == "fastboot" &&
                       android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
              fastboot = true;
            }
            break;
          }
        }
      }
      optind = 1;
     
      if (locale.empty()) {
        if (has_cache) {
          locale = load_locale_from_cache();
        }
     
        if (locale.empty()) {
          static constexpr const char* DEFAULT_LOCALE = "en-US";
          locale = DEFAULT_LOCALE;
        }
      }
     
      static constexpr const char* kDefaultLibRecoveryUIExt = "librecovery_ui_ext.so";
      // Intentionally not calling dlclose(3) to avoid potential gotchas (e.g. `make_device` may have
      // handed out pointers to code or static [or thread-local] data and doesn't collect them all back
      // in on dlclose).
      void* librecovery_ui_ext = dlopen(kDefaultLibRecoveryUIExt, RTLD_NOW);
     
      using MakeDeviceType = decltype(&make_device);
      MakeDeviceType make_device_func = nullptr;
      if (librecovery_ui_ext == nullptr) {
        printf("Failed to dlopen %s: %s\n", kDefaultLibRecoveryUIExt, dlerror());
      } else {
        reinterpret_cast<void*&>(make_device_func) = dlsym(librecovery_ui_ext, "make_device");
        if (make_device_func == nullptr) {
          printf("Failed to dlsym make_device: %s\n", dlerror());
        }
      }
     
      Device* device;
      if (make_device_func == nullptr) {
        printf("Falling back to the default make_device() instead\n");
        device = make_device();
      } else {
        printf("Loading make_device from %s\n", kDefaultLibRecoveryUIExt);
        device = (*make_device_func)();
      }
     
      if (android::base::GetBoolProperty("ro.boot.quiescent", false)) {
        printf("Quiescent recovery mode.\n");
        device->ResetUI(new StubRecoveryUI());
      } else {
        if (!device->GetUI()->Init(locale)) {
          printf("Failed to initialize UI; using stub UI instead.\n");
          device->ResetUI(new StubRecoveryUI());
        }
      }
      ui = device->GetUI();
     
      /*if (!has_cache) {
        device->RemoveMenuItemForAction(Device::WIPE_CACHE);
      }
      if (!android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
        device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT);
      }
      if (!is_ro_debuggable()) {
        device->RemoveMenuItemForAction(Device::ENTER_RESCUE);
      }*/
      ui->SetBackground(RecoveryUI::NONE);
      if (show_text) ui->ShowText(true);
     
      LOG(INFO) << "Starting recovery (pid " << getpid() << ") on " << ctime(&start);
      LOG(INFO) << "locale is [" << locale << "]";
     
      sehandle = selinux_android_file_context_handle();
      selinux_android_set_sehandle(sehandle);
      if (!sehandle) {
        ui->Print("Warning: No file_contexts\n");
      }

从main()方法看出会在
 

    if (!has_cache) {
        device->RemoveMenuItemForAction(Device::WIPE_CACHE);
      }
     
      if (!android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
        device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT);
      }
     
      if (!is_ro_debuggable()) {
        device->RemoveMenuItemForAction(Device::ENTER_RESCUE);
      }

```

经过上述部分的修改，会移除一部分 recovery 选项 如果像上面那样去掉选项 走到这里会崩溃到出现黑屏卡死的现象所以需要注释这些关于移除选项 device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT) 等的方法

第二种去掉 recovery 方法就是在 recovery_main.cpp 中的 main（）方法中调用

device->RemoveMenuItemForAction(Device::ENTER_RESCUE);

来去掉所需要去掉的 recovery 选项

具体修改如下:

```
int main(int argc, char** argv) {
  // We don't have logcat yet under recovery; so we'll print error on screen and log to stdout
  // (which is redirected to recovery.log) as we used to do.
  android::base::InitLogging(argv, &UiLogger);
 
  // Take last pmsg contents and rewrite it to the current pmsg session.
  static constexpr const char filter[] = "recovery/";
  // Do we need to rotate?
  bool do_rotate = false;
 
  __android_log_pmsg_file_read(LOG_ID_SYSTEM, ANDROID_LOG_INFO, filter, logbasename, &do_rotate);
  // Take action to refresh pmsg contents
  __android_log_pmsg_file_read(LOG_ID_SYSTEM, ANDROID_LOG_INFO, filter, logrotate, &do_rotate);
 
  time_t start = time(nullptr);
 
  // redirect_stdio should be called only in non-sideload mode. Otherwise we may have two logger
  // instances with different timestamps.
  redirect_stdio(Paths::Get().temporary_log_file().c_str());
 
  load_volume_table();
  has_cache = volume_for_mount_point(CACHE_ROOT) != nullptr;
 
  std::vector<std::string> args = get_args(argc, argv);
  auto args_to_parse = StringVectorToNullTerminatedArray(args);
 
  static constexpr struct option OPTIONS[] = {
    { "fastboot", no_argument, nullptr, 0 },
    { "locale", required_argument, nullptr, 0 },
    { "show_text", no_argument, nullptr, 't' },
    { nullptr, 0, nullptr, 0 },
  };
 
  bool show_text = false;
  bool fastboot = false;
  std::string locale;
 
  int arg;
  int option_index;
  while ((arg = getopt_long(args_to_parse.size() - 1, args_to_parse.data(), "", OPTIONS,
                            &option_index)) != -1) {
    switch (arg) {
      case 't':
        show_text = true;
        break;
      case 0: {
        std::string option = OPTIONS[option_index].name;
        if (option == "locale") {
          locale = optarg;
        } else if (option == "fastboot" &&
                   android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
          fastboot = true;
        }
        break;
      }
    }
  }
  optind = 1;
 
  if (locale.empty()) {
    if (has_cache) {
      locale = load_locale_from_cache();
    }
 
    if (locale.empty()) {
      static constexpr const char* DEFAULT_LOCALE = "en-US";
      locale = DEFAULT_LOCALE;
    }
  }
 
  static constexpr const char* kDefaultLibRecoveryUIExt = "librecovery_ui_ext.so";
  // Intentionally not calling dlclose(3) to avoid potential gotchas (e.g. `make_device` may have
  // handed out pointers to code or static [or thread-local] data and doesn't collect them all back
  // in on dlclose).
  void* librecovery_ui_ext = dlopen(kDefaultLibRecoveryUIExt, RTLD_NOW);
 
  using MakeDeviceType = decltype(&make_device);
  MakeDeviceType make_device_func = nullptr;
  if (librecovery_ui_ext == nullptr) {
    printf("Failed to dlopen %s: %s\n", kDefaultLibRecoveryUIExt, dlerror());
  } else {
    reinterpret_cast<void*&>(make_device_func) = dlsym(librecovery_ui_ext, "make_device");
    if (make_device_func == nullptr) {
      printf("Failed to dlsym make_device: %s\n", dlerror());
    }
  }
 
  Device* device;
  if (make_device_func == nullptr) {
    printf("Falling back to the default make_device() instead\n");
    device = make_device();
  } else {
    printf("Loading make_device from %s\n", kDefaultLibRecoveryUIExt);
    device = (*make_device_func)();
  }
 
  if (android::base::GetBoolProperty("ro.boot.quiescent", false)) {
    printf("Quiescent recovery mode.\n");
    device->ResetUI(new StubRecoveryUI());
  } else {
    if (!device->GetUI()->Init(locale)) {
      printf("Failed to initialize UI; using stub UI instead.\n");
      device->ResetUI(new StubRecoveryUI());
    }
  }
  ui = device->GetUI();
 
  /*if (!has_cache) {
    device->RemoveMenuItemForAction(Device::WIPE_CACHE);
  }
  if (!android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
    device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT);
  }
  if (!is_ro_debuggable()) {
    device->RemoveMenuItemForAction(Device::ENTER_RESCUE);
  }*/
  device->RemoveMenuItemForAction(Device::REBOOT_BOOTLOADER);
  device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT);
  device->RemoveMenuItemForAction(Device::APPLY_ADB_SIDELOAD);
  device->RemoveMenuItemForAction(Device::APPLY_SDCARD);
  device->RemoveMenuItemForAction(Device::WIPE_DATA);
  device->RemoveMenuItemForAction(Device::WIPE_CACHE);
  device->RemoveMenuItemForAction(Device::MOUNT_SYSTEM);
  device->RemoveMenuItemForAction(Device::VIEW_RECOVERY_LOGS);
  device->RemoveMenuItemForAction(Device::RUN_GRAPHICS_TEST);
  device->RemoveMenuItemForAction(Device::RUN_LOCALE_TEST);
  device->RemoveMenuItemForAction(Device::ENTER_RESCUE);
  ui->SetBackground(RecoveryUI::NONE);
  if (show_text) ui->ShowText(true);
 
  LOG(INFO) << "Starting recovery (pid " << getpid() << ") on " << ctime(&start);
  LOG(INFO) << "locale is [" << locale << "]";
 
  sehandle = selinux_android_file_context_handle();
  selinux_android_set_sehandle(sehandle);
  if (!sehandle) {
    ui->Print("Warning: No file_contexts\n");
  }

```

这样就不会出现问题完整解决问题