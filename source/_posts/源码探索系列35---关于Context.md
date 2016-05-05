title: 源码探索系列35---关于Context
date: 2016-05-05 17:13:46
tags: [android,源码,context]
categories: android

------------------------------------------

对于Context，我们应该算挺“熟悉”的，很多时候都需要用到它。
那么这个context到底实际是什么，为何可以做那么多的功能，又是在哪里创建的呢?
`Context`、`Activity` 和 `Application`这三个又是怎么个关系，为何有些时候都可以替着用？
带着这些问题，我们来探索下，看下背后的实现是怎样的。


<!--more-->

# 起航 
api:23

我们需要说下，每个Activity是对应着一个Context对象的。这个关系在前面我们看Activity的启动流程就有看到对应的代码段，现在我们就从哪里开始看起把。

## performLaunchActivity（）
在启动Activity的流程最后面环节，我们会到ActivityThread里面这个函数去，创建Activity，并创建Context，绑定两者的关系。

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent){

 	  ...
 	  
 
      Context appContext = createBaseContextForActivity(r, activity);
       CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
       Configuration config = new Configuration(mCompatConfiguration);
       
       //attach
       activity.attach(appContext, this, getInstrumentation(), r.token,
               r.ident, app, r.intent, r.activityInfo, title, r.parent,
               r.embeddedID, r.lastNonConfigurationInstances, config,
               r.referrer, r.voiceInteractor);

       if (customIntent != null) {
           activity.mIntent = customIntent;
       }  
                    
	   ... 
    }

我们看下这个函数内容，主要就是创建context，然后attach到我们的activity去，从而建立对应关系
对此我们可以先来看下这个Context怎么创建的

###  createBaseContextForActivity

	private Context createBaseContextForActivity(ActivityClientRecord r, final Activity activity) {
	
	
        int displayId = Display.DEFAULT_DISPLAY;  
        displayId = ActivityManagerNative.getDefault().getActivityDisplayId(r.token); 

        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, displayId, r.overrideConfig);
        appContext.setOuterContext(activity);
        Context baseContext = appContext;

        //下面这大段是留下的“测试代码”，我们某天也可以利用这个来做点什么哈
        final DisplayManagerGlobal dm = DisplayManagerGlobal.getInstance();
        // For debugging purposes, if the activity's package name contains the value of
        // the "debug.use-second-display" system property as a substring, then show
        // its content on a secondary display if there is one.
        String pkgName = SystemProperties.get("debug.second-display.pkg");
        if (pkgName != null && !pkgName.isEmpty()
                && r.packageInfo.mPackageName.contains(pkgName)) {
            for (int id : dm.getDisplayIds()) {
                if (id != Display.DEFAULT_DISPLAY) {
                    Display display =dm.getCompatibleDisplay(id, 
	                            appContext.getDisplayAdjustments(id));
	                            
                    baseContext = appContext.createDisplayContext(display);
                    
                    break;
                }
            }
        }
        return baseContext;
    }
在创建的时候，根据ActivityClientRecord对象的token去AMS拿`displayId`，
然后传ActivityThread，一个LoadedApk类的packageInfo还有多一个配置信息Configuration类。对于这个LoadedApk我们在了解插件化的时候有接触过，我们会需要修改里面的classLoader等信息，这个类就类似于我们的apk本身。
 
构造一个后，然后他设置了OuterContext这个属性，将关联的Activity实例保存在它的内部，这样当有时候需要用到它所关联到的Activity组件的属性或者方法等的时候，以后就可以访问了。 
	 
 看完这个，我们继续`performLaunchActivity`后续步骤`activity.attach（）`
 
### attach()
 
	final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
  
		 //Set the base context for this ContextWrapper.All calls will then be delegated
		 //to the base context. Throws IllegalStateException if a base context has 
		 //already been set.
		 //就是调用父类的这个函数设置context
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
         //重要的一个变量
        mWindow = new PhoneWindow(this); 
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        //做过一些EditText的都知道这个属性把？在配置文件调整这个字段的。
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }        
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        
        mUiThread = Thread.currentThread();
        mMainThread = aThread; 
        ...

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
                
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
    
 通过这环节，基本就建立了对应关系了，把我们创建的一些contextImpl，applicationd，config等信息给存起来了。
 
####  mBase
需要稍微提下的是，这个attachBaseContext()最后是去设置mBase这个变量

	//在他的父类ContextThemeWrapper
	@Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
    }
    //ContextThemeWrapper父类ContextWrapper
	protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    
ContextThemeWrapper主要是维护一个应用程序窗口的**主题(Theme)**，而实际用来描述我们应用程序窗口的运行上下文环境的是`ContextImpl`对象，他就是那个在`ContextWrapper`的`attachBaseContext()`函数设置的这个`mBase`变量。至于这个`ContextWrapper`就是个代理类，只是简单地封装了对其mBase的操作。后面我们在Activity用的很多时候都是调用这个去给我们干活的!具体有哪些可以去看下 ContextWrapper的内容，我们在分析service，广播等时候已经看过不少了，就不在这里展开。
 

#### 事件分发

然后是关于`mWindow.setCallback(this);`这句话，背后还是有点故事的哈。
平时我们聊事件分发流程，都会说从Activity一级一级下沉到View去，那么这个事件一开始是谁给Activity的呢？
答案就是这个window的`setCallback()`函数，我们的Activity是实现了Window.Callback接口的，这接口有十多个回调，其中包括我们比较熟悉的`dispatchKeyEvent`，`dispatchTouchEvent`，`onMenuItemSelected`等函数。

#### Window

这Activity里有一个很有趣的一个类，就是这个PhoneWindow，在很久以前看setContentView背后的代码过程就有接触过，关于他曾经有一个问题，就是Activity和window是个什么数量关系，一个Activity对一个Window吗？还是怎样呢？
另外我们以前设置全屏的时候，也用到下面这行代码：

	this.getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, 
							WindowManager.LayoutParams.FLAG_FULLSCREEN); 

这个getWidnow获取的变量就是这个mWindow变量。
  

 
---

  
 

 
# 比较

关于对比使用上的区别,有一张流传的图

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/35_b1cf36a5-d134-390c-b7a3-4212369a2fcf.png)


## Activity和context 是什么关系呢？

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/35_context_20160505135516.png)

一图说明所有内容，`Context`是一个抽线接口，`Activity`继承`ContextThemeWrapper`,后者又继承`ContextWrapper`,这样其实`Activity`是`Context`的子类来的！所以很多时候可以拿`Activity`来替代`Context`用。
 
## Application 与Context关系

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/35_QQ%E6%88%AA%E5%9B%BE20160505135931.png)

`Application`也是`ContextWrapper`子类。 

----

 

因此，`Activity`和`Application`都是`Context`的子类。这点我们确认无疑。

1. 生命周期不一样：是的，一句废话，它们的生命周期不一样，Activity短些，Application长久。存在的时间长就，会带来了一些不一样的内容。例如有时候会带来泄漏问题，应为application的这个Context活得久一些，利用他创建的东西一般也是"长寿"些的，所以需要小心使用，避免一些泄漏问题！

2. 偏向性不同：Actvity偏其对应的Context也只能访问该activity内的各种资源，后者偏向于Application的生命周期方面内容。你去查看这个application类，里面的代码也不多，大部分代码是关于ActivityLifecycleCallbacks的内容。



# 后记

到这里，整个的上下文环境的创建过程就结束啦！整个还是相对比较简单的流程的。
至于context的实现ContextImpl，我们就不细入介绍了，两千多行，而且功能都还挺分割的。
