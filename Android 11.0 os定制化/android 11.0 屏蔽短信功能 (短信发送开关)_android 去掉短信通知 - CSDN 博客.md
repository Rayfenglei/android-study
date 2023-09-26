> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893914)

### 1. 概述

11.0 定制化开发中, 需要去掉短信发送功能，这就要从发送短信的流程中来分析了，从流程中了解是如何发送短信的，然后从短信的发送部分，根据系统属性来决定是否继续走完发送短信的流程

### 2. 屏蔽短信功能 (短信发送开关) 的代码

```
frameworks/opt/telephony/src/java/com/android/internal/telephony/SMSDispatcher.java
frameworks/base/telephony/java/android/telephony/SmsMessage.java
 /frameworks/opt/telephony/src/java/com/android/internal/telephony/IccSmsInterfaceManager.java


```

### 3. 屏蔽短信功能 (短信发送开关) 的功能分析

SmsManager.java 是负责发送短信的管理类，调  
smsManager.sendTextMessage();

### 3.1SmsManager 发送短信的相关代码分析

就可以发送短信了 继续往下看 sendTextMessage() 调用

```
 @UnsupportedAppUsage
      public void sendTextMessage(
              String destinationAddress, String scAddress, String text,
              PendingIntent sentIntent, PendingIntent deliveryIntent,
              int priority, boolean expectMore, int validityPeriod) {
          sendTextMessageInternal(destinationAddress, scAddress, text, sentIntent, deliveryIntent,
                  true /* persistMessage*/, priority, expectMore, validityPeriod);
      }

```

