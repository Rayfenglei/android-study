> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739284)

### 1. 概述

在定制 11.0 的产品的时候，由于没有音量键 所以用音量键和电源键来选择 recovery 模式就无法实现了 所以当进入 recovery 选择模式界面 就一直停在那里 根据需要 要修改成进入等待用户选择 recovery 模式时，直接[恢复出厂设置](https://so.csdn.net/so/search?q=%E6%81%A2%E5%A4%8D%E5%87%BA%E5%8E%82%E8%AE%BE%E7%BD%AE&spm=1001.2101.3001.7020)就好  
2. 进入 recovery 模式 (等待用户选择 recovery 模式界面) 进入自动恢复出厂设置模式核心类

```
bootable\recovery\recovery_main.cpp

```

先看 recovery_main.cpp 代码如下  
路径: bootable\recovery\recovery_main.cpp  
int main(int argc, char** argv) {  
… 省略

```
redirect_stdio(Paths::Get().temporary_log_file().c_str());
load_volume_table();
has_cache = volume_for_mount_point(CACHE_ROOT) != nullptr;
std::vectorstd::string args = get_args(argc, argv);
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
// Intentionally not calling dlclose(3) to avoid potential gotchas (e.g. make_device may have
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
if (!has_cache) {
device->RemoveMenuItemForAction(Device::WIPE_CACHE);
}
if (!android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
device->RemoveMenuItemForAction(Device::ENTER_FASTBOOT);
}
if (!is_ro_debuggable()) {
device->RemoveMenuItemForAction(Device::ENTER_RESCUE);
}
ui->SetBackground(RecoveryUI::NONE);
if (show_text) ui->ShowText(true);
LOG(INFO) << "Starting recovery (pid " << getpid() << ") on " << ctime(&start);
LOG(INFO) << "locale is [" << locale << "]";
sehandle = selinux_android_file_context_handle();
selinux_android_set_sehandle(sehandle);
if (!sehandle) {
ui->Print("Warning: No file_contexts\n");
}
SetLoggingSehandle(sehandle);
std::atomicDevice::BuiltinAction action;
std::thread listener_thread(ListenRecoverySocket, ui, std::ref(action));
listener_thread.detach();
while (true) {
std::string usb_config = fastboot ? "fastboot" : is_ro_debuggable() ? "adb" : "none";
std::string usb_state = android::base::GetProperty("sys.usb.state", "none");
if (usb_config != usb_state) {
if (!SetUsbConfig("none")) {
LOG(ERROR) << "Failed to clear USB config";
}
if (!SetUsbConfig(usb_config)) {
LOG(ERROR) << "Failed to set USB config to " << usb_config;
}
}
ui->SetEnableFastbootdLogo(fastboot);
auto ret = fastboot ? StartFastboot(device, args) : start_recovery(device, args);
if (ret == Device::KEY_INTERRUPTED) {
ret = action.exchange(ret);
if (ret == Device::NO_ACTION) {
continue;
}
}
switch (ret) {
case Device::SHUTDOWN:
ui->Print("Shutting down...\n");
// TODO: Move all the reboots to reboot(), which should conditionally set quiescent flag.
android::base::SetProperty(ANDROID_RB_PROPERTY, "shutdown,");
break;
case Device::REBOOT_BOOTLOADER:
ui->Print("Rebooting to bootloader...\n");
android::base::SetProperty(ANDROID_RB_PROPERTY, "reboot,bootloader");
break;
case Device::REBOOT_FASTBOOT:
ui->Print("Rebooting to recovery/fastboot...\n");
android::base::SetProperty(ANDROID_RB_PROPERTY, "reboot,fastboot");
break;
case Device::REBOOT_RECOVERY:
ui->Print("Rebooting to recovery...\n");
reboot("reboot,recovery");
break;
case Device::REBOOT_RESCUE: {
// Not using reboot("reboot,rescue"), as it requires matching support in kernel and/or
// bootloader.
bootloader_message boot = {};
strlcpy(boot.command, "boot-rescue", sizeof(boot.command));
std::string err;
if (!write_bootloader_message(boot, &err)) {
LOG(ERROR) << "Failed to write bootloader message: " << err;
// Stay under recovery on failure.
continue;
}
ui->Print("Rebooting to recovery/rescue...\n");
reboot("reboot,recovery");
break;
}
case Device::ENTER_FASTBOOT:
if (logical_partitions_mapped()) {
ui->Print("Partitions may be mounted - rebooting to enter fastboot.");
android::base::SetProperty(ANDROID_RB_PROPERTY, "reboot,fastboot");
} else {
LOG(INFO) << "Entering fastboot";
fastboot = true;
}
break;
case Device::ENTER_RECOVERY:
LOG(INFO) << "Entering recovery";
fastboot = false;
break;
default:
ui->Print("Rebooting...\n");
reboot("reboot,");
break;
}
}
// Should be unreachable.
return EXIT_SUCCESS;
}

Device::BuiltinAction start_recovery(Device* device, const std::vectorstd::string& args) {
static constexpr struct option OPTIONS[] = {
{ "fastboot", no_argument, nullptr, 0 },
{ "fsck_unshare_blocks", no_argument, nullptr, 0 },
{ "just_exit", no_argument, nullptr, 'x' },
{ "locale", required_argument, nullptr, 0 },
{ "prompt_and_wipe_data", no_argument, nullptr, 0 },
{ "reason", required_argument, nullptr, 0 },
{ "rescue", no_argument, nullptr, 0 },
{ "retry_count", required_argument, nullptr, 0 },
{ "security", no_argument, nullptr, 0 },
{ "show_text", no_argument, nullptr, 't' },
{ "shutdown_after", no_argument, nullptr, 0 },
{ "sideload", no_argument, nullptr, 0 },
{ "sideload_auto_reboot", no_argument, nullptr, 0 },
{ "update_package", required_argument, nullptr, 0 },
{ "wipe_ab", no_argument, nullptr, 0 },
{ "wipe_cache", no_argument, nullptr, 0 },
{ "wipe_data", no_argument, nullptr, 0 },
{ "wipe_package_size", required_argument, nullptr, 0 },
{ nullptr, 0, nullptr, 0 },
};
const char* update_package = nullptr;
bool should_wipe_data = false;
bool should_prompt_and_wipe_data = false;
bool should_wipe_cache = false;
bool should_wipe_ab = false;
size_t wipe_package_size = 0;
bool sideload = false;
bool sideload_auto_reboot = false;
bool rescue = false;
bool just_exit = false;
bool shutdown_after = false;
bool fsck_unshare_blocks = false;
int retry_count = 0;
bool security_update = false;
std::string locale;
auto args_to_parse = StringVectorToNullTerminatedArray(args);
int arg;
int option_index;
// Parse everything before the last element (which must be a nullptr). getopt_long(3) expects a
// null-terminated char* array, but without counting null as an arg (i.e. argv[argc] should be
// nullptr).
while ((arg = getopt_long(args_to_parse.size() - 1, args_to_parse.data(), "", OPTIONS,
&option_index)) != -1) {
switch (arg) {
case 't':
// Handled in recovery_main.cpp
break;
case 'x':
just_exit = true;
break;
case 0: {
std::string option = OPTIONS[option_index].name;
if (option == "fsck_unshare_blocks") {
fsck_unshare_blocks = true;
} else if (option == "locale" || option == "fastboot") {
// Handled in recovery_main.cpp
} else if (option == "prompt_and_wipe_data") {
should_prompt_and_wipe_data = true;
} else if (option == "reason") {
reason = optarg;
} else if (option == "rescue") {
rescue = true;
} else if (option == "retry_count") {
android::base::ParseInt(optarg, &retry_count, 0);
} else if (option == "security") {
security_update = true;
} else if (option == "sideload") {
sideload = true;
} else if (option == "sideload_auto_reboot") {
sideload = true;
sideload_auto_reboot = true;
} else if (option == "shutdown_after") {
shutdown_after = true;
} else if (option == "update_package") {
update_package = optarg;
} else if (option == "wipe_ab") {
should_wipe_ab = true;
} else if (option == "wipe_cache") {
should_wipe_cache = true;
} else if (option == "wipe_data") {
should_wipe_data = true;
} else if (option == "wipe_package_size") {
android::base::ParseUint(optarg, &wipe_package_size);
}
break;
}
case '?':
LOG(ERROR) << "Invalid command argument";
continue;
}
}
optind = 1;
printf("stage is [%s]\n", stage.c_str());
printf("reason is [%s]\n", reason);
// Set background string to "installing security update" for security update,
// otherwise set it to "installing system update".
ui->SetSystemUpdateText(security_update);
int st_cur, st_max;
if (!stage.empty() && sscanf(stage.c_str(), "%d/%d", &st_cur, &st_max) == 2) {
ui->SetStage(st_cur, st_max);
}
std::vectorstd::string title_lines =
android::base::Split(android::base::GetProperty("ro.bootimage.build.fingerprint", ""), ":");
title_lines.insert(std::begin(title_lines), "Android Recovery");
ui->SetTitle(title_lines);
ui->ResetKeyInterruptStatus();
device->StartRecovery();
printf("Command:");
for (const auto& arg : args) {
printf(" "%s"", arg.c_str());
}
printf("\n\n");
if (update_package) {
//adupsfota start for replace sdcard path
#ifdef ADUPS_FOTA_SUPPORT
static const char SDCARD_ROOT = "/storage/sdcard0";if (strncmp(update_package, "/storage/", 9) == 0) {int len = strlen(update_package) + 12;char modified_path = (char*)malloc(len);
strlcpy(modified_path, SDCARD_ROOT, len);
if(strstr(update_package, "/Android/data/com.adups.fota/files/adupsfota/update.zip") != NULL){
char const *SDCARD_PATH = "/Android/data/com.adups.fota/files/adupsfota/update.zip";
strlcat(modified_path, SDCARD_PATH, len);
}else if(strstr(update_package, "/Android/data/com.adups.fota/files/LocalSdUpdate.zip") != NULL){
char const *SDCARD_PATH = "/Android/data/com.adups.fota/files/LocalSdUpdate.zip";
strlcat(modified_path, SDCARD_PATH, len);
}
printf("(replacing path "%s" with "%s")\n",update_package, modified_path);
update_package = modified_path;
}
#endif
//adupsfota end
}
printf("\n");
property_list(print_property, nullptr);
printf("\n");
ui->Print("Supported API: %d\n", kRecoveryApiVersion);
int status = INSTALL_SUCCESS;
// next_action indicates the next target to reboot into upon finishing the install. It could be
// overridden to a different reboot target per user request.
Device::BuiltinAction next_action = shutdown_after ? Device::SHUTDOWN : Device::REBOOT;
if (update_package != nullptr) {
// It's not entirely true that we will modify the flash. But we want
// to log the update attempt since update_package is non-NULL.
save_current_log = true;
int required_battery_level;
if (retry_count == 0 && !is_battery_ok(&required_battery_level)) {
ui->Print("battery capacity is not enough for installing package: %d%% needed\n",
required_battery_level);
// Log the error code to last_install when installation skips due to
// low battery.
log_failure_code(kLowBattery, update_package);
status = INSTALL_SKIPPED;
} else if (retry_count == 0 && bootreason_in_blacklist()) {
// Skip update-on-reboot when bootreason is kernel_panic or similar
ui->Print("bootreason is in the blacklist; skip OTA installation\n");
log_failure_code(kBootreasonInBlacklist, update_package);
status = INSTALL_SKIPPED;
} else {
// It's a fresh update. Initialize the retry_count in the BCB to 1; therefore we can later
// identify the interrupted update due to unexpected reboots.
if (retry_count == 0) {
set_retry_bootloader_message(retry_count + 1, args);
}
status = install_package(update_package, should_wipe_cache, true, retry_count, ui);
if (status != INSTALL_SUCCESS) {
ui->Print("Installation aborted.\n");
// When I/O error or bspatch/imgpatch error happens, reboot and retry installation
// RETRY_LIMIT times before we abandon this OTA update.
static constexpr int RETRY_LIMIT = 4;
if (status == INSTALL_RETRY && retry_count < RETRY_LIMIT) {
  copy_logs(save_current_log, has_cache, sehandle);
  retry_count += 1;
  set_retry_bootloader_message(retry_count, args);
  // Print retry count on screen.
  ui->Print("Retry attempt %d\n", retry_count);

  // Reboot and retry the update
  if (!reboot("reboot,recovery")) {
    ui->Print("Reboot failed\n");
  } else {
    while (true) {
      pause();
    }
  }
}
// If this is an eng or userdebug build, then automatically
// turn the text display on if the script fails so the error
// message is visible.
if (is_ro_debuggable()) {
  ui->ShowText(true);
}

}
}
} else if (should_wipe_data) {
save_current_log = true;
bool convert_fbe = reason && strcmp(reason, "convert_fbe") == 0;
if (!WipeData(device, convert_fbe)) {
status = INSTALL_ERROR;
}
} else if (should_prompt_and_wipe_data) {
// Trigger the logging to capture the cause, even if user chooses to not wipe data.
save_current_log = true;
ui->ShowText(true);
ui->SetBackground(RecoveryUI::ERROR);
ui->Print("*NEED TO WIPE USERDATA\nREASON IS:\n%s\n",reason);
status = prompt_and_wipe_data(device);
if (status != INSTALL_KEY_INTERRUPTED) {
ui->ShowText(false);
}
} else if (should_wipe_cache) {
save_current_log = true;
if (!WipeCache(ui, nullptr)) {
status = INSTALL_ERROR;
}
} else if (should_wipe_ab) {
if (!wipe_ab_device(wipe_package_size)) {
status = INSTALL_ERROR;
}
} else if (sideload) {
// 'adb reboot sideload' acts the same as user presses key combinations to enter the sideload
// mode. When 'sideload-auto-reboot' is used, text display will NOT be turned on by default. And
// it will reboot after sideload finishes even if there are errors. This is to enable automated
// testing.
save_current_log = true;
if (!sideload_auto_reboot) {
ui->ShowText(true);
}
status = ApplyFromAdb(device, false /* rescue_mode /, &next_action);ui->Print("\nInstall from ADB complete (status: %d).\n", status);if (sideload_auto_reboot) {status = INSTALL_REBOOT;ui->Print("Rebooting automatically.\n");}} else if (rescue) {save_current_log = true;status = ApplyFromAdb(device, true / rescue_mode */, &next_action);
ui->Print("\nInstall from ADB complete (status: %d).\n", status);
} else if (fsck_unshare_blocks) {
if (!do_fsck_unshare_blocks()) {
status = INSTALL_ERROR;
}
} else if (!just_exit) {
// If this is an eng or userdebug build, automatically turn on the text display if no command
// is specified. Note that this should be called before setting the background to avoid
// flickering the background image.
if (is_ro_debuggable()) {
ui->ShowText(true);
}
status = INSTALL_NONE;  // No command specified
ui->SetBackground(RecoveryUI::NO_COMMAND);
}
if (status == INSTALL_ERROR || status == INSTALL_CORRUPT) {
ui->SetBackground(RecoveryUI::ERROR);
if (!ui->IsTextVisible()) {
sleep(5);
}
}
// Determine the next action.
//  - If the state is INSTALL_REBOOT, device will reboot into the target as specified in
//    next_action.
//  - If the recovery menu is visible, prompt and wait for commands.
//  - If the state is INSTALL_NONE, wait for commands (e.g. in user build, one manually boots
//    into recovery to sideload a package or to wipe the device).
//  - In all other cases, reboot the device. Therefore, normal users will observe the device
//    rebooting a) immediately upon successful finish (INSTALL_SUCCESS); or b) an "error" screen
//    for 5s followed by an automatic reboot.
if (status != INSTALL_REBOOT) {
if (status == INSTALL_NONE || ui->IsTextVisible()) {
Device::BuiltinAction temp = prompt_and_wait(device, status);
if (temp != Device::NO_ACTION) {
next_action = temp;
}
}
}
// Save logs and clean up before rebooting or shutting down.
finish_recovery();
return next_action;
}

```

在 main(int argc, char** argv) 中  
这里会调用 auto ret = fastboot ? StartFastboot(device, args) : start_recovery(device, args);  
来调用 recovery->start_recovery() 来开始 recovery  
在带参数的 args 时会进入相应的 recovery 没有带参数就会进入

```
prompt_and_wait(device, status);
static Device::BuiltinAction prompt_and_wait(Device* device, int status) {
for (;;) {
finish_recovery();
switch (status) {
case INSTALL_SUCCESS:
case INSTALL_NONE:
ui->SetBackground(RecoveryUI::NO_COMMAND);
break;
case INSTALL_ERROR:
case INSTALL_CORRUPT:
ui->SetBackground(RecoveryUI::ERROR);
break;
}
ui->SetProgressType(RecoveryUI::EMPTY);
// 注释掉选择recovery模式代码
/size_t chosen_item = ui->ShowMenu({}, device->GetMenuItems(), 0, false,std::bind(&Device::HandleMenuKey, device, std::placeholders::_1, std::placeholders::_2));// Handle Interrupt keyif (chosen_item == static_cast<size_t>(RecoveryUI::KeyError::INTERRUPTED)) {return Device::KEY_INTERRUPTED;}/
// Device-specific code may take some action here. It may return one of the core actions
// handled in the switch statement below.
Device::BuiltinAction chosen_action = Device::WIPE_DATA; // 直接返回恢复出厂设置
/(chosen_item == static_cast<size_t>(RecoveryUI::KeyError::TIMED_OUT))? Device::REBOOT: device->InvokeMenuItem(chosen_item);/
switch (chosen_action) {
case Device::NO_ACTION:
break;
case Device::ENTER_FASTBOOT:
case Device::ENTER_RECOVERY:
case Device::REBOOT:
case Device::REBOOT_BOOTLOADER:
case Device::REBOOT_FASTBOOT:
case Device::REBOOT_RECOVERY:
case Device::REBOOT_RESCUE:
case Device::SHUTDOWN:
return chosen_action;
case Device::WIPE_DATA:
save_current_log = true;
if (ui->IsTextVisible()) {
// 去掉询问是否进入恢复出厂设置
//if (ask_to_wipe_data(device)) {
WipeData(device, false);
return Device::NO_ACTION;
// }
} else {
WipeData(device, false);
return Device::NO_ACTION;
}
break;
case Device::WIPE_CACHE: {
save_current_log = true;
std::function<bool()> confirm_func = &device {
return yes_no(device, "Wipe cache?", "  THIS CAN NOT BE UNDONE!");
};
WipeCache(ui, ui->IsTextVisible() ? confirm_func : nullptr);
if (!ui->IsTextVisible()) return Device::NO_ACTION;
break;
}
....
}

```

所以在 prompt_and_wait(Device* device, int status) 中可以在 ui->ShowMenu(）这个方法中  
进入到选择 recovery 模式的时候，注释掉这个部分的功能，直接赋值为恢复出厂模式的 Device::WIPE_DATA 模式，然后进入到恢复出厂模式