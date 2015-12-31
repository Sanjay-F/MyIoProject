title: 源码探索系列3---四大金刚之Activity的启动过程完全解析
date: 2015-12-15 01:32 
tags: [android,源码,Activity,启动过程]
categories: android
------------------------------------------

在不同版本API，底层实现有些不一样，所以这里贴出现在在看的API版本号
API: 23

---

关于Activity的四个启动flag，这里下次再说。
先说下我们熟悉的一句吧

	startActivity(new Intent(this,Test2Activity.class));
相信我们对这句太熟悉不过了，不过说到这里，我想说下关于写Intent的一个感觉好的实践
相信你已写腻了一堆这样的`intent`，用起来确实很不好的，就像下面这样

	Intent newIntent= new Intent(this,Test2Activity.class);
	        newIntent.putExtra("data1", "trialData1");
	        Bundle bundle=new Bundle();
	        bundle.putSerializable("key","trialData"); 
	        startActivity(newIntent);
像这种方式我在很多开源项目看到了很多这样的写法，这种不好的方式在于，很多我们要跳到同一个Activity里面去时候，重复写了，这很显然不适合`DRY原则`，而且我们在另外一个Activity获取数据时候，也需要记住这些Key，这不太好！看到的另外一种方式就是直接写一个`IntentUtil`类，里面写满各种Intent，感觉这种也不是很好！虽然没重复了，不过在获取数据方面还不是很好，还需要定义一些Constant数据，这里提供我个人觉得好的方式，在我们的Activity里面写一个静态的`makeIntent（）`函数。
如下：

	public class Test2Activity extends AppCompatActivity {
	
	
	    public static final String EXTRA_DATA = "extra_data";
	
	    public static Intent makeIntent(Context mContext, String data) {
	        return new Intent(mContext, Test2Activity.class).putExtra(EXTRA_DATA, data);
	    }
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_test2);
	         //获取传过来的数据
	        String result = getIntent().getStringExtra(EXTRA_DATA);
	    }
	}

之后我们需要调整到这个界面，只需要这么写

	startActivity(Test2Activity.makeIntent(this,"data"));
通过这个静态方法，我们在数据里面读取方面都很简单，而且启动的时候也知道是要跳到那个界面去,重要是看起来很简单！而且不需要去定义一个常量类来记录这些数据的`Key`

最后的一个大杀器就是直接写成模板。

