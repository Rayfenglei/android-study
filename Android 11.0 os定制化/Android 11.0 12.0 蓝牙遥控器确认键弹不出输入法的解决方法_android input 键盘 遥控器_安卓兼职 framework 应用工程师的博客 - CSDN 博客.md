> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774856)

### 1. 概述

在 android11.0 12.0 设备定制化开发时，遥控器是使用红外遥控器，也有使用蓝牙遥控器的, 所以出现的问题不一定相同，今天遇到个问题就是蓝牙遥控器在输入数据时弹不出输入法的问题  
首选排除输入法的问题，安装其他的输入法，也是同样的问题，这样就确定是系统控件相关的问题了，接下来就从系统输入框控件来寻找原因了

### 2. 蓝牙遥控器确认键弹不出输入法的解决方法的核心类

```
frameworks/base/core/java/android/widget/EditText.java
framework/base/core/java/android/widget/TextView.java

```

### 3. 蓝牙遥控器确认键弹不出输入法的解决方法的核心功能实现分析

### 3.1EditText.java 的源码分析

```
public class EditText extends TextView {
    public EditText(Context context) {
        this(context, null);
    }

    public EditText(Context context, AttributeSet attrs) {
        this(context, attrs, com.android.internal.R.attr.editTextStyle);
    }

    public EditText(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public EditText(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }

    @Override
    public boolean getFreezesText() {
        return true;
    }

    @Override
    protected boolean getDefaultEditable() {
        return true;
    }

     @Override
     protected MovementMethod getDefaultMovementMethod() {
         return ArrowKeyMovementMethod.getInstance();
     }
 
     @Override
     public Editable getText() {
         CharSequence text = super.getText();
         // This can only happen during construction.
         if (text == null) {
             return null;
         }
         if (text instanceof Editable) {
             return (Editable) super.getText();
         }
         super.setText(text, BufferType.EDITABLE);
         return (Editable) super.getText();
     }
 
     @Override
     public void setText(CharSequence text, BufferType type) {
         super.setText(text, BufferType.EDITABLE);
     }
	 ...
}

```

从 EditText 的代码中可以看出 EditText 是 TextView 的子类， 所以最终需要从 TextView.java 中查询问题所在了，看问题在哪里  
路径: framework/base/core/java/android/widget/TextView.java

蓝牙遥控器的确定按键对应的 Android keycode 事件是 KeyEvent.KEYCODE_DPAD_CENTER  
接下来查看源码中的 KeyEvent.KEYCODE_DPAD_CENTER 处理事件

### 3.2TextView.java 的相关代码分析

