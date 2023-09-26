> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124792265)

在 11.0 12.0 设备开发中，遇到奇怪的现象，就是遥控器操作输入框的时候，始终弹不出输入法，刚开始怀疑是输入法的问题，换输入法发现还是一样  
，这时候又连接鼠标来操作发现可以弹出输入法 ，那么就不是输入法的问题，就要从遥控器焦点入手了

1. 首选看 [EditText](https://so.csdn.net/so/search?q=EditText&spm=1001.2101.3001.7020) 有没获取到焦点

```
edittext.setOnFocusChangeListener(new View.OnFocusChangeListener() {
            @Override
            public void onFocusChange(View view, boolean b) {
                     Log.e("EditText","b:"+b);


 
            }
        });

```

注册监听获取焦点 发现 b 始终为 false; 说明获取不到焦点

2. 接下来看 EditText.java 源码类  
路径：framework/base/core/java/android/widget/EditText.java

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
}

```

发现 EditText.java 继承 [TextView](https://so.csdn.net/so/search?q=TextView&spm=1001.2101.3001.7020) 类 没有出来焦点事件

接下来看 TextView.java 类  
路径: framework/base/core/java/android/widget/TextView.java

```
@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    final int which = doKeyDown(keyCode, event, null);
    if (which == KEY_EVENT_NOT_HANDLED) {
        return super.onKeyDown(keyCode, event);
    }

    return true;
}



接下来看doKeyDown(keyCode, event, null);

