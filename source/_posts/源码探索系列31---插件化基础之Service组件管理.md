title: 源码探索系列31---插件化基础之Service组件管理
date: 2016-04-17 17:27:46
tags: [android,源码,DroidPlugin,AMS,AMN,service]
categories: android

------------------------------------------
  
今天我们来看下四大金刚之一的另外一个Sevice。
他和Activity一样，也需要提前占坑，然后HOOK才可以使用。
就让我们来看性下整个过程会涉及到的过程，顺便温习下Service的启动流程吧。

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/service.png)

<!--more-->

#起航
API:23

为了能够成功的HOOK，我们需要找下HOOK点。根据前面在[源码探索系列7---四大金刚之Service](http://sanjay-f.github.io/2015/12/18/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%977---%E5%9B%9B%E5%A4%A7%E9%87%91%E5%88%9A%E4%B9%8BService/)记录，我们了解一个大概的Service启动过程。很有意思的是，他的流程和启动Activity某种程度的神似。我们在这里把启动service过程做个小结，大致流程如下：

当我们调用`context.StartService(serviceIntent)`后，他经过`AMS`和`AS`的一顿处理，最后在AS的`realStartServiceLocked（）`里面会去调用了`ActivityThread`的`scheduleCreateService()`函数 

	public final void scheduleCreateService(IBinder token,
            ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
        updateProcessState(processState, false);
        CreateServiceData s = new CreateServiceData();
        s.token = token;
        s.info = info;
        s.compatInfo = compatInfo;

        sendMessage(H.CREATE_SERVICE, s);
    }
这个函数有重要的参数就是`CreateServiceData`，我们创建service的信息在他里面。这个函数会发送一个`H.CREATE_SERVICE`消息给我们的Handle，由他去执行后面的创建工作，内容如下 

	private void handleCreateService(CreateServiceData data) { 

	    LoadedApk packageInfo = getPackageInfoNoCheck(
	            data.info.applicationInfo, data.compatInfo);
	    Service service = null;
	
	    java.lang.ClassLoader cl = packageInfo.getClassLoader();
	    //创建我们的service！
	    service = (Service) cl.loadClass(data.info.name).newInstance();
	
	    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
	    context.setOuterContext(service);
	
	    Application app = packageInfo.makeApplication(false, mInstrumentation);
	    service.attach(context, this, data.info.name, data.token, app,
	             ActivityManagerNative.getDefault());
	             
		    service.onCreate();//oncreate函数，onStartCommand()在别的地方
	    mServices.put(data.token, service);
	
	    ActivityManagerNative.getDefault().serviceDoneExecuting(
	               data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);	
	               
	}

和我们的Activity有些类似，一个`LoadedApk`参数，另外用`token`标记特定的`service`，放在`mServices`里面缓存起来。不过有点不一样的是，如果我们直接启动没在AndroidManifese里面注册的service，系统是不会崩溃报错的。

**重要：**
根据上面的内容，我们HOOK的一个点就是要确保最后的`data.info.name` 变量是我们的目标`service`！

在`AS`调用完上面的函数后，他会继续调用`sendServiceArgsLocked(r, execInFg, true);` 函数，这函数会调用`ActivityThread`的内部类`ApplicationThread`的 `scheduleServiceArgs()`函数，发送一条`SERVICE_ARGS`消息给我们的handle，最后由他来调用我们的service的`onStartCommand()`函数，把参数发回来。

	public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, 
										int startId,int flags ,Intent args) {
										
            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = taskRemoved;
            s.startId = startId;
            s.flags = flags;
            s.args = args;

            sendMessage(H.SERVICE_ARGS, s);
        }


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
	
（题外话，不知道你有没注意到一点，不少变量都是只有一个字母，例如上面的`“S"`，很好奇为何这么做。这名字起得让人觉得很不舒服，如果你去细看ActivityService的代码，里面不少变量也是只有一个字母，估计都是同一个人写的这部分代码）

这样我们算是快速的温习了一边service的启动过程，在看这个过程，我们已经目色到可以使用的HOOK点了.和我们的Activity类似的，在送去AMS前我们要掉包，接着在回调去创建service时候，需要再处理一下。
也那就是AMS和handle的mCallback组合。

# 前进

根据前面的简单分析，这样我们可以些出这样的最原始的模板代码。
假设我们有两个Service，一个`ProxyService`和`TargetService`，而且我们只在AndroidManifeset注册了ProxyService;
	
	<service android:name=".ProxyService" />

