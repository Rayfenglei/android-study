> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126668723)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.recovery 恢复出厂设置保留某些文件功能实现相关核心代码](#t1)

[3.recovery 恢复出厂设置保留某些文件功能实现相关核心代码功能分析以及功能实现](#t2)

 [3.1 wipe_data.cpp 关于恢复出厂设置清理缓存的相关代码](#t3)

[3.2 logging.cpp 关于判断当前路径是否是 cache 分区的相关代码](#t4)

[3.3 RecoverySystem.java 关于路径的相关判断功能分析以及实现](#t5)

1. 概述
-----

  在进行 [recovery](https://so.csdn.net/so/search?q=recovery&spm=1001.2101.3001.7020) 产品开发中，这个也是比较重要的功能，目前的开发需求中要求某些重要的数据在 recovery 恢复出厂设置时，要保留下来，所以要熟悉 recovery 流程，然后在清除数据的时候，跳过这些目录就可以了

2.recovery [恢复出厂设置](https://so.csdn.net/so/search?q=%E6%81%A2%E5%A4%8D%E5%87%BA%E5%8E%82%E8%AE%BE%E7%BD%AE&spm=1001.2101.3001.7020)保留某些文件功能实现相关核心代码
---------------------------------------------------------------------------------------------------------------------------------------------------

```
  bootable/recovery/install/wipe_data.cpp
  bootable/recovery/recovery_utils/logging.cpp
  frameworks/base/core/java/android/os/RecoverySystem.java
```

3.recovery 恢复出厂设置保留某些文件功能实现相关核心代码功能分析以及功能实现
-------------------------------------------

###   3.1 wipe_data.cpp 关于恢复出厂设置清理缓存的相关代码

```
   static bool EraseVolume(const char* volume, RecoveryUI* ui, bool convert_fbe) {
   bool is_cache = (strcmp(volume, CACHE_ROOT) == 0);
   bool is_data = (strcmp(volume, DATA_ROOT) == 0);
 
   ui->SetBackground(RecoveryUI::ERASING);
   ui->SetProgressType(RecoveryUI::INDETERMINATE);
 
   std::vector<saved_log_file> log_files;
   if (is_cache) {
     // If we're reformatting /cache, we load any past logs (i.e. "/cache/recovery/last_*") and the
     // current log ("/cache/recovery/log") into memory, so we can restore them after the reformat.
     log_files = ReadLogFilesToMemory();
   }
 
   ui->Print("Formatting %s...\n", volume);
 
   ensure_path_unmounted(volume);
 
   int result;
   if (is_data && convert_fbe) {
     constexpr const char* CONVERT_FBE_DIR = "/tmp/convert_fbe";
     constexpr const char* CONVERT_FBE_FILE = "/tmp/convert_fbe/convert_fbe";
     // Create convert_fbe breadcrumb file to signal init to convert to file based encryption, not
     // full disk encryption.
     if (mkdir(CONVERT_FBE_DIR, 0700) != 0) {
       PLOG(ERROR) << "Failed to mkdir " << CONVERT_FBE_DIR;
       return false;
     }
     FILE* f = fopen(CONVERT_FBE_FILE, "wbe");
     if (!f) {
       PLOG(ERROR) << "Failed to convert to file encryption";
       return false;
     }
     fclose(f);
     result = format_volume(volume, CONVERT_FBE_DIR);
     remove(CONVERT_FBE_FILE);
     rmdir(CONVERT_FBE_DIR);
   } else {
     result = format_volume(volume);
   }
 
   if (is_cache) {
     RestoreLogFilesAfterFormat(log_files);
   }
 
   return (result == 0);
 }
 
 bool WipeCache(RecoveryUI* ui, const std::function<bool()>& confirm_func) {
   bool has_cache = volume_for_mount_point("/cache") != nullptr;
   if (!has_cache) {
     ui->Print("No /cache partition found.\n");
     return false;
   }
 
   if (confirm_func && !confirm_func()) {
     return false;
   }
 
   ui->Print("\n-- Wiping cache...\n");
    bool success = EraseVolume("/cache", ui, false);
    ui->Print("Cache wipe %s.\n", success ? "complete" : "failed");
    return success;
  }
  
  bool WipeData(Device* device, bool convert_fbe) {
    RecoveryUI* ui = device->GetUI();
    ui->Print("\n-- Wiping data...\n");
  
    if (!FinishPendingSnapshotMerges(device)) {
      ui->Print("Unable to check update status or complete merge, cannot wipe partitions.\n");
      return false;
    }
  
    bool success = device->PreWipeData();
    if (success) {
      success &= EraseVolume(DATA_ROOT, ui, convert_fbe);
      bool has_cache = volume_for_mount_point("/cache") != nullptr;
      if (has_cache) {
        success &= EraseVolume(CACHE_ROOT, ui, false);
      }
      if (volume_for_mount_point(METADATA_ROOT) != nullptr) {
        success &= EraseVolume(METADATA_ROOT, ui, false);
      }
    }
    if (success) {
      success &= device->PostWipeData();
    }
    ui->Print("Data wipe %s.\n", success ? "complete" : "failed");
    return success;
  }
 
EraseVolume(const char* volume, RecoveryUI* ui, bool convert_fbe)主要是清理相关分析的功能
而   if (is_cache) {
     // If we're reformatting /cache, we load any past logs (i.e. "/cache/recovery/last_*") and the
     // current log ("/cache/recovery/log") into memory, so we can restore them after the reformat.
     log_files = ReadLogFilesToMemory();
   }
是判断当前路径是否是cache分区
```

### 3.2 logging.cpp 关于判断当前路径是否是 cache 分区的相关代码

```
    
  std::vector<saved_log_file> ReadLogFilesToMemory() {
    ensure_path_mounted("/cache");
  
    struct dirent* de;
    std::unique_ptr<DIR, decltype(&closedir)> d(opendir(CACHE_LOG_DIR), closedir);
    if (!d) {
      if (errno != ENOENT) {
        PLOG(ERROR) << "Failed to opendir " << CACHE_LOG_DIR;
      }
      return {};
    }
  
    std::vector<saved_log_file> log_files;
    while ((de = readdir(d.get())) != nullptr) {
      if (strncmp(de->d_name, "last_", 5) == 0 || strcmp(de->d_name, "log") == 0) {
        std::string path = android::base::StringPrintf("%s/%s", CACHE_LOG_DIR, de->d_name);
  
        struct stat sb;
        if (stat(path.c_str(), &sb) != 0) {
          PLOG(ERROR) << "Failed to stat " << path;
          continue;
        }
        // Truncate files to 512kb
        size_t read_size = std::min<size_t>(sb.st_size, 1 << 19);
        std::string data(read_size, '\0');
  
        android::base::unique_fd log_fd(TEMP_FAILURE_RETRY(open(path.c_str(), O_RDONLY)));
        if (log_fd == -1 || !android::base::ReadFully(log_fd, data.data(), read_size)) {
          PLOG(ERROR) << "Failed to read log file " << path;
          continue;
        }
  
        log_files.emplace_back(saved_log_file{ path, sb, data });
      }
    }
  
    return log_files;
  }
  
  bool RestoreLogFilesAfterFormat(const std::vector<saved_log_file>& log_files) {
    // Re-create the log dir and write back the log entries.
    if (ensure_path_mounted(CACHE_LOG_DIR) != 0) {
      PLOG(ERROR) << "Failed to mount " << CACHE_LOG_DIR;
      return false;
    }
  
    if (mkdir_recursively(CACHE_LOG_DIR, 0777, false, logging_sehandle) != 0) {
      PLOG(ERROR) << "Failed to create " << CACHE_LOG_DIR;
      return false;
    }
  
    for (const auto& log : log_files) {
      if (!android::base::WriteStringToFile(log.data, log.name, log.sb.st_mode, log.sb.st_uid,
                                            log.sb.st_gid)) {
        PLOG(ERROR) << "Failed to write to " << log.name;
      }
    }
  
    // Any part of the log we'd copied to cache is now gone.
    // Reset the pointer so we copy from the beginning of the temp
    // log.
    reset_tmplog_offset();
    copy_logs(true /* save_current_log */);
  
    return true;
  }
ReadLogFilesToMemory()是判定当前路径是否是cache分区
而在if (strncmp(de->d_name, "last_", 5) == 0 || strcmp(de->d_name, "log") == 0) 判定路径是否以last_和log开头
然后忽略掉
所以可以修改为:
  std::vector<saved_log_file> ReadLogFilesToMemory() {
    ensure_path_mounted("/cache");
  
    struct dirent* de;
    std::unique_ptr<DIR, decltype(&closedir)> d(opendir(CACHE_LOG_DIR), closedir);
    if (!d) {
      if (errno != ENOENT) {
        PLOG(ERROR) << "Failed to opendir " << CACHE_LOG_DIR;
      }
      return {};
    }
  
    std::vector<saved_log_file> log_files;
    while ((de = readdir(d.get())) != nullptr) {
     - if (strncmp(de->d_name, "last_", 5) == 0 || strcmp(de->d_name, "log") == 0) {
     +if (strncmp(de->d_name, "last_", 5) == 0 || strcmp(de->d_name, "log") == 0 || strcmp(de->d_name, "custom_") == 0) {
        std::string path = android::base::StringPrintf("%s/%s", CACHE_LOG_DIR, de->d_name);
  
        struct stat sb;
        if (stat(path.c_str(), &sb) != 0) {
          PLOG(ERROR) << "Failed to stat " << path;
          continue;
        }
        // Truncate files to 512kb
        size_t read_size = std::min<size_t>(sb.st_size, 1 << 19);
        std::string data(read_size, '\0');
  
        android::base::unique_fd log_fd(TEMP_FAILURE_RETRY(open(path.c_str(), O_RDONLY)));
        if (log_fd == -1 || !android::base::ReadFully(log_fd, data.data(), read_size)) {
          PLOG(ERROR) << "Failed to read log file " << path;
          continue;
        }
  
        log_files.emplace_back(saved_log_file{ path, sb, data });
      }
    }
  
    return log_files;
  }
```

### 3.3 RecoverySystem.java 关于路径的相关判断功能分析以及实现

```
       public static String handleAftermath(Context context) {
          // Record the tail of the LOG_FILE
          String log = null;
          try {
              log = FileUtils.readTextFile(LOG_FILE, -LOG_FILE_MAX_LENGTH, "...\n");
          } catch (FileNotFoundException e) {
              Log.i(TAG, "No recovery log file");
          } catch (IOException e) {
              Log.e(TAG, "Error reading recovery log", e);
          }
  
  
          // Only remove the OTA package if it's partially processed (uncrypt'd).
          boolean reservePackage = BLOCK_MAP_FILE.exists();
          if (!reservePackage && UNCRYPT_PACKAGE_FILE.exists()) {
              String filename = null;
              try {
                  filename = FileUtils.readTextFile(UNCRYPT_PACKAGE_FILE, 0, null);
              } catch (IOException e) {
                  Log.e(TAG, "Error reading uncrypt file", e);
              }
  
              // Remove the OTA package on /data that has been (possibly
              // partially) processed. (Bug: 24973532)
              if (filename != null && filename.startsWith("/data")) {
                  if (UNCRYPT_PACKAGE_FILE.delete()) {
                      Log.i(TAG, "Deleted: " + filename);
                  } else {
                      Log.e(TAG, "Can't delete: " + filename);
                  }
              }
          }
  
          // We keep the update logs (beginning with LAST_PREFIX), and optionally
          // the block map file (BLOCK_MAP_FILE) for a package. BLOCK_MAP_FILE
          // will be created at the end of a successful uncrypt. If seeing this
          // file, we keep the block map file and the file that contains the
          // package name (UNCRYPT_PACKAGE_FILE). This is to reduce the work for
          // GmsCore to avoid re-downloading everything again.
          String[] names = RECOVERY_DIR.list();
          for (int i = 0; names != null && i < names.length; i++) {
              // Do not remove the last_install file since the recovery-persist takes care of it.
              if (names[i].startsWith(LAST_PREFIX) || names[i].equals(LAST_INSTALL_PATH)) continue;
              if (reservePackage && names[i].equals(BLOCK_MAP_FILE.getName())) continue;
              if (reservePackage && names[i].equals(UNCRYPT_PACKAGE_FILE.getName())) continue;
  
              recursiveDelete(new File(RECOVERY_DIR, names[i]));
          }
  
          return log;
      }
 
handleAftermath()中
for (int i = 0; names != null && i < names.length; i++) {
              // Do not remove the last_install file since the recovery-persist takes care of it.
              if (names[i].startsWith(LAST_PREFIX) || names[i].equals(LAST_INSTALL_PATH)) continue;
              if (reservePackage && names[i].equals(BLOCK_MAP_FILE.getName())) continue;
              if (reservePackage && names[i].equals(UNCRYPT_PACKAGE_FILE.getName())) continue;
  
              recursiveDelete(new File(RECOVERY_DIR, names[i]));
          }
来判断哪些不需要清理的 所以可以在这里增加不需要清理的文件路径
具体修改为:
        public static String handleAftermath(Context context) {
          // Record the tail of the LOG_FILE
          String log = null;
          try {
              log = FileUtils.readTextFile(LOG_FILE, -LOG_FILE_MAX_LENGTH, "...\n");
          } catch (FileNotFoundException e) {
              Log.i(TAG, "No recovery log file");
          } catch (IOException e) {
              Log.e(TAG, "Error reading recovery log", e);
          }
  
  
          // Only remove the OTA package if it's partially processed (uncrypt'd).
          boolean reservePackage = BLOCK_MAP_FILE.exists();
          if (!reservePackage && UNCRYPT_PACKAGE_FILE.exists()) {
              String filename = null;
              try {
                  filename = FileUtils.readTextFile(UNCRYPT_PACKAGE_FILE, 0, null);
              } catch (IOException e) {
                  Log.e(TAG, "Error reading uncrypt file", e);
              }
  
              // Remove the OTA package on /data that has been (possibly
              // partially) processed. (Bug: 24973532)
              if (filename != null && filename.startsWith("/data")) {
                  if (UNCRYPT_PACKAGE_FILE.delete()) {
                      Log.i(TAG, "Deleted: " + filename);
                  } else {
                      Log.e(TAG, "Can't delete: " + filename);
                  }
              }
          }
  
          // We keep the update logs (beginning with LAST_PREFIX), and optionally
          // the block map file (BLOCK_MAP_FILE) for a package. BLOCK_MAP_FILE
          // will be created at the end of a successful uncrypt. If seeing this
          // file, we keep the block map file and the file that contains the
          // package name (UNCRYPT_PACKAGE_FILE). This is to reduce the work for
          // GmsCore to avoid re-downloading everything again.
          String[] names = RECOVERY_DIR.list();
          for (int i = 0; names != null && i < names.length; i++) {
              // Do not remove the last_install file since the recovery-persist takes care of it.
              if (names[i].startsWith(LAST_PREFIX) || names[i].equals(LAST_INSTALL_PATH)) continue;
              if (reservePackage && names[i].equals(BLOCK_MAP_FILE.getName())) continue;
              if (reservePackage && names[i].equals(UNCRYPT_PACKAGE_FILE.getName())) continue;
              + names[i].startsWith("custom_") continue;
              recursiveDelete(new File(RECOVERY_DIR, names[i]));
          }
  
          return log;
      }
```