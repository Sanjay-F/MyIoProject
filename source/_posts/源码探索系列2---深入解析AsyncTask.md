title: 源码探索系列2---深入解析AsyncTask
date: 2015-12-09 02:04 
tags: [android,源码,AsyncTask]
categories: android
------------------------------------------

在解析前，我们先来看下一般我们使用的情况是怎样的
下面写了一个简单的demo，用来做个简单的任务，从1数到100，同时调用`publishProgress(i);`来更新下进度。
我想用过的人自己直接阅读下面代码没有任何问题。

	class MyDemoAsyncTask extends AsyncTask<Integer, Integer, String> {
	
	        private TextView textView;
	        private ProgressBar progressBar;
	
	        public MyDemoAsyncTask(TextView textView, ProgressBar progressBar) {
	            super();
	            this.textView = textView;
	            this.progressBar = progressBar;
	        }
	
	        @Override
	        protected void onPreExecute() {
	            textView.setText("开始执行异步线程");
	        }
	
	        @Override
	        protected String doInBackground(Integer... params) {
	            for (int i = 1; i <= 100; i += 1) {
	                publishProgress(i);		 
	               try {
	                    Thread.sleep(1000);
	                } catch (InterruptedException e) {
	                    e.printStackTrace();
	                }
	            }
	            return "resultString";
	        }
	
	        @Override
	        protected void onProgressUpdate(Integer... values) {
	            progressBar.setProgress(values[0]);
	        }
	
	        @Override
	        protected void onPostExecute(String result) {
	            textView.setText("异步操作执行结束" + result);
	        }
	    }

# 问题
好了，这样一个demo，有几点需要解决的问题。

1. 为何他只能执行一次，再调用execute()就会出错

2.   doInBackground()是怎么做到在后台执行的
 