接着在我们的MainActivty里面，我们去启动我们的TargetService。

	public class MainActivity extends Activity {

	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        getApplicationContext().startService(new Intent(this, TargetService.class));
	    }
	
	
	    @Override
	    protected void attachBaseContext(Context newBase) {
	        super.attachBaseContext(newBase);
	        HookFactory.hookActivityThreadHandle();
	        HookFactory.hookActivityManagerNative();         
	    }
	}

然后HOOK得内容我们都很熟悉了，和前面的Activity类似的。

	public static void hookActivityThreadHandle() {


        Class<?> activityThreadClass = null;
        try {
            activityThreadClass = Class.forName("android.app.ActivityThread");

            Field currentActivityThreadField = activityThreadClass.getDeclaredField(
                    "sCurrentActivityThread");
            currentActivityThreadField.setAccessible(true);
            Object currentActivityThread = currentActivityThreadField.get(null);

            //拿我们的H先生
            Field mHField = activityThreadClass.getDeclaredField("mH");
            mHField.setAccessible(true);
            Handler mH = (Handler) mHField.get(currentActivityThread);

            Field mCallBackField = Handler.class.getDeclaredField("mCallback");
            mCallBackField.setAccessible(true);
            //修改它的callback为我们的,从而HOOK掉
            mCallBackField.set(mH, new ActivityThreadHandlerCallback(mH));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    public static void hookActivityManagerNative() {

        Class<?> activityManagerNativeClass = null;
        try {
            activityManagerNativeClass = Class.forName(
			            "android.app.ActivityManagerNative");


            Field gDefaultField = activityManagerNativeClass.getDeclaredField(
												            "gDefault");
            gDefaultField.setAccessible(true);

            Object gDefault = gDefaultField.get(null); 
            Class<?> singleton = Class.forName("android.util.Singleton");
            Field mInstanceField = singleton.getDeclaredField("mInstance");
            mInstanceField.setAccessible(true); 
            
            Object rawIActivityManager = mInstanceField.get(gDefault);             
            Class<?> iActivityManagerInterface = Class.forName(
					            "android.app.IActivityManager");
            Object proxy = Proxy.newProxyInstance(
		            Thread.currentThread().getContextClassLoader(),
                    new Class<?>[]{iActivityManagerInterface}, 
                    new IActivityManagerHook(rawIActivityManager));
                    
            mInstanceField.set(gDefault, proxy);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

接着我们看下IActivityManagerHook 里面的invoke内容

	@Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {

        Log.e(TAG, "invoke() called with: " + "proxy = [" + " " + method.getName());

        if (method.getName().equals("startService")) { 
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof Intent) {
                    Intent ourFakeIntent = new Intent();
                    ComponentName componentName = new ComponentName(
                            "com.example.sanjay.demohook",
                            ProxyService.class.getName());//替换                     
                    ourFakeIntent.setComponent(componentName);
                    ourFakeIntent.putExtra(EXTRA_ORIGINAL_INTENT,
                            (Intent) args[i]);//保存真的intent
                    args[i] = ourFakeIntent;
                    break;
                }
            }
            return method.invoke(originalObject, args);
        }
    }
主要就是替换成我们的proxyService
然后我们来看下那个ActivityThreadHandlerCallback里面的handleMessage内容:

	@Override
    public boolean handleMessage(Message msg) { 
        switch (msg.what) {             
            case CREATE_SERVICE: {
                Object obj = msg.obj;
                try { 
                    Field serviceInfo= obj.getClass().getDeclaredField("info");
                    serviceInfo.setAccessible(true);
                    ServiceInfo info= (ServiceInfo) serviceInfo.get(obj);
                    info.name="com.example.sanjay.demohook.TargetService";
                } catch (Exception e) {
                    e.printStackTrace();
                    Log.e(TAG, "handleMessage() called with: error=" +e.getMessage());
                }
            } 
            break;
        } 
        mOriginalHandle.handleMessage(msg);
        return true;
    }
好了，这样基本就写好了。
需要注意的是，这里采用直接硬编码的方式，把info.name直接换成了我们的targetService，而不是像启动
Activity那样的，采用intent去换，是因为系统传到这里的参数是不一样的。只有到了`handleServiceArgs`时候，系统才会传Intent过来。所以可能我们需要配置下对应的ProxyService和TargetService之间的对应关系。这里为了说明方便，就直接这么做使用了。

最后是我们的service

	public class TargetService extends Service {
	    private static final String TAG = TargetService.class.getSimpleName();
	
	    public TargetService() {
	    }
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	        Log.e(TAG, "ok ,now is onCreate: ");
	    }
	
	    @Override
	    public IBinder onBind(Intent intent) {
	        throw new UnsupportedOperationException("Not yet implemented");
	    }
	
	    @Override
	    public int onStartCommand(Intent intent, int flags, int startId) {
	        Log.e(TAG, " congratulation --onStartCommand success ");
	        return super.onStartCommand(intent, flags, startId);
	    }
	}
			  
让我们来启动一下看看！

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160417155107.jpg)

