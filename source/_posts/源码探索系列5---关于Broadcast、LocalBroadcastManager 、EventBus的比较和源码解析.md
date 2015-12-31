title: 源码探索系列5---关于Broadcast、LocalBroadcastManager 、EventBus的比较和源码解析
date: 2015-12-17 01:24
tags: [android,LocalBroadCast,Broadcast,EventBus]
categories: android
------------------------------------------

这篇文章将会对这三者做简单的综合性的比较，同时对LocalBroadCast深入到源码进行一个探索。看这个类为何相对BroadCast高效！

# 起航

 

# 1. 比较

- **性能 对比 ：  EventBus不差 ** 

      EventBus ~ LocalBroadCast  > Bradocast 
	    
-  **运行的线程环境：    EventBus完胜！**
    
	LocalBroadcastManager 所有调用都是在主线程，
	EventBus 可以定义在调用线程、主线程、后台线程、异步。
 
- **代码量比较：  EventBus完胜！**
		
 <!--more-->
		
1. **Bradocast**
 
				@Override
			    public void onStart() {
			        super.onStart();
			        // 注册广播接收
			        IntentFilter filter = new IntentFilter();
			        filter.addAction(Constants.USER_UPDATE);
			        receiveBroadCast = new ReceiveBroadCast();
			        registerReceiver(receiveBroadCast, filter);
			    }

              // 发送
		       Intent intent = new Intent();
	           intent.setAction(Constants.BP_UPDATE);
                sendBroadcast(intent);

				class ReceiveBroadCast extends BroadcastReceiver {
			        @Override
			        public void onReceive(Context context, Intent intent) {
			            String action = intent.getAction();
			  
			            if ( !TextUtils.isEmpty(action) && action.equals(Constants.USER_UPDATE)) {
			                Bundle bundle = intent.getExtras();
			                if (null != bundle) {
			                    ...do Something 
			                    }
			                }
			            }
			
			        }
		        }        
		        
		        @Override
			    public void onStop() {
			        super.onStop();
			        			 
			        if (receiveBroadCast != null) {
			            unregisterReceiver(receiveBroadCast);//注销
			        }
			    }
		 
2. **EventBus**
 
            EventBus.getDefault().register(this);//注册
             
            EventBus.getInstance().post(ourEvnet());         //发送
	
		    @Subscribe
		    public void onEvent(TokenInvalidEvent event) {
		       .....处理接收到的消息
		    }
		     
 
			EventBus.getDefault().unregister(this); //注销
       
    3. **LocalBroadcastManager**
    
			//注册
			LocalBroadcastReceiver localReceiver = new LocalBroadcastReceiver();
			LocalBroadcastManager.getInstance(context).registerReceiver(localReceiver, new IntentFilter(ACTION_LOCAL_SEND));  
			
            //发送
			LocalBroadcastManager.getInstance(context).sendBroadcast(new Intent(ACTION_LOCAL_SEND));

            //接收广播
			public class LocalBroadcastReceiver extends BroadcastReceiver {
			 
			    @Override
			    public void onReceive(Context context, Intent intent) {
			        localMsg.setText(intent.getStringExtra(MSG_KEY));
			    }
			}    
			
			//取消
			LocalBroadcastManager.getInstance(context).unregisterReceiver(localReceiver);
好啦，我们可以看到，如果追求代码行数，正常人都会选择用`EventBus`，简单好用！
EventBus < BroadCast ~ LocalBroadCast 。  

- **信息量** 
1.  **系统广播：**
 使用广播另外的一个点是，许多**系统级的事件**都是使用广播来进行的，
 像常用的屏幕开闭、网络状态变化、短信发送接收的状态等等。
 对于这些事情，没办法，系统指定了的，就只能用广播来干了。
但广播的另外一个问题就是，他也是**性能和空间消耗**最大的，
如果你不需要接收系统的消息和跨进程的，那用这个就显得有点浪费了。
2.  **接收方数目**
EventBus 还有一些额外功能，比如说定义多个接收方接收的顺序，LocalBroadcastManager对此不支持，
但全局 Broadcast 是支持的 。

