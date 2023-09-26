> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828224)

### 1. 概述

在 11.0 在定制化系统中，默认是没有开机铃声的，有客户提出需要要添加开机铃声，所以为了  
完成需求，就来实现这一个功能  
关于开机铃声 都是在 BootAnimation.cpp 这里面负责管理

### 2. 添加添加开机铃声的核心类

```
frameworks\base\cmds\bootanimation\BootAnimation.h
frameworks\base\cmds\bootanimation\BootAnimation.cpp

```

### 3. 添加开机铃声的核心功能分析和实现

### 3.1 BootAnimation.h 开机动画处理分析

```
class BootAnimation : public Thread, public IBinder::DeathRecipient
{
....
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
    bool preloadAnimation();

    void checkExit();

    void handleViewport(nsecs_t timestep);
...

//add core start
    bool soundplay();
    bool soundstop();
    bool playSoundsAllowed();
    String8     mSoundFileName;
    sp<MediaPlayer> mp;
    int         mfd;
    bool        mSystemCalled;
    bool        mWaitForComplete;
//add core end
};



```

在 BootAnimation.h 中的上述方法中，添加 soundplay(); 和 soundstop(); 这两个方法作为开始播放和停止播放开机铃声的方法,  
而 sp mp 就是播放类， String8 mSoundFileName; 获取播放开机音乐的路径, 在 playSoundsAllowed(); 中  
判断是否允许播放开机音乐的判断，增加这些变量和方法后，需要在 BootAnimation.cpp 中实现这些方法具体实现如下  
从代码中可以看到 是由 BootAnimation 来处理 开机铃声  
接下来分析下 BootAnimation.cpp 的相关源码  
frameworks\base\cmds\bootanimation\BootAnimation.cpp

```
//add core start
static const char OEM_BOOTSOUND_FILE[] = "/oem/media/bootsound.mp3";
static const char PRODUCT_BOOTSOUND_FILE[] = "/product/media/bootsound.mp3";
static const char SYSTEM_BOOTSOUND_FILE[] = "/system/media/bootsound.mp3";
static const char PLAY_SOUND_PROP_NAME[] = "persist.sys.bootanim.play_sound";
static const char BOOTREASON_PROP_NAME[] = "ro.boot.bootreason";
static const std::vector<std::string> PLAY_SOUND_BOOTREASON_BLACKLIST {
  "kernel_panic",
  "Panic",
  "Watchdog",
};

class MPlayerListener : public MediaPlayerListener
{
    void notify(int msg, int /*ext1*/, int /*ext2*/, const Parcel * /*obj*/)
    {
        switch (msg) {
        case MEDIA_PLAYBACK_COMPLETE:
            ALOGD("shutdown media finished, quit");
            if (property_set("service.bootanim.end", "1") != 0) {
                ALOGD("set property failed!");
            }
            break;
        default:
            break;
        }
    }
};

bool BootAnimation::soundplay() {
    mp = NULL;
    if (mSoundFileName.length() == 0) {
        __android_log_print(ANDROID_LOG_WARN, LOG_TAG, "no bootanimation sound resource....");
        return false;
    }
    mfd = open(mSoundFileName.string(), O_RDONLY);
    if(mfd == -1){
        __android_log_print(ANDROID_LOG_WARN, LOG_TAG, "can not find bootanimation sound resource....");
        return false;
    }

    mp = new MediaPlayer();
    SLOGE("new MediaPlayer");
    if (mShuttingDown && mWaitForComplete) {
        sp<MPlayerListener> mListener = new MPlayerListener();
        mp->setListener(mListener);
    }
    mp->setDataSource(mfd, 0, 0x7ffffffffffffffLL);
    mp->setAudioStreamType(/*AUDIO_STREAM_MUSIC*/AUDIO_STREAM_SYSTEM);
    mp->prepare();
    ALOGD("%sAnimation start MediaPlayer,play sound time: %" PRId64 "ms",
        mShuttingDown ? "Shutdown" : "Boot", elapsedRealtime());
    mp->start();
    return false;
}

bool BootAnimation::soundstop()
{
    if (mSoundFileName.length() == 0) {
        __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, "no sound resource ");
        return false;
    }

    ALOGD("%sAnimation stop MediaPlayer time: %" PRId64 "ms",
        mShuttingDown ? "Shutdown" : "Boot", elapsedRealtime());
    if (mp != NULL)mp->stop();
    return false;
}

bool BootAnimation::playSoundsAllowed() {
    // Compatible with Android Things
    // only system called will play sounds in system/media/
    if (!mSystemCalled) {
        return false;
    }

    // Read the system property to see if we should play the sound.
    // If it's not present, default to allowed.
    if (!property_get_bool(PLAY_SOUND_PROP_NAME, 1)) {
        return false;
    }

    // Don't play sounds if this is a reboot due to an error.
    char bootreason[PROPERTY_VALUE_MAX];
    if (property_get(BOOTREASON_PROP_NAME, bootreason, nullptr) > 0) {
        for (const auto& str : PLAY_SOUND_BOOTREASON_BLACKLIST) {
            if (strcasecmp(str.c_str(), bootreason) == 0) {
                return false;
            }
        }
    }
    return true;
}

bool BootAnimation::parseAnimationDesc(Animation& animation)
{
    String8 desString;

    if (!readFile(animation.zip, "desc.txt", desString)) {
        return false;
    }
    char const* s = desString.string();

// add core start
    ALOGE("before allowed, soundplay");
    if (playSoundsAllowed()) {
        ALOGE("soundplay called");
        soundplay();
    }
// add core end
...
}
//add core end

```