根据打印的内容，我们想我们基本是成功了！

但这和前面在介绍关于Activity的启动一样，目前我们的`TargetService`还是在已知包内的。如果我们想启动插件包里面的service，还需要做些额外的工作。例如那个LoadedApk变量等等。

 ---
# 小结
 
目前我们hook方案很”原生态“，DroidPlugin原本也是这样的方案，不过我们看到他把相关的代码都注释了，看来用了另外一种解决方案来处理这个问题，那么它是怎么处理这问题的。下次继续写，今天的就到这里吧


# DroidPlugin的方案

目前我们在`IActivityManagerHookHandle`看到，它会在发送去AMS前对`Intent`做些处理做些。如下


	 private static class startService extends ReplaceCallingPackageHookedMethodHandler{

        public startService(Context hostContext) {
            super(hostContext);
        }
        private ServiceInfo info = null;

        @Override
        protected boolean beforeInvoke(Object receiver, Method method, Object[] args){                 
            info = replaceFirstServiceIntentOfArgs(args);
            return super.beforeInvoke(receiver, method, args);
        }

        @Override
        protected void afterInvoke(Object receiver, Method method, Object[] args,
         Object invokeResult) {
         
            if (invokeResult instanceof ComponentName) {
                if (info != null) {
                   //让startService函数返回这个构造的假结果
                    setFakedResult(new ComponentName(info.packageName, info.name));
                }
            }
            info = null;
            super.afterInvoke(receiver, method, args, invokeResult);
        }
    }

看下`replaceFirstServiceIntentOfArgs`，主要是替换传过来的Intent内容：

	private static ServiceInfo replaceFirstServiceIntentOfArgs(Object[] args){
	
        int intentOfArgIndex = findFirstIntentIndexInArgs(args);
        if (args != null && args.length > 1 && intentOfArgIndex >= 0) {
        
            Intent intent = (Intent) args[intentOfArgIndex];
            ServiceInfo serviceInfo = resolveService(intent);
            
            if (serviceInfo != null && isPackagePlugin(serviceInfo.packageName)) {
                ServiceInfo proxyService = selectProxyService(intent);
                if (proxyService != null) {
                    Intent newIntent = new Intent();
                    //FIXBUG：https://github.com/Qihoo360/DroidPlugin/issues/122
                    //如果插件中有两个Service：ServiceA和ServiceB，在bind ServiceA的时候会调用
                    //ServiceA的onBind并返回其IBinder对象，但是再次bind ServiceA的时候还是会返
                    //回ServiceA的IBinder对象，这是因为插件系统对多个Service使用了同一个
                    //StubService来代理，而系统对StubService的IBinder做了缓存的问题。
                    //这里设置一个Action则会穿透这种缓存。                    
                    newIntent.setAction(proxyService.name + new Random().nextInt());
                  
                    //作者这也是绝啊，弄个随机数来凑action
                                       
                    newIntent.setClassName(proxyService.packageName, proxyService.name);
                    newIntent.putExtra(Env.EXTRA_TARGET_INTENT, intent);
                    newIntent.setFlags(intent.getFlags());
                    args[intentOfArgIndex] = newIntent;
                    return serviceInfo;
                }
            }
        }
        return null;
    }
 
这个和我们的Activity类似，需要替换下`Intent`内容。`resolveService`背后会去查看下是否对应的插件有匹配的Service，整条过程还是挺长的。
 
 ------
 
在发送出这个消息后，我们要看的当然就是之后的启动过程啦，是吧？看下背后是怎么启动我们的目标Service。这个工作我们在`ServiceStub`的父类`AbstarctServiceStub`我们看到一些线索。

	@Override
    public void onStart(Intent intent, int startId) {
 
        if (intent != null) {
            if (intent.getBooleanExtra("ActionKillSelf", false)) {
                startKillSelf();
                if (!ServcesManager.getDefault().hasServiceRunning()) {
                    stopSelf(startId);
                    boolean stopService = getApplication().stopService(intent);
                 } 
            } else {
                mCreator.onStart(this, intent, 0, startId);// <--这句
            }
        } 
        super.onStart(intent, startId);
    }
    
