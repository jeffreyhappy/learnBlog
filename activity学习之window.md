不错的博客

https://blog.csdn.net/yhaolpz/article/details/68936932


window是view的载体,持有了decorView


Activity中,有这两个,先看Window
```
    private Window mWindow;

    private WindowManager mWindowManager;
```

之前看Activity的创建的时候,Activity创建成功后,会调用attach初始化,window就是在这个时候初始化的
```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

		//1 
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

		//2 
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);

        setAutofillCompatibilityEnabled(application.isAutofillCompatibilityEnabled());
        enableAutofillCompatibilityIfNeeded();
    }
```
1 这边初始化了Window,实现类是PhoneWindow,等下看下PhoneWindow
2 这边涉及到了WindowManager,通过getSystemService获得了一个WindowManager对象,这里有个问题:Window是如何被WindowManger添加上的

初始化

```
    public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }

    /**
     * Constructor for main window of an activity.
     */
    public PhoneWindow(Context context, Window preservedWindow,
            ActivityConfigCallback activityConfigCallback) {
        this(context);
        // Only main activity windows use decor context, all the other windows depend on whatever
        // context that was given to them.
        mUseDecorContext = true;
        if (preservedWindow != null) {
            mDecor = (DecorView) preservedWindow.getDecorView();
            mElevation = preservedWindow.getElevation();
            mLoadElevation = false;
            mForceDecorInstall = true;
            // If we're preserving window, carry over the app token from the preserved
            // window, as we'll be skipping the addView in handleResumeActivity(), and
            // the token will not be updated as for a new window.
            getAttributes().token = preservedWindow.getAttributes().token;
        }
        // Even though the device doesn't support picture-in-picture mode,
        // an user can force using it through developer options.
        boolean forceResizable = Settings.Global.getInt(context.getContentResolver(),
                DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
        mSupportsPictureInPicture = forceResizable || context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_PICTURE_IN_PICTURE);
        mActivityConfigCallback = activityConfigCallback;
    }

```
window用的是 new PhoneWindow(this, window, activityConfigCallback)这个构造函数,里面注释了说只有主activity的windows 使用decor context,其他的值要基于它就好了
如果是主Activity的话,mDecor这个对象是暂时没赋值的,赋值是在后面的setContent

```
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
		}
		.....
	}
	
	private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
		    //1
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
		    //2
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
                mDecorContentParent.setWindowCallback(getCallback());
                if (mDecorContentParent.getTitle() == null) {
                    mDecorContentParent.setWindowTitle(mTitle);
                }

                final int localFeatures = getLocalFeatures();
                for (int i = 0; i < FEATURE_MAX; i++) {
                    if ((localFeatures & (1 << i)) != 0) {
                        mDecorContentParent.initFeature(i);
                    }
                }

                mDecorContentParent.setUiOptions(mUiOptions);

                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                        (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                    mDecorContentParent.setIcon(mIconRes);
                } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                        mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                    mDecorContentParent.setIcon(
                            getContext().getPackageManager().getDefaultActivityIcon());
                    mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
                }
                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                        (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                    mDecorContentParent.setLogo(mLogoRes);
                }

                // Invalidate if the panel menu hasn't been created before this.
                // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
                // being called in the middle of onCreate or similar.
                // A pending invalidation will typically be resolved before the posted message
                // would run normally in order to satisfy instance state restoration.
                PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
                if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                    invalidatePanelMenu(FEATURE_ACTION_BAR);
                }
            } else {
                mTitleView = findViewById(R.id.title);
                if (mTitleView != null) {
                    if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                        final View titleContainer = findViewById(R.id.title_container);
                        if (titleContainer != null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }

            if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
                mDecor.setBackgroundFallback(mBackgroundFallbackResource);
            }

            // Only inflate or create a new TransitionManager if the caller hasn't
            // already set a custom one.
            if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
                if (mTransitionManager == null) {
                    final int transitionRes = getWindowStyle().getResourceId(
                            R.styleable.Window_windowContentTransitionManager,
                            0);
                    if (transitionRes != 0) {
                        final TransitionInflater inflater = TransitionInflater.from(getContext());
                        mTransitionManager = inflater.inflateTransitionManager(transitionRes,
                                mContentParent);
                    } else {
                        mTransitionManager = new TransitionManager();
                    }
                }

                mEnterTransition = getTransition(mEnterTransition, null,
                        R.styleable.Window_windowEnterTransition);
                mReturnTransition = getTransition(mReturnTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReturnTransition);
                mExitTransition = getTransition(mExitTransition, null,
                        R.styleable.Window_windowExitTransition);
                mReenterTransition = getTransition(mReenterTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReenterTransition);
                mSharedElementEnterTransition = getTransition(mSharedElementEnterTransition, null,
                        R.styleable.Window_windowSharedElementEnterTransition);
                mSharedElementReturnTransition = getTransition(mSharedElementReturnTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReturnTransition);
                mSharedElementExitTransition = getTransition(mSharedElementExitTransition, null,
                        R.styleable.Window_windowSharedElementExitTransition);
                mSharedElementReenterTransition = getTransition(mSharedElementReenterTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReenterTransition);
                if (mAllowEnterTransitionOverlap == null) {
                    mAllowEnterTransitionOverlap = getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowEnterTransitionOverlap, true);
                }
                if (mAllowReturnTransitionOverlap == null) {
                    mAllowReturnTransitionOverlap = getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowReturnTransitionOverlap, true);
                }
                if (mBackgroundFadeDurationMillis < 0) {
                    mBackgroundFadeDurationMillis = getWindowStyle().getInteger(
                            R.styleable.Window_windowTransitionBackgroundFadeDuration,
                            DEFAULT_BACKGROUND_FADE_DURATION_MS);
                }
                if (mSharedElementsUseOverlay == null) {
                    mSharedElementsUseOverlay = getWindowStyle().getBoolean(
                            R.styleable.Window_windowSharedElementsUseOverlay, true);
                }
            }
        }
    }
	
```
1 那边如果没有decorView就创建,有就把phoneWindow塞给他
2 那边涉及到了mContentParent 
   mDecor是Window下的顶级view
   mContentParent要么是mDecor,要么就是mDecor的子view
```
    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;
    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    ViewGroup mContentParent;
```


window有点绕,以后再看