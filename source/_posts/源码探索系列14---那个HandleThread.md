title: 源码探索系列14---那个HandleThread
date: 2015-12-28 22:05
tags: [android,源码,HandleThread]
categories: android
------------------------------------------


这篇文章被CSDN搞丢了，好在通过使用谷歌搜索，
看到历史的记录版本，现在再根据记忆，补充下内容。
算是恢复了这篇文章的大概的模样。虽然对于CSDN的来说这文章没什么，但对于我来说就是自己花的时间和精力。   
 
# 起航
这次我们来说说HandleThread，听名字好像就是Handler+Thread的感觉啊。 
他和Handler的关系就像Service和IntentService一样，一个不支持耗时的任务，一个可以。 
不过两者还是有点区别，一些是运行在主线程的，一个不在主线程跑。 
很好理解，要是在主线程跑也能搞耗时任务，那不就逆天了？？
对于他的使用，我们就不介绍了，让我们看下他实际做些什么吧。
 
就从我们的HandleThread的声明开始说起吧
<!--more-->

	handlerThread = new MyHandlerThread("MyHanlerThread");
	handlerThread.start();
	handler = new Handler(handlerThread.getLooper(), handlerThread);
一般我们是这样用的，我们在Handler的构造器传了我们的HandleThread的Loop和他的Callback接口，从而替换掉Handler原生的，之后我们利用Handler发送消息时候handler.sendEmptyMessage(MSG_OK);，会跑回到我们的HandlerThread去。这点从我们的Handler的底层代码的 dispatchMessage(）函数可以看到

	public void dispatchMessage(Message msg) {
	        if (msg.callback != null) {
	            handleCallback(msg);
	        } else {
	            if (mCallback != null) {
	                if (mCallback.handleMessage(msg)) {
	                    return; 
	                }
	            }
	            handleMessage(msg);
	        }
	}
那么问题来了，这个到底为何可以执行耗时操作呢，估计和他名字一样，有Thread在里面吧。 
而且很可能是有自己的Looper，不会像Handler一样是MainLooper的。
就让我们开始看下他底层的把

	public class HandlerThread extends Thread 
根据名字，我们看到他确实是继承Thread的，所以可以执行耗时操作。
接着我们看下start（）方法 后的run（）里面是什么内容

	@Override
	public void run() {
	    mTid = Process.myTid();
	    Looper.prepare();
	    synchronized (this) {
	        mLooper = Looper.myLooper();
	        notifyAll();
	    }
	    Process.setThreadPriority(mPriority);
	    onLooperPrepared();
	    Looper.loop();
	    mTid = -1;
	}
我们看到他为自己搞了一个Looper，和我们一开始说的一样，这个当然不是主looper。 
之后在调用Looper.loop之前，他会去调用下onLooperPrepared（）

	/**
	 * Call back method that can be explicitly overridden if needed to execute some
	 * setup before Looper loops.
	 */
	protected void onLooperPrepared() {
	}
	
这是一个空方法，什么也没有，官方给的注释是，如果我们需要再Looper Loops启动前做点什么，就重载这个方法。这个就和AsyncTask的onPreExecute() 类似的方法。不看源码还真没去注意到。
好，这个类基本到这里了，没什么好说的了…很简单的一个，不过AMS里面一个函数长的。
有个问题，为何要有这个东西呢？

因为我们如果开子线程时候，一直都需要在线程run()方法当中先调用Looper.prepare()初始化Looper，
然后在`run()`方法最后调用`Looper.loop()`，这个如果写多几次的话都回觉得挺烦的，所以谷歌 给我们带来了这个类， 一个拥有Looper的方便好用类。

# 后记

 

这个，这次的好短啊，就这么结束了，还不习惯，还没前进彻底就结束了。
找到篇文章，写的是用[这个来模拟一个AsyncTask](http://www.cnblogs.com/over140/p/3607483.html)，挺有意思的，源码像下面

	public abstract class FakeAsyncTask<Params, Progress, Result> {

    private HandlerThread mHandlerThread;
    private TaskHandler mBackgroundHandler;
    private TaskHandler mUiHandler;
    private Params[] mParams;

    public FakeAsyncTask() {
        mHandlerThread = new HandlerThread("HandlerThread", Process.THREAD_PRIORITY_DEFAULT);
        mHandlerThread.start();
        mBackgroundHandler = new TaskHandler(mHandlerThread.getLooper());
        mUiHandler = new TaskHandler(Looper.getMainLooper());
    }

    protected abstract Result doInBackground(Params... params);

    protected void onPreExecute() {
    }

    protected void onProgressUpdate(Progress... values) {
    }

    protected final void publishProgress(Progress... values) {
        mUiHandler.obtainMessage(MESSAGE_PROGRESS, values).sendToTarget();
    }

    protected void onPostExecute(Result result) {
    }

    public final boolean isCancelled() {
        return mHandlerThread.isInterrupted();
    }

    public final void cancel(boolean mayInterruptIfRunning) {
        if (!mHandlerThread.isInterrupted()) {
            try {
                mHandlerThread.quit();
                mHandlerThread.interrupt();
            } catch (SecurityException e) {
                e.printStackTrace();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        onCancelled();
    }

    protected void onCancelled() {
    }

    public void execute(Params... params) {
        mParams = params;
        onPreExecute();
        mBackgroundHandler.sendEmptyMessage(MESSAGE_INBACKGROUND);
    }

    private static final int MESSAGE_INBACKGROUND = 0;
    private static final int MESSAGE_POSTEXECUTE = 1;
    private static final int MESSAGE_PROGRESS = 2;

    private class TaskHandler extends Handler {

        public TaskHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings("unchecked")
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_INBACKGROUND:
                    mUiHandler.obtainMessage(MESSAGE_POSTEXECUTE, doInBackground(mParams)).sendToTarget();
                    break;
                case MESSAGE_POSTEXECUTE:
                    onPostExecute((Result) msg.obj);
                    mHandlerThread.quit();
                    break;
                case MESSAGE_PROGRESS:
                    onProgressUpdate((Progress[]) msg.obj);
                    break;
            }
        }
    }
}