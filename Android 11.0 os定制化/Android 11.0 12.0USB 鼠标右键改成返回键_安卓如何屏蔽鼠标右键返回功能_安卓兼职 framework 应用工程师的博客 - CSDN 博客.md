> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124806968)

### 1. 概述

在 11.0 12.0 设备定制化开发中，产品有好几个 usb 口，用来可以连接外设，所以 USB 鼠标通过 usb 口来控制设  
备也是常见的问题，在 window 系统中，鼠标右键是返回键的功能，可是 android 原生的系统 鼠标右键不是返回键 根据客户需要鼠标修改成右键就需要跟代码，

### 2.USB 鼠标右键改成返回键的核心类

```
frameworks/native/services/inputflinger/reader/InputReader.cpp
frameworks/native/services/inputflinger/reader/mapper/accumulator/CursorButtonAccumulator.cpp
device\sprd\sharkle\sl8541e_1h10\system.prop

```

### 3.USB 鼠标右键改成返回键的核心功能分析和实现

功能分析：  
InputReader 从 EventHub 读取原始事件数据，并将其处理为输入事件，并将其发送到 InputListener。 InputReader 的某些功能（例如低功耗状态下的早期事件过滤）由单独的策略对象控制。  
追踪代码到 InputReader.cpp 文件，位置 frameworks/native/services/inputflinger/reader/InputReader.cpp。InputReader 主要功能是处理 EventHub 传过来的事件，然后加工，再分发给各个 InputDispatcher  
接下来看 InputReader.cpp 源码

### 3.1 InputReader.cpp 关于[鼠标事件](https://so.csdn.net/so/search?q=%E9%BC%A0%E6%A0%87%E4%BA%8B%E4%BB%B6&spm=1001.2101.3001.7020)源码分析

```
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    if(rawEvents->value == 0 || rawEvents->value == 1 || rawEvents->value == -1){ // 0:A and B type key up or A type touch or move end; -1: B type touch or move up
        ALOGD("processEventsLocked: type=%d Count=%zu code=%d value=%d deviceId=%d",
                rawEvents->type, count,rawEvents->code,rawEvents->value,rawEvents->deviceId);
    }
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
#if DEBUG_RAW_EVENTS
            ALOGD_READER("BatchSize: %zu Count: %zu", batchSize, count);
#endif
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            switch (rawEvent->type) {
            case EventHubInterface::DEVICE_ADDED:
                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                handleConfigurationChangedLocked(rawEvent->when);
                break;
            default:
                ALOG_ASSERT(false); // can't happen
                break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
void InputReader::addDeviceLocked(nsecs_t when, int32_t eventHubId) {
      if (mDevices.find(eventHubId) != mDevices.end()) {
          ALOGW("Ignoring spurious device added event for eventHubId %d.", eventHubId);
          return;
      }
  
      InputDeviceIdentifier identifier = mEventHub->getDeviceIdentifier(eventHubId);
      std::shared_ptr<InputDevice> device = createDeviceLocked(eventHubId, identifier);
      device->configure(when, &mConfig, 0);
      device->reset(when);
  
      if (device->isIgnored()) {
          ALOGI("Device added: id=%d, eventHubId=%d, 
                "(ignored non-input device)",
                device->getId(), eventHubId, identifier.name.c_str(), identifier.descriptor.c_str());
      } else {
          ALOGI("Device added: id=%d, eventHubId=%d, ,
                device->getId(), eventHubId, identifier.name.c_str(), identifier.descriptor.c_str(),
                device->getSources());
      }
  
      mDevices.emplace(eventHubId, device);
      bumpGenerationLocked();
  
      if (device->getClasses() & INPUT_DEVICE_CLASS_EXTERNAL_STYLUS) {
          notifyExternalStylusPresenceChanged();
      }
  }
  std::shared_ptr<InputDevice> InputReader::createDeviceLocked(
          int32_t eventHubId, const InputDeviceIdentifier& identifier) {
      auto deviceIt = std::find_if(mDevices.begin(), mDevices.end(), [identifier](auto& devicePair) {
          return devicePair.second->getDescriptor().size() && identifier.descriptor.size() &&
                  devicePair.second->getDescriptor() == identifier.descriptor;
      });
  
      std::shared_ptr<InputDevice> device;
      if (deviceIt != mDevices.end()) {
          device = deviceIt->second;
      } else {
          int32_t deviceId = (eventHubId < END_RESERVED_ID) ? eventHubId : nextInputDeviceIdLocked();
          device = std::make_shared<InputDevice>(&mContext, deviceId, bumpGenerationLocked(),
                                                 identifier);
      }
      device->addEventHubDevice(eventHubId);
      return device;
  }

```