我们在前面的分析文章有提到过，当发送的`intent`走一轮，回调到
`ActivityThread`的`handleCreateService` 时候，发送过来的参数类型是
`CreateServiceData`的，里面是没包含我们一开始发送出去的`intent`的，所以导致我们不能在这个环节做掉包工作，把原来的ProxyService换成我们的目标TargetService，为了能做到掉包工作，我们可能需要做些别的额外的工作，例如弄一个缓存代理service和目标Service的关系等。
但在这里的就换了种做法，在走到代理Service的`onStart()`函数再去做启动工作。 

我们到mCreator.onStart里面看下内容

	public int onStart(Context context, Intent intent, int flags, int startId){
	
        Intent targetIntent = intent.getParcelableExtra(Env.EXTRA_TARGET_INTENT);
        if (targetIntent != null) {
            ServiceInfo targetInfo = PluginManager.getInstance().
						               resolveServiceInfo(targetIntent, 0);
            if (targetInfo != null) {
                Service service = mNameService.get(targetInfo.name);
                if (service == null) {
                    handleCreateServiceOne(context, intent, targetInfo);
                }
                handleOnStartOne(targetIntent, flags, startId);
            }
        }
        return -1;
    }
这里取出了我们的目标intent的内容，然后如果这service没创建过，那么缓存的mNameService就没记录，那么用`handleCreateServiceOne`去创建一个。我们继续看下内容

	//这个需要适配,目前只是适配android api 21
    private void handleCreateServiceOne(Context hostContext, Intent stubIntent, ServiceInfo info) throws Exception {
        //            CreateServiceData data = new CreateServiceData();
        //            data.token = fakeToken;// IBinder
        //            data.info =; //ServiceInfo
        //            data.compatInfo =; //CompatibilityInfo
        //            data.intent =; //Intent
        //            activityThread.handleCreateServiceOne(data);
        //            service = activityThread.mTokenServices.get(fakeToken);
        //            activityThread.mTokenServices.remove(fakeToken);
        ResolveInfo resolveInfo = hostContext.getPackageManager().
										      resolveService(stubIntent, 0);
										      
        ServiceInfo stubInfo = resolveInfo != null ? resolveInfo.serviceInfo : null;
        PluginManager.getInstance().reportMyProcessName(stubInfo.processName, 
												      info.processName, 
													   info.packageName);
													   
        PluginProcessManager.preLoadApk(hostContext, info);
        Object activityThread = ActivityThreadCompat.currentActivityThread();
        
        IBinder fakeToken = new MyFakeIBinder();
        Class CreateServiceData = Class.forName(ActivityThreadCompat.activityThreadClass()
										        .getName()+ "$CreateServiceData");
										        
        Constructor init = CreateServiceData.getDeclaredConstructor();
        if (!init.isAccessible()) {
            init.setAccessible(true);
        }
        Object data = init.newInstance();
        // 自己构造一个CreateServiceData对象
        FieldUtils.writeField(data, "token", fakeToken);
        FieldUtils.writeField(data, "info", info);
        if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB) {
            FieldUtils.writeField(data, "compatInfo", 
						          CompatibilityInfoCompat.DEFAULT_COMPATIBILITY_INFO());
        }

		//然后按流程的调用handleCreateService去创建Service
		//配套的LoadedApk变量已在前面的PluginProcessManager.preLoadApk(hostContext, info)生成了	
		
        Method method = activityThread.getClass().getDeclaredMethod("handleCreateService", 
															        CreateServiceData);         														        
        if (!method.isAccessible()) {
            method.setAccessible(true);
        }
        method.invoke(activityThread, data);
  
        Object mService = FieldUtils.readField(activityThread, "mServices");
        Service service = (Service) MethodUtils.invokeMethod(mService, "get", fakeToken);
        MethodUtils.invokeMethod(mService, "remove", fakeToken);
        
        mTokenServices.put(fakeToken, service);
        mNameService.put(info.name, service);

        if (stubInfo != null) {
          //缓存代理和实际的service对应的关系信息  
          PluginManager.getInstance().onServiceCreated(stubInfo, info);
        }
    }
 
我们看到很有趣的事情，就是他不是去拦截`mCallback`，而是在代理service启动之后，再去实际启动插件的
Service，和我们预期的启动Activity的流程有些不一样的地方！

# 小结

DroidPlugin在解决启动Service方面，采用的是一对多的方案。








 