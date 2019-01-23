## FragmentController

文档上说:这个是给FramgentManager提供一个Framgnet的切入点
他的构造方式只有createController这一种,所以得再了解下FragmentHostCallback
```
public class FragmentController {
    private final FragmentHostCallback<?> mHost;

    /**
     * Returns a {@link FragmentController}.
     */
    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }

    private FragmentController(FragmentHostCallback<?> callbacks) {
        mHost = callbacks;
    }
}
```
fragments可以被任何对象持有,例如Activity,为了持有fragments就要实现FragmentHostCallback并重写它的方法
先看FragmentHostCallback的构造函数,看下来这个类其实还是为了activity准备的,在Activity的代码中E的类就是Activity
```
public abstract class FragmentHostCallback<E> extends FragmentContainer {
    public FragmentHostCallback(Context context, Handler handler, int windowAnimations) {
        this((context instanceof Activity) ? (Activity)context : null, context,
                chooseHandler(context, handler), windowAnimations);
    }

    FragmentHostCallback(Activity activity) {
        this(activity, activity /*context*/, activity.mHandler, 0 /*windowAnimations*/);
    }

    FragmentHostCallback(Activity activity, Context context, Handler handler,
            int windowAnimations) {
        mActivity = activity;
        mContext = context;
        mHandler = handler;
        mWindowAnimations = windowAnimations;
    }
}

```
所以Fragment相关的初始化
Activity中
```
    final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
```
HostCallbacks
```
class HostCallbacks extends FragmentHostCallback<Activity> {
        public HostCallbacks() {
            super(Activity.this /*activity*/);
        }
}
```
FragmentHostCallback
```
public abstract class FragmentHostCallback<E> extends FragmentContainer {
  public FragmentHostCallback(Context context, Handler handler, int windowAnimations) {
        this((context instanceof Activity) ? (Activity)context : null, context,
                chooseHandler(context, handler), windowAnimations);
    }

    FragmentHostCallback(Activity activity) {
        this(activity, activity /*context*/, activity.mHandler, 0 /*windowAnimations*/);
    }

    FragmentHostCallback(Activity activity, Context context, Handler handler,
            int windowAnimations) {
        mActivity = activity;
        mContext = context;
        mHandler = handler;
        mWindowAnimations = windowAnimations;
    }
}
```
FragmentController中
```
    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }

    private FragmentController(FragmentHostCallback<?> callbacks) {
        mHost = callbacks;
    }
```
FragmentController的初始化就是各种赋值,和activity关联起来
初始化完了,我们平时使用就是getFragmentManager(),就以他为入口
```
    public FragmentManager getFragmentManager() {
        return mFragments.getFragmentManager();
    }
```

