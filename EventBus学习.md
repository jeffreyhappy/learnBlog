## 使用

1. 定义个消息事件
   ```
   public static class MessageEvent { /* Additional fields if needed */ }
   ```
2. 定义个消息接收函数
   ```
   @Subscribe(threadMode = ThreadMode.MAIN)  
    public void onMessageEvent(MessageEvent event) {/* Do something */};
   ```
3. 定义注册和反注册
   ```
    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
   ```
4. 发送消息
   ```
   EventBus.getDefault().post(new MessageEvent());
   ```

## 构造函数

也就是EventBus.getDefault()的时候发生了什么

```
 static volatile EventBus defaultInstance;
 public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```
这是典型的双重校验锁的单例模式,volatile可以防止指令重拍,EventBus也是提供了默认构造函数和有参构造函数
```
    
    public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```
默认构造函数就直接用了有参构造函数,有参构造函数的修饰符是默认的,只能自己类,和同一个包下面访问,正常就只有默认构造函数调用了  

**调用一次构造函数后,会生成一个新的对象,各对象之间的消息是不互通的,所以想要一个全局数据中心的话,就用单例模式来使用**

## 成员变量的意思
EventBus的成员变量初始化是在有参构造函数中进行的,先看看入参
```
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```
EventBusBuilder是个典型的建造者模式,里面有一系列的参数,先都给一个默认值,通过默认配置参数就可以构建,想改变配置有可以

```
public class EventBusBuilder {

    //线程池,newCachedThreadPool会产生一个线程池,线程的核心线程数为0,最大线程数量为Inter.MAX_VALUE
    //意思就是没有工作的时候就没有活动的线程,有任务再创建线程,线程不活动后再销毁
    private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();

    //跟踪了下,如果sendSubscriberExceptionEvent为true,在Event的invokeSubscriber中出错的话,会用logSubscriberExceptions觉得是否打印出来,可以看下面
    boolean logSubscriberExceptions = true;
    //找不到订阅者的时候,是否打印日志
    boolean logNoSubscriberMessages = true;
    //post消息的时候报错后是否发送订阅出错消息
    boolean sendSubscriberExceptionEvent = true;
    //是否找不到订阅者的时候,发送NoSubscriberEvent这个事件
    boolean sendNoSubscriberEvent = true;
    //默认为false,true的话,post异常直接抛出EventBusException,从handleSubscriberExceptionkey看到
    boolean throwSubscriberException;
    //是否往上找,找父类的接收者
    boolean eventInheritance = true;

    //true:只使用反射获取订阅方法.false:通过编译时注解的方法获取订阅对象
    boolean ignoreGeneratedIndex;

    //true严格模式,函数参数只能一个,需要是public ,non-static,non-abstract
    boolean strictMethodVerification;

    //默认的线程次就是DEFAULT_EXECUTOR_SERVICE,用户也可以自己设置
    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;

    //这个看了下 没看到用到的地方
    List<Class<?>> skipMethodVerificationForClasses;
    
    //编译时注解提供的订阅,getSubscriberInfo中如果没找到订阅,就从这个列表里再找找
    List<SubscriberInfoIndex> subscriberInfoIndexes;

    EventBusBuilder() {
    }

    //删除了set方法
    .........



    //通过这个方法也可以直接初始化一个全局的EventBus对象,但是只能初始化一次,不然会报错
    /**
     * Installs the default EventBus returned by {@link EventBus#getDefault()} using this builders' values. Must be
     * done only once before the first usage of the default EventBus.
     *
     * @throws EventBusException if there's already a default EventBus instance in place
     */
    public EventBus installDefaultEventBus() {
        synchronized (EventBus.class) {
            if (EventBus.defaultInstance != null) {
                throw new EventBusException("Default instance already exists." +
                        " It may be only set once before it's used the first time to ensure consistent behavior.");
            }
            EventBus.defaultInstance = build();
            return EventBus.defaultInstance;
        }
    }

    /** Builds an EventBus based on the current configuration. */
    public EventBus build() {
        return new EventBus(this);
    }

}
```

### handleSubscriberException
handleSubscriberException会在post的报错的时候进入,在EventBusBuilder的默认配置中
1. throwSubscriberException没有初始化,所以默认值是false,不会抛出异常
2. logSubscriberExceptions为ture,会打印下日志
3. sendSubscriberExceptionEvent为true,所以构建了SubscriberExceptionEvent并且继续调用了post,然后一路调用还会回到这个方法里,接下来就打印下日志
```
private void handleSubscriberException(Subscription subscription, Object event, Throwable cause) {
        if (event instanceof SubscriberExceptionEvent) {
            if (logSubscriberExceptions) {
                // Don't send another SubscriberExceptionEvent to avoid infinite event recursion, just log
                Log.e(TAG, "SubscriberExceptionEvent subscriber " + subscription.subscriber.getClass()
                        + " threw an exception", cause);
                SubscriberExceptionEvent exEvent = (SubscriberExceptionEvent) event;
                Log.e(TAG, "Initial event " + exEvent.causingEvent + " caused exception in "
                        + exEvent.causingSubscriber, exEvent.throwable);
            }
        } else {
        
            if (throwSubscriberException) {
                throw new EventBusException("Invoking subscriber failed", cause);
            }
            if (logSubscriberExceptions) {
                Log.e(TAG, "Could not dispatch event: " + event.getClass() + " to subscribing class "
                        + subscription.subscriber.getClass(), cause);
            }
            if (sendSubscriberExceptionEvent) {
                SubscriberExceptionEvent exEvent = new SubscriberExceptionEvent(this, cause, event,
                        subscription.subscriber);
                post(exEvent);
            }
        }
    }
```


