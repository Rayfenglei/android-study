> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124459519)

### 1. 概述

在 9.0 10.0 11.0 的产品定制开发中，在开机流程中，是在开机 kenel 部分都是播放的开机 log, 等 kenel 启动完成后进入系统后这时播放的是开机动画，由于开发需要要求开机动画换成支持 mp4 视频，可以播放自定义的开机视频来替代开机动画功能  
这就需要从开机动画相关的类中定制了用来播放开机视频不用播放开机动画了

### 2.9.0 10.0 11.0 开机动画支持 mp4 视频播放核心类

```
frameworks/base/cmds/bootanimation/BootAnimation.h
frameworks/base/cmds/bootanimation/BootAnimation.cpp

```

### 3.9.0 10.0 11.0 开机动画支持 mp4 视频播放的核心功能分析和实现

实现功能的准备工作  
1. 准备 bootvideo.mp4 视频文件 编译到 / system/media 目录下 和开机动画放在一个目录下  
2. 现在就要从开机 BootAnimation.cpp 中看怎么完成需求

### 3.1 BootAnimation.h 中增加播放开机视频的方法

```
    private:
      virtual bool        threadLoop();
      virtual status_t    readyToRun();
      virtual void        onFirstRef();
      virtual void        binderDied(const wp<IBinder>& who);
  
      bool                updateIsTimeAccurate();
  
      class TimeCheckThread : public Thread {
      public:
          explicit TimeCheckThread(BootAnimation* bootAnimation);
          virtual ~TimeCheckThread();
      private:
          virtual status_t    readyToRun();
          virtual bool        threadLoop();
          bool                doThreadLoop();
          void                addTimeDirWatch();
  
          int mInotifyFd;
          int mSystemWd;
          int mTimeWd;
          BootAnimation* mBootAnimation;
      };
  
      // Display event handling
      class DisplayEventCallback;
      int displayEventCallback(int fd, int events, void* data);
      void processDisplayEvents();
  
      status_t initTexture(Texture* texture, AssetManager& asset, const char* name);
      status_t initTexture(FileMap* map, int* width, int* height);
      status_t initFont(Font* font, const char* fallback);
      bool android();
      bool movie();
      void drawText(const char* str, const Font& font, bool bold, int* x, int* y);
      void drawClock(const Font& font, const int xPos, const int yPos);
      bool validClock(const Animation::Part& part);
      Animation* loadAnimation(const String8&);
      bool playAnimation(const Animation&);
      void releaseAnimation(Animation*) const;
      bool parseAnimationDesc(Animation&);
      bool preloadZip(Animation &animation);
      void findBootAnimationFile();
      bool findBootAnimationFileInternal(const std::vector<std::string>& files);
      bool preloadAnimation();
      + bool video();
      + bool mVideo;

  };
  

```

增加 bool video() 和 bool mVideo; 这两个关于播放开机视频的方法和变量来  
作为开机视频播放的相关方法  
接下来看 BootAnimation.cpp 关于视频播放的定义

### 3.2BootAnimation.cpp 关于开机视频功能的增加

```
status_t BootAnimation::readyToRun() {
...
// 判断mp4视频是否存在
const char* bootvideofile = "/system/media/bootvideo.mp4";
if(access(bootvideofile, R_OK) == 0)
mVideo = true;
else
mVideo = false;
...
}
bool BootAnimation::threadLoop(){
...
// 如果视频存在 就播放视频 不播放动画
#if 0
if (mZipFileName.isEmpty()) {
r = android();
} else {
r = movie();
}
#else
if (!mVideo) {
r = android();
} else {
r = video();
}
#endif
...
}

```

在 readyToRun() 中定义播放开机视频的路径，而在 threadLoop() 中判断讲开机动画换成播放开机视频的相关方法中  
实现对开机视频的播放  
增加播放视频的方法

```
bool BootAnimation::video(){
eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
eglDestroySurface(mDisplay, mSurface);
const char* boot_videofile = "/system/media/bootvideo.mp4";
int fd = open(boot_videofile, O_RDONLY);
sp<MediaPlayer> mediaplayer = new MediaPlayer();
mediaplayer->reset();
mediaplayer->setDataSource(fd, 0, 0x7ffffffffffffffLL);
mediaplayer->setLooping(false);
mediaplayer->setVideoSurfaceTexture(mFlingerSurface->getIGraphicBufferProducer());
mediaplayer->prepare();
mediaplayer->start();
while(true) {
    if(exitPending())
        break;
    usleep(100);
    if(!mediaplayer->isPlaying())
        checkExit();
}

mediaplayer->stop();
mediaplayer->release();
return false;

}

```

