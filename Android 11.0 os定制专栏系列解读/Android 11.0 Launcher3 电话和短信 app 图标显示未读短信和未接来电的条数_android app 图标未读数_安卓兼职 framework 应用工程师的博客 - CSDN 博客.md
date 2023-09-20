> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124792252)

### 1. 概述

在 11.0 产品开发中，最近客户有需求要求在电话 app 图标显示未接来电的条数 在短信 app 图标上显示未读信息的条数  
根据需求首选要在 Launcher3 的 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020).java 中，启动 launcher 时, 查询未读短信和未接来电  
在有未接来电时，更新未接来电的数量 在有未读短信时，更新未读短信的数量

效果图如下:  
![](https://img-blog.csdnimg.cn/e4f6c9e4008a4239816483549f68b25a.png#pic_center)

1.Launcher.java 中，添加监听未接短信和未接来电，

```
public class SMSContentObserver extends ContentObserver {
		private Handler mHandler;

		public SMSContentObserver(Context context, Handler handler) {
			super(handler);
			mHandler = handler;
		}

		@Override
		public void onChange(boolean selfChange) {
			Log.e("Launcher-","SMSContentObserver onChange");
			mHandler.removeMessages(UPDATE_MMS_ICON);
			Message msg = mHandler.obtainMessage(UPDATE_MMS_ICON);
			msg.obj = getMissMmsCount();
			mHandler.sendMessage(msg);
		}
	}

	public class CallContentObserver extends ContentObserver {
		private Handler mHandler;
		public CallContentObserver(Context context, Handler handler) {
			super(handler);
			mHandler = handler;
		}

		@Override
		public void onChange(boolean selfChange) {
			Log.e("Launcher-","CallContentObserver onChange");
			mHandler.removeMessages(UPDATE_CALL_ICON);
			Message msg = mHandler.obtainMessage(UPDATE_CALL_ICON);
			msg.obj = getMissCallCount();
			mHandler.sendMessage(msg);
		}
	}

```

然后注册 监听未读短信和未接来电

```
import android.database.ContentObserver;
import android.provider.CallLog;
import android.net.Uri;
import android.database.Cursor;
import android.graphics.Bitmap;
import android.content.ComponentName;
import android.content.pm.ShortcutInfo;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageManager;
import android.graphics.drawable.Drawable;
import android.graphics.drawable.BitmapDrawable;
private SMSContentObserver smsContentObserver = null;  
private CallContentObserver callContentObserver = null;  

smsContentObserver = new SMSContentObserver(this,mHandler);
callContentObserver =new CallContentObserver(this,mHandler);

getContentResolver().registerContentObserver(CallLog.Calls.CONTENT_URI,true,callContentObserver);
getContentResolver().registerContentObserver(Uri.parse("content://mms-sms/"),true,smsContentObserver);

mHandler处理监听到的消息
@Thunk
final Handler mHandler = new Handler(new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        Log.i("Launcher-","mHandler msg.what = " + msg.what);
        //ADD BY Bruce Yang FOR SHOW UNREAD MMS
        else if (msg.what == UPDATE_MMS_ICON) {
            setMmsOrPhoneNum(MMS_ICON_NAME, getMissMmsCount());
        } else if (msg.what == UPDATE_CALL_ICON) {
            setMmsOrPhoneNum(PHONE_ICON_NAME, getMissCallCount());
        }
        return true;
    }
});

```

在 OnDestroy() 注销监听

```
getContentResolver().unregisterContentObserver(smsContentObserver);
getContentResolver().unregisterContentObserver(callContentObserver);

```

查询未接来电和未读短信

```
   private final static int UPDATE_MMS_ICON = 826;  
    private final static int UPDATE_CALL_ICON = 1206;
	// 这两个 ICON_NAME 根据自己实际系统短信和电话页面对应包名填写
	private final static String PHONE_ICON_NAME = "com.android.dialer.main.impl.MainActivity";
	private final static String MMS_ICON_NAME = "com.android.messaging.ui.conversationlist.ConversationListActivity";

private int getMissMmsCount() {
		Log.e("Launcher-","getMissMmsCount");
		int missSmsCount = 0;
		Cursor cursorSMS = null;
		Cursor cursorMMS = null;
		try {
			cursorSMS = getContentResolver().query(
					Uri.parse("content://sms"), null, "(read=0 and type=1)",
					null, null);
			cursorMMS = getContentResolver().query(
					Uri.parse("content://mms"), null, "(read=0)", null,
					null);
		} catch (Exception e) {
			return missSmsCount;
		}
		if (cursorSMS != null) {
			missSmsCount = cursorSMS.getCount();
			cursorSMS.close();
		}
		if (cursorMMS != null) {
				missSmsCount = missSmsCount + cursorMMS.getCount();
			cursorMMS.close();
		}

		Log.e("Launcher-","getMissMmsCount  missSmsCount = " + missSmsCount);
		return missSmsCount;
	}

	private int getMissCallCount() {
		Log.e("Launcher-","getMissCallCount");
		int missCallCount = 0;
		Uri missingCallUri = CallLog.Calls.CONTENT_URI;
		String where = CallLog.Calls.TYPE + "='" + CallLog.Calls.MISSED_TYPE + "'"
				+ " AND new=1";
		Cursor cursorCall = null;
		try {
			cursorCall = getContentResolver().query(missingCallUri,
					null, where, null, null);
		} catch (Exception e) {
			return missCallCount;
		}

		if (cursorCall != null) {
			missCallCount = cursorCall.getCount();
			cursorCall.close();
		}
		Log.e("Launcher-","getMissCallCount  missCallCount = " + missCallCount);
		return missCallCount;
	}



在OnResume()中读取未读短信和未接来电

        mHandler.postDelayed(new Runnable() {
                                 @Override
                                 public void run() {
                                     //ADD BY Bruce Yang
                                     int missCall = getMissCallCount();
                                     int missMms = getMissMmsCount();
if(missCall != 0) {
                                     setMmsOrPhoneNum(PHONE_ICON_NAME, missCall);
}
if(missMms != 0) {
                                     setMmsOrPhoneNum(MMS_ICON_NAME, missMms);
}
                                 }
                             },1000);



设置未读信息的方法如下:

/**
    *
    * @param flag 更新电话或短信 ICON
    * @param missCount 未读数
*/
private void setMmsOrPhoneNum(final String flag, final int missCount) {
    Log.e("Launcher-","flag = "+flag +" missCount = "+missCount);
    if(mWorkspace == null) return;
	 CellLayout[] cellLayouts = mWorkspace.getWorkspaceAndHotseatCellLayouts();
    for (final CellLayout layoutParent: cellLayouts) {
        final ViewGroup shortcutAndWidgetContainer = layoutParent.getShortcutsAndWidgets();

        mWorkspace.post(new Runnable() {
            public void run() {
                //找到电话和信息的图标更新带未读信息的图标
                int childCount = shortcutAndWidgetContainer.getChildCount();
				Log.e("Launcher-","childCount:"+childCount);
                for (int j = 0; j <childCount; j++) {
                    View view = shortcutAndWidgetContainer.getChildAt(j);

                    Object tag = view.getTag();
                    if (tag instanceof WorkspaceItemInfo) {
                        final WorkspaceItemInfo info = (WorkspaceItemInfo) tag;
                        final Intent intent = info.getIntent();
                        if (intent != null) {
                            final ComponentName name = intent.getComponent();
							//Log.i("Launcher-","ComponentName:"+name+"---classname:"+name.getClassName());
                            if (name != null && name.getClassName().equals(flag)) {
                                BubbleTextView bv = (BubbleTextView) view;
				FastBitmapDrawable oldIcon = (FastBitmapDrawable)(bv.getIcon());
                                Bitmap defaultIconBitmap = Bitmap.createBitmap(oldIcon.getBitmap());
                                Bitmap bitmap = Utilities.createIconBitmap(defaultIconBitmap, missCount);
								//Log.e("Launcher-","defaultIconBitmap:"+defaultIconBitmap+"---bitmap:"+bitmap);
                                bv.setCompoundDrawablesWithIntrinsicBounds(null,
                                        new FastBitmapDrawable(bitmap),
                                        null, null);
                            }
                        }
                    }
                }
            }
        });
    }
}

```

Utilities.java 中增加设置未读信息的方法  
路径: packages/apps/Launcher3/src/com/android/launcher3/Utilities.java

```
//add by Bruce Yang for ...
public static Bitmap createIconBitmap(Bitmap b, int count) {
    Bitmap bitmap = b.copy(Bitmap.Config.ARGB_8888,true);
    Log.i("Launcher-","b.isMutable() = "+b.isMutable()); // 如果为 false 就会抛出 java.lang.IllegalStateException 异常， http://bbs.csdn.net/topics/370021698
    if (count == 0) return b;
    int textureWidth = bitmap.getWidth();
    final Canvas canvas = new Canvas();
    Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DEV_KERN_TEXT_FLAG);
    canvas.setBitmap(bitmap);

    paint.setColor(Color.RED);
    canvas.drawCircle(textureWidth - 17-6, 16+6, 16+6, paint);
    paint.setColor(Color.WHITE);
    paint.setStyle(Paint.Style.STROKE);
    paint.setStrokeWidth(2);
    canvas.drawCircle(textureWidth - 17-6, 16+6, 16+6, paint);

    Paint countPaint = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DEV_KERN_TEXT_FLAG);
    countPaint.setColor(Color.WHITE);
    countPaint.setTextSize(26f);
    countPaint.setTypeface(Typeface.DEFAULT_BOLD);

    float x = textureWidth - 24-4;
    if (count > 9) x -= 4+6;

    if (count > 99) {
        countPaint.setTextSize(22f);
        String text = String.valueOf(99) + "+";
        canvas.drawText(text, x-2, 25+5, countPaint);
    } else {
        String text = String.valueOf(count);
        canvas.drawText(text,x, 25+5, countPaint);
    }
    return bitmap;
}

```

在 FastBitmapDrawable.java 增加获取 [Bitmap](https://so.csdn.net/so/search?q=Bitmap&spm=1001.2101.3001.7020) 的方法  
路径: packages/apps/Launcher3/src/com/android/launcher3/FastBitmapDrawable.java

```
public Bitmap getBitmap(){
	return mBitmap;
}

```