## 注册
也就是EventBus.getDefault().register(this)的时候发生了什么

```
public void register(Object subscriber) {
        //1 获取subscriber的类
        Class<?> subscriberClass = subscriber.getClass();
        //2 通过类找到订阅者中的订阅方法列表
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            //3 把subscriber的类和订阅方法一一绑定
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
#### 查找类中注解的订阅方法列表

```
//通过类信息去找注解过的方法
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //之前就查找过,缓存里有了,这是个ConcurrentHashMap
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }


        if (ignoreGeneratedIndex) {
            //通过反射来找订阅方法
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

##### 通过反射来找订阅方法
```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        //循环查找,查找完一次后就再找父类的
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            //把clazz设置为自己的父类
            findState.moveToSuperclass();
        }
        //释放查找器
        return getMethodsAndRelease(findState);
    }
```
集中精力看这个findUsingReflectionInSingleClass
1. 通过反射获取方法列表,然后遍历
1. 获取修饰符
1. 判断方法参数只有1个
1. 获取方法的Subscribe注解,如果注解不为空就继续
1. 通过参数,获取注册事件的类信息
1. 检查是否已经添加过
1. 获取注解的ThreadMode成员对象,这个ThreadMode有四个枚举  
   POSTING:订阅者处理的线程就是发送者的线程**这个是默认值**  
   MAIN: 订阅者在主线程处理  
   BACKGROUND: 订阅者在子线程处理,如果发送者是子线程,订阅者直接就在发送者线程处理,如果发送者是主线程,那么就会在子线程顺序处理  
   ASYNC:发送者发完之后,直接就返回了,订阅者是异步处理的,可以处理耗时任务
1. 构建个SubscriberMethod对象,通过订阅着的方法,订阅参数的类,线程模式,优先级,是否是粘性时间
1. 这样就查找好了,然后把查找到的 SubscriberMethod列表对象返回 
```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

看下SubscriberMethod类,就是把订阅的方法,订阅的事件类封装起来
```
public class SubscriberMethod {
    //订阅的方法
    final Method method;
    //线程模式
    final ThreadMode threadMode;
    //事件类,就是订阅方法的参数事件
    final Class<?> eventType;
    //优先级
    final int priority;
    //是否是粘性事件
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;
}
```

#### 不通过反射来查找订阅方法(通过编译时注解)
上面是通过反射,再看看findUsingInfo

```
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //准备查找器
        FindState findState = prepareFindState();
        //初始化
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //看看这个是啥
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //不行就用反射的方法
                findUsingReflectionInSingleClass(findState);
            }
            //把clazz设置为父类,继续查找
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
看看getSubscriberInfo是干啥了
```
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        //如果已经有了信息就直接返回
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }

        //没有就从这个里面找,subscriberInfoIndexes是啥呢?
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```
其实subscriberInfoIndexes是EventBus的编译时注解加进去的
```
/** Adds an index generated by EventBus' annotation preprocessor. */
    public EventBusBuilder addIndex(SubscriberInfoIndex index) {
        if(subscriberInfoIndexes == null) {
            subscriberInfoIndexes = new ArrayList<>();
        }
        subscriberInfoIndexes.add(index);
        return this;
}

```
#### 保存订阅方法
上面找到对应的订阅信息后,然后遍历保存起来,这里涉及了Subscription,Subscription就是封装了订阅者的对象和订阅者中的订阅方法
```
final class Subscription {
    //订阅对象,例如xxxActivity
    final Object subscriber;
    //订阅方法,里面包含了方法,参数,优先级等等
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;
}
```
然后看subscribe方法

```
 // Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //获取订阅事件类信息
        Class<?> eventType = subscriberMethod.eventType;
        //创建subscription对象
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //看看订阅map表里有没有订阅事件类
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        //根据优先级插入下
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        //放到typesBySubscriber中
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
        //如果是粘性时间,再特殊处理下
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
这里涉及到
```
    //key是发送事件的类信息,value是Subscription列表,一个事件类,绑定的订阅信息
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    //key是订阅对象,例如xxxActivity,value是发送事件的类列表,这个订阅对象绑定了多少个事件类
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    //key是发送事件类信息,value是发送事件对象
    private final Map<Class<?>, Object> stickyEvents;

```


这样register方法就完成了

### post发送事件

```
public void post(Object event) {
        //获取当前线程的postingState ,currentPostingThreadState是个ThreadLocal类
        PostingThreadState postingState = currentPostingThreadState.get();
        //加入事件列表
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
        //如果当前不是posting
        if (!postingState.isPosting) {
            //判断是否主线程
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //循环抛出,知道列表为空
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
继续往下postSingleEvent
```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        //获取事件的类信息
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //eventInheritance默认为true
        if (eventInheritance) {

            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```
通过事件,事件类,postingState来找到对应的订阅者
```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        //知道了对应的subscriptions,就可以遍历抛出去
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }

```
根据线程类型抛出去,正常我们就是用MAIN和BACKGROUND
MAIN:如果post线程是main,就直接invokeSubscriber,否则就往主线程队列里塞入
BACKGROUND:如果post线程是main,就往子线程队列里塞入,否则直接调用
```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
最终调用具体方法的地方
```
void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
subscription.subscriber是订阅者对象就是XXXactivity
event就是post的事件对象
```
subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
```
就是XXXactivity调用对应的订阅方法,这样流程就走完了







