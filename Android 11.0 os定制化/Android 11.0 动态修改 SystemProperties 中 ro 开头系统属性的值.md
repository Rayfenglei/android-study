> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127018864)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 动态修改 SystemProperties 中 ro 开头系统属性的值的核心代码](#t1)

[3. 动态修改 SystemProperties 中 ro 开头系统属性的值的功能分析和实现功能](#t2)

 [3.1 关于系统属性 SystemProperty 分析](#t3)

[3.2 property_service.cpp 关于系统属性的相关方法设置](#t4)

1. 概述
-----

  在 11.0 的产品开发中，对于定制功能的需求很多，有些机型要求可以修改系统属性值，对于系统本身在 10.0 以后为了系统安全性，不允许修改 ro 开头的 SystemProperties 的值，所以如果要求修改 ro 的相关系统属性，就得看在设置 SystemProperties 的相关值的要求，修改这部分要求就可以了

2. 动态修改 SystemProperties 中 ro 开头系统属性的值的核心代码
-------------------------------------------

```
  frameworks/base/core/java/android/os/SystemProperties.java
  /system/core/init/property_service.cpp
```

3. 动态修改 SystemProperties 中 ro 开头系统属性的值的功能分析和实现功能
------------------------------------------------

###   3.1 关于系统属性 SystemProperty 分析

     系统属性，肯定对整个系统全局共享。通常程序的执行以进程为单位各自相互独立，如何实现全局共享呢？属性系统是 android 的一个重要特性。它作为一个服务运行，管理系统配置和状态。所有这些配置和状态都是属性。SystemProperties.java 每个属性是一个键值对（key/value pair），其类型都是字符串  
   接下来看下 SystemProperties.java 相关方法

```
 
    private static native String native_get(String key);
    private static native String native_get(String key, String def);
    private static native int native_get_int(String key, int def);
    @UnsupportedAppUsage
    private static native long native_get_long(String key, long def);
    private static native boolean native_get_boolean(String key, boolean def);
    private static native void native_set(String key, String def);
 
    /**
     * Get the String value for the given {@code key}.
     *
     * @param key the key to lookup
     * @param def the default value in case the property is not set or empty
     * @return if the {@code key} isn't found, return {@code def} if it isn't null, or an empty
     * string otherwise
     * @hide
     */
    @NonNull
    @SystemApi
    @TestApi
    public static String get(@NonNull String key, @Nullable String def) {
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        return native_get(key, def);
    }
 
    /**
     * Get the value for the given {@code key}, and return as an integer.
     *
     * @param key the key to lookup
     * @param def a default value to return
     * @return the key parsed as an integer, or def if the key isn't found or
     *         cannot be parsed
     * @hide
     */
    @SystemApi
    public static int getInt(@NonNull String key, int def) {
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        return native_get_int(key, def);
    }
 
    /**
     * Get the value for the given {@code key}, and return as a long.
     *
     * @param key the key to lookup
     * @param def a default value to return
     * @return the key parsed as a long, or def if the key isn't found or
     *         cannot be parsed
     * @hide
     */
    @SystemApi
    public static long getLong(@NonNull String key, long def) {
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        return native_get_long(key, def);
    }
 
    @SystemApi
    @TestApi
    public static boolean getBoolean(@NonNull String key, boolean def) {
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        return native_get_boolean(key, def);
    }
 
    /**
     * Set the value for the given {@code key} to {@code val}.
     *
     * @throws IllegalArgumentException if the {@code val} exceeds 91 characters
     * @hide
     */
    @UnsupportedAppUsage
    public static void set(@NonNull String key, @Nullable String val) {
        if (val != null && !val.startsWith("ro.") && val.length() > PROP_VALUE_MAX) {
            throw new IllegalArgumentException("value of system property '" + key
                    + "' is longer than " + PROP_VALUE_MAX + " characters: " + val);
        }
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        native_set(key, val);
    }
```

通过上述 get() 和 set() 发现 最终是通过 jni 的方法设置相关的值  
最终通过查阅资料发现是在 property_service.cpp 中来设置系统相关属性的 接下来看下 property_service.cpp  
设置属性的相关代码

### 3.2 property_service.cpp 关于系统属性的相关方法设置

```
     static uint32_t PropertySet(const std::string& name, const std::string& value, std::string* error) {
      size_t valuelen = value.size();
  
      if (!IsLegalPropertyName(name)) {
          *error = "Illegal property name";
          return PROP_ERROR_INVALID_NAME;
      }
  
      if (auto result = IsLegalPropertyValue(name, value); !result.ok()) {
          *error = result.error().message();
          return PROP_ERROR_INVALID_VALUE;
      }
  
      prop_info* pi = (prop_info*) __system_property_find(name.c_str());
      if (pi != nullptr) {
          // ro.* properties are actually "write-once".
         // 判断是否ro开头的属性 这只为读的属性 不能修改值
          if (StartsWith(name, "ro.")) {
              *error = "Read-only property was already set";
              return PROP_ERROR_READ_ONLY_PROPERTY;
          }
  
          __system_property_update(pi, value.c_str(), valuelen);
      } else {
          int rc = __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
          if (rc < 0) {
              *error = "__system_property_add failed";
              return PROP_ERROR_SET_FAILED;
          }
      }
  
      // Don't write properties to disk until after we have read all default
      // properties to prevent them from being overwritten by default values.
      if (persistent_properties_loaded && StartsWith(name, "persist.")) {
          WritePersistentProperty(name, value);
      }
      // If init hasn't started its main loop, then it won't be handling property changed messages
      // anyway, so there's no need to try to send them.
      auto lock = std::lock_guard{accept_messages_lock};
      if (accept_messages) {
          PropertyChanged(name, value);
      }
      return PROP_SUCCESS;
  }
  // This returns one of the enum of PROP_SUCCESS or PROP_ERROR*.
  uint32_t HandlePropertySet(const std::string& name, const std::string& value,
                             const std::string& source_context, const ucred& cr,
                             SocketConnection* socket, std::string* error) {
      if (auto ret = CheckPermissions(name, value, source_context, cr, error); ret != PROP_SUCCESS) {
          return ret;
      }
  
      if (StartsWith(name, "ctl.")) {
          return SendControlMessage(name.c_str() + 4, value, cr.pid, socket, error);
      }
  
      // sys.powerctl is a special property that is used to make the device reboot.  We want to log
      // any process that sets this property to be able to accurately blame the cause of a shutdown.
      if (name == "sys.powerctl") {
          std::string cmdline_path = StringPrintf("proc/%d/cmdline", cr.pid);
          std::string process_cmdline;
          std::string process_log_string;
          if (ReadFileToString(cmdline_path, &process_cmdline)) {
              // Since cmdline is null deliminated, .c_str() conveniently gives us just the process
              // path.
              process_log_string = StringPrintf(" (%s)", process_cmdline.c_str());
          }
          LOG(INFO) << "Received sys.powerctl='" << value << "' from pid: " << cr.pid
                    << process_log_string;
          if (value == "reboot,userspace" && !is_userspace_reboot_supported().value_or(false)) {
              *error = "Userspace reboot is not supported by this device";
              return PROP_ERROR_INVALID_VALUE;
          }
      }
  
      // If a process other than init is writing a non-empty value, it means that process is
      // requesting that init performs a restorecon operation on the path specified by 'value'.
      // We use a thread to do this restorecon operation to prevent holding up init, as it may take
      // a long time to complete.
      if (name == kRestoreconProperty && cr.pid != 1 && !value.empty()) {
          static AsyncRestorecon async_restorecon;
          async_restorecon.TriggerRestorecon(value);
          return PROP_SUCCESS;
      }
  
      return PropertySet(name, value, error);
  }
 uint32_t InitPropertySet(const std::string& name, const std::string& value) {
      uint32_t result = 0;
      ucred cr = {.pid = 1, .uid = 0, .gid = 0};//获取cr参数
      std::string error;
      result = HandlePropertySet(name, value, kInitContext, cr, nullptr, &error);
      if (result != PROP_SUCCESS) {
          LOG(ERROR) << "Init cannot set '" << name << "' to '" << value << "': " << error;
      }
  
      return result;
  }
```

在进入 property_service 修改系统属性时，先调用 InitPropertySet(const std::string& name, const std::string& value)  
在通过 HandlePropertySet(name, value, kInitContext, cr, nullptr, &error) 来获取返回值  
在 HandlePropertySet(）中根据参数 name 的值分别调用相关的方法来设置值 ro 开头的值 最终是由  
PropertySet(name, value, error); 来设置值时会判断 ro 开头的值 只读不让修改 所以要修改就去了这个限制  
具体修改为:

```
      static uint32_t PropertySet(const std::string& name, const std::string& value, std::string* error) {
      size_t valuelen = value.size();
  
      if (!IsLegalPropertyName(name)) {
          *error = "Illegal property name";
          return PROP_ERROR_INVALID_NAME;
      }
  
      if (auto result = IsLegalPropertyValue(name, value); !result.ok()) {
          *error = result.error().message();
          return PROP_ERROR_INVALID_VALUE;
      }
  
      prop_info* pi = (prop_info*) __system_property_find(name.c_str());
      if (pi != nullptr) {
          // ro.* properties are actually "write-once".
         // 判断是否ro开头的属性 这只为读的属性 不能修改值
        // 修改开始 注释掉这部分代码
        /*  if (StartsWith(name, "ro.")) {
              *error = "Read-only property was already set";
              return PROP_ERROR_READ_ONLY_PROPERTY;
          }*/
 
     // 修改完毕
  
          __system_property_update(pi, value.c_str(), valuelen);
      } else {
          int rc = __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
          if (rc < 0) {
              *error = "__system_property_add failed";
              return PROP_ERROR_SET_FAILED;
          }
      }
  
      // Don't write properties to disk until after we have read all default
      // properties to prevent them from being overwritten by default values.
      if (persistent_properties_loaded && StartsWith(name, "persist.")) {
          WritePersistentProperty(name, value);
      }
      // If init hasn't started its main loop, then it won't be handling property changed messages
      // anyway, so there's no need to try to send them.
      auto lock = std::lock_guard{accept_messages_lock};
      if (accept_messages) {
          PropertyChanged(name, value);
      }
      return PROP_SUCCESS;
  }
```