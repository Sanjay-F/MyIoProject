title: 源码探索系列1---Handler与HandlerLeak的那些事
date: 2015-12-08 23:55 
tags: [android,源码,Handler,HandlerLeak]
categories: android
------------------------------------------

开始学安卓的时候，我对一些异步操作都是用Handler和AsyncTask的。

现在那个Handler被挂上了泄漏的名字！
最近在设计一个功能的时候，像借鉴于handler的设计模式，所以顺便顺便写篇文章记录下。

<!--more-->
一开始我调用Handler像下面这样，对这种，系统提示会导致泄漏，为人懒惰，
就直接加了这个`@SuppressLint("HandlerLeak")`，然后忽略掉它。


	public class MainActivity extends Activity {
	
		private Handler mHandler = new Handler() {
		        @Override
		        public void handleMessage(Message msg) {
		            super.handleMessage(msg);
		            if (msg.what == 0x12345) {
		                Log.e(TAG, " log");
		            }
		        }
		    };
		    
		 @Override
		  protected void onCreate(Bundle savedInstanceState) {
		    super.onCreate(savedInstanceState);
		   
		    mHandler.postDelayed(new Runnable() {
		      @Override
		      public void run() { /* ... */ }
			    }, 60 * 1000); //延迟一分钟发送这个消息
		    
		    finish();
		  } 
	}

后来遇到多了，就还是乖乖根据提示写成一个静态内部类的形式来

	public class MainActivity extends Activity {
	
	
		static class MyHandler extends Handler {         
	        private WeakReference<MainActivity> mOuter;
	
	        public MyHandler(MainActivity activity) {
	            mOuter = new WeakReference<MainActivity>(activity);
	        }
	
	        @Override
	        public void handleMessage(Message msg) {
	            MainActivity outer = mOuter.get();
	            if (outer != null) {
	                //// TODO: 2015/12/8  
	            }
	        }
	    }
	}
	
**那么问题来了，是什么导致写内部类就没事呢？**

看下官方的解释:

> Since this Handler is declared as an inner class,
>  it may prevent the outer class from being garbage collected. 
>  
> If the Handler is using a Looper or MessageQueue for a thread other than the main thread, 
> then  there is no issue.
> 
> If the Handler is using the Looper or MessageQueue of the main thread, 
> you need to fix your Handler declaration, as  follows: 
> Declare the Handler as a static class; In the outer class,
> instantiate a WeakReference to the outer class and pass this object to  your Handler 
> when you instantiate the Handler; 
> Make all references to members of the outer class using the WeakReference object.

结合下面这个例子来说下原因

	mHandler.postDelayed(new Runnable() {
			      @Override
			      public void run() { /* ... */ }
			    }, 60 * 1000); 
			    
			    finish();
当finish后，延时消息会继续存在主线程消息队列中1分钟，然后处理消息。
而该消息引用了`Activity`的`Handler`对象，然后这个`Handler`又引用了这个`Activity`。
这些引用对象会保持到该消息被处理完，这样就导致该Activity对象**无法被回收**，
从而导致了上面说的 Activity泄露。
所以他说如果Handler是使用主线程的`Looper`或者`MessageQueue`，那么就需要注意咯 。

嗯，主线程的Looper或MessageQueue，可我没看到有用到啊，怎么回事？

好了，还是让我们开始看下Handler的实现吧，肯定可以找到答案。
  

# 看下Hanlder的源码

 让我们从调用的第一句，无参构造器看起吧
  
	public Handler() {
	    this(null, false);
	}
	    
	//上面那个无参构造器调用了下面这个
	public Handler(Callback callback, boolean async) {
 
             ....
	
	        mLooper = Looper.myLooper();
	        if (mLooper == null) {
	            throw new RuntimeException(
	                "Can't create handler inside thread that has not called Looper.prepare()");
	        }
	        mQueue = mLooper.mQueue;
	        mCallback = callback;  //处理消息的回调，后面用到
	        mAsynchronous = async;
	    }
很好，瞬间我们就看到了`Looper`和`Queue`这两个关键的字眼东西啦，
既然我们说引用了主线程的Looper和MessageQueue，那理论上这两句
`mLooper = Looper.myLooper();`和 `mQueue = mLooper.mQueue;`
返回的结果就很可能就是了。我们继续深入看下

	public static @Nullable Looper myLooper() {
	        return sThreadLocal.get();
	    }