<!--more-->
哈哈，之后就可以快捷的自己生成了！！
![这里写图片描述](http://img.blog.csdn.net/20151216174523169)

**以后我们就这样，就生成了，不再需要重复写了！**

![这里写图片描述](http://img.blog.csdn.net/20151216174637003)

**是不是很好！！！**
			
---


---

#StartActivity
上面分享个人觉得好的最佳实战，下面就从这个开始剖析底层是如何启动这个Activity的。

	@Override
	public void startActivity(Intent intent) {
		this.startActivity(intent, null);
	}
	 
	@Override
	public void startActivity(Intent intent, @Nullable Bundle options) {
	   if (options != null) {
	       startActivityForResult(intent, -1, options);
	   } else {
	        startActivityForResult(intent, -1);
	    }
    }
    
	public void startActivityForResult(Intent intent, int requestCode) {
	        startActivityForResult(intent, requestCode, null);
	 }
 
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
从上面代码，我们发现一个很有趣的事实，就是他底层居然调用的是`startActivityForResult()`函数,而且RequestCode是-1。

	if (requestCode >= 0) {
	    mStartedActivity = true;
	}
结合上面这个判断条件，这样我们就理解，为何在startActivity的RequestCode 有这么句解释

> requestCode - If >= 0, this code will be returned in
> onActivityResult() when the activity exits.

 我们需要注意到这么一句，这句实际的执行了intent程序
 
	 Instrumentation.ActivityResult ar =
	                mInstrumentation.execStartActivity(
	                    this, mMainThread.getApplicationThread(), mToken, this,
	                    intent, requestCode, options);
不知道对`MainThread`这个单词还记得嘛，在前篇文章说Looper的时候里面有介绍到这个单词，创说中的主线程，他get回ApplicationThread，我们的app线程。另外这个`Instrumentation`居然出现在这里，我们在介绍测试教程的时候，很多类都是基础于它的，就让我们继续看这个`execStartActivity`到底做了什么

	 public ActivityResult execStartActivity(
	            Context who, IBinder contextThread, IBinder token, Activity target,
	            Intent intent, int requestCode, Bundle options) {
	        IApplicationThread whoThread = (IApplicationThread) contextThread;
	        Uri referrer = target != null ? target.onProvideReferrer() : null;
	        if (referrer != null) {
	            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
	        }
	        if (mActivityMonitors != null) {
	            synchronized (mSync) {
	                final int N = mActivityMonitors.size();
	                for (int i=0; i<N; i++) {
	                    final ActivityMonitor am = mActivityMonitors.get(i);
	                    if (am.match(who, null, intent)) {
	                        am.mHits++;
	                        if (am.isBlocking()) {
	                            return requestCode >= 0 ? am.getResult() : null;
	                        }
	                        break;
	                    }
	                }
	            }
	        }
	        try {
	            intent.migrateExtraStreamToClipData();
	            intent.prepareToLeaveProcess();
	            int result = ActivityManagerNative.getDefault()
	                .startActivity(whoThread, who.getBasePackageName(), intent,
	                        intent.resolveTypeIfNeeded(who.getContentResolver()),
	                        token, target != null ? target.mEmbeddedID : null,
	                        requestCode, 0, null, options);
	            checkStartActivityResult(result, intent);
	        } catch (RemoteException e) {
	            throw new RuntimeException("Failure from system", e);
	        }
	        return null;
	    }
（题外话：当年看《Android系统源代码情景分析》的时候吐槽里面都是贴代码的，现在自己分析时候也贴了好多，哈哈）
我们看到里面结尾处的最重要的一句，好长的一句ActivityManagerNative实际执行了这个startActivity的工作。

	int result = ActivityManagerNative.getDefault()
	                .startActivity(whoThread, who.getBasePackageName(), intent,
	                        intent.resolveTypeIfNeeded(who.getContentResolver()),
	                        token, target != null ? target.mEmbeddedID : null,
	                        requestCode, 0, null, options);

再近一步的看，我们看到这个`ActivityManagerNative` 居然是个**抽象类**，而且基础于`Binder`，实现了`IActivityManager`这个接口。另外要提一点就是这个`@hide`，他的作用很神奇的，可以使类在编译时不对外开放，但是运行的时候这些类和API都是可以访问的。所以要直接使用是会报错的，需要修改下设置的，具体有兴趣的，关于他的经一步内容，找下资料吧

	/** {@hide} */
	public abstract class ActivityManagerNative extends Binder implements IActivityManager
	{
		
	    /**
	     * Retrieve the system's default/global activity manager.
	     */
	    static public IActivityManager getDefault() {
	        return gDefault.get();
	    }
	    
		 private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
		        protected IActivityManager create() {
		            IBinder b = ServiceManager.getService("activity");
		            if (false) {
		                Log.v("ActivityManager", "default service binder = " + b);
		            }
		            IActivityManager am = asInterface(b);
		            if (false) {
		                Log.v("ActivityManager", "default service = " + am);
		            }
		            return am;
		        }
		    };
    }
 好吧，一个神奇的Binder，让我们继续看下去
很遗憾的是，现在都凌晨一点多了，我居然还没找到传说中的`ActivityManagerService`这个类在哪里！！
好消息是，这玩意经过一番折腾，终于发现一个事实，这各类在Android.jar里面没有，得去源码里面找。呵呵。初学，这类坑难免。
位置是`你的SDK位置\sources\android-23\com\android\server\am` 
打开一看，简直疯了，这个类居然

有**20640**行
有**20640**行
有**20640**行
有**20640**行

简直不敢相信，怪不得一个文件有871KB大小，服了，这代码量，简直够一个简单的app的整个代码量了！
先上一张偷来的大图，整个启动的流程。

![这里写图片描述](http://pic002.cnblogs.com/images/2012/328668/2012040716440386.jpg)

还没看完一半的代码。呵呵 
一折腾都一点半了，明天继续写吧。。。
 ---
让我们继续看下了这神奇的`ActivityManagerService`

	@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
这些参数也太多了把，系统的源码这样如此的复杂。

	  final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
 
         ...

            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);


          ...
	}
