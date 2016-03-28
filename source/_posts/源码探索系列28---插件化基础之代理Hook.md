title: 源码探索系列28---插件化基础之代理Hook
date: 2016-03-28 18:41:46
tags: [android,源码,Hook]
categories: android

------------------------------------------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160328184052.png)

代理有两种，一种是常见的静态代理模型，这个在[设计模式系列2---幕后黑手的代理模式](http://sanjay-f.github.io/2015/12/30/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%E7%B3%BB%E5%88%972---%E5%B9%95%E5%90%8E%E9%BB%91%E6%89%8B%E7%9A%84%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F/)就有写过，他可以让我们在调用实际的api时做一些别的工作，从而达到某种程度的AOP效果。
但当时并没有提到动态代理的概念，因为目前自己的开发需要用到的场景偏少。不过在DroidPlugin里面就很多，应为他需要hook很多的内容。

<!--more-->

# 起航
API:23

在做插件化，需要的一件事情就是Hook掉部分的内容，由我们来做接替。
所以我们先从继承的PluginApplicatiion开始看起

	public class PluginApplication extends Application {	 
	
	    @Override
	    public void onCreate() {
	        super.onCreate();	        
	        PluginHelper.getInstance().applicationOnCreate(getBaseContext());
	    } 
	    ...
	}
	
	
	public void applicationOnCreate(final Context baseContext) {
	        mContext = baseContext;
	        initPlugin(baseContext);
    }
	
//HOOK起点    

	private void initPlugin(Context baseContext) {
				...
       PluginPatchManager.getInstance().init(baseContext);
       PluginProcessManager.installHook(baseContext);
    
       if (PluginProcessManager.isPluginProcess(baseContext)) {
           PluginProcessManager.setHookEnable(true);
       } else {
           PluginProcessManager.setHookEnable(false);
       }             
	   ...
     }
我们跳到PluginProcessManager看下里面内容，是调用Hook工厂类去安装这个HOOK内容，里面内容很多

	public static void installHook(Context hostContext) throws Throwable {
	        HookFactory.getInstance().installHook(hostContext, null);
	    }


传说的熟悉各个版本的差异，兼容问题就在下面有所体现了：

	public final void installHook(Context context, ClassLoader classLoader) throws Throwable {
        installHook(new IClipboardBinderHook(context), classLoader);
        //for ISearchManager
        installHook(new ISearchManagerBinderHook(context), classLoader);
        //for INotificationManager
        installHook(new INotificationManagerBinderHook(context), classLoader);
        installHook(new IMountServiceBinder(context), classLoader);
        installHook(new IAudioServiceBinderHook(context), classLoader);
        installHook(new IContentServiceBinderHook(context), classLoader);
        installHook(new IWindowManagerBinderHook(context), classLoader);
        if (VERSION.SDK_INT >= VERSION_CODES.LOLLIPOP_MR1) {
            installHook(new IGraphicsStatsBinderHook(context), classLoader);
        }
        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            installHook(new WebViewFactoryProviderHook(context), classLoader);
        }
        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            installHook(new IMediaRouterServiceBinderHook(context), classLoader);
        }
        if (VERSION.SDK_INT >= VERSION_CODES.LOLLIPOP) {
            installHook(new ISessionManagerBinderHook(context), classLoader);
        }
        if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN_MR2) {
            installHook(new IWifiManagerBinderHook(context), classLoader);
        }

        if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN_MR2) {
            installHook(new IInputMethodManagerBinderHook(context), classLoader);
        }
        if (VERSION.SDK_INT >= VERSION_CODES.ICE_CREAM_SANDWICH_MR1) {
            installHook(new ILocationManagerBinderHook(context), classLoader);
        }
        installHook(new IPackageManagerHook(context), classLoader);
        installHook(new IActivityManagerHook(context), classLoader);
        installHook(new PluginCallbackHook(context), classLoader);
        installHook(new InstrumentationHook(context), classLoader);
        installHook(new LibCoreHook(context), classLoader);

        installHook(new SQLiteDatabaseHook(context), classLoader);
    }
    
	 public void installHook(Hook hook, ClassLoader cl) {         
        hook.onInstall(cl);
        synchronized (mHookList) {
	        mHookList.add(hook);          
        }
		...
    }
我们举个粒子进去看下，我们挑选InstrumentationHook，因为这个好说，与我们熟悉的Activity调用有关。

	@Override
    protected void onInstall(ClassLoader classLoader) throws Throwable {

        Object target = ActivityThreadCompat.currentActivityThread();
        Class ActivityThreadClass = ActivityThreadCompat.activityThreadClass();

         /*替换ActivityThread.mInstrumentation，拦截组件调度消息*/
        Field mInstrumentationField = FieldUtils.getField(ActivityThreadClass, 
								        "mInstrumentation");
        
        Instrumentation mInstrumentation = (Instrumentation) FieldUtils.readField(
									        mInstrumentationField, target);
        
        if (!PluginInstrumentation.class.isInstance(mInstrumentation)) {
            PluginInstrumentation pit = new PluginInstrumentation(
							            mHostContext, mInstrumentation);
            pit.setEnable(isEnable());
            mPluginInstrumentations.add(pit);
            FieldUtils.writeField(mInstrumentationField, target, pit);
            
         } else {
            Log.i(TAG, "Instrumentation has installed,skip");
        }
    }