一个神奇的东西上场了`sThreadLocal`，这家伙很有故事，后面有时间细说，先看下Looper里面初始化这个sThreadLocal的代码。

	public final class Looper {

		static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

		public static void prepare() {
		        prepare(true);
		}
		
	    private static void prepare(boolean quitAllowed) {
                 ...
	        sThreadLocal.set(new Looper(quitAllowed));
	    }

	    private Looper(boolean quitAllowed) {
	        mQueue = new MessageQueue(quitAllowed);
	        mThread = Thread.currentThread();
	    }
	    	    
	}

看到这里，那个sThreadLocal确实保存了一个`Thread.currentThread();`，
如果我们就是在主线程调用的，那会保存主线程的引用。

看到这里，可以确认handler底层是用到了主线程的Looper和MessageQueue

---
# 进一步的看下


	mHandler.postDelayed(new Runnable() {	
		      @Override
		      public void run() { 
		        //do something...
		      }}, 60 * 1000);   //延迟一分钟发送这个消息 
			    
			    
我们先来看下我们从发送消息`mHandler.sendMessage(msg)`，
到返回`handleMessage(Message msg)`  的流程在底层的具体实现，
然后结合的来分析 。


		public final boolean postDelayed(Runnable r, long delayMillis){
	        return sendMessageDelayed(getPostMessage(r), delayMillis);
	    }
	    
	    private static Message getPostMessage(Runnable r) {
	       Message m = Message.obtain();
	       m.callback = r;//把我们的回调设置到了callback。后面分发消息时候用到
	       return m;
	    }
	   
		public final boolean sendMessageDelayed(Message msg, long delayMillis){
	        if (delayMillis < 0) {
	            delayMillis = 0;
	        }
	        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
	    }
	
		public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	        MessageQueue queue = mQueue;
	        if (queue == null) {
	            RuntimeException e = new RuntimeException(
	                    this + " sendMessageAtTime() called with no mQueue");
	            Log.w("Looper", e.getMessage(), e);
	            return false;
	        }
	        return enqueueMessage(queue, msg, uptimeMillis);
	    }
	    
		private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
	        msg.target = this;
	        if (mAsynchronous) {
	            msg.setAsynchronous(true);
	        }
	        return queue.enqueueMessage(msg, uptimeMillis);
	    }

我们看到，我们发送的Runnable被打包成Message，然后层层调用，
最后调用`queue.enqueueMessage(msg, uptimeMillis);`,
把我们的消息扔到一个队列里面去了，那我们来看下队里里面具体做了什么

	boolean enqueueMessage(Message msg, long when) {
 
             ...
             	
	        synchronized (this) {	             
	            ...
	            
	            msg.markInUse();
	            msg.when = when; //设定时间
	            Message p = mMessages;
	            boolean needWake;
	            
	            if (p == null || when == 0 || when < p.when) {
	                // New head, wake up the event queue if blocked.
	                msg.next = p;
	                mMessages = msg;
	                needWake = mBlocked;
	            } else {
	                // Inserted within the middle of the queue.  Usually we don't have to wake
	                // up the event queue unless there is a barrier at the head of the queue
	                // and the message is the earliest asynchronous message in the queue.
	                needWake = mBlocked && p.target == null && msg.isAsynchronous();
	                Message prev;
	                for (;;) {
	                    prev = p;
	                    p = p.next;
	                    if (p == null || when < p.when) {
	                        break;
	                    }
	                    if (needWake && p.isAsynchronous()) {
	                        needWake = false;
	                    }
	                }
	                msg.next = p; // invariant: p == prev.next
	                prev.next = msg;
	            }
	
	            // We can assume mPtr != 0 because mQuitting is false.
	            if (needWake) {
	                nativeWake(mPtr);
	            }
	        }
	        return true;
	    }

看了这么长，就是根据这个msg的时间，把这个消息插入到队伍的合适的位置里面去。
好了，这样好像线索就断了，我们没有看到哪里在消耗这些消息。
很可能在这个过程我们错过了什么，跳回去看下整个过程。