就这样实现了开机视频的播放，替换 开机动画的播放功能  
在 BootAnimation.h 中增加 video() 和 mVideo  
最后补丁为:

```
diff --git a/frameworks/base/cmds/bootanimation/BootAnimation.h b/frameworks/base/cmds/bootanimation/BootAnimation.h
old mode 100644
new mode 100755
index 8d82f01ed3..3fbf737904
--- a/frameworks/base/cmds/bootanimation/BootAnimation.h
+++ b/frameworks/base/cmds/bootanimation/BootAnimation.h
@@ -164,7 +164,8 @@ private:
bool preloadZip(Animation &animation);
void findBootAnimationFile();
bool preloadAnimation();
bool video();
bool mVideo;void checkExit();void handleViewport(nsecs_t timestep);
diff --git a/frameworks/base/cmds/bootanimation/BootAnimation.cpp b/frameworks/base/cmds/bootanimation/BootAnimation.cpp
index 3adb349a8d..2c023cd6b8 100755
--- a/frameworks/base/cmds/bootanimation/BootAnimation.cpp
+++ b/frameworks/base/cmds/bootanimation/BootAnimation.cpp
@@ -65,7 +65,7 @@
#define ALOGD_ANIMATION(...) if (gAnimationLog) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, VA_ARGS)
#define DEFAULT_SUPPORT_RESOLUTION_NUM 1
namespace android {
static const char OEM_BOOTANIMATION_FILE[] = "/oem/media/bootanimation.zip";
@@ -94,6 +94,7 @@ static const char SYSTEM_BOOTSOUND_FILE[] = "/system/media/bootsound.mp3";
static const char OEM_SHUTDOWNSOUND_FILE[] = "/oem/media/shutdownsound.mp3";
static const char PRODUCT_SHUTDOWNSOUND_FILE[] = "/product/media/shutdownsound.mp3";
static const char SYSTEM_SHUTDOWNSOUND_FILE[] = "/system/media/shutdownsound.mp3";
#endif
static const char SYSTEM_DATA_DIR_PATH[] = "/data/system";
@@ -390,7 +391,11 @@ status_t BootAnimation::readyToRun() {
mFlingerSurfaceControl = control;
mFlingerSurface = s;
mTargetInset = -1;
  const char* bootvideofile = "/system/media/bootvideo.mp4";

if(access(bootvideofile, R_OK) == 0)
   mVideo = true;

else
   mVideo = false;
SLOGD("Get Surface mWidth is %d,mHeight is %d",mWidth,mHeight);
return NO_ERROR;
}
@@ -467,12 +472,19 @@ bool BootAnimation::threadLoop()
bool r;
// We have no bootanimation file, so we use the stock android logo
// animation.
if (mZipFileName.isEmpty()) {
   r = android();

} else {
   r = movie();

}
  #if 0

          if (mZipFileName.isEmpty()) {

                  r = android();

          } else {

                  r = movie();

          }

#else
          if (!mVideo) {

             r = android();

          } else {

             r = video();

          }

  #endif
eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
eglDestroyContext(mDisplay, mContext);
eglDestroySurface(mDisplay, mSurface);
@@ -484,6 +496,32 @@ bool BootAnimation::threadLoop()
return r;
}
  
+bool BootAnimation::video(){
eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
eglDestroySurface(mDisplay, mSurface);
  const char* boot_videofile = "/system/media/bootvideo.mp4";

int fd = open(boot_videofile, O_RDONLY);
sp<MediaPlayer> mediaplayer = new MediaPlayer();
mediaplayer->reset();
mediaplayer->setDataSource(fd, 0, 0x7ffffffffffffffLL);
mediaplayer->setLooping(false);
mediaplayer->setVideoSurfaceTexture(mFlingerSurface->getIGraphicBufferProducer());
mediaplayer->prepare();
mediaplayer->start();
while(true) {
   if(exitPending())

       break;

   usleep(100);

   if(!mediaplayer->isPlaying())

       checkExit();

}
mediaplayer->stop();
  mediaplayer->release();

return false;
+}
+
bool BootAnimation::android()
{
SLOGD("%sAnimationShownTiming start time: %" PRId64 "ms", mShuttingDown ? "Shutdown" : "Boot",

```