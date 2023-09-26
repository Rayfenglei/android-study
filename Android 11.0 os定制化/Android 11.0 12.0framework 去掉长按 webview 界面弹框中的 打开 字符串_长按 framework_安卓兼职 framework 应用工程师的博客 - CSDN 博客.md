> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754675)

### 1. 概述

在 11.0 12.0 系统定制化开发中，在 app 中进入网页 [WebView](https://so.csdn.net/so/search?q=WebView&spm=1001.2101.3001.7020) 控件加载网页后，长按网页会弹出分享和打开等等字样，但是产品需要觉得不想要这些选项 所以要求去掉这些选项所以就要从 WebView 控件开始寻找相关的代码，所以要求去掉这些选项

### 2. framework 去掉长按 webview 界面弹框中的 打开 字符串的核心类

```
frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java

```

### 3. framework 去掉长按 webview 界面弹框中的 打开 字符串的核心功能分析和实现

在 WebView 控件中，并没有发现分享的文字，于是就只能用 Android Studio 中，通过 Tools 下的 Layout Inspector  
来寻找布局文件 终于找到 FloatingToolbar 字样  
于是就全局搜索 FloatingToolbar 查看它的布局  
路径为：frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java

接下来就来看源码分析问题

```
public final class FloatingToolbar {
 
     // This class is responsible for the public API of the floating toolbar.
     // It delegates rendering operations to the FloatingToolbarPopup.
 
     public static final String FLOATING_TOOLBAR_TAG = "floating_toolbar";
 
     private static final MenuItem.OnMenuItemClickListener NO_OP_MENUITEM_CLICK_LISTENER =
             item -> false;
 
     private final Context mContext;
     private final Window mWindow;
     private final FloatingToolbarPopup mPopup;
 
     private final Rect mContentRect = new Rect();
     private final Rect mPreviousContentRect = new Rect();
 
     private Menu mMenu;
     private List<MenuItem> mShowingMenuItems = new ArrayList<>();
     private MenuItem.OnMenuItemClickListener mMenuItemClickListener = NO_OP_MENUITEM_CLICK_LISTENER;
 
     private int mSuggestedWidth;
      private boolean mWidthChanged = true;
  
      private final OnLayoutChangeListener mOrientationChangeHandler = new OnLayoutChangeListener() {
  
          private final Rect mNewRect = new Rect();
          private final Rect mOldRect = new Rect();
  
          @Override
          public void onLayoutChange(
                  View view,
                  int newLeft, int newRight, int newTop, int newBottom,
                  int oldLeft, int oldRight, int oldTop, int oldBottom) {
              mNewRect.set(newLeft, newRight, newTop, newBottom);
              mOldRect.set(oldLeft, oldRight, oldTop, oldBottom);
              if (mPopup.isShowing() && !mNewRect.equals(mOldRect)) {
                  mWidthChanged = true;
                  updateLayout();
              }
          }
      };
	  
	   private final Comparator<MenuItem> mMenuItemComparator = (menuItem1, menuItem2) -> {
          // Ensure the assist menu item is always the first item:
          if (menuItem1.getItemId() == android.R.id.textAssist) {
              return menuItem2.getItemId() == android.R.id.textAssist ? 0 : -1;
          }
          if (menuItem2.getItemId() == android.R.id.textAssist) {
              return 1;
          }
  
          // Order by SHOW_AS_ACTION type:
          if (menuItem1.requiresActionButton()) {
              return menuItem2.requiresActionButton() ? 0 : -1;
          }
          if (menuItem2.requiresActionButton()) {
              return 1;
          }
          if (menuItem1.requiresOverflow()) {
              return menuItem2.requiresOverflow() ? 0 : 1;
          }
          if (menuItem2.requiresOverflow()) {
              return -1;
          }
  
          // Order by order value:
          return menuItem1.getOrder() - menuItem2.getOrder();
      };
  
      /**
       * Initializes a floating toolbar.
       */
      public FloatingToolbar(Window window) {
          // TODO(b/65172902): Pass context in constructor when DecorView (and other callers)
          // supports multi-display.
          mContext = applyDefaultTheme(window.getContext());
          mWindow = Objects.requireNonNull(window);
          mPopup = new FloatingToolbarPopup(mContext, window.getDecorView());
      }
  
      /**
       * Sets the menu to be shown in this floating toolbar.
       * NOTE: Call {@link #updateLayout()} or {@link #show()} to effect visual changes to the
       * toolbar.
       */
      public FloatingToolbar setMenu(Menu menu) {
          mMenu = Objects.requireNonNull(menu);
          return this;
      }
  
      /**
       * Sets the custom listener for invocation of menu items in this floating toolbar.
       */
      public FloatingToolbar setOnMenuItemClickListener(
              MenuItem.OnMenuItemClickListener menuItemClickListener) {
          if (menuItemClickListener != null) {
              mMenuItemClickListener = menuItemClickListener;
          } else {
              mMenuItemClickListener = NO_OP_MENUITEM_CLICK_LISTENER;
          }
          return this;
      }
/**
 * Shows this floating toolbar.
 */
public FloatingToolbar show() {
    registerOrientationHandler();
    doShow();
    return this;
}

private void doShow() {
    List<MenuItem> menuItems = getVisibleAndEnabledMenuItems(mMenu);
    menuItems.sort(mMenuItemComparator);
    if (!isCurrentlyShowing(menuItems) || mWidthChanged) {
        mPopup.dismiss();
        mPopup.layoutMenuItems(menuItems, mMenuItemClickListener, mSuggestedWidth);
        mShowingMenuItems = menuItems;
    }
    if (!mPopup.isShowing()) {
        mPopup.show(mContentRect);
    } else if (!mPreviousContentRect.equals(mContentRect)) {
        mPopup.updateCoordinates(mContentRect);
    }
    mWidthChanged = false;
    mPreviousContentRect.set(mContentRect);
}

```

在 FloatingToolbar 中的 show() 方法就是长按 webview，弹窗的相关功能，所以在 show() 中调用  
发现 doShow() 来负责构建显示的菜单 menuItems 里面保存显示的菜单 于是就在这里来去掉打开  
通过过滤 MenuItem 里的值来去掉 字符串

修改如下:

```
 private void doShow() {
        List<MenuItem> menuItems = getVisibleAndEnabledMenuItems(mMenu);
        menuItems.sort(mMenuItemComparator);
       // 这里添加代码如下:
        int size = menuItems.size();
       for(int i=0;i<size;i++){
              MenuItem menuItem = menuItems.get(i);
               String title = menuItem.getTitle().toString();

                 if (!TextUtils.isEmpty(title)) {

                     if (title.equals(mContext.getResources().getString(R.string.websearch)) ||

                            title.equals(mContext.getResources().getString(R.string.share)) ||

                                                       title.equals(mContext.getResources().getString(R.string.browse))) {

                         menuItems.remove(i);
                 }
       }
 // 代码添加结束
        if (!isCurrentlyShowing(menuItems) || mWidthChanged) {
            mPopup.dismiss();
            mPopup.layoutMenuItems(menuItems, mMenuItemClickListener, mSuggestedWidth);
            mShowingMenuItems = menuItems;
        }
        if (!mPopup.isShowing()) {
            mPopup.show(mContentRect);
        } else if (!mPreviousContentRect.equals(mContentRect)) {
            mPopup.updateCoordinates(mContentRect);
        }
        mWidthChanged = false;
        mPreviousContentRect.set(mContentRect);
    }

```

主要的修改记录如下：

```
diff --git a/frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java b/frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java

index 97cc22e..37c3333 100755 (executable)

--- a/frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java

+++ b/frameworks/base/core/java/com/android/internal/widget/FloatingToolbar.java

@@ -285,7 +285,8 @@ public final class FloatingToolbar {


+                   String title = menuItem.getTitle().toString();

 +               if (!TextUtils.isEmpty(title)) {

 +                    if (title.equals(mContext.getResources().getString(R.string.websearch)) ||

+                            title.equals(mContext.getResources().getString(R.string.share)) ||

+                                                       title.equals(mContext.getResources().getString(R.string.browse))) {

                         menuItems.remove(i);

                     }

                 }

```