好吧，原来我们的系统会调用`Looper.loop()`方法。
这点的具体使用的地方还没看到，先就这么确认了，我们看下loop（）里面都做了什么

	public static void loop() {
	        final Looper me = myLooper(); //又一个获取myLooper的方法
	        if (me == null) {
	            throw new RuntimeException(
	            "No Looper; Looper.prepare() wasn't called on this thread."); 
	             //这个错误信息，应该有遇到过吧，我们在子线程没有调用Looper.prepare()的时候。
	        }

           //重要的一句来了，获取队列
	        final MessageQueue queue = me.mQueue;
	
	        Binder.clearCallingIdentity();
	        final long ident = Binder.clearCallingIdentity();
	
	        for (;;) {
	        
	            Message msg = queue.next(); // might block

	            if (msg == null) {
	                // 没有消息表示队列停止了,滚回去等待下个消息
	                return;
	            }
	
	            // This must be in a local variable, in case a UI event sets the logger
	            Printer logging = me.mLogging;
	            if (logging != null) {
	                logging.println(">>>>> Dispatching to " + msg.target + " " +
	                        msg.callback + ": " + msg.what);
	            }
	
                //来了，分发消息的一句，看到这个就有我们自定义View时候的分发event的即视感
	            msg.target.dispatchMessage(msg);
	
	            if (logging != null) {
	                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
	            }	 
	            
	            ...	
	            
	            msg.recycleUnchecked();
	        }
	    }



#### 小插曲
看到这一部分代码，应该可以理解我们开一个新线程的时候的代码要先这么写了吧.

	new Thread(){          
	            @Override
	            public  void run(){
	                Looper.prepare();                
	                Handler handler=new Handler();                
	                Looper.loop();	                
                      ....
                      
	            }
	}.start();
 当然这样写需要记得的一件事情是，我们结束完工作后，记得调用`quit();`来结束。