这个方法也好长，这里贴下最重要的一句`startActivityLocked()`

	 final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
 
        ...
        
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);

        if (err < 0) {
            // If someone asked to have the keyguard dismissed on the next
            // activity start, but we are not actually doing an activity
            // switch...  just dismiss the keyguard now, because we
            // probably want to see whatever is behind it.
            notifyActivityDrawnForKeyguard();
        }
        return err;
    }
越到后面的类都好变态，参数看起来可怕，整个类的行数都用千来做单位，看起来真的好辛苦，那些设计这个系统的一开始得花了多少功夫啊！根据查看，他底部是这个方法

        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
                
好想跳过。。。这个方法里面内容也是超级庞大的。看到好难过。。
里面调整到`resumeTopActivitiesLocked()`接着到了

	 boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        ...
        return result;
    }

下面这个是ActivityStack的resumeTopActivityLocked

	final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
继续坚持看下去。。

	private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        if (DEBUG_LOCKSCREEN) mService.logLockScreen("");

          ....
         
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
         ...
  
    }

苍天，终于这个方法短一点了，都敢直接贴上来了，前面都是几百行的，而且重点是很多操作是没深入看，都晕晕的！！

	  void startSpecificActivityLocked(ActivityRecord r,
	            boolean andResume, boolean checkConfig) {
	        // Is this activity's application already running?
	        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
	                r.info.applicationInfo.uid, true);
	
	        r.task.stack.setLaunchTime(r);
	
	        if (app != null && app.thread != null) {
	            try {
	                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
	                        || !"android".equals(r.info.packageName)) {
	                    // Don't add this if it is a platform component that is marked
	                    // to run in multiple processes, because this is actually
	                    // part of the framework so doesn't make sense to track as a
	                    // separate apk in the process.
	                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
	                            mService.mProcessStats);
	                }
	                realStartActivityLocked(r, app, andResume, checkConfig);
	                return;
	            } catch (RemoteException e) {
	                Slog.w(TAG, "Exception when starting activity "
	                        + r.intent.getComponent().flattenToShortString(), e);
	            }
	
	            // If a dead object exception was thrown -- fall through to
	            // restart the application.
	        }
	
	        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
	                "activity", r.intent.getComponent(), false, false, true);
	    }
好日子到头了，又继续看下吧

	final boolean realStartActivityLocked(ActivityRecord r,
	            ProcessRecord app, boolean andResume, boolean checkConfig)
	            throws RemoteException {
	
	           ...
	           
	            app.forceProcessStateUpTo(mService.mTopProcessState);
	            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
	                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
	                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
	                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
	                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
	        ...
	
	        return true;
	    }

看这些，突然想说，知道个整个流程真不容易，好复杂的流程，一开始搭架构的人能力也太犀利了！设计这么多！
看到这里我们的线索中断了下，因为`app.thread`的这个`thread`是一个接口，继承IInterface，成了个Binder，我们需要找到真正干活的人

	public interface IApplicationThread extends IInterface { 
	
	}
我们搜索了这个ProcessRecord类，发现在他的makeActivie里面对他进行了赋值

	 public void makeActive(IApplicationThread _thread, ProcessStatsService tracker) {
      
         ....
         
        thread = _thread;
    }
    
是时候了，使用传说中的Alt+Ctrl+Shift+F7组合键，我们发现他的来源是在AMS里面，我的跪了,感觉新新苦苦回到革命前的感觉啊。

![这里写图片描述](http://img.blog.csdn.net/20151215222112105)
接着我们继续追寻，发现下面这个入口

	@Override
	    public final void attachApplication(IApplicationThread thread) {
	        synchronized (this) {
	            int callingPid = Binder.getCallingPid();
	            final long origId = Binder.clearCallingIdentity();
	            attachApplicationLocked(thread, callingPid);
	            Binder.restoreCallingIdentity(origId);
	        }
	    }
既然是Override的方法，我们回来看的声明

	public final class ActivityManagerService extends ActivityManagerNative
	        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	
	}
	
好吧，一个熟悉的东西，我们在`ActivityManagerNative`发现了最重要的信息，居然跳回了这里！

	case ATTACH_APPLICATION_TRANSACTION: {
	            data.enforceInterface(IActivityManager.descriptor);
	            IApplicationThread app = ApplicationThreadNative.asInterface(
	                    data.readStrongBinder());
	            if (app != null) {
	                attachApplication(app);
	            }
	            reply.writeNoException();
	            return true;
	        }
        
