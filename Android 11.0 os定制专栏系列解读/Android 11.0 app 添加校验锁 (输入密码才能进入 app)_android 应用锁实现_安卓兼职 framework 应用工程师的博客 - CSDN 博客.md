> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774627)

### 1. 概述

11.0 定制化开发中，由功能需求要求给 app 添加锁，就是点击 app 图标时，会弹出 Dialog, 需要输入密码才能进入 app 中, 就是应用校验锁，这样就能通过密码限制访问某些 app 了  
最开始想到在 Launcher3 中实现，但是如果更换了默认 Launcher 又不行了 所以最终还是得拦截 app 启动 Activity 从这里入手了

### 2.app 添加校验锁 (输入密码才能进入 app) 的核心类

```
/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

```

### 3.app 添加校验锁 (输入密码才能进入 app) 核心功能分析和实现

Launcher 与 APP 是在两个不同的进程中，他们之间的通信是通过 [Binder](https://so.csdn.net/so/search?q=Binder&spm=1001.2101.3001.7020) 完成的，点击 Launcher 上的某个 APP，这时会调用 Launcher 的 startActivitySafely 方法。

### 3.1Launcher 关于启动 app 的方法调用

```
     @Override
     public boolean startActivitySafely(View v, Intent intent, ItemInfo item,
             @Nullable String sourceContainer) {
         if (!hasBeenResumed()) {
             // Workaround an issue where the WM launch animation is clobbered when finishing the
             // recents animation into launcher. Defer launching the activity until Launcher is
             // next resumed.
             addOnResumeCallback(() -> startActivitySafely(v, intent, item, sourceContainer));
             if (mOnDeferredActivityLaunchCallback != null) {
                 mOnDeferredActivityLaunchCallback.run();
                 mOnDeferredActivityLaunchCallback = null;
             }
             return true;
         }
 
         boolean success = super.startActivitySafely(v, intent, item, sourceContainer);
         if (success && v instanceof BubbleTextView) {
             // This is set to the view that launched the activity that navigated the user away
             // from launcher. Since there is no callback for when the activity has finished
             // launching, enable the press state and keep this reference to reset the press
             // state when we return to launcher.
             BubbleTextView btv = (BubbleTextView) v;
             btv.setStayPressed(true);
             addOnResumeCallback(btv);
         }
         return success;
     }

```

上面代码我省略了一些不太重要的，重点看上面两行代码，第一行给 intent 设置 Flag 为 Intent.FLAG_ACTIVITY_NEW_TASK，Activity 会在新的任务栈中启动，第二行代码调用 startActivity 方法，很简单，就是启动 APP 中的 Activity。

最终会调用 Activity 的 startActivity 方法，Intent 中携带的就是需要启动的 APP 的 Activity 信息。startActivity 方法最终会调用 startActivityForResult  
在 [AMS](https://so.csdn.net/so/search?q=AMS&spm=1001.2101.3001.7020) 中最后会调用 ActivityStarter.java 的 executeRequest(Request request) 来启动 Activity 所以在这里拦截启动是最好的方法

```
// add core start
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
			showAppCheckPasswordWindow(mService.mContext);
        }
    };
// add core end

    /**
     * Executing activity start request and starts the journey of starting an activity. Here
     * begins with performing several preliminary checks. The normally activity launch flow will
     * go through {@link #startActivityUnchecked} to {@link #startActivityInner}.
     */
    private int executeRequest(Request request) {
        if (TextUtils.isEmpty(request.reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        mLastStartReason = request.reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord = null;

        final IApplicationThread caller = request.caller;
        Intent intent = request.intent;
        NeededUriGrants intentGrants = request.intentGrants;
        String resolvedType = request.resolvedType;
        ActivityInfo aInfo = request.activityInfo;
        ResolveInfo rInfo = request.resolveInfo;
        final IVoiceInteractionSession voiceSession = request.voiceSession;
        final IBinder resultTo = request.resultTo;
        String resultWho = request.resultWho;
        int requestCode = request.requestCode;
        int callingPid = request.callingPid;
        int callingUid = request.callingUid;
        String callingPackage = request.callingPackage;
        String callingFeatureId = request.callingFeatureId;
        final int realCallingPid = request.realCallingPid;
        final int realCallingUid = request.realCallingUid;
        final int startFlags = request.startFlags;
        final SafeActivityOptions options = request.activityOptions;
        Task inTask = request.inTask;

        int err = ActivityManager.START_SUCCESS;
        // Pull the optional Ephemeral Installer-only bundle out of the options early.
        final Bundle verificationBundle =
                options != null ? options.popAppVerificationBundle() : null;

        WindowProcessController callerApp = null;
        if (caller != null) {
            callerApp = mService.getProcessController(caller);
            if (callerApp != null) {
                callingPid = callerApp.getPid();
                callingUid = callerApp.mInfo.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller + " (pid=" + callingPid
                        + ") when starting: " + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId = aInfo != null && aInfo.applicationInfo != null
                ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
        }

// add core start
        ComponentName component = intent.getComponent();
        if (component != null) {
            String activityEnabled = "com.bdjy.bedakid:com.bdjy.bedakid.mvp.ui.activity.FlashActivity";
			boolean isHavePassword = intent.getBooleanExtra("ishaspsd",false);
            Slog.i(TAG, "isHavePassword :" + isHavePassword + "--component.getClassName():" + component.getClassName());
            if (!isHavePassword&&!TextUtils.isEmpty(activityEnabled) && activityEnabled.contains(component.getPackageName()+":"+component.getClassName())) {
	mHandler.sendEmptyMessage(0);
                return START_ABORTED;
            }
        }
// add core end

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
......
}

```

启动 Activity 最终是在 ActivityStarter 的 executeRequest(Request request) 中启动 Activity 来调用 activity 入栈  
在这个方法里面 可以追踪到 当前具体启动哪个 Activity 所以在这里拦截最合适

具体实现方案如下:

```
import android.content.Context;
import android.graphics.Color;
import android.graphics.Typeface;
import android.text.InputType;
import android.view.Gravity;
import android.view.View;
import android.view.WindowManager;
import android.widget.Button;
import android.widget.EditText;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;
import android.widget.RelativeLayout;
import android.view.ViewGroup;
import android.graphics.PixelFormat;

   

// 添加输入密码窗口 输入正确密码后 进入app

private void showAppCheckPasswordWindow(Context mContext) {

    final WindowManager.LayoutParams mLayoutParams =  new WindowManager.LayoutParams();
    mLayoutParams.width = 400;
    mLayoutParams.height = 250;
    mLayoutParams.dimAmount =0.5f;
    mLayoutParams.format = PixelFormat.TRANSLUCENT;
    mLayoutParams.type = WindowManager.LayoutParams.TYPE_KEYGUARD_DIALOG;
    WindowManager mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);

    final LinearLayout parentLayout = new LinearLayout(mContext);
    parentLayout.setOrientation(LinearLayout.VERTICAL);
    parentLayout.setBackgroundColor(Color.WHITE);
    LinearLayout.LayoutParams layoutParams
            = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT,LinearLayout.LayoutParams.MATCH_PARENT);
    parentLayout.setLayoutParams(layoutParams);

    TextView titleText = new TextView(mContext);
    LinearLayout.LayoutParams contentParams
            = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT);
    titleText.setLayoutParams(contentParams);
    titleText.setText("check password");
    titleText.setTextColor(Color.BLACK);
    titleText.setTypeface(Typeface.create(titleText.getTypeface(), Typeface.NORMAL), Typeface.BOLD);
    titleText.setPadding(10, 10, 0, 0);
    parentLayout.addView(titleText);

    EditText passEdtTxt = new EditText(mContext);
    passEdtTxt.setLayoutParams(contentParams);
	passEdtTxt.setHint("Please input password");
	passEdtTxt.setTextSize(14);
    passEdtTxt.setInputType(InputType.TYPE_CLASS_TEXT | InputType.TYPE_TEXT_VARIATION_PASSWORD);
    passEdtTxt.setTextColor(Color.BLACK);
	
    parentLayout.addView(passEdtTxt);
    RelativeLayout reLayout = new RelativeLayout(mContext);
    RelativeLayout.LayoutParams rightReal = new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
    rightReal.addRule(RelativeLayout.ALIGN_PARENT_TOP);
    rightReal.addRule(RelativeLayout.ALIGN_PARENT_RIGHT, RelativeLayout.TRUE);
    rightReal.setMargins(0,10,15,0);
    
    Button confirmBtn = new Button(mContext);
    LinearLayout.LayoutParams btnParams
            = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
    confirmBtn.setLayoutParams(btnParams);
    confirmBtn.setText("ok");
    confirmBtn.setTextColor(Color.parseColor("#BEBEBE"));
    confirmBtn.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            String password = passEdtTxt.getText().toString();

            if ("123456".equals(password)) {
                if (parentLayout!=null){
                    mWindowManager.removeViewImmediate(parentLayout);
                    //parentLayout = null;
                }
                Intent intent = new Intent();
                intent.setClassName("com.bdjy.bedakid","com.bdjy.bedakid.mvp.ui.activity.FlashActivity");
                intent.putExtra("ishaspsd",true);
				intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                mService.mContext.startActivity(intent);
            }else {
                Toast.makeText(mContext,"密码错误,请重新输入...",Toast.LENGTH_SHORT).show();
            }
        }
    });
	reLayout.addView(confirmBtn, rightReal);
	RelativeLayout.LayoutParams leftReal = new RelativeLayout.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
    leftReal.addRule(RelativeLayout.ALIGN_PARENT_TOP);
    leftReal.addRule(RelativeLayout.ALIGN_PARENT_LEFT, RelativeLayout.TRUE);
    leftReal.setMargins(15,10,0,0);
    Button cancelBtn = new Button(mContext);
    LinearLayout.LayoutParams cancelbtnParams
            = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
    cancelBtn.setLayoutParams(cancelbtnParams);
    cancelBtn.setText("cancel");
    cancelBtn.setTextColor(Color.parseColor("#BEBEBE"));
    cancelBtn.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
               if (parentLayout!=null){
                    mWindowManager.removeViewImmediate(parentLayout);
                    //parentLayout = null;
                }
        }
    });
	reLayout.addView(cancelBtn, leftReal);
    parentLayout.addView(reLayout);
    try {
        mWindowManager.addView(parentLayout, mLayoutParams);
    } catch (WindowManager.BadTokenException e) {
        e.printStackTrace();
    }
}
// add core end

```

于是通过验证密码的方法来判断是否进入 Activity 从此来实现 app 应用锁的功能, 重点都是在 showAppCheckPasswordWindow(Context mContext) 处理[密码锁](https://so.csdn.net/so/search?q=%E5%AF%86%E7%A0%81%E9%94%81&spm=1001.2101.3001.7020)页面输入密码验证密码的功能