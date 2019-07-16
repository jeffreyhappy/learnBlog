看看queryAll生成的代码是啥

```
    @Query("select * from TB_RECORD")
    Flowable<List<RecordEntity>> queryAll();
```
上面的实现
```
@Override
  public Flowable<List<RecordEntity>> queryAll() {
    final String _sql = "select * from TB_RECORD";
    final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 0);
    return RxRoom.createFlowable(__db, new String[]{"TB_RECORD"}, new Callable<List<RecordEntity>>() {
      @Override
      public List<RecordEntity> call() throws Exception {
        final Cursor _cursor = __db.query(_statement);
        try {
          final int _cursorIndexOfNRecordID = _cursor.getColumnIndexOrThrow("nRecordID");
          final int _cursorIndexOfStrUserDomainCode = _cursor.getColumnIndexOrThrow("strUserDomainCode");
          final int _cursorIndexOfStrUserID = _cursor.getColumnIndexOrThrow("strUserID");
          final int _cursorIndexOfStrUserName = _cursor.getColumnIndexOrThrow("strUserName");
          final int _cursorIndexOfStrRecordStartTime = _cursor.getColumnIndexOrThrow("strRecordStartTime");
          final int _cursorIndexOfNRecordDuration = _cursor.getColumnIndexOrThrow("nRecordDuration");
          final int _cursorIndexOfNRecordType = _cursor.getColumnIndexOrThrow("nRecordType");
          final List<RecordEntity> _result = new ArrayList<RecordEntity>(_cursor.getCount());
          while(_cursor.moveToNext()) {
            final RecordEntity _item;
            _item = new RecordEntity();
            _item.nRecordID = _cursor.getLong(_cursorIndexOfNRecordID);
            _item.strUserDomainCode = _cursor.getString(_cursorIndexOfStrUserDomainCode);
            _item.strUserID = _cursor.getString(_cursorIndexOfStrUserID);
            _item.strUserName = _cursor.getString(_cursorIndexOfStrUserName);
            _item.strRecordStartTime = _cursor.getString(_cursorIndexOfStrRecordStartTime);
            _item.nRecordDuration = _cursor.getInt(_cursorIndexOfNRecordDuration);
            _item.nRecordType = _cursor.getInt(_cursorIndexOfNRecordType);
            _result.add(_item);
          }
          return _result;
        } finally {
          _cursor.close();
        }
      }

      @Override
      protected void finalize() {
        _statement.release();
      }
    });
  }
```

主要是RxRoom.createFlowable这个静态方法,入参是RoomDatabase,表名列表,回调
```
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static <T> Flowable<T> createFlowable(final RoomDatabase database,
            final String[] tableNames, final Callable<T> callable) {
        return createFlowable(database, tableNames).observeOn(sAppToolkitIOScheduler)
                .map(new Function<Object, Optional<T>>() {
                    @Override
                    public Optional<T> apply(@NonNull Object o) throws Exception {
                        T data = callable.call();
                        return new Optional<>(data);
                    }
                }).filter(new Predicate<Optional<T>>() {
                    @Override
                    public boolean test(@NonNull Optional<T> optional) throws Exception {
                        return optional.mValue != null;
                    }
                }).map(new Function<Optional<T>, T>() {
                    @Override
                    public T apply(@NonNull Optional<T> optional) throws Exception {
                        return optional.mValue;
                    }
                });
    }
```
1. createFlowable(database, tableNames)创建一个Flowable
2. observeOn(sAppToolkitIOScheduler)观察线程在子线程
3. .map 调用callable这个回调,并放入Optional,这个就是个包装类
    ```
    static class Optional<T> {
            @Nullable
            final T mValue;

            Optional(@Nullable T value) {
                this.mValue = value;
            }
        }
    ```  
4. .filter 过滤下,如果包装的value为null就不继续往下走了,只有非Null的可以继续
5. .map 再取出value的值
它这一套操作就是调用callable然后剔除掉为null的结果
看看重载的createFlowable(database, tableNames)
```
public static Flowable<Object> createFlowable(final RoomDatabase database,
            final String... tableNames) {
        return Flowable.create(new FlowableOnSubscribe<Object>() {
            @Override
            public void subscribe(final FlowableEmitter<Object> emitter) throws Exception {
                final InvalidationTracker.Observer observer = new InvalidationTracker.Observer(
                        tableNames) {
                    @Override
                    public void onInvalidated(
                            @android.support.annotation.NonNull Set<String> tables) {
                        if (!emitter.isCancelled()) {
                            emitter.onNext(NOTHING);
                        }
                    }
                };
                if (!emitter.isCancelled()) {
                    database.getInvalidationTracker().addObserver(observer);
                    emitter.setDisposable(Disposables.fromAction(new Action() {
                        @Override
                        public void run() throws Exception {
                            database.getInvalidationTracker().removeObserver(observer);
                        }
                    }));
                }

                // emit once to avoid missing any data and also easy chaining
                if (!emitter.isCancelled()) {
                    emitter.onNext(NOTHING);
                }
            }
        }, BackpressureStrategy.LATEST);
    }
```
我看下的意思就是新建了个无效观察器,如果数据库无效了就发送个NOTHING
如果有效的话,会在订阅的时候调用call,看看最上面的call就是调用数据库的query,然后用游标读取数据然后返回