```
 public class TextView extends View implements ViewTreeObserver.OnPreDrawListener {
      static final String LOG_TAG = "TextView";
      static final boolean DEBUG_EXTRACT = false;
      static final boolean DEBUG_CURSOR = false;
public TextView(Context context) {
          this(context, null);
      }
  
      public TextView(Context context, @Nullable AttributeSet attrs) {
          this(context, attrs, com.android.internal.R.attr.textViewStyle);
      }
  
      public TextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
          this(context, attrs, defStyleAttr, 0);
      }
 @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (!isEnabled()) {
            return super.onKeyUp(keyCode, event);
        }

        if (!KeyEvent.isModifierKey(keyCode)) {
            mPreventDefaultMovement = false;
        }

        switch (keyCode) {
            case KeyEvent.KEYCODE_DPAD_CENTER:
                if (event.hasNoModifiers()) {
                    /*
                     * If there is a click listener, just call through to
                     * super, which will invoke it.
                     *
                     * If there isn't a click listener, try to show the soft
                     * input method.  (It will also
                     * call performClick(), but that won't do anything in
                     * this case.)
                     */
                    if (!hasOnClickListeners()) {
                        if (mMovement != null && mText instanceof Editable
                                && mLayout != null && onCheckIsTextEditor()) {
                            InputMethodManager imm = getInputMethodManager();
                            viewClicked(imm);
                            if (imm != null && getShowSoftInputOnFocus()) {
                                imm.showSoftInput(this, 0);
                            }
                        }
                    }
                }
                return super.onKeyUp(keyCode, event);

            case KeyEvent.KEYCODE_ENTER:
                if (event.hasNoModifiers()) {
                    if (mEditor != null && mEditor.mInputContentType != null
                            && mEditor.mInputContentType.onEditorActionListener != null
                            && mEditor.mInputContentType.enterDown) {
                        mEditor.mInputContentType.enterDown = false;
                        if (mEditor.mInputContentType.onEditorActionListener.onEditorAction(
                                this, EditorInfo.IME_NULL, event)) {
                            return true;
                        }
                    }

                    if ((event.getFlags() & KeyEvent.FLAG_EDITOR_ACTION) != 0
                            || shouldAdvanceFocusOnEnter()) {
                        /*
                         * If there is a click listener, just call through to
                         * super, which will invoke it.
                         *
                         * If there isn't a click listener, try to advance focus,
                         * but still call through to super, which will reset the
                         * pressed state and longpress state.  (It will also
                         * call performClick(), but that won't do anything in
                         * this case.)
                         */
                        if (!hasOnClickListeners()) {
                            View v = focusSearch(FOCUS_DOWN);

                            if (v != null) {
                                if (!v.requestFocus(FOCUS_DOWN)) {
                                    throw new IllegalStateException("focus search returned a view "
                                            + "that wasn't able to take focus!");
                                }

                                /*
                                 * Return true because we handled the key; super
                                 * will return false because there was no click
                                 * listener.
                                 */
                                super.onKeyUp(keyCode, event);
                                return true;
                            } else if ((event.getFlags()
                                    & KeyEvent.FLAG_EDITOR_ACTION) != 0) {
                                // No target for next focus, but make sure the IME
                                // if this came from it.
                                InputMethodManager imm = getInputMethodManager();
                                if (imm != null && imm.isActive(this)) {
                                    imm.hideSoftInputFromWindow(getWindowToken(), 0);
                                }
                            }
                        }
                    }
                    return super.onKeyUp(keyCode, event);
                }
                break;
        }

        if (mEditor != null && mEditor.mKeyListener != null) {
            if (mEditor.mKeyListener.onKeyUp(this, (Editable) mText, keyCode, event)) {
                return true;
            }
        }

        if (mMovement != null && mLayout != null) {
            if (mMovement.onKeyUp(this, mSpannable, keyCode, event)) {
                return true;
            }
        }

        return super.onKeyUp(keyCode, event);
    }

```

在 TextView 的 onKeyUp 中可以看出判断如果是 hasOnClickListeners() 为 false 才会执行  
调用 imm.showSoftInput(this, 0); 启用软键盘

```
if (!hasOnClickListeners()) {
if (mMovement != null && mText instanceof Editable
&& mLayout != null && onCheckIsTextEditor()) {
InputMethodManager imm = getInputMethodManager();
viewClicked(imm);
if (imm != null && getShowSoftInputOnFocus()) {
imm.showSoftInput(this, 0);
}
}
}

```

调用输入法  
就是说没有设置 OnClickListeners 事件才会被调用  
所以找到问题所在了  
解决方法就是去掉这个条件判断 修改为:

```
case KeyEvent.KEYCODE_DPAD_CENTER:
                if (event.hasNoModifiers()) {
                    /*
                     * If there is a click listener, just call through to
                     * super, which will invoke it.
                     *
                     * If there isn't a click listener, try to show the soft
                     * input method.  (It will also
                     * call performClick(), but that won't do anything in
                     * this case.)
                     */
                    //if (!hasOnClickListeners()) {
                        if (mMovement != null && mText instanceof Editable
                                && mLayout != null && onCheckIsTextEditor()) {
                            InputMethodManager imm = getInputMethodManager();
                            viewClicked(imm);
                            if (imm != null && getShowSoftInputOnFocus()) {
                                imm.showSoftInput(this, 0);
                            }
                        }
                   // }
                }
                return super.onKeyUp(keyCode, event);

```