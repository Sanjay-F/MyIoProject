title: 源码探索系列7---四大金刚之Service
date: 2015-12-18 12:57
tags: [android,源码,service]
categories: android
------------------------------------------

今天我们来看下这安卓的四大组件的另外一个Service，
按套路应该先列点我们在探索过程需要注意的问题，不过现在一时没想到有什么，
让我们边看说解释，看下有什么需要注意的

# 起航----开启服务

API ： 23

我们启动服务一般有两种调用`StartService`和`BindService`,这里我们先从StartService开始。
首先去到ContextWrapper里面的startService函数

	@Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
	
<!--more-->
	
这个我们在前一篇也分析广播时候也遇到过类似情况，这个mBase的具体实现是ContextImpl，我们继续看下

	@Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser);
    }
    
    private ComponentName startServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess();
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
              
               ...
            return cn;
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
    }
好吧，看到这个AMN，最近我们都看到他，已经在熟悉不过了，他的实现是AMS，我们就跑去哪里再看下那个startService里面做了什么吧

	    @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId)
            throws TransactionTooLargeException {
        enforceNotIsolatedCaller("startService");
        
          ...
         
        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
他在底层去调用了`ActivityService.startServiceLocked()`,感觉这个命名格式也很有熟悉感啊，来我们继续看下他做了什么

	 ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, int userId)
            throws TransactionTooLargeException {

		ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg);
     
        ServiceRecord r = res.record;
 
         ...
        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
经过一对的处理，他最后调用了startServiceInnerLocked（）；

	 ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
         ...
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false);
        if (error != null) {
            return new ComponentName("!!", error);
        }

        ...
        return r.name;
    }
继续看下里面的内容

	private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting) throws TransactionTooLargeException {
 
           ...
	       app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
           realStartServiceLocked(r, app, execInFg);
           ...
 
    }
哈，`realStartServiceLocked`这名字看起来，前面都是骗人的启动过程，现在才开始干真活的样子

    private final void realStartServiceLocked(ServiceRecord r,
           ProcessRecord app, boolean execInFg) throws RemoteException {
           
            ... 
       try {
           ...
           
           mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
           app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
           app.thread.scheduleCreateService(r, r.serviceInfo,
                   mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                   app.repProcState);
           r.postNotification();
           created = true;
       } catch (DeadObjectException e) {
           ...
       }

       requestServiceBindingsLocked(r, execInFg);

       updateServiceClientActivitiesLocked(app, null, true);

       // If the service is in the started state, and there are no
       // pending arguments, then fake up one so its onStartCommand() will
       // be called.
       if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
           r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                   null, null));
       }

       sendServiceArgsLocked(r, execInFg, true);

       ...
    }
   好了，截取掉一部分不相干的消息，我们现在看下他实际做了什么，我们看到了他是去调用app.thread.scheduleCreateService(）函数去执行service的，这个我们前篇文章有看到过这个app.thread，最终干活的是ActivityThread里的一个ApplicationThread。让我们去看下他实际干了什么。
 另外结尾也有注释，如果service已启动且没Pending arguments，那是去调用了他`onStartCommand()`
好了，还是就像去看下主线任务吧

	public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
我们的”H“显示似乎又要登场了，靠他来发送一个创建服务的消息来干活。

	private void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }
    
    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
 
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
确实是去调用我们的H朋友去发送消息去了，我们看下消息里面调用的`handleCreateService((CreateServiceData)msg.obj)`具体做了什么

	private void handleCreateService(CreateServiceData data) { 

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
 
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
  
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                 ActivityManagerNative.getDefault());
        service.onCreate();
        mServices.put(data.token, service);
        
        ActivityManagerNative.getDefault().serviceDoneExecuting(
                   data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);

    }
为何方便看，精简了下代码，从上面的代码可以看到，到这里用ClassLoader去启动了服务，配置了下Context，这个就是我们然后调用了服务的onCreate方法。好了，这样我们的启动过程基本就到这里了。