替换主线程`ActivityThread`的`mInstrumentation`变量为我们的`PluginInstrumentation`。
那么问题来了，为何要这样呢？

## Context - > StartActivity

之所以要去替换他，是因为当我们调用Context.StartActivity的时候，他背后的内容如下：

	@Override
    public void startActivity(Intent intent, Bundle options) {
		  ...
         mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity)null, intent, -1, options);
    }
                   
调用它的execStartActivity()函数去执行任务，所以通过拦截它，我们就可以算拦截了Activity启动的钥匙，但为何是这个还是另有原因。

>我们找Hook点，一般寻找**静态变量和单例**，**静态变量和单例**，**静态变量和单例**；

因为在**一个进程**之内，静态变量和单例的是相对不变的，因此容易做到hook一次，永久有效。就像把守关口，咽喉要道一样的道理，因此你应该也就知道为何选他了。

一个进程只有一个主线程(ActivityThread/UI线程)，而这个是它的变量，因此我们拿这个“单例”的变量来做HOOK点。就很自然而然了。这样下次当我们调用像下面这样的代码的时候：

	getBaseContext().startActivity(intent2);

我们就可以做一些处理。不过以上是调用context来启动的。

## Activity - >StartActivity

有时我们是直接调用Activity的启动，就是自己再Activity里面调用它，对这个情况需要我们另外处理，我们看整个启动流程

![enter image description here](http://pic002.cnblogs.com/images/2012/328668/2012040716440386.jpg)

（题外话： 这时候终于觉得看源码，写个记录文章[源码探索系列3---四大金刚之Activity的启动过程完全解析](http://sanjay-f.github.io/2015/12/15/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%973---%E5%9B%9B%E5%A4%A7%E9%87%91%E5%88%9A%E4%B9%8BActivity%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90/) 有点用了。帮助明白了DroidPlugin里面代码是什么鬼，不会觉得陌生。）


我们分析下`StartActivity`的调用链，这个AMN会是一个可以的点。
根据提到的那篇文章，我们知道他启动背后是调用AMN的getDefault的启动方法startActivity()

	int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);

是的，这里是个好地方。
getDefault作为一个静态方法，返回的是IActivityManager这个Binder。
这样我们回到DroidPlugin里面对AMS的处理

	installHook(new IActivityManagerHook(context), classLoader);

背后的是

	@Override
    public void onInstall(ClassLoader classLoader) throws Throwable {
    
        Class cls = ActivityManagerNativeCompat.Class();
        Object obj = FieldUtils.readStaticField(cls, "gDefault");
        
        if (obj == null) {
            ActivityManagerNativeCompat.getDefault();
            obj = FieldUtils.readStaticField(cls, "gDefault");
        }

        if (IActivityManagerCompat.isIActivityManager(obj)) {
            setOldObj(obj);
            Class<?> objClass = mOldObj.getClass();
            List<Class<?>> interfaces = Utils.getAllInterfaces(objClass);
            Class[] ifs = interfaces != null && interfaces.size() > 0 ? 
				            interfaces.toArray(new Class[interfaces.size()]) : 
				            new Class[0];
				            
            Object proxiedActivityManager = MyProxy.newProxyInstance(
								            objClass.getClassLoader(), ifs, this);
								            
            FieldUtils.writeStaticField(cls, "gDefault", proxiedActivityManager);
            
            Log.i(TAG, "Install ActivityManager Hook 1 old=%s,new=%s", mOldObj, 
		           proxiedActivityManager);	         
	       }
	        ...
    }
下面是用到的ActivityManagerNativeCompat的内容

	public class ActivityManagerNativeCompat {

	    private static Class sClass;
	
	    public static Class Class() throws ClassNotFoundException {
	        if (sClass == null) {
	            sClass = Class.forName("android.app.ActivityManagerNative");
	        }
	        return sClass;
	    }
	
	    public static Object getDefault() throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException {
	        return MethodUtils.invokeStaticMethod(Class(), "getDefault");
	    }
	}

这样也就明白了为何这段代码是这么写的了。


# 后记

动态代理在自己的开发过程还是很少用到，就算是静态代理。目前用得还是相对少，不过在这个项目过程，重复利用Java这个代理来给我们干活，替我们去拦截一些方法。
整个项目涉及到了很多FrameWork的内容，看完也算不错的一次学习机会！



 