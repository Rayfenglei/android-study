> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132951872)

1. 前言
-----

在 11.0 的[产品开发](https://so.csdn.net/so/search?q=%E4%BA%A7%E5%93%81%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，在对于一些产品开发需求中，在一些教育产品中，对系统截屏和录屏功能 要求去掉这些功能，不让用户截屏和录屏 保护  
一个 app 的资源，所以就需要在系统中做限制不让截屏录屏

2. 系统 [framework](https://so.csdn.net/so/search?q=framework&spm=1001.2101.3001.7020) 禁用截屏和录屏功能的核心类
--------------------------------------------------------------------------------------------------

```
    frameworks\native\services\surfaceflinger\Layer.cpp
    frameworks\base\core\java\android\app\ActivityThread.java
```

3. 系统 framework 禁用截屏和录屏功能的核心功能分析和实现
-----------------------------------

ActivityThread 是一个非常重要的组件, 它的作用就像是 Android 应用程序的灵魂, 它处理着应用程序和活动中的大部分工作, 例如创建新的应用程序实例、加载和管理类、创建新的活动  
ActivityThread 是 [Android 应用](https://so.csdn.net/so/search?q=Android%E5%BA%94%E7%94%A8&spm=1001.2101.3001.7020)的主线程 (UI 线程), 说起 ActivityThread, 不得不提到 Activity 的创建、启动过程以及 ActivityManagerService

 在系统中可以在 app 中禁用录屏和截屏功能，同时也可以通过在系统源码中禁止截屏和录屏的功能，首先我们看下如何在 app 应用中禁止截屏录屏的的功能，app 中禁止录屏和截屏功能的相关源码如下

            @Override  
            protected void onCreate(Bundle savedInstanceState) {  
           
                getWindow().addFlags(WindowManager.LayoutParams.FLAG_SECURE);  
           
                super.onCreate(savedInstanceState);  
                setContentView(R.layout.activity_layout);  
            }  
系统 framework 禁用截屏和录屏功能的实现中，  
通过在单个应用中禁止截屏录屏功能，系统提供了对应的接口，如上代码添加 FLAG_SECURE 即可。  
设置后，在此 activity 界面，截图时会提示无法截图，录屏时界面是全黑的。通过上面的实现方法  
可以得知就是给 activity 的窗口的 window 窗口添加 WindowManager.LayoutParams.FLAG_SECURE  
属性就可以了，所以可以在 ActivityThread.java 启动 activity 的时候添加 flags 属性就可以了

3.1 ActivityThread.java 相关源码分析
------------------------------

  
ActivityThread 它管理应用进程的主线程的执行 (相当于普通 Java 程序的 main 入口函数)，并根据 AMS 的要求（通过 IApplicationThread 接口，AMS 为 Client、ActivityThread.ApplicationThread 为 Server）负责调度和执行 activities、broadcasts 和其它操作  
系统 framework 禁用截屏和录屏功能的实现中，  
在上述的分析中，可以得知，在系统 app 应用开发中，可以通过在 oncreate 中启动 activity 中，设置 window 窗口的 flag 属性来禁用录屏截屏功能，就是设置 WindowManager.LayoutParams.FLAG_SECURE 这个属性，所以也同时可以在 ActivityThread.java 中, 构建 activity 窗口的时候设置这个 flag 属性来实现功能

```
        @Override
        public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                String reason) {
            // If we are getting ready to gc after going to the background, well
            // we are back active so skip it.
            unscheduleGcIdler();
            mSomeActivitiesChanged = true;
     
            // TODO Push resumeArgs into the activity for consideration
            final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
            if (r == null) {
                // We didn't actually resume the activity, so skipping any follow-up actions.
                return;
            }
            if (mActivitiesToBeDestroyed.containsKey(token)) {
                // Although the activity is resumed, it is going to be destroyed. So the following
                // UI operations are unnecessary and also prevents exception because its token may
                // be gone that window manager cannot recognize it. All necessary cleanup actions
                // performed below will be done while handling destruction.
                return;
            }
     
            final Activity a = r.activity;
            clearDisplayCacheIfNeeded(r);//add code
            if (localLOGV) {
                Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                        + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
            }
     
            final int forwardBit = isForward
                    ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
     
            // If the window hasn't yet been added to the window manager,
            // and this guy didn't finish itself or start another activity,
            // then go ahead and add the window.
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
     
    //add core start
                ComponentName comptname = r.activity.getComponentName();
    			if(comptname!=null){
    			 String pkg = comptname.getPackageName();
    			 String cls = comptname.getClassName();
    			 android.util.Log.e("ActivityThread","pkg:"+pkg+"--cls:"+cls);
    			 if(!TextUtils.isEmpty(pkg)&&!pkg.equals("android")&&!pkg.equals("com.android.launcher3")){
    			    l.flags = WindowManager.LayoutParams.FLAG_SECURE;
    			 }
    			}
     
    //add core end 
     
     
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient) {
                    if (!a.mWindowAdded) {
                        a.mWindowAdded = true;
                        wm.addView(decor, l);
                    } else {
                        // The activity will get a callback for this {@link LayoutParams} change
                        // earlier. However, at that time the decor will not be set (this is set
                        // in this method), so no action will be taken. This call ensures the
                        // callback occurs with the decor set.
                        a.onWindowAttributesChanged(l);
                    }
                }
     
                // If the window has already been added, but during resume
                // we started another activity, then don't yet make the
                // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
     
            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r, false /* force */);
    ...
    }
```

系统 framework 禁用截屏和录屏功能的实现中，  
在上述 ActivityThread.java 的源码中，onCreate 回调完成后就开始准备显示页面然后回调 onResume。这个操作从 handleResumeActivity 方法开始。  
可以在 handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,  
            String reason) 中这个构建 activity 的方法中，在绑定 PhoneWindow 的窗口的时候  
添加 flags 属性 WindowManager.LayoutParams.FLAG_SECURE; 通过上述的这些修改  
这样就可以禁用截屏功能了

3.2 Layer.cpp 中实现禁用录屏功能
-----------------------

  
系统 framework 禁用截屏和录屏功能的实现中，在关于对系统录屏的功能源码分析中，得知，在  
Layer.cpp 中实现查询是否 SECURE 模式，然后判断是否可以录屏，所以接下来具体分析下相关源码

```
 Hwc2::IComposerClient::Composition Layer::getCompositionType(const DisplayDevice& display) const {
        const auto outputLayer = findOutputLayerForDisplay(&display);
        if (outputLayer == nullptr) {
            return Hwc2::IComposerClient::Composition::INVALID;
        }
        if (outputLayer->getState().hwc) {
            return (*outputLayer->getState().hwc).hwcCompositionType;
        } else {
            return Hwc2::IComposerClient::Composition::CLIENT;
        }
    }
     
    bool Layer::addSyncPoint(const std::shared_ptr<SyncPoint>& point) {
        if (point->getFrameNumber() <= mCurrentFrameNumber) {
            // Don't bother with a SyncPoint, since we've already latched the
            // relevant frame
            return false;
        }
        if (isRemovedFromCurrentState()) {
            return false;
        }
     
        Mutex::Autolock lock(mLocalSyncPointMutex);
        mLocalSyncPoints.push_back(point);
        return true;
    }
     
    // ----------------------------------------------------------------------------
    // local state
    // ----------------------------------------------------------------------------
     
    bool Layer::isSecure() const {
        const State& s(mDrawingState);
        return (s.flags & layer_state_t::eLayerSecure);
    }
```

系统 framework 禁用截屏和录屏功能的实现中，  
在 Layer.cpp 中的上述源码中，在通过分析得知，在这里构建视图的时候，会首先通过在这里处理录屏的方法中，主要是根据 isSecure() 来判断  
当前是否是 SECURE 模式，如果是 SECURE 模式就不能录屏，所以可以修改这个 isSecure() 的返回值，直接返回 false 就表示禁止录屏，所以可以具体修改为:

    bool Layer::isSecure() const {  
        /*const State& s(mDrawingState);*/  
        return true/*(s.flags & layer_state_t::eLayerSecure)*/;  
    }  
系统 framework 禁用截屏和录屏功能的实现中，  
在 Layer.cpp 中的上述源码中，通过对 Layer::isSecure() 的返回值返回 true，就表示当前禁用录屏功能，从而实现了这个功能