再看看插入

```
  @Override
  public void insertRecord(RecordEntity recordEntity) {
    __db.beginTransaction();
    try {
      __insertionAdapterOfRecordEntity.insert(recordEntity);
      __db.setTransactionSuccessful();
    } finally {
      __db.endTransaction();
    }
  }
```
这个是个EntityInsertionAdapter,




先看看RoomDatabase,这是个抽象类,注释说这个可以直接访问数据库,但是还是建议优先使用dao类来访问

成员对象不多
```
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static final int MAX_BIND_PARAMETER_CNT = 999;
    // set by the generated open helper.
    protected volatile SupportSQLiteDatabase mDatabase;
    private SupportSQLiteOpenHelper mOpenHelper;
    private final InvalidationTracker mInvalidationTracker;
    private boolean mAllowMainThreadQueries;
    boolean mWriteAheadLoggingEnabled;

    @Nullable
    protected List<Callback> mCallbacks;

    private final ReentrantLock mCloseLock = new ReentrantLock();

```

### SupportSQLiteDatabase
是个数据库的抽象,就可以移除掉框架的依赖,是模仿的android.database.sqlite.SQLiteDatabase
抽象方法和SQLiteDatabase一样

### SupportSQLiteOpenHelper
就跟android.database.sqlite.SQLiteOpenHelper一样,
```
    SupportSQLiteDatabase getWritableDatabase();
    SupportSQLiteDatabase getReadableDatabase();
```
比较让我瞩目的是这里有个Callback
这个Callback提供了很多回调
```
public abstract void onUpgrade(SupportSQLiteDatabase db, int oldVersion, int newVersion);
public void onDowngrade(SupportSQLiteDatabase db, int oldVersion, int newVersion) 
public void onCorruption(SupportSQLiteDatabase db) 
```