另外，我们知道onCreate函数执行完后，就是到了onStartCommand（）啦，不过这个过程好像没看到啊？跑去哪里了呢？

这是因为还有一部分代码没说，重新回到我们的 真 · 干活的`realStartServiceLocked`函数的地步，有这么一个函数我留着  `sendServiceArgsLocked(r, execInFg, true);`　，他的作用就是发送参数，让我们看下这部分内容

	 private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
       ...
       r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
       ．．． 　
    }
只留下最重要的一句话，就是调度发送service的参数

	public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags ,Intent args) {
            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = taskRemoved;
            s.startId = startId;
            s.flags = flags;
            s.args = args;

            sendMessage(H.SERVICE_ARGS, s);
        }
这个复杂发送了一个消息回去，代码在下面

	private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {

            if (data.args != null) {
                data.args.setExtrasClassLoader(s.getClassLoader());
                data.args.prepareToEnterProcess();
            }
            int res;
            if (!data.taskRemoved) {
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }
			...
       }
    }
好了，这样我们的onStartCommannd也就集齐了。
整个流程算跑完一半了。

# 前进--- --- 停止服务

在开始看过程前，我们来猜下他打开的过程，应该最后也是跑回到H里面执行一个什么`STOP_SERVICE`消息。带着这个猜想，我们偷下懒，直接从我们的H先生哪里看下消息列表，看下有没我们想要的

	 case CREATE_SERVICE:
         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceCreate");
         handleCreateService((CreateServiceData)msg.obj);
         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
         break;
     case BIND_SERVICE:
         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
         handleBindService((BindServiceData)msg.obj);
         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
         break;
     case UNBIND_SERVICE:
         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceUnbind");
         handleUnbindService((BindServiceData)msg.obj);
         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
         break;
     case SERVICE_ARGS:
         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStart");
         handleServiceArgs((ServiceArgsData)msg.obj);
         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
         break;
     case STOP_SERVICE:
         Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStop");
         handleStopService((IBinder)msg.obj);
         maybeSnapshot();
         Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
         break;
	
哈哈，真的中了，我们看到还有`BIND_SERVICE`和`UNBIND_SERVICE`，全部在这里，那么中间过程我们就可以猜想啦

    private void handleStopService(IBinder token) {
        Service s = mServices.remove(token);
        if (s != null) {
            try { 
                s.onDestroy();
                Context context = s.getBaseContext();
                if (context instanceof ContextImpl) {
                    final String who = s.getClassName();
                    ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
                }
                QueuedWork.waitToFinish();
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                            token, SERVICE_DONE_EXECUTING_STOP, 0, 0); 
            } catch (Exception e) {
              ...
            }     
       }
    } 

# 大跃进--- --- 绑定服务

现在我们直接来看下绑定服务是怎么回事


	@Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }

老套路，我们继续看下ContextImpl的内容

	@Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }

	private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                    mMainThread.getHandler(), flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess();
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
    }
我们这里看到，最重要的一句还是上面第十行的位置，跑去调用AMS的`bindService`了。
另外这里有点要补充的内容就是，我们的这个绑定可能是跨进程的情况啊，所以我们看到上面，他把我们的
ServiceConnection被再次打包了一下，弄成了`IServiceConnection`，显然这个是一个`Binder`，和我们的广播类似啊，被打包成了`IIntentReceiver`，从而达到能够跨进程的效果。这个IServiceConnection也是在LoadedApk里面啊，保存着一个map，存储两者的关系，这里就不细细去看了，继续看我们的主线吧。

	public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {
        enforceNotIsolatedCaller("bindService");

        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);
        }
    }