在 BootAnimation.cpp 中的上述方法和参数变量中，都是新增的关于播放开机铃声的相关方法和变量，  
在 soundplay() 中，主要是获取开机铃声的地址，然后播放开机铃声，而在 soundstop() 中，就是在播放开机动画结束后，  
停止对开机铃声的播放，而 playSoundsAllowed() 判断是否允许播放开机铃声，在 parseAnimationDesc([Animation](https://so.csdn.net/so/search?q=Animation&spm=1001.2101.3001.7020)& animation)  
在解析开机动画的时候，开始播放开机铃声, 接下来分析下其他开机铃声的相关方法

```
void BootAnimation::findBootAnimationFile() {
    // If the device has encryption turned on or is in process
    // of being encrypted we show the encrypted boot animation.
    char decrypt[PROPERTY_VALUE_MAX];
    property_get("vold.decrypt", decrypt, "");

    bool encryptedAnimation = atoi(decrypt) != 0 ||
        !strcmp("trigger_restart_min_framework", decrypt);

    if (!mShuttingDown && encryptedAnimation) {
        static const char* encryptedBootFiles[] =
            {PRODUCT_ENCRYPTED_BOOTANIMATION_FILE, SYSTEM_ENCRYPTED_BOOTANIMATION_FILE};
        for (const char* f : encryptedBootFiles) {
            if (access(f, R_OK) == 0) {
                mZipFileName = f;
                return;
            }
        }
    }

    const bool playDarkAnim = android::base::GetIntProperty("ro.boot.theme", 0) == 1;
// add core start
    static const char* bootSoundFiles[] =
        {PRODUCT_BOOTSOUND_FILE, OEM_BOOTSOUND_FILE, SYSTEM_BOOTSOUND_FILE};

    for (const char* f : bootFiles) {
        if (access(f, R_OK) == 0) {
            mZipFileName = f;
            return;
        }
    }
// add core end
}

BootAnimation::BootAnimation(sp<Callbacks> callbacks)

        : Thread(false), mClockEnabled(true), mTimeIsAccurate(false),
        mTimeFormat12Hour(false), mTimeCheckThread(nullptr), mCallbacks(callbacks) {
//add core start
    ALOGD("System Called %d ", systemCalled);
    mSystemCalled = true;
    mfd = -1;
    if (!mShuttingDown) {
        mShuttingDown = android::base::GetProperty("service.bootanim.shutdown", "").compare("1") == 0;
    }
    if (mShuttingDown) {
        mWaitForComplete = android::base::GetBoolProperty("service.wait_for_bootanim", false);
    }
//add core end
}

BootAnimation::~BootAnimation() {
//add core start
    if(mfd != -1){ close(mfd); }
//add core end
}


bool BootAnimation::playAnimation(const Animation& animation)
{
    const size_t pcount = animation.parts.size();
    nsecs_t frameDuration = s2ns(1) / animation.fps;
    const int animationX = (mWidth - animation.width) / 2;
    const int animationY = (mHeight - animation.height) / 2;
....
    // Free textures created for looping parts now that the animation is done.
    for (const Animation::Part& part : animation.parts) {
        if (part.count != 1) {
            const size_t fcount = part.frames.size();
            for (size_t j = 0; j < fcount; j++) {
                const Animation::Frame& frame(part.frames[j]);
                glDeleteTextures(1, &frame.tid);
            }
        }
    }

//add core start
    // SPRD: stop soundplay
    if (playSoundsAllowed()) {
        soundstop();
    }
//add core end
    return true;
}

```

在 BootAnimation.cpp 中的上述方法中，在 findBootAnimationFile() 中，初始化获取到的开机铃声的地址，  
BootAnimation(sp [callbacks](https://so.csdn.net/so/search?q=callbacks&spm=1001.2101.3001.7020)) 构造方法中，初始化相关参数，在~ BootAnimation() 中释放开机铃声  
播放器的相关参数，playAnimation(const Animation& animation) 中就是在播放完开机动画后，开始  
停止开机铃声的播放，以上方法和参数 就是实现整个开机铃声的相关方法，  
开机铃声添加路径为：  
路径：device/sprd/common/customer/system/media/bootsound.mp3  
和开机动画放在同一个目录下就可以了