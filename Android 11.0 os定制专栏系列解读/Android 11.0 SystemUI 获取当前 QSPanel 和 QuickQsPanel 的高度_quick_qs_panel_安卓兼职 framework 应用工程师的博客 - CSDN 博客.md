> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754443)

在 11.0 的定制 SystemUI 下拉状态栏 UI 的时候，要求下拉展开 QuickQsPanel, 和展开通知栏  
就是说一次下拉就要展开 QuickQsPanel 不需要二次展开 QsPanel 所以就需要认真了解第二次展开  
QsPanel 的机制, 要获取第一次和第二次展开 QsPanel 的高度 好做调整

在 10.0 的原生下拉状态栏中 第一次下拉会展示 QuickQsPanel 第二次下拉会展开 QSPanel 的界面  
同时会收缩通知栏 因为 QSPanel 的高度会比 QuickQsPanel 的高度高出许多，所以会第二次展开  
QsPanel 的时候 会同时收缩通知栏

而在 StatusBar.java 中第一次创建状态栏的时候 会有 QSFragment.java 负责管理

```
  protected void setUpQuickSettingsTilePanel() {
         ....省略
        View container = mStatusBarWindow.findViewById(R.id.qs_frame);
        if (container != null) {
            FragmentHostManager fragmentHostManager = FragmentHostManager.get(container);
            ExtensionFragmentListener.attachExtensonToFragment(container, QS.TAG, R.id.qs_frame,
                    Dependency.get(ExtensionController.class)
                            .newExtension(QS.class)
                            .withPlugin(QS.class)
                            .withDefault(this::createDefaultQSFragment)
                            .build());
            mBrightnessMirrorController = new BrightnessMirrorController(mStatusBarWindow,
                    (visible) -> {
                        mBrightnessMirrorVisible = visible;
                        updateScrimController();
                    });
            fragmentHostManager.addTagListener(QS.TAG, (tag, f) -> {
                QS qs = (QS) f;
                if (qs instanceof QSFragment) {
                    mQSPanel = ((QSFragment) qs).getQsPanel();
                    mQSPanel.setBrightnessMirror(mBrightnessMirrorController);
                }
            });
        }
        ....省略
    }

```

对于获取当前 QuickQsPanel 和 Qspanel.java 的高度可以从 QsFragement.java 中查看  
接下来就来看 QsFragement.java 如何获取高度值

```
    protected QuickStatusBarHeader mHeader;
    protected QSPanel mQSPanel;
 @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        mQSPanel = view.findViewById(R.id.quick_settings_panel);
        mQSDetail = view.findViewById(R.id.qs_detail);
        mHeader = view.findViewById(R.id.header);
        mFooter = view.findViewById(R.id.qs_footer);
        mContainer = view.findViewById(id.quick_settings_container);

        mQSDetail.setQsPanel(mQSPanel, mHeader, (View) mFooter);
        mQSAnimator = new QSAnimator(this,
                mHeader.findViewById(R.id.quick_qs_panel), mQSPanel);

        mQSCustomizer = view.findViewById(R.id.qs_customize);
        mQSCustomizer.setQs(this);
        if (savedInstanceState != null) {
            setExpanded(savedInstanceState.getBoolean(EXTRA_EXPANDED));
            setListening(savedInstanceState.getBoolean(EXTRA_LISTENING));
            setEditLocation(view);
            mQSCustomizer.restoreInstanceState(savedInstanceState);
            if (mQsExpanded) {
                mQSPanel.getTileLayout().restoreInstanceState(savedInstanceState);
            }
        }
        setHost(mHost);
        mStatusBarStateController.addCallback(this);
        onStateChanged(mStatusBarStateController.getState());
    }

```

在 getDesiredHeight(）中完全可以通过打印日志的方法来获取当前的高度

```
  @Override
    public int getDesiredHeight() {
        if (mQSCustomizer.isCustomizing()) {
            return getView().getHeight();
        }
        if (mQSDetail.isClosingDetail()) {
            LayoutParams layoutParams = (LayoutParams) mQSPanel.getLayoutParams();
            int panelHeight = layoutParams.topMargin + layoutParams.bottomMargin +
                    + mQSPanel.getMeasuredHeight();
            return panelHeight + getView().getPaddingBottom();
        } else {
           android.util.Log.e("QsFragment","mQSPanel:"+mQSPanel.getMeasuredHeight()+"---mHeader:"+mHeader.getMeasuredHeight());
            return getView().getMeasuredHeight();
        }
    }

```

这样只要能设置 mQSPanel 和 mHeader 的高度一样 就不会在第二次展开时会展开 QSPanel