嫌疑犯`ApplicationThreadNative`出现了，这名字感觉很有规律啊，XXXNative和我们的ActivityManagerNative是类似的，万恶的Binder又出现了。他的具体实现是下面这个`ApplicationThread`，居然是`ActivityThread` 里面的内部类，不要问我怎么知道的


	private class ApplicationThread extends ApplicationThreadNative
 

     ....
     
	public abstract class ApplicationThreadNative extends Binder
        implements IApplicationThread {
        
    }
     
既然回到了ApplicationThread，我们就继续吧，我快看腻了，不知道看这个干什么。。
但最少我们找到了真正干活的人！！！希望你看这么久，不会忘了我们是要找下面这句
 
     app.thread.scheduleLaunchActivity(）
好了，我们继续看他的具体内容把
 	              
	public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
                Bundle state, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
                String profileName, ParcelFileDescriptor profileFd, boolean autoStopProfiler) {
            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profileFile = profileName;
            r.profileFd = profileFd;
            r.autoStopProfiler = autoStopProfiler;

            updatePendingConfiguration(curConfig);

            queueOrSendMessage(H.LAUNCH_ACTIVITY, r);
        }
发送消息，又事坑

	private void queueOrSendMessage(int what, Object obj) {
	       queueOrSendMessage(what, obj, 0, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1) {
        queueOrSendMessage(what, obj, arg1, 0);
    }

    private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        } 
    }
    
	final H mH = new H();
	
我们看到，他最后居然用一个Handler发送消息，呵呵，而且这个handler居然那么简单的名字

	private class H extends Handler {
	
        public static final int LAUNCH_ACTIVITY         = 100;
        public static final int PAUSE_ACTIVITY          = 101; 

	    ...
     
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + msg.what);
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                } break;
                case RELAUNCH_ACTIVITY: {
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                    handleRelaunchActivity(r);
                } break;
                
                ...
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + msg.what);
        }
    }


继续看下去吧，`handleLaunchActivity()`,写到这里，很想说，没什么动力，估计没什么人没事会把整个系统的源码看一遍。实在好累的感觉

    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        

        if (r.profileFd != null) {
            mProfiler.setProfiler(r.profileFile, r.profileFd);
            mProfiler.startProfiling();
            mProfiler.autoStopProfiler = r.autoStopProfiler;
        }

        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward);

            
                r.paused = true;
            }
        }  
        
        ...
    }
    
恭喜你，看到这里基本到尾声了，我们的Activity被创建出来了，这个函数一百多行，实在太长了，就不贴了
，一个函数长度都够我平时写的一个类的长度了。
在performLaunchActivity(）函数里面，对我们的Activity粘贴到window，设置主题，设置Context ，调用OnCreate函数等等操作。 

然后就是在handleResumeActivity里面，会调用我们最熟悉不过的`onResume()`函数！
好了，到这里结束！！！！！

关于别的Activity的生命周期函数是怎么调用到的，下次有空再继续写！
就这点内容，看了我几个小时，得洗澡先了！！


---

# 后记
第一次自己看完这么长的一个流程，真的是不容易，写这篇花了我两个晚上
虽然觉得挺好，不过也不知道有什么意义呢，看完整个启动的生命周期！
写系统代码的人也是牛逼，绕来绕去的，这么长流程，想做到没bug的话真的不容易！
我看着就累了，特别是看到那个AMS里面居然**两万多行！！！**，
这一个类，真的完全都够写一个中小型的项目的代码量了
发现自己贴了好多代码，整篇文章都16，000多了，好长！！
估计一般人都不会看，就当记录下整个流程，
下次遇到问题，方便自己查阅

-------------------再次更新--------------------

**补充下一个笑话**

深夜，交警查车，突然插到一部车的后尾箱放了几百万美元，在这样一个夜黑风高之晚，显然很有嫌疑，所以警察质问司机：“为何这么晚还带这么多钱跑？”，实际虽然有点紧张，但就回答了句，“嗯，我能”。
这个是以前在知乎看到的。具体大意入手，最终想说的就是那么两个字，因为我能。>.<


---


参考：
整个流程图，来自下面这篇文章
[Activity启动创建 (AcitivtyManageService,ActivityThread,Activity)](http://www.cnblogs.com/lijunamneg/p/3573093.html)
 