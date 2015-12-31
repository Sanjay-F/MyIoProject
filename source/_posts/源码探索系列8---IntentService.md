title: 源码探索系列8---IntentService
date: 2015-12-18 14:57
tags: [android,源码,IntentService]
categories: android
------------------------------------------

我们知道，我们的`Service`如果要执行一些耗时的操作，需要开单独的线程去干活，而`IntentService`却不用，
在他的`onHandleIntent`函数里面我们可以执行耗时操作，啊，到底神奇在哪里了呢？
让我们去看看他的源码，背后到底做了什么好事情！

# 起航 --- IntentService
API：23

	public abstract class IntentService extends Service 
我们的`IntentService`是一个继承`Service`的抽象类，那关于他的启动，停止等就和Service一样，前一篇就解释过了，我们现在就不说了。整个类也就164行，不大，还不够AMS里面一个函数的一半长度，哈，我们从他的`OnCreate`函数开始吧

	@Override
    public void onCreate() {
	    // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.
 
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
我们看到他开了一个子线程和Handler。这里我们可以猜测了，之所以在我们的`onHandleIntent`很可能是因为他们两个！
<!--more-->

让我们继续看下去
 
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
    
	@Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
好了，我们看到了线索了，发送过来的消息，通过onStartCommand(）跑到了onStart，他里面调用了Handler去发送一个包含我们的Intent的消息。让我们去看下这个消息背后被怎么处理

	@Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
我们看到他就直接调用我们的`onHandleIntent（）`，让我们去处理发送过来的Intent。
处理完后调用`stopSelf（int id）；`，这个函数需要解释下，他和我们的`stopSelf（）`是有区别的，前者是判断下看是否还有没处理的消息，有就继续干活，没有才结束，后者就不管啦，死就死的。
 

# 后记

 这里有些需要补充的：
 
 1. **排队**
 我们看到，这个IntentService是借助消息队列实现的，这种单线程，排队的任务队列，每次只执行一个，后面来的就需要等前面一个做完。
 这点可能有时不适合我们的实际需求情况，我们可能还想一次干多点，这时候还是我们自己在Service里面做吧！
 但他有些好的，就是他处理完就会去调用        `stopSelf(msg.arg1);`
 
 
 2.  **WakeLock**
 不知道你有没注意到，在他的OnCreate函数他注视里面有个todo的内容
>  **TODO:** It would be nice to have an option to hold a partial wakelock 
>  during processing,  and to have a static startService(Context, Intent) 
>    method that would launch the service & hand off a wakelock.

 关于这个wakelock，可以补充点内容在这里，因为这篇突然好短，哈哈，都不习惯了，所以使劲加点内容
他的作用可以让屏幕保持唤醒，不过得先声明下权限

	```
	<uses-permission android:name="android.permission.WAKE_LOCK" />
	```
 曾经使用他的一个情景就是在做IM功能的时候，在做那个声音的功能的时候，需要用他来保持唤醒。
		
		wakeLock = ((PowerManager) getSystemService(Context.POWER_SERVICE)).
					newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "demo");

		wakeLock.acquire(); //设置保持唤醒
		
		...
		
		wakeLock.release(); //解除保持唤醒
另外我们平时使用手机看电影的时候，也会遇到过，要保持屏幕常量嘛。