private int doKeyDown(int keyCode, KeyEvent event, KeyEvent otherEvent) {
        if (!isEnabled()) {
            return KEY_EVENT_NOT_HANDLED;
        }

        // If this is the initial keydown, we don't want to prevent a movement away from this view.
        // While this shouldn't be necessary because any time we're preventing default movement we
        // should be restricting the focus to remain within this view, thus we'll also receive
        // the key up event, occasionally key up events will get dropped and we don't want to
        // prevent the user from traversing out of this on the next key down.
        if (event.getRepeatCount() == 0 && !KeyEvent.isModifierKey(keyCode)) {
            mPreventDefaultMovement = false;
        }

        switch (keyCode) {
            case KeyEvent.KEYCODE_ENTER:
                if (event.hasNoModifiers()) {
                    // When mInputContentType is set, we know that we are
                    // running in a "modern" cupcake environment, so don't need
                    // to worry about the application trying to capture
                    // enter key events.
                    if (mEditor != null && mEditor.mInputContentType != null) {
                        // If there is an action listener, given them a
                        // chance to consume the event.
                        if (mEditor.mInputContentType.onEditorActionListener != null
                                && mEditor.mInputContentType.onEditorActionListener.onEditorAction(
                                        this, EditorInfo.IME_NULL, event)) {
                            mEditor.mInputContentType.enterDown = true;
                            // We are consuming the enter key for them.
                            return KEY_EVENT_HANDLED;
                        }
                    }

                    // If our editor should move focus when enter is pressed, or
                    // this is a generated event from an IME action button, then
                    // don't let it be inserted into the text.
                    if ((event.getFlags() & KeyEvent.FLAG_EDITOR_ACTION) != 0
                            || shouldAdvanceFocusOnEnter()) {
                        if (hasOnClickListeners()) {
                            return KEY_EVENT_NOT_HANDLED;
                        }
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_DPAD_CENTER:
                if (event.hasNoModifiers()) {
                    if (shouldAdvanceFocusOnEnter()) {
                        return KEY_EVENT_NOT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_TAB:
                if (event.hasNoModifiers() || event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                    if (shouldAdvanceFocusOnTab()) {
                        return KEY_EVENT_NOT_HANDLED;
                    }
                }
                break;

                // Has to be done on key down (and not on key up) to correctly be intercepted.
            case KeyEvent.KEYCODE_BACK:
                if (mEditor != null && mEditor.getTextActionMode() != null) {
                    stopTextActionMode();
                    return KEY_EVENT_HANDLED;
                }
                break;

            case KeyEvent.KEYCODE_CUT:
                if (event.hasNoModifiers() && canCut()) {
                    if (onTextContextMenuItem(ID_CUT)) {
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_COPY:
                if (event.hasNoModifiers() && canCopy()) {
                    if (onTextContextMenuItem(ID_COPY)) {
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_PASTE:
                if (event.hasNoModifiers() && canPaste()) {
                    if (onTextContextMenuItem(ID_PASTE)) {
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_FORWARD_DEL:
                if (event.hasModifiers(KeyEvent.META_SHIFT_ON) && canCut()) {
                    if (onTextContextMenuItem(ID_CUT)) {
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;

            case KeyEvent.KEYCODE_INSERT:
                if (event.hasModifiers(KeyEvent.META_CTRL_ON) && canCopy()) {
                    if (onTextContextMenuItem(ID_COPY)) {
                        return KEY_EVENT_HANDLED;
                    }
                } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON) && canPaste()) {
                    if (onTextContextMenuItem(ID_PASTE)) {
                        return KEY_EVENT_HANDLED;
                    }
                }
                break;
        }

        if (mEditor != null && mEditor.mKeyListener != null) {
            boolean doDown = true;
            if (otherEvent != null) {
                try {
                    beginBatchEdit();
                    final boolean handled = mEditor.mKeyListener.onKeyOther(this, (Editable) mText,
                            otherEvent);
                    hideErrorIfUnchanged();
                    doDown = false;
                    if (handled) {
                        return KEY_EVENT_HANDLED;
                    }
                } catch (AbstractMethodError e) {
                    // onKeyOther was added after 1.0, so if it isn't
                    // implemented we need to try to dispatch as a regular down.
                } finally {
                    endBatchEdit();
                }
            }

            if (doDown) {
                beginBatchEdit();
                final boolean handled = mEditor.mKeyListener.onKeyDown(this, (Editable) mText,
                        keyCode, event);
                endBatchEdit();
                hideErrorIfUnchanged();
                if (handled) return KEY_DOWN_HANDLED_BY_KEY_LISTENER;
            }
        }

        // bug 650865: sometimes we get a key event before a layout.
        // don't try to move around if we don't know the layout.

        if (mMovement != null && mLayout != null) {
            boolean doDown = true;
            if (otherEvent != null) {
                try {
                    boolean handled = mMovement.onKeyOther(this, mSpannable, otherEvent);
                    doDown = false;
                    if (handled) {
                        return KEY_EVENT_HANDLED;
                    }
                } catch (AbstractMethodError e) {
                    // onKeyOther was added after 1.0, so if it isn't
                    // implemented we need to try to dispatch as a regular down.
                }
            }
            if (doDown) {
                if (mMovement.onKeyDown(this, mSpannable, keyCode, event)) {
                    if (event.getRepeatCount() == 0 && !KeyEvent.isModifierKey(keyCode)) {
                        mPreventDefaultMovement = true;
                    }
                    return KEY_DOWN_HANDLED_BY_MOVEMENT_METHOD;
                }
            }
            // Consume arrows from keyboard devices to prevent focus leaving the editor.
            // DPAD/JOY devices (Gamepads, TV remotes) often lack a TAB key so allow those
            // to move focus with arrows.
            if (event.getSource() == InputDevice.SOURCE_KEYBOARD
                    && isDirectionalNavigationKey(keyCode)) {
                return KEY_EVENT_HANDLED;
            }
        }

        return mPreventDefaultMovement && !KeyEvent.isModifierKey(keyCode)
                ? KEY_EVENT_HANDLED : KEY_EVENT_NOT_HANDLED;
    }

```

而这段代码从注释可以看出会消耗掉焦点

```
// Consume arrows from keyboard devices to prevent focus leaving the editor.
            // DPAD/JOY devices (Gamepads, TV remotes) often lack a TAB key so allow those
            // to move focus with arrows.
         /*   if (event.getSource() == InputDevice.SOURCE_KEYBOARD
                    && isDirectionalNavigationKey(keyCode)) {
                return KEY_EVENT_HANDLED;
            }*/

```

所以注释掉这段代码  
然后编译 发现遥控器已经可以在输入框弹出输入法了 完美解决问题