我们看到结尾这里，继续调用了ActiveServices的bindServiceLocked(）。想说的是，这个ActivityService是负责给AMS打工，管理Service的启动，绑定，解绑和停止等工作的，终于想说这个AMS里面有点像组合模型的地方，把一些业务分割出去了。应该在分割出去点的，虽然很多都是判断和处理语句，太庞大了，整个AMS类

我们继续看下那个`bindServiceLock`，他和我们启动任务的时候类似，去调用了`bringUpServiceLocked`，调用完后就去`realStartServiceLocked`，后面流程类似，就不重复，唯一看完全部，不一样的地方当然是我们是绑定啊，要返回连接的啊！
在`RealStartServiceLock`里面，他继续去执行了函数结尾处的下面的函数

	requestServiceBindingsLocked(r, execInFg);
    updateServiceClientActivitiesLocked(app, null, true);

	private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
 
          ...
          
        if ((!i.requested || rebind) && i.apps.size() > 0) {
             
	        bumpServiceExecutingLocked(r, execInFg, "bind");
            r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                    r.app.repProcState);
            if (!rebind) {
                i.requested = true;
            }
            i.hasBound = true;
            i.doRebind = false;
             
        }
        ...
        return true;
    }
 这样我们就看到他去调用多了ApplicationThread里面的scheduleBindService去绑定多服务！

	public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
            sendMessage(H.BIND_SERVICE, s);
        }
通过这里去绑定我们的服务

	private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
 
        if (s != null) {            
	         data.intent.setExtrasClassLoader(s.getClassLoader());
	         data.intent.prepareToEnterProcess();
	         try {
	             if (!data.rebind) {
	                 IBinder binder = s.onBind(data.intent);
	                 ActivityManagerNative.getDefault().publishService(
	                         data.token, data.intent, binder);
	             } else {
	                 s.onRebind(data.intent);
	                 ActivityManagerNative.getDefault().serviceDoneExecuting(
	                         data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
	             }
	             ensureJitEnabled();
	         } catch (RemoteException ex) {
	         }         
           
        }
    }
在这里我们看到，对于多次绑定，他调用了我们服务的onRebind函数！而对于第一次绑定，我们是调用了我们的服务的onBind函数，这和我们以前的开发经验也很对，不过好像缺少了跑到回调去的步骤啊，
看下去，嗯，应该是用AMS的`publishService`去做的，让我们探索下去吧！。

	public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
你看，我们的小助手mService又出现了，真的是管理Service的好助手啊

	void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity(); 　　
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {　
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = 
                        r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);                            
                            ...                                                                                     
                            c.conn.connected(r.name, service);                             
                        }
                    }
                }
              　serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
           } 　
	 }
	 
　看到这里，需要解释下，还记得前面说，我们的ServiceConnection是被打包起来，弄成了`IServiceConnection`，才来达到跨进程的效果的，这里的`c.conn`就是原来那个iServiceConnection。
　很显然他的这个connected方法就是回调，像前面说的广播一样，再在里面去执行我们的方法的回调。

	private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service);
                }
            }
        }
这个我们看到跑到了`sd.connected();`不小心看成了连接sb.

	public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);
            }
        }
好了，这里有看到一些熟悉的内容啦！！！
这个坑爹的mActivyThread！！实际上就是那个“H”先生！！，这里就不解释了！通过他来运行回主线程！既然是调用它的post方法去执行，继续看下那个RunConnection里面的内容吧

	public void run() {
      if (mCommand == 0) {
               doConnected(mName, mService);
           } else if (mCommand == 1) {
               doDeath(mName, mService);
           }
    }
好了，我们去看下doConencted的内容把

	public void doConnected(ComponentName name, IBinder service) {
               ...
             if (service != null) {
                mConnection.onServiceConnected(name, service);
            }
    }
到这里我们就回调到我们绑定的时候的的回调接口去。
好了，贴了这么多代码，终于进入尾声了，我们的整个绑定过程！
　
　 

---
　
　
# 后记

偷下懒，别的过程就先不写了。其余的过程基本最后也是跑回到那个H先生那里去执行下的，没什么大的问题的。 
另外，说到Service，都会想到他的朋友IntentService，为何它可以执行一些耗时的操作呢？
这部分没写下来。

这样就剩下最后一个金刚没写他的故事啦，今晚抽空写下，就算对四大金刚有一个进一步的了解了！