查找了下SupportSQLiteOpenHelper的实现,找到了FrameworkSQLiteOpenHelper
```
 FrameworkSQLiteOpenHelper(Context context, String name,
            Callback callback) {
        mDelegate = createDelegate(context, name, callback);
    }

    private OpenHelper createDelegate(Context context, String name, Callback callback) {
        final FrameworkSQLiteDatabase[] dbRef = new FrameworkSQLiteDatabase[1];
        return new OpenHelper(context, name, dbRef, callback);
    }
    
    
    @Override
    public SupportSQLiteDatabase getWritableDatabase() {
        return mDelegate.getWritableSupportDatabase();
    }

    @Override
    public SupportSQLiteDatabase getReadableDatabase() {
        return mDelegate.getReadableSupportDatabase();
    }
```
之后的一些实现都是基于mDelgate,这个就是代理设计模式吧,这个OpenHelper具体使用的啥呢?就是继承了SQLiteOpenHelper
```
static class OpenHelper extends SQLiteOpenHelper {
        /**
         * This is used as an Object reference so that we can access the wrapped database inside
         * the constructor. SQLiteOpenHelper requires the error handler to be passed in the
         * constructor.
         */
        final FrameworkSQLiteDatabase[] mDbRef;
        final Callback mCallback;
        // see b/78359448
        private boolean mMigrated;

        OpenHelper(Context context, String name, final FrameworkSQLiteDatabase[] dbRef,
                final Callback callback) {
            super(context, name, null, callback.version,
                    new DatabaseErrorHandler() {
                        @Override
                        public void onCorruption(SQLiteDatabase dbObj) {
                            FrameworkSQLiteDatabase db = dbRef[0];
                            if (db != null) {
                                callback.onCorruption(db);
                            }
                        }
                    });
            mCallback = callback;
            mDbRef = dbRef;
        }

        synchronized SupportSQLiteDatabase getWritableSupportDatabase() {
            mMigrated = false;
            SQLiteDatabase db = super.getWritableDatabase();
            if (mMigrated) {
                // there might be a connection w/ stale structure, we should re-open.
                close();
                return getWritableSupportDatabase();
            }
            return getWrappedDb(db);
        }

        synchronized SupportSQLiteDatabase getReadableSupportDatabase() {
            mMigrated = false;
            SQLiteDatabase db = super.getReadableDatabase();
            if (mMigrated) {
                // there might be a connection w/ stale structure, we should re-open.
                close();
                return getReadableSupportDatabase();
            }
            return getWrappedDb(db);
        }

        FrameworkSQLiteDatabase getWrappedDb(SQLiteDatabase sqLiteDatabase) {
            FrameworkSQLiteDatabase dbRef = mDbRef[0];
            if (dbRef == null) {
                dbRef = new FrameworkSQLiteDatabase(sqLiteDatabase);
                mDbRef[0] = dbRef;
            }
            return mDbRef[0];
        }

        @Override
        public void onCreate(SQLiteDatabase sqLiteDatabase) {
            mCallback.onCreate(getWrappedDb(sqLiteDatabase));
        }

        @Override
        public void onUpgrade(SQLiteDatabase sqLiteDatabase, int oldVersion, int newVersion) {
            mMigrated = true;
            mCallback.onUpgrade(getWrappedDb(sqLiteDatabase), oldVersion, newVersion);
        }

        @Override
        public void onConfigure(SQLiteDatabase db) {
            mCallback.onConfigure(getWrappedDb(db));
        }

        @Override
        public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
            mMigrated = true;
            mCallback.onDowngrade(getWrappedDb(db), oldVersion, newVersion);
        }

        @Override
        public void onOpen(SQLiteDatabase db) {
            if (!mMigrated) {
                // if we've migrated, we'll re-open the db so we  should not call the callback.
                mCallback.onOpen(getWrappedDb(db));
            }
        }

        @Override
        public synchronized void close() {
            super.close();
            mDbRef[0] = null;
        }
    }
```
当收到回调的时候,调用callback抛出去,这样就能实现状态的转移
回到Room的初始化,是通过建造者模式构建的
```
db = Room.databaseBuilder(context, AppDatabase.class, dbFileName)
                    .fallbackToDestructiveMigration()
                    .allowMainThreadQueries()
                    .build();
```
看下build的内容
```
public T build() {
            //noinspection ConstantConditions
            if (mContext == null) {
                throw new IllegalArgumentException("Cannot provide null context for the database.");
            }
            //noinspection ConstantConditions
            if (mDatabaseClass == null) {
                throw new IllegalArgumentException("Must provide an abstract class that"
                        + " extends RoomDatabase");
            }

            if (mMigrationStartAndEndVersions != null && mMigrationsNotRequiredFrom != null) {
                for (Integer version : mMigrationStartAndEndVersions) {
                    if (mMigrationsNotRequiredFrom.contains(version)) {
                        throw new IllegalArgumentException(
                                "Inconsistency detected. A Migration was supplied to "
                                        + "addMigration(Migration... migrations) that has a start "
                                        + "or end version equal to a start version supplied to "
                                        + "fallbackToDestructiveMigrationFrom(int... "
                                        + "startVersions). Start version: "
                                        + version);
                    }
                }
            }

            if (mFactory == null) {
                mFactory = new FrameworkSQLiteOpenHelperFactory();
            }
            DatabaseConfiguration configuration =
                    new DatabaseConfiguration(mContext, mName, mFactory, mMigrationContainer,
                            mCallbacks, mAllowMainThreadQueries,
                            mJournalMode.resolve(mContext),
                            mRequireMigration, mMigrationsNotRequiredFrom);
            T db = Room.getGeneratedImplementation(mDatabaseClass, DB_IMPL_SUFFIX);
            db.init(configuration);
            return db;
        }
```
我们正常都不会设置mFactory,所以他就用的FrameworkSQLiteOpenHelperFactory,这样就串起来了
```
if (mFactory == null) {
      mFactory = new FrameworkSQLiteOpenHelperFactory();
}
```
这个就是根据对应类+DB_IMPL_SUFFIX去找生成的实现类
```
T db = Room.getGeneratedImplementation(mDatabaseClass, DB_IMPL_SUFFIX);
```
我们的数据库类都是抽象类例如
```
@Database(entities = {ExampleEntity.class}, version = AppDatas.EXTERNAL_DB_VERSION_CODE)
public abstract class AppDatabase extends RoomDatabase {
    public abstract ExampleDao exampleDao();
}
```
生成的类如
```
public class AppDatabase_Impl extends AppDatabase 
```