```
   public FragmentManager getFragmentManager() {
        return mHost.getFragmentManagerImpl();
    }
```
看了下Activity的HostCallbacks中没有重写这个getFragmentManagerImpl方法,那就得去基类里面找了
```
public abstract class FragmentHostCallback<E> extends FragmentContainer {
    final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();
	FragmentManagerImpl getFragmentManagerImpl() {
        return mFragmentManager;
    }
}	
```
FragmentManagerImpl是直接在基类中创建的,就在这里看到的熟悉的FragmentManager
```
final class FragmentManagerImpl extends FragmentManager implements LayoutInflater.Factory2{
}
```
我们之前使用是这样的,就以它为切入口
```
getFragmentManager().beginTransaction().add().commit();
```
对于FragmentManager这个方法是个抽象发放,而FragmentManagerImpl重写了这个方法
```
    @Override
    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }
```
又牵扯到了BackStackRecord,它继承了FragmentTransaction,FragmentTransaction是个抽象类,定义了add,attach,remove,replace,commit等方法.
我一直就对add,attach这几个方法的使用区别比较模糊
```
final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManagerImpl.OpGenerator {
	public BackStackRecord(FragmentManagerImpl manager) {
        mManager = manager;
        mReorderingAllowed = mManager.getTargetSdk() > Build.VERSION_CODES.N_MR1;
    }
}		
```
看add的代码,各种检查,然后走进了addOp
```
public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
    }

    private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        if (mManager.getTargetSdk() > Build.VERSION_CODES.N_MR1) {
            final Class fragmentClass = fragment.getClass();
            final int modifiers = fragmentClass.getModifiers();
            if ((fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers)
                    || (fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers)))) {
                throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName()
                        + " must be a public static class to be  properly recreated from"
                        + " instance state.");
            }
        }
        fragment.mFragmentManager = mManager;

        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }

        if (containerViewId != 0) {
            if (containerViewId == View.NO_ID) {
                throw new IllegalArgumentException("Can't add fragment "
                        + fragment + " with tag " + tag + " to container view with no id");
            }
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }

        addOp(new Op(opcmd, fragment));
    }
```
Op的参数cmd是OP_ADD,fragment就是对应的fragment,放进了mOps这个列表里,还定义了op的动画,接下来看commit
```
    void addOp(Op op) {
        mOps.add(op);
        op.enterAnim = mEnterAnim;
        op.exitAnim = mExitAnim;
        op.popEnterAnim = mPopEnterAnim;
        op.popExitAnim = mPopExitAnim;
    }

```
commit就是这个实现
```
 int commitInternal(boolean allowStateLoss) {
        if (mCommitted) {
            throw new IllegalStateException("commit already called");
        }
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Commit: " + this);
            LogWriter logw = new LogWriter(Log.VERBOSE, TAG);
            PrintWriter pw = new FastPrintWriter(logw, false, 1024);
            dump("  ", null, pw, null);
            pw.flush();
        }
        mCommitted = true;
		//1
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
		//2
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
```
注释1这里,如果要加入返回栈里,就调用FragmentManagerImpl来加入,并返回对于的index
```
public int allocBackStackIndex(BackStackRecord bse) {
        synchronized (this) {
            if (mAvailBackStackIndices == null || mAvailBackStackIndices.size() <= 0) {
                if (mBackStackIndices == null) {
                    mBackStackIndices = new ArrayList<BackStackRecord>();
                }
                int index = mBackStackIndices.size();
                if (DEBUG) Log.v(TAG, "Setting back stack index " + index + " to " + bse);
                mBackStackIndices.add(bse);
                return index;

            } else {
                int index = mAvailBackStackIndices.remove(mAvailBackStackIndices.size()-1);
                if (DEBUG) Log.v(TAG, "Adding back stack index " + index + " with " + bse);
                mBackStackIndices.set(index, bse);
                return index;
            }
        }
    }
```
注释2应该就是真的添加了,BackStackRecord是实现了OpGenerator接口的

```
    public void enqueueAction(OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                if (allowStateLoss) {
                    // This FragmentManager isn't attached, so drop the entire transaction.
                    return;
                }
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<>();
            }
            mPendingActions.add(action);
            scheduleCommit();
        }
    }
	
	 private void scheduleCommit() {
        synchronized (this) {
            boolean postponeReady =
                    mPostponedTransactions != null && !mPostponedTransactions.isEmpty();
            boolean pendingReady = mPendingActions != null && mPendingActions.size() == 1;
            if (postponeReady || pendingReady) {
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit);
            }
        }
    }
```
通过FragmentHostCallBack来运行一个Runnable, execPendingActions()上的注释明确要求要在主线程运行
```
    Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions();
        }
    };  
```

```
  public boolean execPendingActions() {
        ensureExecReady(true);

        boolean didSomething = false;
        while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
            mExecutingActions = true;
            try {
                removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
            } finally {
                cleanupExec();
            }
            didSomething = true;
        }

        doPendingDeferredStart();
        burpActive();

        return didSomething;
    }
```

