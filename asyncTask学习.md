AsyncTask是系统提供可以方便切换线程的类,学习下里面的代码熟悉下线程切换

官方文档给的示例
```
 private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
     protected Long doInBackground(URL... urls) {
         int count = urls.length;
         long totalSize = 0;
         for (int i = 0; i < count; i++) {
             totalSize += Downloader.downloadFile(urls[i]);
             publishProgress((int) ((i / (float) count) * 100));
             // Escape early if cancel() is called
             if (isCancelled()) break;
         }
         return totalSize;
     }

     protected void onProgressUpdate(Integer... progress) {
         setProgressPercent(progress[0]);
     }

     protected void onPostExecute(Long result) {
         showDialog("Downloaded " + result + " bytes");
     }
 }
 
```
然后调用如下
```
 new DownloadFilesTask().execute(url1, url2, url3);
```
三个泛型依次是输入类型,进度类型,结果类型
doInBackground是在子线程,onProgressUpdate和onPostExecute在主线程,所以可以很方便的实现下载时在界面上显示进度

构造函数
```
    public AsyncTask() {
        this((Looper) null);
    }

	
    public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

```
默认构造函数创建的mHandler就是通过mainLooper创建的
```
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
```
InternalHandler就是从子线程切换到主线的关键,当子线程要通知主线程的时候,就发个消息然后InternalHandler就调用对应的方法
```
private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
mWorker就如他的名字一样,是用来完成工作流程的,先把参数给doInBackground,然后等待执行完成后postResult,它继承了Callable接口,这个接口和runnable接口很像,只是他有返回
```
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
}

public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

```

还有个mFuture,FutureTask继承了Future和Runnable,是一个可以取消的异步计算类,他包装一个Callable类,其实就和Thread,Runnable的关系一样
```
		mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
这里重写了done方法,为什么?我试验了下,如果任务被取消了mWorker可能不调用postResut方法,对于AsyncTask来说流程没走完,如果要最后确认下是否走完了,在done里面有个get()
这个get会获取callable的返回,postResultIfNotInvoked就如名字一样,抛出过了就不抛了,如果任务被取消了的话CancellationException会抓到,然后调用postResult(null),这样流程就完整了
构造函数完成了,不过类的初始化还没完成,类的初始化应该是类的static成员对象先调用,AsyncTask就有static成员对象
一看吓一跳 ,还蛮多的,这个主要是为了不要重复创建线程
```
	//核心线程的大小最小2,最大4
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    //最大线程数,cpu的数量*2+1
	private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    //线程空闲后,最多存活多久
	private static final int KEEP_ALIVE_SECONDS = 30;

	//线程的创建工厂,作用就是Thread命名下
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

	//存放线程的队列,最大128个等待任务,如果一下放太多,就爆掉了
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;
	
	//创建线程池
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

	//创建每次只跑一个任务的Executor,AsyncTask的exec调用的就是它
    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;
	//看,默认Executor
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static InternalHandler sHandler;
```
再看下SerialExecutor,每次execute的时候,就放到mTasks里,new一个Runnable包装r,然后放到队列的尾部,如果当前的mActivity为空的话就调用scheduleNext,scheduleNext会从头拿一个runnable出来丢给线程池运行
sDefaultExecutor就是这个SerialExecutor类,并且他还是个static,对于这个jvm就这一个执行者,所以就是通过这个来实现了AsyncTask的顺序执行
为何要顺序执行,可能是因为防止大量使用AsyncTask执行耗时任务,影响性能吧
```
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
看下执行,默认就是用的SerialExecutor顺序执行的,参数来了,给任务赋值下运行参数,然后丢给Exector执行
```
	public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
	
	public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
最后附上我模仿写的简陋版AsyncTask
```
public abstract class MyTask<Params,Progress,Result>{

    private final int POST_PROGRESS = 1;
    private final int POST_RESULT   = 2;
    private Params[] params;
    //这个用来在子线程运行的,还需要个handler往主线程回抛
    //这个不是按顺序运行的,等下需要改下
    private static ThreadPoolExecutor threadPoolExecutor;

    //这个handler往主线程抛
    private Handler handler;

    static {
        int process = Runtime.getRuntime().availableProcessors();
        int coreNum = Math.max(2,Math.min(4,process-1));
        BlockingQueue<Runnable> blockingQueue = new LinkedBlockingQueue<>(100);
        threadPoolExecutor = new ThreadPoolExecutor(coreNum, 100, 30, TimeUnit.SECONDS, blockingQueue, new ThreadFactory() {
            AtomicInteger atomicInteger = new AtomicInteger();
            @Override
            public Thread newThread(@NonNull Runnable r) {
                return new Thread(r,"MyTask# " + atomicInteger.incrementAndGet());
            }
        });
        threadPoolExecutor.allowCoreThreadTimeOut(true);

    }
    //线程池是共享的,但一个Async只运行一个任务,所以futureTask在初始化的时候赋值
    FutureTask<Result> futureTask;
    Callable<Result> workCallable;

    void exec(Params... params){
        //需要一个task,来完成流程
        //AsyncTask如何监听任务的状态的?用FutureTask

        this.params = params;
        threadPoolExecutor.execute(futureTask);
    }


    public MyTask(){
        Looper looper = Looper.myLooper();
        if (looper == null){
            looper = Looper.getMainLooper();
        }
        handler = new Handler(looper){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what){
                    case POST_PROGRESS:
                        onProgress((Progress) msg.obj);
                        break;
                    case POST_RESULT:
                        onResult((Result) msg.obj);
                        break;
                }
            }
        };
        workCallable = new Callable<Result>() {
            @Override
            public Result call() throws Exception {
                Result result = null;
                try {
                    result =  doInBackground(params);
                }catch (Throwable t){
                    Log.d("MyTask","workCallable throwable " + t);
                }finally {
                    postResult(result);
                }
                return result;
            }
        };
        futureTask = new FutureTask<Result>(workCallable){
            @Override
            protected void done() {
                try {
                   Result result =   get();
                } catch (ExecutionException e) {
                    Log.d("MyTask","futureTask done ExecutionException " + e);
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    Log.d("MyTask","futureTask done InterruptedException " + e);
                    e.printStackTrace();
                } catch (CancellationException e){
                    Log.d("MyTask","futureTask done CancellationException " + e);
                    e.printStackTrace();
                }
                Log.d("MyTask","futureTask done");

                super.done();
            }
        };
    }


    abstract void  onProgress(Progress progress);

    abstract Result doInBackground(Params... params);
    abstract void onResult(Result result);

    protected void postProgress(Progress progress){
        handler.obtainMessage(POST_PROGRESS,progress).sendToTarget();
    }

    private void postResult(Result result){
        handler.obtainMessage(POST_RESULT,result).sendToTarget();

    }
    public void cancel(){
        futureTask.cancel(true);
    }

}
```
测试
```
		myTask = new MyTask<Integer, Integer, Integer>() {
            @Override
            void onProgress(Integer integer) {
                Log.d("MyTask","onProgress " + Thread.currentThread().getName());
            }

            @Override
            Integer doInBackground(Integer... params) {
                Log.d("MyTask","doInBackground " +Thread.currentThread().getName());
                int result = 0;
                for (int i = 0 ; i < 10 ; i++){
                    postProgress(i);
                }
                return result;
            }

            @Override
            void onResult(Integer integer) {
                Log.d("MyTask","onResult " + Thread.currentThread().getName());
            }
        };
        myTask.exec(1,2);
```


