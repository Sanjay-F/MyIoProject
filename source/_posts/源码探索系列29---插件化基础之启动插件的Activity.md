title: 源码探索系列29---插件化基础之启动插件的Activity
date: 2016-04-01 17:41:46
tags: [android,源码,DroidPlugin,Activity]
categories: android

------------------------------------------

昨天的那篇简答的描述了如何利用HOOK来劫持Intent事件，从何来控制关于Activity的开启，
在这个基础上，我们需要来看下怎么开启我们的插件中的Activity

<!--more-->

# 起航

目前DroidPlugin采用的是预先注册占坑，预先注册权限的方式。
利用假的Activity来做“运行”真的PluginActivity。

为何要这样做呢？因为我们在看整个Activity的启动源码就有看到，对于没在AndroidManifest.XML文件注册的Activity，人家是会抛下面这个错误：

	case ActivityManager.START_CLASS_NOT_FOUND:
        if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
            throw new ActivityNotFoundException(
            "Unable to find explicit activity class "
            +((Intent)intent).getComponent().toShortString()+ "; 
            +have you declared this activity in your AndroidManifest.xml?");

我相信刚开始开发的或多或少有遇到过这个bug的了.
因此，作者对这个占坑问题也说了会预注册一堆各种Launch Mode的Activity.然后是Hook了startActivity和handleLaunchActivity		     

---
那么，现在我们应该怎么处理这件事呢？从而让我们的插件在不做大的改动下，也可以正常运行呢？
这先让我们来回顾下大概的Activity启动流程：

![图盗自老罗哥的博客](http://hi.csdn.net/attachment/201108/14/0_1313305334OkCc.gif)

当我们调用Actvity的时候，会经过一大段的流程，最后再回到我们的ActivityThread里面的Handle去，在里面启动我们想要的Activity！详情可以看我写的这篇，[源码探索系列3---四大金刚之Activity的启动过程完全解析](http://sanjay-f.github.io/2015/12/15/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%973---%E5%9B%9B%E5%A4%A7%E9%87%91%E5%88%9A%E4%B9%8BActivity%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%AE%8C%E5%85%A8%E8%A7%A3%E6%9E%90/)具体就不贴出来，我们挑着说。

在被缩略的整个一大段流程中重要的内容是处于系统进程(system_server)的
AMS会对我们想启动的Activity做个检测（例如上面提到的bug），看是否在我们的AndroidManifes.xml里面有注册了，这样某种程度避免我们程序受到注入攻击。不过由于系统的**进程隔离**的存在，我们没办法HOOK掉这个系统进程，但这让我们挑选HOOK点显得好找。去掉中间，那就是两头咯！


## 思路：

  因此我们就这样的思路出来了：

	当我们要启动一个插件Activity的时候，把这个intent修改下，传一个真的注册了的Activity信息过去给AMS做检测等工作，等处理完回调到我们的ActivityThread去调用启动Activity的时候，我们再Hook，去启动我们最终想启动的！
 

## demo

有了这样的思路，我们来测试下效果如果

	

我们创建两个Activity，一个叫RealActivity，另外一个叫StubActvity
然后我们我们在AndroidManifes.xml里面只保留StubActvity的注册信息。

	<application 
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name" 
        android:theme="@style/AppTheme">
        
        <activity android:name=".ui.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>         

        <activity android:name=".ui.StubActivity">  //<-只有Stub没有realActivity
        </activity> 
    </application>

准备好这些后，我们在MainActivity里面发 startActivity到StubActivity

	public class MainActivity extends Activity { 
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);	
	        //现在我们要去我们的插件Activity的，类似于这个RealActivity
	        startActivity(new Intent(MainActivity.this, RealActivity.class));
	    } 
	}


这些准备好后，我们来看下怎么HOOK掉两头的内容，让发到AMS的是真的，但到了另外一头Handler里面就是启动plugin的activty。

假认你看了上一篇[源码探索系列28---插件化基础之代理Hook](http://sanjay-f.github.io/2016/03/28/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9728---%E6%8F%92%E4%BB%B6%E5%8C%96%E5%9F%BA%E7%A1%80%E4%B9%8B%E4%BB%A3%E7%90%86Hook/)，知道如何写HOOK这个startActivity的方法。

那么我们就只重点的看下invoke里面的内容，后续再贴整个项目的参考代码地址

    public class IActivityManagerHook implements InvocationHandler {
    
	    private static final String TAG = IActivityManagerHook.class.getSimpleName();
	    private Object originalObject;
	    public static final String EXTRA_ORIGINAL_INTENT = "EXTRA_INTENT_data_2";

	    public IActivityManagerHook(Object originalObject) {
	        this.originalObject = originalObject;
	    }
	
	    @Override
	    public Object invoke(Object proxy, Method method, Object[] args) 
	     throws Throwable {
	
	        Log.e(TAG, "invoke() called with: " + "proxy = [" + " " + method.getName());
	        	
	        if (method.getName().equals("startActivity")) {//我们的starActivity方法
	            for (int i = 0; i < args.length; i++) {
	                if (args[i] instanceof Intent) {//找到传过来的intent了！
	
	                    Intent ourFakeIntent = new Intent();
	                    ComponentName componentName = new ComponentName(
						                    "com.example.administrator.testdemo", 
						                    StubActivity.class.getName());
	                    ourFakeIntent.setComponent(componentName);
	                    ourFakeIntent.putExtra(EXTRA_ORIGINAL_INTENT, 
							                    (Intent) args[i]);//存真的intent	
	                    args[i] = ourFakeIntent;//替换过去给AMS检测	
	                    break;
	                }
	            }
	            
	            return method.invoke(originalObject, args);//  <- 送检
	            
	        } else {
	            return method.invoke(originalObject, args);
	        }
	    }
	}

根据这部分内容，我们之后需要去hook掉回到我们ActivityThread部分的内容！
依据我们前面在看Activity的启动过程的时候，我们知道，最终他会回到ActivityThread的一个名叫mH的变量的H类（H继承Handle.不知为何要搞这么短的名字，记得大学的时候看到一个日本人写的关于C语言的极简编程的书，大量的利用编译器的特性，这家伙不会也是个日本人吧！还是说受到他的影响呢？），接着他会收到一条`LAUNCH_ACTIVITY`的消息.

	private class H extends Handler {
        public static final int LAUNCH_ACTIVITY         = 100;
        ...
        
		public void handleMessage(Message msg) {	     
	         switch (msg.what) {
               case LAUNCH_ACTIVITY: {
                   Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, 
							                    "activityStart");
                   final ActivityClientRecord r = (ActivityClientRecord) msg.obj;	
                   r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo,					                    	                    	                             
					                    	             r.compatInfo);
                   handleLaunchActivity(r, null);//启动Activity
                   Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
               } break;
		}
	}
因此我们可以Hook掉的点就来了。在去调用`handleLaunchActivity()`前，我们得替换回我们实际的Activty去！这样就可以达到借尸还魂了。那么得怎么做呢？
好在我们以前有看过Handle的源码，知道这个handleMessage是一个之类需要实现的方法，

	/**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) { 
    
	 }    
	 
嗯，我记得看[DexClassLoader](http://sanjay-f.github.io/2016/03/16/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9727---%E6%8F%92%E4%BB%B6%E5%8C%96%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8DexClassLoader/)里面的有一个片段代码，就是下面这个

	protected Class<?> findClass(String className) throws ClassNotFoundException {
	   throw new ClassNotFoundException(className);			  
	   
		//我们的子类必须实现他！所以我们就去看下我们的BaseDexClassLoder
		//不要问我为何他不设置抽象,尽管这个类就是抽象类!
	}
			
在里面也有一个需要之类重写的方法，不然就直接抛异常的。
显然这不是同一个人呢写的。风格不一样。

在他前面还有别人调用它，类似下面这样

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

我们看下这个Handle类的这段代码，如果这个消息自带了callback的话，就去调用它自带的，要不然看有没设置mCallback这个属性，没有才轮到我们重写的 handleMessage()部分!!而ActivityThread的H先生是没设置这个的，我也是看源码才发现有这个属性设置的！所以，我们可以在这里做点文章，这是我们去调用我们的真实Activity的地方！

	public static void hookHandler() throws Exception {
	
	    // 先获取到当前的ActivityThread对象
        Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
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
        
    }
好了，有这样的铺垫，我们来看下那个`ActivityThreadHandlerCallback`的内容

	class ActivityThreadHandlerCallback implements Handler.Callback {

	    private static final String TAG = ActivityThreadHandlerCallback.
										   class.getSimpleName();
	    public static final int LAUNCH_ACTIVITY = 100;	
	    Handler mOriginalHandle;	
	    public ActivityThreadHandlerCallback(Handler base) {
	        mOriginalHandle = base;
	    }	
	    
	    @Override
	    public boolean handleMessage(Message msg) {
	     
	        switch (msg.what) {
	            case LAUNCH_ACTIVITY: {
	                Object obj = msg.obj;
	                try {
	                    Field intent = obj.getClass().getDeclaredField("intent");
	                    intent.setAccessible(true);
	                    Intent raw = (Intent) intent.get(obj);
	                    Intent target = raw.getParcelableExtra(
				                    IActivityManagerHook.EXTRA_ORIGINAL_INTENT);
	                    ComponentName component = new ComponentName(
							                    target.getComponent().getPackageName(), 
							                    target.getComponent().getClassName());
							                    
	                    //替换我们原本送去AMS的stubActivity为真的RealActivity的 intent!
	                    raw.setComponent(component);
	                    
	                } catch (Exception e) {
	                    e.printStackTrace();
	                }
	            }
	            break;
	        }
	
			//替换好后，再用原来的ActivityThread的mH去处理
			//到此大功告成!
	        mOriginalHandle.handleMessage(msg);
	        return true;
	    }
	
	}


## 之后的生命和周期呢？

我们前面已经成功做到了启动一个没有在AndroidManifest中注册的Activity，那么接下来对这个Activity的别的生命周期的调用还有效吗？
	在看Activity的启动过程我们知道，。AMS与ActivityThread之间对于Activity的生命周期的交互，是使用一个**Token**来标识，而不是直接使用Activity对象来进行操作管理的！而这个token是**Binder**对象，因此可以跨进程传递。Activity里面有一个成员变量`mToken`代表的就是它，token唯一地标识一个Activity对象，它在Activity的attach方法里面被初始化；
	我们传给AMS的是我们的假Activity，当在AMS处理Activity的任务栈的时候，使用这个token标记我们传过去的Activity，因此实际上AMS进程里面的token对应的是StubActivity。但到了我们的进程里面时候，我们在他去调用哪
	个`handle`的`handleMessage`前，掉包了那个Activity，换成为了我们的
	`RealActivity`，因此token对应的却是RealActivity！
	所以ActivityThread执行回调的时候，能正确地回调到RealActivity相应的方法。因此我们可以确认，通过这种方式启动的Activity有它自己完整生命周期！和真的没什么区别！
	这样的设计和我们的服务器的token一样，只要偷到这个，我们就可以做`中间人攻击`，从而让两端被骗的人以为没事，一切正常进行。

### AppCompatActivity与Activity


	 Caused by: android.content.pm.PackageManager$NameNotFoundException:	  
	 ComponentInfo{com.example.administrator.testdemo.ui/
	 com.example.administrator.testdemo.ui.RealActivity}	 
	 atandroid.app.ApplicationPackageManager.getActivityInfo(ApplicationPackageManager.
	 java:342)
	 at android.support.v4.app.NavUtils.getParentActivityName(NavUtils.java:301) 
     at android.support.v4.app.NavUtils.getParentActivityName(NavUtils.java:281)
	 atandroid.support.v7.app.AppCompatDelegateImplV7.onCreate(AppCompatDelegateImplV7.
	java:152) 
	at android.support.v7.app.AppCompatActivity.onCreate(AppCompatActivity.java:60) 
	at com.example.administrator.testdemo.ui.RealActivity.onCreate(RealActivity.java:12) 

写这篇文章的时候，用AS建的模板程序，写完一直是这个bug！看了好久才发现我继承的是
`AppCompatActivity`而不是`Activity`！！！！差点哭晕在厕所，这么低级的错误。

# 后记

在明白了整个插件的Activity启动的原理流程后，还有些细节需要我们去了解的，例如去获取资源时候涉及到用的R部分的内容，有些是编号规则来的，其中也涉及到AssetsManager等内容！再详细看吧

另外我们都知道Activity是有四种启动模式的，我们都需要占坑，因此会在AndroidManifest写下面的内容 

	<activity
            android:name=".StandardActivity"
            android:label="@string/title_activity_standard" />
            
	<activity
            android:name=".SingleTopActivity"
            android:label="@string/title_activity_single_top"
            android:launchMode="singleTop" />

	<activity
            android:name=".SingleTaskActivity"
            android:label="@string/title_activity_single_task"
            android:launchMode="singleTask" />
	 <activity
            android:name=".SingleTaskActivity$SingleTaskActivity1"
            android:label="@string/title_activity_single_task1"
            android:launchMode="singleTask" />
            
      <activity
            android:name=".SingleInstanceActivity"
            android:label="@string/title_activity_single_instance"
            android:launchMode="singleInstance" />
     <activity
            android:name=".SingleInstanceActivity$SingleInstanceActivity1"
            android:label="@string/title_activity_single_instance1"
            android:launchMode="singleInstance" />

为何有这些.SingleInstanceActivity$SingleInstanceActivity1这样的名字的类，应该和混淆有关。

另外不可避免的是在插件会有很多的Activity被新启动，这需要我们根据业务需要来看下最大的数目，从而确定占坑的数目。

另外，这里的那个RealActivity是在同一个包内的，所以我们前面可以用那样的方式去直接写是没问题的。但显然的，插件时候是不会再在同一个包内的，这时候需要一个类似Java的ClassLoader加载类的工具，在Android上就是DexClassLoader啦。这个在上一篇 [源码探索系列27---插件化基础之类加载器DexClassLoader](http://sanjay-f.github.io/2016/03/16/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9727---%E6%8F%92%E4%BB%B6%E5%8C%96%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8DexClassLoader/) 有做过简单的介绍。下次再看怎么修改这部分内容。


 