通过对 sendTextMessageInternal(）代码跟踪最终发现最后调用的是  
IccSmsInterfaceManager.java  
的 sendText() 方法  
来实现短信发送功能

### 3.2 IccSmsInterfaceManager.java 关于短信发送功能

路径： /frameworks/opt/telephony/src/java/com/android/internal/telephony/IccSmsInterfaceManager.java

```
        public void sendText(String callingPackage, String destAddr, String scAddr,
                String text, PendingIntent sentIntent, PendingIntent deliveryIntent,
                boolean persistMessageForNonDefaultSmsApp) {
            sendTextInternal(callingPackage, destAddr, scAddr, text, sentIntent, deliveryIntent,
                    persistMessageForNonDefaultSmsApp, SMS_MESSAGE_PRIORITY_NOT_SPECIFIED,
                    false /* expectMore */, SMS_MESSAGE_PERIOD_NOT_SPECIFIED, false /* isForVvm */);
        }
     
        /**
         * A permissions check before passing to {@link IccSmsInterfaceManager#sendTextInternal}.
         * This method checks if the calling package or itself has the permission to send the sms.
         */
        public void sendTextWithSelfPermissions(String callingPackage, String destAddr, String scAddr,
                String text, PendingIntent sentIntent, PendingIntent deliveryIntent,
                boolean persistMessage, boolean isForVvm) {
            if (!mSmsPermissions.checkCallingOrSelfCanSendSms(callingPackage, "Sending SMS message")) {
                returnUnspecifiedFailure(sentIntent);
                return;
            }
            sendTextInternal(callingPackage, destAddr, scAddr, text, sentIntent, deliveryIntent,
                    persistMessage, SMS_MESSAGE_PRIORITY_NOT_SPECIFIED, false /* expectMore */,
                    SMS_MESSAGE_PERIOD_NOT_SPECIFIED, isForVvm);
        }
     
        /**
         * Send a text based SMS.
         *
         * @param destAddr the address to send the message to
         * @param scAddr is the service center address or null to use
         *  the current default SMSC
         * @param text the body of the message to send
         * @param sentIntent if not NULL this <code>PendingIntent</code> is
         *  broadcast when the message is successfully sent, or failed.
         *  The result code will be <code>Activity.RESULT_OK<code> for success,
         *  or one of these errors:<br>
         *  <code>RESULT_ERROR_GENERIC_FAILURE</code><br>
         *  <code>RESULT_ERROR_RADIO_OFF</code><br>
         *  <code>RESULT_ERROR_NULL_PDU</code><br>
         *  For <code>RESULT_ERROR_GENERIC_FAILURE</code> the sentIntent may include
         *  the extra "errorCode" containing a radio technology specific value,
         *  generally only useful for troubleshooting.<br>
         *  The per-application based SMS control checks sentIntent. If sentIntent
         *  is NULL the caller will be checked against all unknown applications,
         *  which cause smaller number of SMS to be sent in checking period.
         * @param deliveryIntent if not NULL this <code>PendingIntent</code> is
         *  broadcast when the message is delivered to the recipient.  The
         *  raw pdu of the status report is in the extended data ("pdu").
         * @param persistMessageForNonDefaultSmsApp whether the sent message should
         *  be automatically persisted in the SMS db. It only affects messages sent
         *  by a non-default SMS app. Currently only the carrier app can set this
         *  parameter to false to skip auto message persistence.
         * @param priority Priority level of the message
         *  Refer specification See 3GPP2 C.S0015-B, v2.0, table 4.5.9-1
         *  ---------------------------------
         *  PRIORITY      | Level of Priority
         *  ---------------------------------
         *      '00'      |     Normal
         *      '01'      |     Interactive
         *      '10'      |     Urgent
         *      '11'      |     Emergency
         *  ----------------------------------
         *  Any Other values including negative considered as Invalid Priority Indicator of the message.
         * @param expectMore is a boolean to indicate the sending messages through same link or not.
         * @param validityPeriod Validity Period of the message in mins.
         *  Refer specification 3GPP TS 23.040 V6.8.1 section 9.2.3.12.1.
         *  Validity Period(Minimum) -> 5 mins
         *  Validity Period(Maximum) -> 635040 mins(i.e.63 weeks).
         *  Any Other values including negative considered as Invalid Validity Period of the message.
         */
     
        private void sendTextInternal(String callingPackage, String destAddr, String scAddr,
                String text, PendingIntent sentIntent, PendingIntent deliveryIntent,
                boolean persistMessageForNonDefaultSmsApp, int priority, boolean expectMore,
                int validityPeriod, boolean isForVvm) {
            if (Rlog.isLoggable("SMS", Log.VERBOSE)) {
                log("sendText: destAddr=" + destAddr + " scAddr=" + scAddr
                        + " text='" + text + "' sentIntent=" + sentIntent + " deliveryIntent="
                        + deliveryIntent + " priority=" + priority + " expectMore=" + expectMore
                        + " validityPeriod=" + validityPeriod + " isForVVM=" + isForVvm);
            }
            destAddr = filterDestAddress(destAddr);
            mDispatchersController.sendText(destAddr, scAddr, text, sentIntent, deliveryIntent,
                    null/*messageUri*/, callingPackage, persistMessageForNonDefaultSmsApp,
                    priority, expectMore, validityPeriod, isForVvm);
        }

接下来看SmsDispatchersController的sendText();

        public void sendText(String destAddr, String scAddr, String text, PendingIntent sentIntent,
                PendingIntent deliveryIntent, Uri messageUri, String callingPkg, boolean persistMessage,
                int priority, boolean expectMore, int validityPeriod, boolean isForVvm) {
            if (mImsSmsDispatcher.isAvailable() || mImsSmsDispatcher.isEmergencySmsSupport(destAddr)) {
                mImsSmsDispatcher.sendText(destAddr, scAddr, text, sentIntent, deliveryIntent,
                        messageUri, callingPkg, persistMessage, SMS_MESSAGE_PRIORITY_NOT_SPECIFIED,
                        false /*expectMore*/, SMS_MESSAGE_PERIOD_NOT_SPECIFIED, isForVvm);
            } else {
                if (isCdmaMo()) {
                    mCdmaDispatcher.sendText(destAddr, scAddr, text, sentIntent, deliveryIntent,
                            messageUri, callingPkg, persistMessage, priority, expectMore,
                            validityPeriod, isForVvm);
                } else {
                    mGsmDispatcher.sendText(destAddr, scAddr, text, sentIntent, deliveryIntent,
                            messageUri, callingPkg, persistMessage, priority, expectMore,
                            validityPeriod, isForVvm);
                }
            }
        }

```

在 SmsDispatchersController 的 sendText 中又调用 GsmSMSDispatcher 类 mDispatche 的 sendText()  
实现短信发送功能，  
继续往下看 调用 SMSDispatcher 的 sendSubmitPdu() 方法 完成发送功能  
在 sendSubmitPdu() 通过系统属性来控制短信的发送  
具体修改如下:

```
--- a/frameworks/opt/telephony/src/java/com/android/internal/telephony/SMSDispatcher.java
+++ b/frameworks/opt/telephony/src/java/com/android/internal/telephony/SMSDispatcher.java
@@ -83,7 +83,8 @@ import com.android.internal.telephony.cdma.sms.UserData;
 import com.android.internal.telephony.gsm.GsmSMSDispatcher;
 import com.android.internal.telephony.uicc.UiccCard;
 import com.android.internal.telephony.uicc.UiccController;
-
+import android.provider.Settings;
+import android.util.Log;
 import java.util.ArrayList;
 import java.util.HashMap;
 import java.util.List;
@@ -646,6 +647,14 @@ public abstract class SMSDispatcher extends Handler {
      */
     @UnsupportedAppUsage
     private void sendSubmitPdu(SmsTracker tracker) {
+           boolean isIntercept = Settings.Global.getInt(mContext.getContentResolver(),
+                "roco_sms_disable", 0) == 1;
+               Log.d("dong","getBlockStatus:"+isIntercept);
+               if(isIntercept){
+                         Rlog.d(TAG, "Block SMS in Emergency Callback mode");
+            tracker.onFailed(mContext, SmsManager.RESULT_ERROR_NO_SERVICE, 0/*errorCode*/);
+            return ;
+                               }
         if (shouldBlockSmsForEcbm()) {
             Rlog.d(TAG, "Block SMS in Emergency Callback mode");
             tracker.onFailed(mContext, SmsManager.RESULT_ERROR_NO_SERVICE, 0/*errorCode*/);

```