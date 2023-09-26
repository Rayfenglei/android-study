> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124617541)

### 1. 概述

在 11.0 12.0 的系统产品开发中，由于默认的原生系统的拨打接听电话免提功能是关闭掉的，产品主要是用于老年机，由于产品开发需要根据产品需要要求开启默认免提模式，所以在设置通话模式的时候打开免提模式

### 2. 拨打接听电话默认开启免提的核心类

```
framework/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
framework/av/services/audiopolicy/enginedefault/src/Engine.cpp

```

### 3. 拨打接听电话默认开启免提核心功能分析和实现

在 AudioPolicyManager 是系统对于音频策略的管理类，所以在设置通话模式这块的相关功能也是在这里  
做设置的，所以需要看 AudioPolicyManager 的相关代码  
通话底层主要部分由 AudioPolicyManager.cpp 处理  
路径: framework/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp  
处理流程:

```
AudioSystem::setForceUse--AudioPolicyService::setForceUse（AudioPolicyInterfaceImpl.cpp）--AudioPolicyManager::setForceUse
void AudioPolicyManager::setForceUse(audio_policy_force_use_t usage,
audio_policy_forced_cfg_t config)
{
....
checkForDeviceAndOutputChanges();
// force client reconnection to reevaluate flag AUDIO_FLAG_AUDIBILITY_ENFORCED
if (usage == AUDIO_POLICY_FORCE_FOR_SYSTEM) {
    mpClientInterface->invalidateStream(AUDIO_STREAM_SYSTEM);
    mpClientInterface->invalidateStream(AUDIO_STREAM_ENFORCED_AUDIBLE);
}

//FIXME: workaround for truncated touch sounds
// to be removed when the problem is handled by system UI
uint32_t delayMs = 0;
uint32_t waitMs = 0;
if (usage == AUDIO_POLICY_FORCE_FOR_COMMUNICATION) {
    delayMs = TOUCH_SOUND_FIXED_DELAY_MS;
}
....

}
void AudioPolicyManager::checkForDeviceAndOutputChanges(std::function<bool()> onOutputsChecked)
{
// checkA2dpSuspend must run before checkOutputForAllStrategies so that A2DP
// output is suspended before any tracks are moved to it
checkA2dpSuspend();
checkOutputForAllStrategies();
checkSecondaryOutputs();
if (onOutputsChecked != nullptr && onOutputsChecked()) checkA2dpSuspend();
updateDevicesAndOutputs();
if (mHwModules.getModuleFromName(AUDIO_HARDWARE_MODULE_ID_MSD) != 0) {
setMsdPatch();
}
}
void AudioPolicyManager::updateDevicesAndOutputs()
{
mEngine->updateDeviceSelectionCache();
mPreviousOutputs = mOutputs;
}

```

在上述代码中 updateDevicesAndOutputs() 负责设置设备音频输出模式  
接下来看下 mEngine->updateDeviceSelectionCache();  
路径: framework/av/services/audiopolicy/enginedefault/src/Engine.cpp