一路往下跟,最终
```
void executeOps() {
        final int numOps = mOps.size();
        for (int opNum = 0; opNum < numOps; opNum++) {
            final Op op = mOps.get(opNum);
            final Fragment f = op.fragment;
            if (f != null) {
                f.setNextTransition(mTransition, mTransitionStyle);
            }
            switch (op.cmd) {
                case OP_ADD:
                    f.setNextAnim(op.enterAnim);
                    mManager.addFragment(f, false);
                    break;
                case OP_REMOVE:
                    f.setNextAnim(op.exitAnim);
                    mManager.removeFragment(f);
                    break;
                case OP_HIDE:
                    f.setNextAnim(op.exitAnim);
                    mManager.hideFragment(f);
                    break;
                case OP_SHOW:
                    f.setNextAnim(op.enterAnim);
                    mManager.showFragment(f);
                    break;
                case OP_DETACH:
                    f.setNextAnim(op.exitAnim);
                    mManager.detachFragment(f);
                    break;
                case OP_ATTACH:
                    f.setNextAnim(op.enterAnim);
                    mManager.attachFragment(f);
                    break;
                case OP_SET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(f);
                    break;
                case OP_UNSET_PRIMARY_NAV:
                    mManager.setPrimaryNavigationFragment(null);
                    break;
                default:
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
            }
            if (!mReorderingAllowed && op.cmd != OP_ADD && f != null) {
                mManager.moveFragmentToExpectedState(f);
            }
        }
        if (!mReorderingAllowed) {
            // Added fragments are added at the end to comply with prior behavior.
            mManager.moveToState(mManager.mCurState, true);
        }
    }
```
搞了半天还是调用FragmentManagerImp的对应方法,去看下addFragment

```
    public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
			//将fragment加入list
            synchronized (mAdded) {
                mAdded.add(fragment);
            }
            fragment.mAdded = true;
            fragment.mRemoving = false;
            if (fragment.mView == null) {
                fragment.mHiddenChanged = false;
            }
            if (fragment.mHasMenu && fragment.mMenuVisible) {
                mNeedMenuInvalidate = true;
            }
            if (moveToStateNow) {
                moveToState(fragment);
            }
        }
    }
	
	void moveToState(Fragment f) {
        moveToState(f, mCurState, 0, 0, false);
    }
```
最终还是调用了moveToState,Fragment的生命周期相关的都在这里,这个函数太长了,就先看看初始化
```
    void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive) {
        if (DEBUG && false) Log.v(TAG, "moveToState: " + f
            + " oldState=" + f.mState + " newState=" + newState
            + " mRemoving=" + f.mRemoving + " Callers=" + Debug.getCallers(5));

        // Fragments that are not currently added will sit in the onCreate() state.
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
            if (f.mState == Fragment.INITIALIZING && f.isInBackStack()) {
                // Allow the fragment to be created so that it can be saved later.
                newState = Fragment.CREATED;
            } else {
                // While removing a fragment, we can't change it to a higher state.
                newState = f.mState;
            }
        }
        // Defer start if requested; don't allow it to move to STARTED or higher
        // if it's not already started.
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.STOPPED) {
            newState = Fragment.STOPPED;
        }
        if (f.mState <= newState) {
            // For fragments that are created from a layout, when restoring from
            // state we don't want to allow them to be created until they are
            // being reloaded from the layout.
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }
            if (f.getAnimatingAway() != null) {
                // The fragment is currently being animated...  but!  Now we
                // want to move our state back up.  Give up on waiting for the
                // animation, move to whatever the final state should be once
                // the animation is done, and then we can proceed from there.
                f.setAnimatingAway(null);
                moveToState(f, f.getStateAfterAnimating(), 0, 0, true);
            }
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {
                        if (DEBUG) Log.v(TAG, "moveto CREATED: " + f);
                        if (f.mSavedFragmentState != null) {
                            f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                    FragmentManagerImpl.VIEW_STATE_TAG);
                            f.mTarget = getFragment(f.mSavedFragmentState,
                                    FragmentManagerImpl.TARGET_STATE_TAG);
                            if (f.mTarget != null) {
                                f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                        FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                            }
                            f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                    FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                            if (!f.mUserVisibleHint) {
                                f.mDeferStart = true;
                                if (newState > Fragment.STOPPED) {
                                    newState = Fragment.STOPPED;
                                }
                            }
                        }

                        f.mHost = mHost;
                        f.mParentFragment = mParent;
                        f.mFragmentManager = mParent != null
                                ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();

                        // If we have a target fragment, push it along to at least CREATED
                        // so that this one can rely on it as an initialized dependency.
                        if (f.mTarget != null) {
                            if (mActive.get(f.mTarget.mIndex) != f.mTarget) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTarget
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (f.mTarget.mState < Fragment.CREATED) {
                                moveToState(f.mTarget, Fragment.CREATED, 0, 0, true);
                            }
                        }

                        dispatchOnFragmentPreAttached(f, mHost.getContext(), false);
                        f.mCalled = false;
						//这里调用Fragment的onAttach
                        f.onAttach(mHost.getContext());
                        if (!f.mCalled) {
                            throw new SuperNotCalledException("Fragment " + f
                                    + " did not call through to super.onAttach()");
                        }
                        if (f.mParentFragment == null) {
                            mHost.onAttachFragment(f);
                        } else {
                            f.mParentFragment.onAttachFragment(f);
                        }
                        dispatchOnFragmentAttached(f, mHost.getContext(), false);
						
						
						//这里调用Fragment的performCreate
                        if (!f.mIsCreated) {
                            dispatchOnFragmentPreCreated(f, f.mSavedFragmentState, false);
                            f.performCreate(f.mSavedFragmentState);
                            dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
                        } else {
                            f.restoreChildFragmentState(f.mSavedFragmentState, true);
                            f.mState = Fragment.CREATED;
                        }
                        f.mRetaining = false;
                    }
                    // fall through
               
        
        if (f.mState != newState) {
            Log.w(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                    + "expected state " + newState + " found " + f.mState);
            f.mState = newState;
        }
    }
```
case Fragment.INITIALIZING:中执行了onAttach和performCreate

