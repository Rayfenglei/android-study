> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125549113)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 开机动画横屏显示的核心代码部分](#t1)

[3. 开机动画横屏显示的核心代码部分分析以及实现功能](#t2)

[3.1BootAnimation.cpp 的代码分析](#t3)

[3.2 分析源码](#t4)

1. 概述
=====

在 11.0 的原生系统中，默认的都是竖屏显示的，由于要定制化一些横屏显示的产品，所以要对开机动画显示要做一些旋转才能满足需求，本篇主要讲解开机动画横屏显示所要作的定制

2. 开机动画横屏显示的核心代码部分
==================

```
核心代码部分:
/frameworks/base/cmds/bootanimation/BootAnimation.cpp
```

3. 开机动画横屏显示的核心代码部分分析以及实现功能
==========================

### 3.1BootAnimation.cpp 的代码分析

核心代码部分就是在 status_t BootAnimation::readyToRun() 中  
在运行开机动画的时候会在这里绘制每一帧动画来显示

```
 
 
status_t BootAnimation::readyToRun() {
mAssets.addDefaultAssets();
 
mDisplayToken = SurfaceComposerClient::getInternalDisplayToken();
if (mDisplayToken == nullptr)
return NAME_NOT_FOUND;
 
DisplayConfig displayConfig;
const status_t error =
SurfaceComposerClient::getActiveDisplayConfig(mDisplayToken, &displayConfig);
if (error != NO_ERROR)
return error;
 
mMaxWidth = android::base::GetIntProperty("ro.surface_flinger.max_graphics_width", 0);
mMaxHeight = android::base::GetIntProperty("ro.surface_flinger.max_graphics_height", 0);
ui::Size resolution = displayConfig.resolution;
resolution = limitSurfaceSize(resolution.width, resolution.height);
// create the native surface
sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
resolution.getWidth(), resolution.getHeight(), PIXEL_FORMAT_RGB_565);
 
SurfaceComposerClient::Transaction t;
 
// this guest property specifies multi-display IDs to show the boot animation
// multiple ids can be set with comma (,) as separator, for example:
// setprop persist.boot.animation.displays 19260422155234049,19261083906282754
Vector<uint64_t> physicalDisplayIds;
char displayValue[PROPERTY_VALUE_MAX] = "";
property_get(DISPLAYS_PROP_NAME, displayValue, "");
bool isValid = displayValue[0] != '\0';
if (isValid) {
char *p = displayValue;
while (*p) {
if (!isdigit(*p) && *p != ',') {
isValid = false;
break;
}
p ++;
}
if (!isValid)
SLOGE("Invalid syntax for the value of system prop: %s", DISPLAYS_PROP_NAME);
}
if (isValid) {
std::istringstream stream(displayValue);
for (PhysicalDisplayId id; stream >> id; ) {
physicalDisplayIds.add(id);
if (stream.peek() == ',')
stream.ignore();
}
 
// In the case of multi-display, boot animation shows on the specified displays
// in addition to the primary display
auto ids = SurfaceComposerClient::getPhysicalDisplayIds();
constexpr uint32_t LAYER_STACK = 0;
for (auto id : physicalDisplayIds) {
if (std::find(ids.begin(), ids.end(), id) != ids.end()) {
sp<IBinder> token = SurfaceComposerClient::getPhysicalDisplayToken(id);
if (token != nullptr)
t.setDisplayLayerStack(token, LAYER_STACK);
}
}
t.setLayerStack(control, LAYER_STACK);
}
 
t.setLayer(control, 0x40000000)
apply();
 
sp<Surface> s = control->getSurface();
 
// initialize opengl and egl
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
eglInitialize(display, nullptr, nullptr);
EGLConfig config = getEglConfig(display);
EGLSurface surface = eglCreateWindowSurface(display, config, s.get(), nullptr);
EGLContext context = eglCreateContext(display, config, nullptr, nullptr);
EGLint w, h;
eglQuerySurface(display, surface, EGL_WIDTH, &w);
eglQuerySurface(display, surface, EGL_HEIGHT, &h);
 
if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)
return NO_INIT;
 
mDisplay = display;
mContext = context;
mSurface = surface;
mWidth = w;
mHeight = h;
mFlingerSurfaceControl = control;
mFlingerSurface = s;
mTargetInset = -1;
 
// Register a display event receiver
mDisplayEventReceiver = std::make_unique<DisplayEventReceiver>();
status_t status = mDisplayEventReceiver->initCheck();
SLOGE_IF(status != NO_ERROR, "Initialization of DisplayEventReceiver failed with status: %d",
status);
mLooper->addFd(mDisplayEventReceiver->getFd(), 0, Looper::EVENT_INPUT,
new DisplayEventCallback(this), nullptr);
 
return NO_ERROR;
}
```

### 3.2 分析源码

在进入 readyToRun() 中 就来加载显示每一帧动画 readyToRun() 负责构建开机动画 createSurface() 负责绘制开机动画页面 sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"), resolution.getWidth(), resolution.getHeight(), PIXEL_FORMAT_RGB_565); 就是每一帧动画的宽高等参数

3.3 适配开机动画横屏的修改部分

```
status_t BootAnimation::readyToRun() {
mAssets.addDefaultAssets();
 
mDisplayToken = SurfaceComposerClient::getInternalDisplayToken();
if (mDisplayToken == nullptr)
return NAME_NOT_FOUND;
 
DisplayConfig displayConfig;
const status_t error =
SurfaceComposerClient::getActiveDisplayConfig(mDisplayToken, &displayConfig);
if (error != NO_ERROR)
return error;
 
mMaxWidth = android::base::GetIntProperty("ro.surface_flinger.max_graphics_width", 0);
mMaxHeight = android::base::GetIntProperty("ro.surface_flinger.max_graphics_height", 0);
ui::Size resolution = displayConfig.resolution;
resolution = limitSurfaceSize(resolution.width, resolution.height);
 
 
// create the native surface 
// 重点修改部分
- sp<SurfaceControl> control = session()->createSurface(String8("BootAnimation"),
- resolution.getWidth(), resolution.getHeight(), PIXEL_FORMAT_RGB_565);
+ sp<SurfaceControl> control;
+ if(resolution.getWidth() < resolution.getHeight())
+     control = session()->createSurface(String8("BootAnimation"),
+         resolution.getHeight(), resolution.getWidth(), PIXEL_FORMAT_RGB_565);
+ else
+ control = session()->createSurface(String8("BootAnimation"),
+         resolution.getWidth(), resolution.getHeight(), PIXEL_FORMAT_RGB_565);
//这样就会根据横竖屏显示横竖屏开机动画
 
SurfaceComposerClient::Transaction t;
 
// this guest property specifies multi-display IDs to show the boot animation
// multiple ids can be set with comma (,) as separator, for example:
// setprop persist.boot.animation.displays 19260422155234049,19261083906282754
Vector<uint64_t> physicalDisplayIds;
char displayValue[PROPERTY_VALUE_MAX] = "";
property_get(DISPLAYS_PROP_NAME, displayValue, "");
bool isValid = displayValue[0] != '\0';
if (isValid) {
char *p = displayValue;
while (*p) {
if (!isdigit(*p) && *p != ',') {
isValid = false;
break;
}
p ++;
}
if (!isValid)
SLOGE("Invalid syntax for the value of system prop: %s", DISPLAYS_PROP_NAME);
}
if (isValid) {
std::istringstream stream(displayValue);
for (PhysicalDisplayId id; stream >> id; ) {
physicalDisplayIds.add(id);
if (stream.peek() == ',')
stream.ignore();
}
 
// In the case of multi-display, boot animation shows on the specified displays
// in addition to the primary display
auto ids = SurfaceComposerClient::getPhysicalDisplayIds();
constexpr uint32_t LAYER_STACK = 0;
for (auto id : physicalDisplayIds) {
if (std::find(ids.begin(), ids.end(), id) != ids.end()) {
sp<IBinder> token = SurfaceComposerClient::getPhysicalDisplayToken(id);
if (token != nullptr)
t.setDisplayLayerStack(token, LAYER_STACK);
}
}
t.setLayerStack(control, LAYER_STACK);
}
 
t.setLayer(control, 0x40000000)
apply();
 
sp<Surface> s = control->getSurface();
 
// initialize opengl and egl
EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
eglInitialize(display, nullptr, nullptr);
EGLConfig config = getEglConfig(display);
EGLSurface surface = eglCreateWindowSurface(display, config, s.get(), nullptr);
EGLContext context = eglCreateContext(display, config, nullptr, nullptr);
EGLint w, h;
eglQuerySurface(display, surface, EGL_WIDTH, &w);
eglQuerySurface(display, surface, EGL_HEIGHT, &h);
 
if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)
return NO_INIT;
 
mDisplay = display;
mContext = context;
mSurface = surface;
mWidth = w;
mHeight = h;
mFlingerSurfaceControl = control;
mFlingerSurface = s;
mTargetInset = -1;
 
// Register a display event receiver
mDisplayEventReceiver = std::make_unique<DisplayEventReceiver>();
status_t status = mDisplayEventReceiver->initCheck();
SLOGE_IF(status != NO_ERROR, "Initialization of DisplayEventReceiver failed with status: %d",
status);
mLooper->addFd(mDisplayEventReceiver->getFd(), 0, Looper::EVENT_INPUT,
new DisplayEventCallback(this), nullptr);
 
return NO_ERROR;
}
 
 
```

第二种方法    
也可以不用修改代码  这需要将 每一帧的开机动画旋转 90 度也可以 仁者见仁智者见智 两种方式都可以实现功能