```
void Engine::updateDeviceSelectionCache()
{
for (const auto &iter : getProductStrategies()) {
const auto &strategy = iter.second;
auto devices = getDevicesForProductStrategy(strategy->getId());
mDevicesForStrategies[strategy->getId()] = devices;
strategy->setDeviceTypes(devices.types());
strategy->setDeviceAddress(devices.getFirstValidAddress().c_str());
}
}
DeviceVector Engine::getDevicesForProductStrategy(product_strategy_t strategy) const {
DeviceVector availableOutputDevices = getApmObserver()->getAvailableOutputDevices();
  // check if this strategy has a preferred device that is available,
  // if yes, give priority to it
  AudioDeviceTypeAddr preferredStrategyDevice;
  const status_t status = getPreferredDeviceForStrategy(strategy, preferredStrategyDevice);
  if (status == NO_ERROR) {
      // there is a preferred device, is it available?
      sp<DeviceDescriptor> preferredAvailableDevDescr = availableOutputDevices.getDevice(
              preferredStrategyDevice.mType,
              String8(preferredStrategyDevice.mAddress.c_str()),
              AUDIO_FORMAT_DEFAULT);
      if (preferredAvailableDevDescr != nullptr) {
         ALOGVV("%s using pref device 0x%08x/%s for strategy %u",
                 __func__, preferredStrategyDevice.mType,
                preferredStrategyDevice.mAddress.c_str(), strategy);
          return DeviceVector(preferredAvailableDevDescr);
      }
  }

 DeviceVector availableInputDevices = getApmObserver()->getAvailableInputDevices();
  const SwAudioOutputCollection& outputs = getApmObserver()->getOutputs();
 auto legacyStrategy = mLegacyStrategyMap.find(strategy) != end(mLegacyStrategyMap) ?
                        mLegacyStrategyMap.at(strategy) : STRATEGY_NONE;
  return getDevicesForStrategyInt(legacyStrategy,
                                  availableOutputDevices,
                                  availableInputDevices, outputs);

}
DeviceVector Engine::getDevicesForStrategyInt(legacy_strategy strategy,
DeviceVector availableOutputDevices,
DeviceVector availableInputDevices,
const SwAudioOutputCollection &outputs) const
{
.....
  case STRATEGY_PHONE:
      // Force use of only devices on primary output if:
      // - in call AND
      //   - cannot route from voice call RX OR
     //   - audio HAL version is < 3.0 and TX device is on the primary HW module
      if (getPhoneState() == AUDIO_MODE_IN_CALL) {
          audio_devices_t txDevice = getDeviceForInputSource(
                  AUDIO_SOURCE_VOICE_COMMUNICATION)->type();
          sp<AudioOutputDescriptor> primaryOutput = outputs.getPrimaryOutput();
          LOG_ALWAYS_FATAL_IF(primaryOutput == nullptr, "Primary output not found");
         DeviceVector availPrimaryInputDevices =
                 availableInputDevices.getDevicesFromHwModule(primaryOutput->getModuleHandle());

          // TODO: getPrimaryOutput return only devices from first module in
         // audio_policy_configuration.xml, hearing aid is not there, but it's
          // a primary device
          // FIXME: this is not the right way of solving this problem
          DeviceVector availPrimaryOutputDevices = availableOutputDevices.getDevicesFromTypes(
                 primaryOutput->supportedDevices().types());
          availPrimaryOutputDevices.add(
                  availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_HEARING_AID));

          if ((availableInputDevices.getDevice(AUDIO_DEVICE_IN_TELEPHONY_RX,
                 String8(""), AUDIO_FORMAT_DEFAULT) == nullptr) ||
                  ((availPrimaryInputDevices.getDevice(
                          txDevice, String8(""), AUDIO_FORMAT_DEFAULT) != nullptr) &&
                        (primaryOutput->getPolicyAudioPort()->getModuleVersionMajor() < 3))) {
              availableOutputDevices = availPrimaryOutputDevices;
          }
    }
      // for phone strategy, we first consider the forced use and then the available devices by
      // order of priority
      switch (getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION)) {
      case AUDIO_POLICY_FORCE_BT_SCO:
          if (!isInCall() || strategy != STRATEGY_DTMF) {
              devices = availableOutputDevices.getDevicesFromType(
                     AUDIO_DEVICE_OUT_BLUETOOTH_SCO_CARKIT);
              if (!devices.isEmpty()) break;
          }
         devices = availableOutputDevices.getFirstDevicesFromTypes({
                  AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET, AUDIO_DEVICE_OUT_BLUETOOTH_SCO});
          if (!devices.isEmpty()) break;
          // if SCO device is requested but no SCO device is available, fall back to default case
          FALLTHROUGH_INTENDED;

      default:    // FORCE_NONE
          devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_HEARING_AID);
          if (!devices.isEmpty()) break;
          // when not in a phone call, phone strategy should route STREAM_VOICE_CALL to A2DP
          if (!isInCall() &&
                 (getForceUse(AUDIO_POLICY_FORCE_FOR_MEDIA) != AUDIO_POLICY_FORCE_NO_BT_A2DP) &&
                   outputs.isA2dpSupported()) {
              devices = availableOutputDevices.getFirstDevicesFromTypes({
                      AUDIO_DEVICE_OUT_BLUETOOTH_A2DP,
                     AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES});
             if (!devices.isEmpty()) break;
         }
          devices = availableOutputDevices.getFirstDevicesFromTypes({
                  AUDIO_DEVICE_OUT_WIRED_HEADPHONE, AUDIO_DEVICE_OUT_WIRED_HEADSET,
                  AUDIO_DEVICE_OUT_LINE, AUDIO_DEVICE_OUT_USB_HEADSET,
                  AUDIO_DEVICE_OUT_USB_DEVICE});
         if (!devices.isEmpty()) break;
          if (!isInCall()) {
              devices = availableOutputDevices.getFirstDevicesFromTypes({
                      AUDIO_DEVICE_OUT_USB_ACCESSORY, AUDIO_DEVICE_OUT_DGTL_DOCK_HEADSET,
                      AUDIO_DEVICE_OUT_AUX_DIGITAL, AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET});
             if (!devices.isEmpty()) break;
          }
          devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_EARPIECE);
          break;

      case AUDIO_POLICY_FORCE_SPEAKER:
         // when not in a phone call, phone strategy should route STREAM_VOICE_CALL to
          // A2DP speaker when forcing to speaker output
         if (!isInCall() &&
                  (getForceUse(AUDIO_POLICY_FORCE_FOR_MEDIA) != AUDIO_POLICY_FORCE_NO_BT_A2DP) &&
                 outputs.isA2dpSupported()) {
             devices = availableOutputDevices.getDevicesFromType(
                    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER);
             if (!devices.isEmpty()) break;
          }
          if (!isInCall()) {
              devices = availableOutputDevices.getFirstDevicesFromTypes({
                     AUDIO_DEVICE_OUT_USB_ACCESSORY, AUDIO_DEVICE_OUT_USB_DEVICE,
                     AUDIO_DEVICE_OUT_DGTL_DOCK_HEADSET, AUDIO_DEVICE_OUT_AUX_DIGITAL,
                      AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET});
              if (!devices.isEmpty()) break;
          }
          devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER);
        break;
      }
  break;
 ......

}

```

getDeviceForStrategyInt() 处理听筒的输出方式 主要是根据 type 类型判断使用什么音频输出模式  
默认是 devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_EARPIECE); 这个模式  
所以就需要改成听筒模式  
修改为:

```
case STRATEGY_PHONE
....
switch (getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION)) {
 ...

default:    // FORCE_NONE

....

-devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_EARPIECE);
+devices = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER);
break；

```