```
     void performCreate(Bundle savedInstanceState) {
        if (mChildFragmentManager != null) {
            mChildFragmentManager.noteStateNotSaved();
        }
        mState = CREATED;
        mCalled = false;
        onCreate(savedInstanceState);
        mIsCreated = true;
        if (!mCalled) {
            throw new SuperNotCalledException("Fragment " + this
                    + " did not call through to super.onCreate()");
        }
        final Context context = getContext();
        final int version = context != null ? context.getApplicationInfo().targetSdkVersion : 0;
        if (version < Build.VERSION_CODES.N) {
            restoreChildFragmentState(savedInstanceState, false);
        }
    }
```
熟悉的onCreate就是在这时调用的
再回到moveToState这里对应的几个状态
```
    static final int INITIALIZING = 0;     // Not yet created.
    static final int CREATED = 1;          // Created.
    static final int ACTIVITY_CREATED = 2; // The activity has finished its creation.
    static final int STOPPED = 3;          // Fully created, not started.
    static final int STARTED = 4;          // Created and started, not resumed.
    static final int RESUMED = 5;          // Created started and resumed.

    int mState = INITIALIZING;

```
他会调用Fragment的performXXXX


之前有个问题,fragment不是View,他是怎么和Activity的View绑定的,答案就在moveToState的created状态中
```
case Fragment.CREATED:
                    // This is outside the if statement below on purpose; we want this to run
                    // even if we do a moveToState from CREATED => *, CREATED => CREATED, and
                    // * => CREATED as part of the case fallthrough above.
                    ensureInflatedFragmentView(f);

                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                if (f.mContainerId == View.NO_ID) {
                                    throwException(new IllegalArgumentException(
                                            "Cannot create fragment "
                                                    + f
                                                    + " for a container view with no id"));
                                }
                                container = mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    String resName;
                                    try {
                                        resName = f.getResources().getResourceName(f.mContainerId);
                                    } catch (NotFoundException e) {
                                        resName = "unknown";
                                    }
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                            + Integer.toHexString(f.mContainerId) + " ("
                                            + resName
                                            + ") for fragment " + f));
                                }
                            }
							//上面找到container对应的ViewGroup
                            f.mContainer = container;
							//执行Fragment的onCreateView
                            f.mView = f.performCreateView(f.performGetLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mView.setSaveFromParentEnabled(false);
								//这里!!!都不为空就把view放到container里面
                                if (container != null) {
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) {
                                    f.mView.setVisibility(View.GONE);
                                }
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                                dispatchOnFragmentViewCreated(f, f.mView, f.mSavedFragmentState,
                                        false);
                                // Only animate the view if it is visible. This is done after
                                // dispatchOnFragmentViewCreated in case visibility is changed
                                f.mIsNewlyAdded = (f.mView.getVisibility() == View.VISIBLE)
                                        && f.mContainer != null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
```