关键点在db.init里面
```
public void init(@NonNull DatabaseConfiguration configuration) {
        mOpenHelper = createOpenHelper(configuration);
        boolean wal = false;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            wal = configuration.journalMode == JournalMode.WRITE_AHEAD_LOGGING;
            mOpenHelper.setWriteAheadLoggingEnabled(wal);
        }
        mCallbacks = configuration.callbacks;
        mAllowMainThreadQueries = configuration.allowMainThreadQueries;
        mWriteAheadLoggingEnabled = wal;
}
```
在createOpenHelper里面调用了
```
final SupportSQLiteOpenHelper _helper = configuration.sqliteOpenHelperFactory.create(_sqliteConfig);
return _helper;
```
这样就建好了



至于配合LiveData可以实现,数据插入后自动通知数据变动,先要研究下LiveData,了解LiveData后再来看看
看下queryAll的实现
```
@Override
  public LiveData<List<ChatMessageBean>> queryAll(String meetID) {
    final String _sql = "select * from tb_chat_message where groupID=? order by insertTime";
    final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 1);
    int _argIndex = 1;
    if (meetID == null) {
      _statement.bindNull(_argIndex);
    } else {
      _statement.bindString(_argIndex, meetID);
    }
    return new ComputableLiveData<List<ChatMessageBean>>() {
      private Observer _observer;

      @Override
      protected List<ChatMessageBean> compute() {
        if (_observer == null) {
          _observer = new Observer("tb_chat_message") {
            @Override
            public void onInvalidated(@NonNull Set<String> tables) {
              invalidate();
            }
          };
          __db.getInvalidationTracker().addWeakObserver(_observer);
        }
        ....
          return _result;
        } finally {
          _cursor.close();
        }
      }

      @Override
      protected void finalize() {
        _statement.release();
      }
    }.getLiveData();
  }
```

重点看ComputableLiveData的实现,其中有
```
 if (_observer == null) {
          _observer = new Observer("tb_chat_message") {
            @Override
            public void onInvalidated(@NonNull Set<String> tables) {
              invalidate();
            }
          };
          __db.getInvalidationTracker().addWeakObserver(_observer);
    }
```
关注InvalidationTracker

```
   @WorkerThread
    public void addObserver(@NonNull Observer observer) {
        final String[] tableNames = observer.mTables;
        int[] tableIds = new int[tableNames.length];
        final int size = tableNames.length;
        long[] versions = new long[tableNames.length];

        // TODO sync versions ?
        for (int i = 0; i < size; i++) {
            Integer tableId = mTableIdLookup.get(tableNames[i].toLowerCase(Locale.US));
            if (tableId == null) {
                throw new IllegalArgumentException("There is no table with name " + tableNames[i]);
            }
            tableIds[i] = tableId;
            versions[i] = mMaxVersion;
        }
        ObserverWrapper wrapper = new ObserverWrapper(observer, tableIds, tableNames, versions);
        ObserverWrapper currentObserver;
        synchronized (mObserverMap) {
            currentObserver = mObserverMap.putIfAbsent(observer, wrapper);
        }
        if (currentObserver == null && mObservedTableTracker.onAdded(tableIds)) {
            syncTriggers();
        }
    }
```
重点就在syncTriggers,会在beginTransaction的时候被调用
```
    /**
     * Called by RoomDatabase before each beginTransaction call.
     * <p>
     * It is important that pending trigger changes are applied to the database before any query
     * runs. Otherwise, we may miss some changes.
     * <p>
     * This api should eventually be public.
     */
    void syncTriggers() {
        if (!mDatabase.isOpen()) {
            return;
        }
        syncTriggers(mDatabase.getOpenHelper().getWritableDatabase());
    }


    void syncTriggers(SupportSQLiteDatabase database) {
        if (database.inTransaction()) {
            // we won't run this inside another transaction.
            return;
        }
        try {
            // This method runs in a while loop because while changes are synced to db, another
            // runnable may be skipped. If we cause it to skip, we need to do its work.
            while (true) {
                Lock closeLock = mDatabase.getCloseLock();
                closeLock.lock();
                try {
                    // there is a potential race condition where another mSyncTriggers runnable
                    // can start running right after we get the tables list to sync.
                    final int[] tablesToSync = mObservedTableTracker.getTablesToSync();
                    if (tablesToSync == null) {
                        return;
                    }
                    final int limit = tablesToSync.length;
                    try {
                        database.beginTransaction();
                        for (int tableId = 0; tableId < limit; tableId++) {
                            switch (tablesToSync[tableId]) {
                                case ObservedTableTracker.ADD:
                                    startTrackingTable(database, tableId);
                                    break;
                                case ObservedTableTracker.REMOVE:
                                    stopTrackingTable(database, tableId);
                                    break;
                            }
                        }
                        database.setTransactionSuccessful();
                    } finally {
                        database.endTransaction();
                    }
                    mObservedTableTracker.onSyncCompleted();
                } finally {
                    closeLock.unlock();
                }
            }
        } catch (IllegalStateException | SQLiteException exception) {
            // may happen if db is closed. just log.
            Log.e(Room.LOG_TAG, "Cannot run invalidation tracker. Is the db closed?",
                    exception);
        }
    }
```
接着就到了startTrackingTable,这里面新建了个触发器,原来sqlite也是支持触发器的
```
private void startTrackingTable(SupportSQLiteDatabase writableDb, int tableId) {
        final String tableName = mTableNames[tableId];
        StringBuilder stringBuilder = new StringBuilder();
        for (String trigger : TRIGGERS) {
            stringBuilder.setLength(0);
            stringBuilder.append("CREATE TEMP TRIGGER IF NOT EXISTS ");
            appendTriggerName(stringBuilder, tableName, trigger);
            stringBuilder.append(" AFTER ")
                    .append(trigger)
                    .append(" ON `")
                    .append(tableName)
                    .append("` BEGIN INSERT OR REPLACE INTO ")
                    .append(UPDATE_TABLE_NAME)
                    .append(" VALUES(null, ")
                    .append(tableId)
                    .append("); END");
            writableDb.execSQL(stringBuilder.toString());
        }
    }