现在我们来看下这个`Message msg = queue.next();`获取队列的一个Message里面都发生了什么，
才让他说了句 `// might block`

	Message next() {
	 
	        ...
	        
	        int pendingIdleHandlerCount = -1; // -1 only during first iteration
	        int nextPollTimeoutMillis = 0;
	        for (;;) {  //居然又有一个死循环
	        
	            if (nextPollTimeoutMillis != 0) {
	                Binder.flushPendingCommands();
	            }
                   	
	            nativePollOnce(ptr, nextPollTimeoutMillis);
	
	            synchronized (this) {
	                // Try to retrieve the next message.  Return if found.
	                final long now = SystemClock.uptimeMillis();
	                Message prevMsg = null;
	                Message msg = mMessages;
	                if (msg != null && msg.target == null) {
	                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
	                    do {
	                        prevMsg = msg;
	                        msg = msg.next;
	                    } while (msg != null && !msg.isAsynchronous());
	                }
	                if (msg != null) {
	                    if (now < msg.when) {
	                        // Next message is not ready.  Set a timeout to wake up when it is ready.
	                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
	                    } else {
	                        // Got a message.
	                        mBlocked = false;
	                        if (prevMsg != null) {
	                            prevMsg.next = msg.next;
	                        } else {
	                            mMessages = msg.next;
	                        }
	                        msg.next = null;
	                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
	                        msg.markInUse();
	                        return msg;
	                    }
	                } else {
	                    // No more messages.
	                    nextPollTimeoutMillis = -1;
	                }
	
	                ....
	        }
	    }
好了，节选了一部分代码，我们可以看到，这个next方法就是一个无限循环的方法，如果消息队列有消息，而且时间到了，那么就返回，如果时间还没到，那就等待下一次的唤醒。如果没有消息，那么就一直堵塞在哪里。

	

> 那么为何这个死循环没有卡死程序呢？


我们的安卓是基于Linux的，其中有`pipe/epoll`机制，所以在主线程的MessageQueue没有消息时，便**阻塞**`queue.next()`中的`nativePollOnce()`方法。此时主线程会释放CPU资源进入`休眠状态`，直到下个消息到达或者有事务**发生**，通过往pipe管道写端写入数据来唤醒主线程工作。
所以可以简单的理解epoll机制是一种 **观察者模型**，（背后是一种IO多路复用机制，可以同时监控多个描述符，当描述符就绪，则通知相应 **消费者**进行特定操作。
所以实质上，主线程其实基本是**休眠状态**，并不会消耗大量CPU资源。从而卡死程度！
 
 ---


	if (now < msg.when) {
      // Next message is not ready.  Set a timeout to wake up when it is ready.
       nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
    } else {
       // Got a message. 
       ...
        msg.markInUse();
       return msg;
    }




好了，我们来看下我们的消息分发`msg.target.dispatchMessage(msg);`通过这句话，就成功把代码逻辑切换到指定的线程中去执行。

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
还记得我们前面在发送消息时候，我们把Runnable封装成Message的时候，设置了callback吗？就是下面这个。

	 private static Message getPostMessage(Runnable r) {
		       Message m = Message.obtain();
		       m.callback = r;
		       return m;
	}
好了，我们继续看下`handleCallback(msg);`

	private static void handleCallback(Message message) {
	        message.callback.run();
	    }
	    
很直接的就是直接运行这个run方法啦！


接着对于那些没有callback的Message，如果一开始没有设置`callback`的，
就会落入到hendleMessage里面去，这个就是我们重写的方法啦。
 好啦，整个的发送到消费，再回到我们的Activity里面的整个过程，基本的就到这里了。
 
	public Handler(Callback callback, boolean async) {	 
             ...	             
	        mLooper = Looper.myLooper(); 
	        mQueue = mLooper.mQueue;
	        
	        mCallback = callback;//就是这个
	}
额外补充的一句就是，我们平时遇到的代码比较少用到这个callback，因为我们一般都是派生出一个子类来，但有时候我们其实并不需要一个子类，重写handleMessage()方法，我们完全可以直接传个Callback就可以了。	
	
-----

# 后记
 
- 这个过程，其实有很多细节是忽略没有提及的，例如我们的MessageQueue里面涉及到的`nativeInit();`,这个的背后是什么，有兴趣的可以看下面参考地址的第二篇，他有对此做进一步的介绍。
 
		private Looper(boolean quitAllowed) {
	        mQueue = new MessageQueue(quitAllowed);
	        mThread = Thread.currentThread();
	    }
	       
		MessageQueue(boolean quitAllowed) {
		        mQuitAllowed = quitAllowed;
		        mPtr = nativeInit();
	 	 }
   
- msg.target.dispatchMessage(msg);
这个Message的tartget即Handle是什么时候设置的。

- ~~我们的Activity里面的Loop等方法是什么时候调用的，这些都没有近一步探索。~~

- ThreadLocal这个神奇的东西，也没有做进一步介绍

- 另外那个MessageQueue也有不少内容可以说，下次有空再继续介绍吧


好吧，虽然还有很多细节没说，但大抵的关系我们都知道啦，就像下面这张

![这里写图片描述](http://pic002.cnblogs.com/images/2012/328668/2012060815460614.jpg)


---

# 更新补充内充

1. **我们的Activity里面的Loop等方法是什么时候调用的。**
 
   好了，这个通过查阅点资料，我们找到了点内容啦，我们的Activity继承于android.app.Activity这个类，这个类里面有一样东西叫ActivityThread，有图有真相的。
   ![这里写图片描述](http://img.blog.csdn.net/20151208214652271)
这个就是传送中的Android的**主线程**啦，我们到他的入口方法**main**里面看到了我们想要的答案。

	  
		  public static void main(String[] args) {	
	          ...
	          
	        AndroidKeyStoreProvider.install();
	
	        Looper.prepareMainLooper();
	
	        ActivityThread thread = new ActivityThread();
	        thread.attach(false);
	
	        if (sMainThreadHandler == null) {
	            sMainThreadHandler = thread.getHandler();
	        }
	
	        if (false) {
	            Looper.myLooper().setMessageLogging(new
	                    LogPrinter(Log.DEBUG, "ActivityThread"));
	        }
	
	        // End of event ActivityThreadMain.
	        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
	        Looper.loop();

 
	    }

这个ActivityThread来头不小，5421行代码，呵呵



---

参考资料：

1. [内部Handler类引起内存泄露](http://blog.chengyunfeng.com/?p=468#ixzz3tiWUgbOW%20http://blog.chengyunfeng.com/?p=468)

2. [Android消息处理机制(Handler、Looper、MessageQueue与Message)](http://www.cnblogs.com/angeldevil/p/3340644.html)
 
  