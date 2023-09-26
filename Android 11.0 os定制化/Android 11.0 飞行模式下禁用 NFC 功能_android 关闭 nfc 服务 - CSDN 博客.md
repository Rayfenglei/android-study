> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124792285)

11.0 产品在测试中发现，打开飞行模式以后，wifi 和蓝牙都关闭了，[nfc](https://so.csdn.net/so/search?q=nfc&spm=1001.2101.3001.7020) 还是能打开的，这也是原生系统的一个 bug, 客户要求开启飞行模式的时候 禁用 nfc, 就是  
不能打开 nfc, 既然发现了就必须解决掉这个问题  
通过代码发现飞行模式打开后，控制这个的参数就是 airplane_mode_toggleable_radios，然后根据这个参数来设置哪些可以打开使用  
接下来 看下这个参数  
路径:  
/frameworks/base/packages/SettingsProvider/res/values/defaults.xml

```
<string >bluetooth,wifi,nfc</string>

```

所以可以去掉 nfc 修改为:

```
<string >bluetooth,wifi</string>

```

这样在 / packages/apps/Settings/src/com/android/settings/nfc/NfcEnabler.java 中

```
    @VisibleForTesting
    boolean isToggleable() {
        if (NfcPreferenceController.isToggleableInAirplaneMode(mContext)
                || !NfcPreferenceController.shouldTurnOffNFCInAirplaneMode(mContext)) {
            return true;
        }
        final int airplaneMode = Settings.Global.getInt(
                mContext.getContentResolver(), Settings.Global.AIRPLANE_MODE_ON, 0);
        return airplaneMode != 1;
    }

NfcPreferenceController.isToggleableInAirplaneMode(mContext) 就为false;

@Override
    protected void handleNfcStateChanged(int newState) {
        switch (newState) {
            case NfcAdapter.STATE_OFF:
                mPreference.setChecked(false);
                mPreference.setEnabled(isToggleable());
                break;
}

```

nfc 的 [setEnabled](https://so.csdn.net/so/search?q=setEnabled&spm=1001.2101.3001.7020)(false；就不可用了

但是这种修改方式有些缺陷，就是只有进入 nfc 界面时，才会让开关为不可用状态 在开启飞行模式时，这里的相关代码都没执行  
所以要想实时更新 nfc 开关状态 就得在 ConnectivityManager.java 中 当开启飞行模式时 让 nfc 不可用  
路径为:/frameworks/base/core/java/android/net/ConnectivityManager.java

接下来 在 @SystemApi

```
public void setAirplaneMode(boolean enable) {
    try {
        mService.setAirplaneMode(enable);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}

```

飞行模式中添加 NFC 相关代码

```
private NfcAdapter mNFCAdapter;
static final int NFC_STATUS_DISABLED                      = 0;
static final int NFC_STATUS_ENABLED                       = 1;
private static final int NFC_DISABLED_AIRPLANE_ON          = 2;
private int  mPersistNfcState = NFC_STATUS_DISABLED;

public void setAirplaneMode(boolean enable) {
    try {
        mService.setAirplaneMode(enable);
       
       // add code start;
       if (!enable) {
            if ( mPersistNfcState == NFC_DISABLED_AIRPLANE_ON) {
                getAdapter().enable();
            }
        } else {
            if (getAdapter().isEnabled()) {
                mPersistNfcState = NFC_DISABLED_AIRPLANE_ON; 
                getAdapter().disable();
            } else {
                mPersistNfcState = NFC_STATUS_DISABLED;
                getAdapter().disable();
            }
        }
       //add code end;

    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}

private NfcAdapter getAdapter() {
    if (mNFCAdapter == null) {
        try {
            mNFCAdapter = NfcAdapter.getNfcAdapter(mContext);
        } catch (UnsupportedOperationException e) {
            mNFCAdapter = null;
        }
    }
    return mNFCAdapter;
}

```

最后编译以后发现满足了自己的项目需求