```
会在表更新,删除,插入的时候生成名为   room_table_modification_trigger_表名_触发类型 的这个条目(这个sql有点没看懂,可能有误)

目光回到endTransaction
```
 public void endTransaction() {
        mOpenHelper.getWritableDatabase().endTransaction();
        if (!inTransaction()) {
            // enqueue refresh only if we are NOT in a transaction. Otherwise, wait for the last
            // endTransaction call to do it.
            mInvalidationTracker.refreshVersionsAsync();
        }
    }


public void refreshVersionsAsync() {
        // TODO we should consider doing this sync instead of async.
        if (mPendingRefresh.compareAndSet(false, true)) {
            ArchTaskExecutor.getInstance().executeOnDiskIO(mRefreshRunnable);
        }
}
```
在mRefreshRunnable中有个checkUpdatedTable方法
```
Runnable mRefreshRunnable = new Runnable() {
        @Override
        public void run() {
            final Lock closeLock = mDatabase.getCloseLock();
            boolean hasUpdatedTable = false;
            try {
                closeLock.lock();

             ......
                if (mDatabase.mWriteAheadLoggingEnabled) {
                    // This transaction has to be on the underlying DB rather than the RoomDatabase
                    // in order to avoid a recursive loop after endTransaction.
                    SupportSQLiteDatabase db = mDatabase.getOpenHelper().getWritableDatabase();
                    try {
                        db.beginTransaction();
                        hasUpdatedTable = checkUpdatedTable();
                        db.setTransactionSuccessful();
                    } finally {
                        db.endTransaction();
                    }
                } else {
                    hasUpdatedTable = checkUpdatedTable();
                }
        
            if (hasUpdatedTable) {
                synchronized (mObserverMap) {
                    for (Map.Entry<Observer, ObserverWrapper> entry : mObserverMap) {
                        entry.getValue().checkForInvalidation(mTableVersions);
                    }
                }
            }
        }

        private boolean checkUpdatedTable() {
            boolean hasUpdatedTable = false;
            Cursor cursor = mDatabase.query(SELECT_UPDATED_TABLES_SQL, mQueryArgs);
            //noinspection TryFinallyCanBeTryWithResources
            try {
                while (cursor.moveToNext()) {
                    final long version = cursor.getLong(0);
                    final int tableId = cursor.getInt(1);

                    mTableVersions[tableId] = version;
                    hasUpdatedTable = true;
                    // result is ordered so we can safely do this assignment
                    mMaxVersion = version;
                }
            } finally {
                cursor.close();
            }
            return hasUpdatedTable;
        }
    };
```
流程就是事务结束的时候,拉起一个线程检查下是否有table更新,就是通过之前触发器来判断是否有新的更新,如果有新的更新,就调用checkForInvalidation,最终调用mObserver.onInvalidated(invalidatedTables);这就返回到了queryAll里面,
```
        if (_observer == null) {
          _observer = new Observer("tb_chat_message") {
            @Override
            public void onInvalidated(@NonNull Set<String> tables) {
              invalidate();
            }
          };
          __db.getInvalidationTracker().addWeakObserver(_observer);
        }
```
invalidate会调用ComputableLiveData的compute方法,并抛出最新的数据,这样就可以实现数据的插入通知观察方法