- **灵活性** 
EventBus的调度灵活，不依赖于 Context，使用时无需像广播一样关注 Context 的注入与传递。
父类对于通知的监听和处理可以`继承`给子类，**这对于简化代码至关重要**；
通知的优先级，能够保证 Subscriber 关注最重要的通知；
粘滞事件（sticky events）能够保证通知不会因 Subscriber 的不在场而忽略。
**可继承、优先级、粘滞，是 EventBus 比之于广播、观察者等方式最大的优点，它们使得创建结构良好组织紧密的通知系统成为可能。**


 
 
---
 
 综上，请根据自己需要做选择，已经说的明了，我们来看下源码上的事吧
 
# LocalBroadcastManager 源码解析
API：23
## 获取单例
首先，我们从getInstance（）开始说起，整个类就两百多行，算不复杂的了。

	private static LocalBroadcastManager mInstance;

	public static LocalBroadcastManager getInstance(Context context) {
	        synchronized (mLock) {
	            if (mInstance == null) {
	                mInstance = new LocalBroadcastManager(context.getApplicationContext());
	            }
	            return mInstance;
	        }
	    }

	  private LocalBroadcastManager(Context context) {
        mAppContext = context;
        mHandler = new Handler(context.getMainLooper()) {

            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_EXEC_PENDING_BROADCASTS:
                        executePendingBroadcasts();
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
我们看到些有意思的内容，他的内部使用了`Handler`  ，还记得前面比较说的工作环境吗？我们的本地广播是在主线程的，因为他内部用Handler的时候用的是`MainLooper`，在`handleMessage（）`中会调用接收器对广播的消息进行处理。 

另外我们看到了他的单例是这样些的，这种的效率是偏低的，但实际我们的程序一般没那么高的效率要求，这种也没太大问题，要改的话，可以改成`DCL模式`，或者`静态内部类`的形式。

    public static LocalBroadcastManager getInstance(Context context) {
        if (mInstance == null) {
	        synchronized (mLock) {
	            if (mInstance == null) {
	                mInstance = new LocalBroadcastManager(context.getApplicationContext());
	            }	              
	        }
	     } 
	     
       return mInstance;	        
    }

## 注册广播
	
	private final HashMap<BroadcastReceiver, ArrayList<IntentFilter>> mReceivers
	            = new HashMap<BroadcastReceiver, ArrayList<IntentFilter>>();
    private final HashMap<String, ArrayList<ReceiverRecord>> mActions
	            = new HashMap<String, ArrayList<ReceiverRecord>>();
	            
    public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
                filters = new ArrayList<IntentFilter>(1);
                mReceivers.put(receiver, filters);
            }
            filters.add(filter);
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
我们看到他把广播信息存储下来，我猜应该就是用于后面有收到广播的时候，去查看是否有匹配的消息，然后调用接收去执行的把。 同时mReceivers 是接收器和IntentFilter的对应表，主要作用是方便在unregisterReceiver(…)取消注册。 
mActions 以Action为 key，注册这个Action的BroadcastReceiver链表为 value。我猜是为了方便在广播发送后快速得到可以接收它的BroadcastReceiver。
有这种猜测是因为EventBus的源码里面也有类似的做法，在有程序发送Post消息时候，可以查找然后执行。

## 注销
就让我们来看下他取消注册的时候是怎么干的，来验证下我们前面的猜想把

	 public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
            ArrayList<IntentFilter> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            for (int i=0; i<filters.size(); i++) {
                IntentFilter filter = filters.get(i);
                for (int j=0; j<filter.countActions(); j++) {                     
                    String action = filter.getAction(j);
                    ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    //这几句说明我们猜对了
                    if (receivers != null) {
                        for (int k=0; k<receivers.size(); k++) {
                            if (receivers.get(k).receiver == receiver) {
                                receivers.remove(k);
                                k--;
                            }
                        }
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
## 发送广播消息

为了方便阅读，把一些debug的内容给砍掉了，来让我们看下我们前面的猜想，以及消息是如何传递到目标Receiver吧

	 public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();
            ...
            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            if (entries != null) { 

                ArrayList<ReceiverRecord> receivers = null;
                for (int i=0; i<entries.size(); i++) {
                    ReceiverRecord receiver = entries.get(i); 

                    if (receiver.broadcasting) { 
                        continue;
                    }

                    int match = receiver.filter.match(action, type, scheme, data,
                            categories, "LocalBroadcastManager");
                    if (match >= 0) {
                        if (debug) Log.v(TAG, "  Filter matched!  match=0x" +
                                Integer.toHexString(match));
                        if (receivers == null) {
                            receivers = new ArrayList<ReceiverRecord>();
                        }
                        receivers.add(receiver);
                        receiver.broadcasting = true;
                    }  
	                     ...
                }

                if (receivers != null) {
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    return true;
                }
            }
        }
        return false;
    }
    
开头我们就看到，他取出Action对应的ReceiverRecord列表，遍历每个ReceiverRecord是否 匹配，是的话则保存到receivers中去，接着发送`MSG_EXEC_PENDING_BROADCASTS`消息，通过 Handler 去处理。
对于匹配，我们看到好长一行，`match(action, type, scheme, data,categories, "LocalBroadcastManager")` ，对于匹配规则，估计对于一些人还是很模糊，欢迎百度查阅下。

我们继续来看下这个消息对应的函数执行了什么

	private void executePendingBroadcasts() {
        while (true) {
            BroadcastRecord[] brs = null;
            synchronized (mReceivers) {
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return;
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);
                mPendingBroadcasts.clear();
            }
            for (int i=0; i<brs.length; i++) {
                BroadcastRecord br = brs[i];
                for (int j=0; j<br.receivers.size(); j++) {
                    br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
    
我们看到他对mReceiver加了`synchronized`，防止后面还有人发送广播时候再修改它，导致bug。
最后再做遍历，去调用各个receive的onReceive方法。这样整个广播流程基本就跑完了

另外他还提供了个同步的方法来发送广播，这个有意思。

	* Like {@link #sendBroadcast(Intent)}, but if there are any receivers for
    * the Intent this function will block and immediately dispatch them before
    * returning.
	 public void sendBroadcastSync(Intent intent) {
	        if (sendBroadcast(intent)) {
	            executePendingBroadcasts();
	        }
	    }


## 一些观点:

对于这个本地广播LocalBroadcastManager，我们就有个大致的理解了，怎么说也把人家给看了遍了，
他的底层实现是基于Handler来做应用内的通信，自然安全性更好，相比与基于Binder的系统广播来说，效率更高。
**另外比较坑的是**，
**另外比较坑的是**，
**另外比较坑的是**，
这个类要我们写的接收是`BroadcastReceiver`，但我们阅读完可以知道，完全没必要啊，自己定义一个接口都行啦 ，直接就是匹配然后就执行。这也很好解释他是没有优先级等概念的原因，这个就是"轻量级"的系统广播嘛。

而且，像上面那也不是使用静态内部类的做法不是很好，虽然在这个上下文情况看是不会有大问题。
关于Handler的正确姿势，欢迎查看这篇文章[源码探索系列---Handler与HandlerLeak的那些事](http://blog.csdn.net/sanjay_f/article/details/50208847) 

#最后想说的

虽然比较了这么多，但实际想说的是，`本地广播`和`EventBus`其实对于很多人写的那项目来说是差不多，只是写起来啰嗦点，像我这种追求简洁和效率点的（其实就是懒），应该都会用EventBus，最少从我看的很多项目里面有看到他的影子，就这样^_^。

# 后记
不知道为啥，有个习惯就是有个后记，哈哈，虽然上面写的就是最后想说了。
想在还缺了EventBus的源码解析，虽然这库看了两次吧，不过也没记录过，因为网上分析这个太多而且写得太好了。虽然这个本地广播也有大把人写过，不过自己没看过，就顺便写在这里吧。
接下来有空再去看下系统广播是怎么实现，再 贴出来。

参考来源：

[LocalBroadcastManager](http://developer.android.com/reference/android/support/v4/content/LocalBroadcastManager.html)
 

[Android 应用内全局通知那种方案更好？观察者、eventbus、本地广播](https://www.zhihu.com/question/33030182)
 