processEventsLocked（）处理鼠标的输入事件  
switch (rawEvent->type) 根据不同状态处理 挂载 卸载输入设备事件 通过 addDeviceLocked 和 createDeviceLocked 来  
挂载输入设备鼠标

真正处理鼠标事件的就是在这里

```
uint32_t CursorButtonAccumulator::getButtonState() const {
    uint32_t result = 0;
    if (mBtnLeft) {
        result |= AMOTION_EVENT_BUTTON_PRIMARY;
    }
   // 鼠标右键处理
    if (mBtnRight) {
        result |= AMOTION_EVENT_BUTTON_SECONDARY;
    }
    if (mBtnMiddle) {
        result |= AMOTION_EVENT_BUTTON_TERTIARY;
    }
    if (mBtnBack || mBtnSide) {
        result |= AMOTION_EVENT_BUTTON_BACK;
    }
    if (mBtnForward || mBtnExtra) {
        result |= AMOTION_EVENT_BUTTON_FORWARD;
    }
    return result;
}

    

所以就可以在这里修改:

uint32_t CursorButtonAccumulator::getButtonState() const {
    uint32_t result = 0;
    if (mBtnLeft) {
        result |= AMOTION_EVENT_BUTTON_PRIMARY;
    }
   // 鼠标右键处理
    if (mBtnRight) {
       - result |= AMOTION_EVENT_BUTTON_SECONDARY;
      +  result |= AMOTION_EVENT_BUTTON_BACK;
    }
    if (mBtnMiddle) {
        result |= AMOTION_EVENT_BUTTON_TERTIARY;
    }
    if (mBtnBack || mBtnSide) {
        result |= AMOTION_EVENT_BUTTON_BACK;
    }
    if (mBtnForward || mBtnExtra) {
        result |= AMOTION_EVENT_BUTTON_FORWARD;
    }
    return result;
}

```

不过这样就写死了 如果想通过属性开关来控制的话也可以这样  
首先在 system.prop 中添加一个 prop 属性：persist.sys.right.back=false  
上层通过设置属性值来控制是否 back 事件  
所以可以修改成

```
uint32_t CursorButtonAccumulator::getButtonState() const {
    uint32_t result = 0;
    if (mBtnLeft) {
        result |= AMOTION_EVENT_BUTTON_PRIMARY;
    }
   // 鼠标右键处理
    if (mBtnRight) {
       - result |= AMOTION_EVENT_BUTTON_SECONDARY;
      +  char rightback[PROPERTY_VALUE_MAX]=“”；
       +     __system_property_get("persist.sys.right.back", rightback);
       +     if(strcmp(rightback,"true") == 0){
        +         result |= AMOTION_EVENT_BUTTON_BACK;
        +    }else{
         +        result |= AMOTION_EVENT_BUTTON_SECONDARY;
         +   }
    }
    if (mBtnMiddle) {
        result |= AMOTION_EVENT_BUTTON_TERTIARY;
    }
    if (mBtnBack || mBtnSide) {
        result |= AMOTION_EVENT_BUTTON_BACK;
    }
    if (mBtnForward || mBtnExtra) {
        result |= AMOTION_EVENT_BUTTON_FORWARD;
    }
    return result;
}

```