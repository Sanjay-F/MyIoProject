title: JetPack源码探索系列4---WorkManager

date: 2018-09-05 20:30:41

tags: [android,JetPack,WorkManager]

categories: android
 

------------------------------------------
 

# WorkManager

The WorkManager API makes it easy to specify deferrable, asynchronous tasks and when they should run. These APIs let you create a task and hand it off to WorkManager to run immediately or at an appropriate time.

在早期，为了做一些延迟定时的任务，做了个基于闹钟的定时任务队列SchedulerQueue。现在谷歌官方帮我们把一切都封装好了。后续你不用再自己造轮子了.像这样的轮子在后台等都有很完全的库了，只是安卓系统支持不太好，搞到现在才出一个吧。

整体原理如下
 ![](https://i.imgur.com/jGx5ihc.png)

使用流程如下：![](https://i.imgur.com/4G2rqGk.png)

<!--more-->

# 起航

下面缩写成WorkManager为WM,然后我们来看下个demo：
	
		OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(MyWorker.class).setInitialDelay(100, TimeUnit.SECONDS).build();
        WorkManager.getInstance().enqueue(request);
			
		public class MyWorker extends Worker {
        
	        public MyWorker(@NonNull Context context, @NonNull WorkerParameters workerParams) {
	            super(context, workerParams);
	        }
	 
	        @Override
	        public Result doWork() {
	             // do something
	            return null;
	        }
	    }

这是一个最简单的延时100s然后启动执行的小demo，我们用这个来做探索：
在详细的进去看源码前，还是建议继续看下文档的内容：

	/**
	 * WorkManager is a library used to enqueue deferrable work that is guaranteed to execute sometime
	 * after its {@link Constraints} are met.  WorkManager allows observation of work status and the
	 * ability to create complex chains of work.

	 * 类是RX可以弄成一堆链式的条子一样，顺序执行一堆任务。定时启动，监听状态，功能非常强大.我们后面再看下是如何做到链式执行任务的
	 * <p>
	 * WorkManager uses an underlying job dispatching service when available based on the following
	 * criteria:
	 * <p><ul>
	 * <li>Uses JobScheduler for API 23+
	 * <li>Uses a custom AlarmManager + BroadcastReceiver implementation for API 14-22</ul>
	 * <p>
	 * All work must be done in a {@link ListenableWorker} class.  A simple implementation,
	 * {@link Worker}, is recommended as the starting point for most developers.  With the optional
	 * dependencies, you can also use {@code CoroutineWorker} or {@code RxWorker}.
	 * <p>
	 * 
	 * There are two types of work supported by WorkManager: {@link OneTimeWorkRequest} and
	 * {@link PeriodicWorkRequest}.  You can enqueue requests using WorkManager as follows:
	 *
		=====只有两种类型的WORK
	 * <pre>
	 * {@code
	 * WorkManager workManager = WorkManager.getInstance();
	 * workManager.enqueue(new OneTimeWorkRequest.Builder(FooWorker.class).build());}</pre>
	 *
	 * A {@link WorkRequest} has an associated id that can be used for lookups and observation as
	 * follows:
	 *
	 * <pre>
	 * {@code
	 * WorkRequest request = new OneTimeWorkRequest.Builder(FooWorker.class).build();
	 * workManager.enqueue(request);
	 * LiveData<WorkInfo> status = workManager.getWorkInfoByIdLiveData(request.getId());
	 * status.observe(...);}</pre>
	 *
	 * You can also use the id for cancellation:
	 *
	 * <pre>
	 * {@code
	 * WorkRequest request = new OneTimeWorkRequest.Builder(FooWorker.class).build();
	 * workManager.enqueue(request);
	 * workManager.cancelWorkById(request.getId());}</pre>

		#每个request都有一个id，就像发网络请求一样
	 * You can chain work as follows:
	 *
	 * <pre>
	 * {@code
	 * WorkRequest request1 = new OneTimeWorkRequest.Builder(FooWorker.class).build();
	 * WorkRequest request2 = new OneTimeWorkRequest.Builder(BarWorker.class).build();
	 * WorkRequest request3 = new OneTimeWorkRequest.Builder(BazWorker.class).build();
	 * workManager.beginWith(request1, request2).then(request3).enqueue();}</pre>
	 *
	 *  ==支持链式的任务，很棒的特性
	 * Each call to {@link #beginWith(OneTimeWorkRequest)} or {@link #beginWith(List)} returns a
	 * {@link WorkContinuation} upon which you can call
	 * {@link WorkContinuation#then(OneTimeWorkRequest)} or {@link WorkContinuation#then(List)} to
	 * chain further work.  This allows for creation of complex chains of work.  For example, to create
	 * a chain like this:
	 *
	 * <pre>
	 *            A
	 *            |
	 *      +----------+
	 *      |          |
	 *      B          C
	 *      |
	 *   +----+
	 *   |    |
	 *   D    E             </pre>
	 *
	 * you would enqueue them as follows:
	 *
	 * <pre>
	 * {@code
	 * WorkContinuation continuation = workManager.beginWith(A);
	 * continuation.then(B).then(D, E).enqueue();  // A is implicitly enqueued here
	 * continuation.then(C).enqueue();}</pre>
	 *
	 * Work is eligible for execution when all of its prerequisites are complete.  If any of its
	 * prerequisites fail or are cancelled, the work will never run.

     * ==我怎么知道失败呢？
	 * <p>
	 * WorkRequests can accept {@link Constraints}, inputs (see {@link Data}), and backoff criteria.
	 * WorkRequests can be tagged with human-readable Strings
	 * (see {@link WorkRequest.Builder#addTag(String)}), and chains of work can be given a
	 * uniquely-identifiable name (see
	 * {@link #beginUniqueWork(String, ExistingWorkPolicy, OneTimeWorkRequest)}).
	 *
	 * <p>
	 * <b>Manually initializing WorkManager</b>
	 * <p>
	 * You can manually initialize WorkManager and provide a custom {@link Configuration} for it.
	 * Please see {@link #initialize(Context, Configuration)}.
	 */
	
	public abstract class WorkManager {}

看我上面的文档注释，我们知道这个库功能还是很强大的，完全的写完的话，文字可能会很长，毕竟支持的特定太多，功能太强大，全部写下来不容易，我还是筛选部分来说吧。有需要再去细看


## uuid 

我们先看下是怎么生成id的


	 public abstract static class Builder<B extends Builder, W extends WorkRequest> {

        boolean mBackoffCriteriaSet = false;
        UUID mId;
        WorkSpec mWorkSpec;
        Set<String> mTags = new HashSet<>();

        Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
            mId = UUID.randomUUID();
			。。。。
		}

## randomUUID
	 /**
     * Static factory to retrieve a type 4 (pseudo randomly generated) UUID.
     *
     * The {@code UUID} is generated using a cryptographically strong pseudo
     * random number generator.
     *
     * @return  A randomly generated {@code UUID}
     */
	public static UUID randomUUID() {
        SecureRandom ng = Holder.numberGenerator;

        byte[] randomBytes = new byte[16];
        ng.nextBytes(randomBytes);
        randomBytes[6]  &= 0x0f;  /* clear version        */
        randomBytes[6]  |= 0x40;  /* set to version 4     */
        randomBytes[8]  &= 0x3f;  /* clear variant        */
        randomBytes[8]  |= 0x80;  /* set to IETF variant  */
        return new UUID(randomBytes);
    }
	static final SecureRandom numberGenerator = new SecureRandom();

看下记录这个，是平时也有生成唯一ID的需求，以后也可以借鉴这个来改下。



# enqueue
继续看enqueue的内容，在WorkManagerImpl类里面：

	public final Operation enqueue(@NonNull WorkRequest workRequest) {
        return enqueue(Collections.singletonList(workRequest));
    }
		
	public Operation enqueue(
            @NonNull List<? extends WorkRequest> workRequests) {

        // This error is not being propagated as part of the Operation, as we want the
        // app to crash during development. Having no workRequests is always a developer error.
        if (workRequests.isEmpty()) {
            throw new IllegalArgumentException(
                    "enqueue needs at least one WorkRequest.");
        }
        return new WorkContinuationImpl(this, workRequests).enqueue();
    }
	
## WorkContinuationImpl
我们去到WorkContinuationImpl看了，这个构造函数，就是new多了一个新的WorkContinuationImpl出来，然后enqueue（）
	 WorkContinuationImpl(
            @NonNull WorkManagerImpl workManagerImpl,
            @NonNull List<? extends WorkRequest> work) {
        this(
                workManagerImpl,
                null,
                ExistingWorkPolicy.KEEP,
                work,
                null);
    }

    WorkContinuationImpl(
            @NonNull WorkManagerImpl workManagerImpl,
            String name,
            ExistingWorkPolicy existingWorkPolicy,
            @NonNull List<? extends WorkRequest> work) {
        this(workManagerImpl, name, existingWorkPolicy, work, null);
    }

    WorkContinuationImpl(@NonNull WorkManagerImpl workManagerImpl,
            String name,
            ExistingWorkPolicy existingWorkPolicy,
            @NonNull List<? extends WorkRequest> work,
            @Nullable List<WorkContinuationImpl> parents) {
        mWorkManagerImpl = workManagerImpl;
        mName = name;
        mExistingWorkPolicy = existingWorkPolicy;
        mWork = work;
        mParents = parents;
        mIds = new ArrayList<>(mWork.size());
        mAllIds = new ArrayList<>();
        if (parents != null) {
            for (WorkContinuationImpl parent : parents) {
                mAllIds.addAll(parent.mAllIds);
            }
        }
        for (int i = 0; i < work.size(); i++) {
            String id = work.get(i).getStringId();
            mIds.add(id);
            mAllIds.add(id);
        }
    }

看下enqueue

	 public @NonNull Operation enqueue() {
        // Only enqueue if not already enqueued.
        if (!mEnqueued) {
            // The runnable walks the hierarchy of the continuations
            // and marks them enqueued using the markEnqueued() method, parent first.
            EnqueueRunnable runnable = new EnqueueRunnable(this);
            mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
            mOperation = runnable.getOperation();
        } else {
            Logger.get().warning(TAG,
                    String.format("Already enqueued work ids (%s)", TextUtils.join(", ", mIds)));
        }
        return mOperation;
    }

我们看这里先普通的new一个新的runnable，然后放到线程池去执行；

	mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
	
这个**getWorkTaskExecutor**实际实现对应的是WorkManagerTaskExecutor类，我们进去看下里面。	
	@Override
    public void executeOnBackgroundThread(Runnable r) {
        mBackgroundExecutor.execute(r);
    }
	private final ExecutorService mBackgroundExecutor =
            Executors.newSingleThreadExecutor(mBackgroundThreadFactory);

确实是，去线程池执行这个。那到底是如何做到定时的呢？

我们去这个EnqueueRunnable看下，他是一个真的Runnable：

	public class EnqueueRunnable implements Runnable {

	    private static final String TAG = "EnqueueRunnable";
	
	    private final WorkContinuationImpl mWorkContinuation;
	    private final OperationImpl mOperation;
	
	    public EnqueueRunnable(@NonNull WorkContinuationImpl workContinuation) {
	        mWorkContinuation = workContinuation;
	        mOperation = new OperationImpl();
	    }
	
	    @Override
	    public void run() {
	        try {
	            if (mWorkContinuation.hasCycles()) {
	                throw new IllegalStateException(
	                        String.format("WorkContinuation has cycles (%s)", mWorkContinuation));
	            }
	            boolean needsScheduling = addToDatabase();
	            if (needsScheduling) {
	                // Enable RescheduleReceiver, only when there are Worker's that need scheduling.
	                final Context context =
	                        mWorkContinuation.getWorkManagerImpl().getApplicationContext();
	                PackageManagerHelper.setComponentEnabled(context, RescheduleReceiver.class, true);
	                scheduleWorkInBackground();
	            }
	            mOperation.setState(Operation.SUCCESS);
	        } catch (Throwable exception) {
	            mOperation.setState(new Operation.State.FAILURE(exception));
	        }
	    }
	}

这个我们进去看下

## addToDatabase（）
	public boolean addToDatabase() {
        WorkManagerImpl workManagerImpl = mWorkContinuation.getWorkManagerImpl();
        WorkDatabase workDatabase = workManagerImpl.getWorkDatabase();
        workDatabase.beginTransaction();
        try {
            boolean needsScheduling = processContinuation(mWorkContinuation);
            workDatabase.setTransactionSuccessful();
            return needsScheduling;
        } finally {
            workDatabase.endTransaction();
        }
    }

把这个任务持久化的保存到数据库里面去了

## processContinuation

	  private static boolean processContinuation(@NonNull WorkContinuationImpl workContinuation) {
        boolean needsScheduling = false;
        List<WorkContinuationImpl> parents = workContinuation.getParents();
        if (parents != null) {
            for (WorkContinuationImpl parent : parents) {
                // When chaining off a completed continuation we need to pay
                // attention to parents that may have been marked as enqueued before.
                if (!parent.isEnqueued()) {
                    needsScheduling |= processContinuation(parent);
                } else {
                    Logger.get().warning(TAG, String.format("Already enqueued work ids (%s).",
                            TextUtils.join(", ", parent.getIds())));
                }
            }
        }
        needsScheduling |= enqueueContinuation(workContinuation);
        return needsScheduling;
    }

## enqueueContinuation

	 private static boolean enqueueContinuation(@NonNull WorkContinuationImpl workContinuation) {
        Set<String> prerequisiteIds = WorkContinuationImpl.prerequisitesFor(workContinuation);

        boolean needsScheduling = enqueueWorkWithPrerequisites(
                workContinuation.getWorkManagerImpl(),
                workContinuation.getWork(),
                prerequisiteIds.toArray(new String[0]),
                workContinuation.getName(),
                workContinuation.getExistingWorkPolicy());

        workContinuation.markEnqueued();
        return needsScheduling;
    }


## enqueueWorkWithPrerequisites
到这个地方，我们看到核心的计算的地方了，去拿当前的系统时间去做计算处理，逻辑还是很复杂的，需要点耐心看下。

		/**
	     * Enqueues the {@link WorkSpec}'s while keeping track of the prerequisites.
	     *
	     * @return {@code true} If there is any scheduling to be done.
	     */
	    private static boolean enqueueWorkWithPrerequisites(
	            WorkManagerImpl workManagerImpl,
	            @NonNull List<? extends WorkRequest> workList,
	            String[] prerequisiteIds,
	            String name,
	            ExistingWorkPolicy existingWorkPolicy) {
	
	        boolean needsScheduling = false;
	
	        long currentTimeMillis = System.currentTimeMillis();
	        WorkDatabase workDatabase = workManagerImpl.getWorkDatabase();
	
	        boolean hasPrerequisite = (prerequisiteIds != null && prerequisiteIds.length > 0);
	        boolean hasCompletedAllPrerequisites = true;
	        boolean hasFailedPrerequisites = false;
	        boolean hasCancelledPrerequisites = false;
	
	        if (hasPrerequisite) {
	            // If there are prerequisites, make sure they actually exist before enqueuing
	            // anything.  Prerequisites may not exist if we are using unique tags, because the
	            // chain of work could have been wiped out already.
	            for (String id : prerequisiteIds) {
	                WorkSpec prerequisiteWorkSpec = workDatabase.workSpecDao().getWorkSpec(id);
	                if (prerequisiteWorkSpec == null) {
	                    Logger.get().error(TAG,
	                            String.format("Prerequisite %s doesn't exist; not enqueuing", id));
	                    return false;
	                }
	
	                WorkInfo.State prerequisiteState = prerequisiteWorkSpec.state;
	                hasCompletedAllPrerequisites &= (prerequisiteState == SUCCEEDED);
	                if (prerequisiteState == FAILED) {
	                    hasFailedPrerequisites = true;
	                } else if (prerequisiteState == CANCELLED) {
	                    hasCancelledPrerequisites = true;
	                }
	            }
	        }
	
	        boolean isNamed = !TextUtils.isEmpty(name);
	
	        // We only apply existing work policies for unique tag sequences that are the beginning of
	        // chains.
	        boolean shouldApplyExistingWorkPolicy = isNamed && !hasPrerequisite;
	        if (shouldApplyExistingWorkPolicy) {
	            // Get everything with the unique tag.
	            List<WorkSpec.IdAndState> existingWorkSpecIdAndStates =
	                    workDatabase.workSpecDao().getWorkSpecIdAndStatesForName(name);
	
	            if (!existingWorkSpecIdAndStates.isEmpty()) {
	                // If appending, these are the new prerequisites.
	                if (existingWorkPolicy == APPEND) {
	                    DependencyDao dependencyDao = workDatabase.dependencyDao();
	                    List<String> newPrerequisiteIds = new ArrayList<>();
	                    for (WorkSpec.IdAndState idAndState : existingWorkSpecIdAndStates) {
	                        if (!dependencyDao.hasDependents(idAndState.id)) {
	                            hasCompletedAllPrerequisites &= (idAndState.state == SUCCEEDED);
	                            if (idAndState.state == FAILED) {
	                                hasFailedPrerequisites = true;
	                            } else if (idAndState.state == CANCELLED) {
	                                hasCancelledPrerequisites = true;
	                            }
	                            newPrerequisiteIds.add(idAndState.id);
	                        }
	                    }
	                    prerequisiteIds = newPrerequisiteIds.toArray(prerequisiteIds);
	                    hasPrerequisite = (prerequisiteIds.length > 0);
	                } else {
	                    // If we're keeping existing work, make sure to do so only if something is
	                    // enqueued or running.
	                    if (existingWorkPolicy == KEEP) {
	                        for (WorkSpec.IdAndState idAndState : existingWorkSpecIdAndStates) {
	                            if (idAndState.state == ENQUEUED || idAndState.state == RUNNING) {
	                                return false;
	                            }
	                        }
	                    }
	
	                    // Cancel all of these workers.
	                    // Don't allow rescheduling in CancelWorkRunnable because it will happen inside
	                    // the current transaction.  We want it to happen separately to avoid race
	                    // conditions (see ag/4502245, which tries to avoid work trying to run before
	                    // it's actually been committed to the database).
	                    CancelWorkRunnable.forName(name, workManagerImpl, false).run();
	                    // Because we cancelled some work but didn't allow rescheduling inside
	                    // CancelWorkRunnable, we need to make sure we do schedule work at the end of
	                    // this runnable.
	                    needsScheduling = true;
	
	                    // And delete all the database records.
	                    WorkSpecDao workSpecDao = workDatabase.workSpecDao();
	                    for (WorkSpec.IdAndState idAndState : existingWorkSpecIdAndStates) {
	                        workSpecDao.delete(idAndState.id);
	                    }
	                }
	            }
	        }
	
	        for (WorkRequest work : workList) {
	            WorkSpec workSpec = work.getWorkSpec();
	
	            if (hasPrerequisite && !hasCompletedAllPrerequisites) {
	                if (hasFailedPrerequisites) {
	                    workSpec.state = FAILED;
	                } else if (hasCancelledPrerequisites) {
	                    workSpec.state = CANCELLED;
	                } else {
	                    workSpec.state = BLOCKED;
	                }
	            } else {
	                // Set scheduled times only for work without prerequisites. Dependent work
	                // will set their scheduled times when they are unblocked.
	                workSpec.periodStartTime = currentTimeMillis;
	            }
	
	            if (Build.VERSION.SDK_INT >= 23 && Build.VERSION.SDK_INT <= 25) {
	                tryDelegateConstrainedWorkSpec(workSpec);
	            }
	
	            // If we have one WorkSpec with an enqueued state, then we need to schedule.
	            if (workSpec.state == ENQUEUED) {
	                needsScheduling = true;
	            }
				
	            workDatabase.workSpecDao().insertWorkSpec(workSpec);
				
				//  居然是通过保存数据库来做的，感觉队列执行，
				//  可以简单理解是保存对应的后续队列到表格，前序执行，查询数据库后续的内容去执行
	            if (hasPrerequisite) {
	                for (String prerequisiteId : prerequisiteIds) {
	                    Dependency dep = new Dependency(work.getStringId(), prerequisiteId);
	                    workDatabase.dependencyDao().insertDependency(dep);
	                }
	            }
	
	            for (String tag : work.getTags()) {
	                workDatabase.workTagDao().insert(new WorkTag(tag, work.getStringId()));
	            }
	
	            if (isNamed) {
	                workDatabase.workNameDao().insert(new WorkName(name, work.getStringId()));
	            }
	        }
	        return needsScheduling;
	    }

	
回到开头的地方，返回了needsScheduling后，走的去做对应你的处理了

# scheduleWorkInBackground

	public void scheduleWorkInBackground() {
        WorkManagerImpl workManager = mWorkContinuation.getWorkManagerImpl();
        Schedulers.schedule(
                workManager.getConfiguration(),
                workManager.getWorkDatabase(),
                workManager.getSchedulers());
    }

掉用schedule去做定时操作？
## schedule

	public static void schedule(
            @NonNull Configuration configuration,
            @NonNull WorkDatabase workDatabase,
            List<Scheduler> schedulers) {
        if (schedulers == null || schedulers.size() == 0) {
            return;
        }

        WorkSpecDao workSpecDao = workDatabase.workSpecDao();
        List<WorkSpec> eligibleWorkSpecs;

        workDatabase.beginTransaction();
        try {
            eligibleWorkSpecs = workSpecDao.getEligibleWorkForScheduling(
                    configuration.getMaxSchedulerLimit());
            if (eligibleWorkSpecs != null && eligibleWorkSpecs.size() > 0) {
                long now = System.currentTimeMillis();

                // Mark all the WorkSpecs as scheduled.
                // Calls to Scheduler#schedule() could potentially result in more schedules
                // on a separate thread. Therefore, this needs to be done first.
                for (WorkSpec workSpec : eligibleWorkSpecs) {
                    workSpecDao.markWorkSpecScheduled(workSpec.id, now);
                }
            }
            workDatabase.setTransactionSuccessful();
        } finally {
            workDatabase.endTransaction();
        }

        if (eligibleWorkSpecs != null && eligibleWorkSpecs.size() > 0) {
            WorkSpec[] eligibleWorkSpecsArray = eligibleWorkSpecs.toArray(new WorkSpec[0]);
            // Delegate to the underlying scheduler.
            for (Scheduler scheduler : schedulers) {
                scheduler.schedule(eligibleWorkSpecsArray);
            }
        }
    }

看我这一串问题来了，什么时候来唤醒我的worker，让我干活呢？