3. 为何 onPreExecute()，onProgressUpdate(）和onPostExecute(）能运行在主线程

4. 为何Task的实例必须在UI thread中创建,execute方法必须在UI thread中调用；

5.  为何不能手动的调用onPreExecute()，onPostExecute()，doInBackground(),，onProgressUpdate()这几个方法；

带着这么几个问题，我们开始深入的看下AsyncTask的源代码。

<!--more-->

# 起源
让我们根据线索，先看下execute（）方法吧。

	@MainThread
	public final AsyncTask<Params, Progress, Result> execute(Params... params) {
	     return executeOnExecutor(sDefaultExecutor, params);
	}
	    
	@MainThread
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

1. 看到这里，我们似乎得到了**第一个问题的答案**，为何只能执行一次，因为AsyncTask会对自己的状态做一些标记，如果已经是`RUNNING`或者`FINISHED`状态，那么就会抛出异常，那么为何需要状态呢？
  1. 主要因为不允许重用！当我们重用的话，一些变量可能被上一次给修改了，再用就有可能出错！就像我们写Junit一样，需要销毁掉，再开始新的测试用例一样。而且当有竞争并发的情况出现后，处理起来的问题就复杂多了。所以还是直接干脆让你在new一个好了。
  1. 另外是为了避免内存泄漏问题！
     当我们做那么多后台任务，这就有可能导致你的Activity都关了，人家还在干活，这就有可能导致内存泄漏问题啊！返回结果还没人要..

2. 在调用了`Exccute()`后，我们看到他顺势也调用了`onPreExecute();`，这个运行在主线程也好理解了。
3. 我们还看到，他使用本地的`sDefaultExecutor`执行器来执行了一个`mFuture`,我们来看下这个`sDefaultExecutor`，因为我们都听说AsyncTask底层是一个线程池，那到底是怎样的呢？
	 
		public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
		
		private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
		 
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
		    
		private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
	    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
	    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
	    private static final int KEEP_ALIVE = 1;
		
		public static final Executor THREAD_POOL_EXECUTOR
		            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
		                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
	
	通过代码我们看到，这个`sDefaultExecutor`是最终用了一个静态的内部类`SerialExecutor`，这个人如其名，是个线性执行的，一次只执行一个。 	
	**而且每次只运行一个异步任务**
	**而且每次只运行一个异步任务**
	**而且每次只运行一个异步任务**
    当然了你也可以根据自己的需要，指定自己的线程池
为了加深你对这个的理解，我写了个简单的demo,我们生成10个task，然后执行，看打印的时间是怎样的
 
		for (int i = 0; i < 10; i++) {
		    new MySerialAsyncTask().execute(i);
		 } 
	
	    class MySerialAsyncTask extends AsyncTask<Integer, Integer, String> {	
	        @Override
	        protected String doInBackground(Integer... params) {
	            Log.e(TAG, " task " + params[0] + " isRuning");
	            try {
	                Thread.sleep(3000);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	            return "" + params[0];
	        }
	
	        @Override
	        protected void onPostExecute(String result) {
	            Log.e(TAG, " task " + result + " finish");
	        }
	    }
打印结果

		12-09 03:09:30.760 23235-23267/org.test.demo E/LoginActivty:  task 0 isRuning
		12-09 03:09:33.780 23235-23235/org.test.demo E/LoginActivty:  task 0 finish
		12-09 03:09:33.780 23235-23307/org.test.demo E/LoginActivty:  task 1 isRuning
		12-09 03:09:36.784 23235-23235/org.test.demo E/LoginActivty:  task 1 finish
		12-09 03:09:36.784 23235-23348/org.test.demo E/LoginActivty:  task 2 isRuning
		12-09 03:09:39.792 23235-23235/org.test.demo E/LoginActivty:  task 2 finish
		12-09 03:09:39.808 23235-23395/org.test.demo E/LoginActivty:  task 3 isRuning
		12-09 03:09:42.812 23235-23235/org.test.demo E/LoginActivty:  task 3 finish
		12-09 03:09:42.812 23235-23438/org.test.demo E/LoginActivty:  task 4 isRuning
		12-09 03:09:45.812 23235-23438/org.test.demo E/LoginActivty:  task 5 isRuning
		12-09 03:09:45.812 23235-23235/org.test.demo E/LoginActivty:  task 4 finish
		12-09 03:09:48.816 23235-23438/org.test.demo E/LoginActivty:  task 6 isRuning
		12-09 03:09:48.816 23235-23235/org.test.demo E/LoginActivty:  task 5 finish
		12-09 03:09:51.820 23235-23438/org.test.demo E/LoginActivty:  task 7 isRuning
		12-09 03:09:51.820 23235-23235/org.test.demo E/LoginActivty:  task 6 finish
		12-09 03:09:54.820 23235-23438/org.test.demo E/LoginActivty:  task 8 isRuning
		12-09 03:09:54.820 23235-23235/org.test.demo E/LoginActivty:  task 7 finish
		12-09 03:09:57.824 23235-23438/org.test.demo E/LoginActivty:  task 9 isRuning
		12-09 03:09:57.824 23235-23235/org.test.demo E/LoginActivty:  task 8 finish
		12-09 03:10:00.824 23235-23235/org.test.demo E/LoginActivty:  task 9 finish
  
 我们看到，严格的没三秒后才执行下一个任务
 当然，我们完全可以并发执行，用`AsyncTask.THREAD_POOL_EXECUTOR`，
根据参数的值，我们可知道是四种常见的线程池之一的**延迟连接池**（newScheduledThreadPool），
我们直接用`asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, task);`
就可以多任务并发执行了。当然你也可以自定义一个。
之所以这样，是谷歌基于性能的考虑，所以在3.0版本的时候，就改成这个默认的串行。
另外当如果我们开太多进程并发的去处理网络请求，挤在一起，那么很容易会把网速都霸占完，结果每个进程分到的网速就那么点， 导致延迟超时问题。

好了，跳了这么远，我们继续回到主线任务上去
**主线**：

	  exec.execute(mFuture);
我们回到上面看到execute()，最后调用了这句，去执行一个`mFuture`的东东，这个`mFuture`里面很可能就是执行我们业务内容的地方。让我们去看望下它

	public AsyncTask() {
	        mWorker = new WorkerRunnable<Params, Result>() {
	            public Result call() throws Exception {
	                mTaskInvoked.set(true);
	
	                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
	                //noinspection unchecked
	                Result result = doInBackground(mParams);
	                Binder.flushPendingCommands();
	                return postResult(result);
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
	    
	private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
	        Params[] mParams;
	    }
	    
在我们的构造器里面，我们看到了我们的变量`mFuture`,它包着我们的`worker`，这个worker背后是实现了`Callback`接口，
而worker里面有一个关键的的一句`Result result = doInBackground(mParams);`,在这个worker里面执行了我们的`doInBackground`方法。然后执行的结果Result通过`PostResult()`做了分发。
对于`Runnable`，`Callback`，`FutureTask` 和 `Future` 这几个凑在一起，有时候还真忘了他们是什么关系。

如果你不是清楚，那么记得这么个结论，我们的Executor会调用FutureTask里面的run()方法，因为FutureTask内部是实现了接口RunnableFuture的Run()方法的，在这个run()方法里面，会调用这个worker的`call()`方法。
当这个worker工作完了时候，结果就被保存起来了，这时候`FutureTask`的`done()`方法会被调用，这时候我们可以通过`get()`  方法得到结果啦。（呵呵，好想把代码贴出来，所谓有代码有真像的）

所以看到上面的`done（）`里面这句`postResultIfNotInvoked(get());`，就是用来获取我们的worker的运算结果的。


但最少，看到这里我们的**第二个问题**，doInBackground是怎么做到在后台执行的就知道了，他在`call（）`里面执行的。然后结果用Handler来post出去的。

另外这个解决了我们的**第一个问题**，为何他只能执行一次，再调用execute()就会出错。
因为我们的mFuture只会被执行一次，再执行是没有效果的，如果我没记错的话   -_-


	private Result postResult(Result result) {
	        @SuppressWarnings("unchecked")
	        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
	                new AsyncTaskResult<Result>(this, result));
	        message.sendToTarget();
	        return result;
	    }
	    
	 private static class AsyncTaskResult<Data> {
	        final AsyncTask mTask;
	        final Data[] mData;
	
	        AsyncTaskResult(AsyncTask task, Data... data) {
	            mTask = task;
	            mData = data;
	        }
	    }
再来看下我们的getHandler();

	private static Handler getHandler() {
	        synchronized (AsyncTask.class) {
	            if (sHandler == null) {
	                sHandler = new InternalHandler();
	            }
	            return sHandler;
	        }
	    }
我们的`getHandler()`返回了一个内部静态`Handler`类,[还记得上一篇文章说为何要写成静态内部类吗？](http://blog.csdn.net/liulipuo/article/details/39029643)
通过这段代码，我们了解到，由于 Handler 需要和主线程交互，而 Handler 作为静态内部类（静态成员在加载类的时候初始化）内置于 AsyncTask 中的，所以，AsyncTask 的创建必须在主线程。这样我们的**第四个问题**就解决了

	private static class InternalHandler extends Handler {
	
	        public InternalHandler() {
	            super(Looper.getMainLooper());
	             //使用的是一个主线程的looper，这样就把消息切换到了主线程去了
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
看到这个Handler的内部构造，我们看到一个`MESSAGE_POST_PROGRESS`消息下调用`onProgressUpdate()`，在看下我们的publishProgress()方法，确实发的就是这个消息

	@WorkerThread
	protected final void publishProgress(Progress... values) {
	     if (!isCancelled()) {
	         getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
	                 new AsyncTaskResult<Progress>(this, values)).sendToTarget();
	     }
    }

这样我们的**第三个问题**，为何`publishProgress()`	可以更主线程UI就知道了。


 **继续回到我们的主任务**
在这个结果，我们看到了他最后调用了`result`的`mTask`里面的`finish`方法。我们看下具体的内容是什么

	private void finish(Result result) {
       if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
在这里我们看到如果是被取消了，就调用onCancelled（），没取消就对`onPostExecute(result);`的调用，这样基本的流程我们就跑完了。从`execute()`到`onPostExecute()`

---
#后记 

到这里我们就基本把AsyncTask的主要部分代码看完了，在实现上使用到了FutureTask，Callback这两个平时比较少用的类，温习了下，对他的使用以后就可以更有把握啦。

看完觉得好和不好，欢迎评论下，知道改进，谢谢。

虽然这个类应该用的很少了，不过也算了解，避免以后自己写类似的代码，从中也学到不少内容了