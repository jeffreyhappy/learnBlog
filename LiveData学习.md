## 基础使用
#### gradle依赖
```
    def lifecycle_version = "2.0.0"

    // ViewModel and LiveData
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
```
#### ViewModel
ViewModel可以感知Activity的生命周期,可以方便的分离数据的所有者,简单使用如下
```
public class NumberModel extends ViewModel {
    
    private MutableLiveData<Integer> numbers;
    private int num;
    
    public LiveData<Integer> getNumbers(){
        if (numbers == null){
            numbers = new MutableLiveData<>();
            num = 0;
        }
        return numbers;
    }
    
    public void add(){
        num++;
        numbers.postValue(num);
    }

    public void minus(){
        num--;
        numbers.postValue(num);

    }
}
```

对于Activity的代码
```
public class MainActivity extends FragmentActivity {

    NumberModel numberModel;
    TextView tvNum;
    Button btnAdd;
    Button btnMinus;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tvNum = findViewById(R.id.tv);
        btnAdd = findViewById(R.id.btn_add);
        btnMinus = findViewById(R.id.btn_minus);


        numberModel = ViewModelProviders.of(this).get(NumberModel.class);
        numberModel.getNumbers().observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(Integer integer) {
                tvNum.setText(integer+"");
            }
        });

        btnAdd.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                numberModel.add();
            }
        });

        btnMinus.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                numberModel.minus();
            }
        });
    }
}
```

### 深入了解下
看下ViewModel的创建
```
        ViewModelProvider viewModelProvider = ViewModelProviders.of(this);
        numberModel = viewModelProvider.get(NumberModel.class);

```
#### of

```
  public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        //1 确认activity/fragment已经被加载入application        
        Application application = checkApplication(activity);
        //获取创建单例,可以使用系统的工厂,也可以自定义工厂
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        //创建ViewModelProvider
        return new ViewModelProvider(activity.getViewModelStore(), factory);
    }
```
上面的AndroidViewModelFactory是一个单例工厂,用来创建ViewModel的所在
```
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

        private static AndroidViewModelFactory sInstance;

        /**
         * Retrieve a singleton instance of AndroidViewModelFactory.
         *
         * @param application an application to pass in {@link AndroidViewModel}
         * @return A valid {@link AndroidViewModelFactory}
         */
        @NonNull
        public static AndroidViewModelFactory getInstance(@NonNull Application application) {
            if (sInstance == null) {
                sInstance = new AndroidViewModelFactory(application);
            }
            return sInstance;
        }

        private Application mApplication;

        /**
         * Creates a {@code AndroidViewModelFactory}
         *
         * @param application an application to pass in {@link AndroidViewModel}
         */
        public AndroidViewModelFactory(@NonNull Application application) {
            mApplication = application;
        }

        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
                //noinspection TryWithIdenticalCatches
                try {
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (IllegalAccessException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InstantiationException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InvocationTargetException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                }
            }
            return super.create(modelClass);
        }
    }
```


#### ViewModelProvider
这是一个工具类,给ViewModel一个范围,默认的activity或者fragment的ViewModelProvider可以从androidx.lifecycle.ViewModelProviders获取

获取对应的ViewModel,就是通过get
```
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

     @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }

        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```
通过ViewModelStore来获取,找到了就返回,不然就创建一下,然后再put进去
问:如何多次get只获取一个对象呢?
像这种多个Fragment,共享一个Model的情况
```
public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // Update the UI.
        });
    }
}
```
1. get是从ViewModelStore获取的对象,多次获取ViewModel肯定是同一个,所以mViewModelStore肯定是同一个

```
@MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }

        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
```
2. mViewModelStore是构造函数的入参,看of中,ViewModelStore是获取的activity.getViewModelStore()
```   
    public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(activity.getViewModelStore(), factory);
    }
```
3. getViewModelStore做了啥?  
如果mViewModelStore为null就创建,不然就直接返回,这里就保证了一个Activity就只持有一个ViewModelStore,
```
 public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
```  
4. 一个activity就只有一个ViewModelStore,用这个ViewModelStore来存储已经创建的ViewModel,所以多次获取ViewModel都是获取的同一个对象

## ViewModel如何销毁
ViewModel由ViewModelStore管理,在FragmentActivity的代码中,在onCreate的创建了
```
 protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.attachHost(null /*parent*/);

        super.onCreate(savedInstanceState);

        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null && nc.viewModelStore != null && mViewModelStore == null) {
            mViewModelStore = nc.viewModelStore;
        }
 }
```

在onDestory中调用了ViewModelStore的clear
```
    @Override
    protected void onDestroy() {
        super.onDestroy();

        if (mViewModelStore != null && !isChangingConfigurations()) {
            mViewModelStore.clear();
        }

        mFragments.dispatchDestroy();
    }
```
viewModelStore的clear就是调用把Map表里的所有ViewMoel的onCleared,并清除map表
```
public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.onCleared();
        }
        mMap.clear();
    }
```
所以自定义ViewModel的时候,onCleared一定要重写,并清除一些引用

## LiveData的生命周期
ViewMode的销毁我们知道了,是在Activity的OnDestory里,但是正常使用的LiveData的时候,也需要传入LifeCycle相关的东西
例如:
```
numberModel.getNumbers().observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(Integer integer) {
                tvNum.setText(integer+"");
            }
});
```
### observer
1. 当前生命周期DESTROYED就直接返回什么都不做了
2. 将owner和observer包装起来
3. 放进去
```
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
```

#### LifecycleBoundObserver

LifecycleBoundObserver继承了ObserverWrapper,实现了GenericLifecycleObserver
```
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }
```
GenericLifecycleObserver接口就一个方法onStateChanged,这个会在生命周期变动的时候回调,看看LifecycleBoundObserver这个的实现
1. 如果DESTROYED,销毁了就移除观察者,否则就走activeStateChanged(shouldBeActive());
2. shouldBeActive返回true的条件是当前的状态要大于等于start,对于activity的创建来说的话 onCreate,onStart,onResume,就从onStart开始就可以激活了
3. activieStateChanged是ObserverWrapper的方法,看看ObserverWrapper
```
private abstract class ObserverWrapper {
        final Observer<? super T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<? super T> observer) {
            mObserver = observer;
        }

        abstract boolean shouldBeActive();

        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }

        void detachObserver() {
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            if (wasInactive && mActive) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            if (mActive) {
                dispatchingValue(this);
            }
        }
    }
```
4. ObserverWrapper就是包装了Observer,含有一个是否激活的状态,mLastVersion是记录数据改变的次数,如果数据变了就+1,为了防止同次数据多次抛出的

