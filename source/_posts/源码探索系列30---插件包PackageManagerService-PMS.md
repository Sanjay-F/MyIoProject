title: 源码探索系列30---插件包PackageManagerService/PMS
date: 2016-04-14 19:53:46
tags: [android,源码,DroidPlugin,PMS]
categories: android

------------------------------------------

**PMS**主要管理了手机上的所有APK,可提供一些例如安装与卸载，查询包的信息（如我们在AndroidManifest.xml写的内容），清数据等功能。

做插件化，想欺骗系统不让他们知道我们悄悄安装多了个APP，对PMS做点手脚当然不可避免的啊，今天就让我们来看下他在插件化过程担当的角色和作用吧！

<!--more-->

# 起航

API:23

在这前，我们看下大致关系，当调用Context.getPackageManager()时,背后代码如下：

	@Override
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
            // Doesn't matter if we make more than one instance. why?
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }

        return null;
    }
    
基于此，我们知道抽象类PM的实现是APM，而这个APM有个Binder的对象，IpackageManger，这个就是PMS啦，我们的安装等都是靠它PMS去搞定的。大致如下：

![enter image description here](http://img.blog.csdn.net/20131022154703437?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmV3X2FiYw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

基于上面这个关系，我们感觉是看到了一个不错的HOOK点，因为这个我们前面的AMS就是HOOK掉了ActivityThread的，这里应该类似，我们去看下

	public final class ActivityThread {
		static IPackageManager sPackageManager;
		...
		public static IPackageManager getPackageManager() {
	        if (sPackageManager != null) {	             
	            return sPackageManager;
	        }
	        IBinder b = ServiceManager.getService("package");	     
	        sPackageManager = IPackageManager.Stub.asInterface(b);

	        return sPackageManager;
	    }
	 }
是啊，看到曙光了，静态的sPackageManager变量。这样我们可以确定这是一个可以的Hook点了。
因此我们可以确定
1. 除了要HOOK掉ActivtyThread的sPackageManger外
2. 还需要的是替换下构造APM时候传进去的参数对应的变量mPM.

	ApplicationPackageManager(ContextImpl context,
                              IPackageManager pm) {
        mContext = context;
        mPM = pm;
    }

这样我们就可以来看下DroidPlugin里面的IpackageMangerHook的内容了，在他的onInstall里面


	 @Override
    protected void onInstall(ClassLoader classLoader) throws Throwable {
        Object currentActivityThread = ActivityThreadCompat.currentActivityThread();
        setOldObj(FieldUtils.readField(currentActivityThread, "sPackageManager"));
        Class<?> iPmClass = mOldObj.getClass();
        List<Class<?>> interfaces = Utils.getAllInterfaces(iPmClass);
        Class[] ifs = interfaces != null && interfaces.size() > 0 ? interfaces.toArray(new Class[interfaces.size()]) : new Class[0];
        Object newPm = MyProxy.newProxyInstance(iPmClass.getClassLoader(), ifs, this);
        FieldUtils.writeField(currentActivityThread, "sPackageManager", newPm);
        //1. hook了ActivityThread里面的变量   

        PackageManager pm = mHostContext.getPackageManager();
        Object mPM = FieldUtils.readField(pm, "mPM");
        if (mPM != newPm) {
            FieldUtils.writeField(pm, "mPM", newPm);
        }
        // 2. hook掉了APM里面的mPm变量
    }

# 前进

根据前面的介绍，我们基本完成了HOOK的工作。做好了一些准备工作啦！
在这样的基础上，我们可以开始尝试做安装APK插件的事情。让我们来看下`DroidPlugin`的内容
看他是如何做插件安装部分的内容的,截取自`IPluginManagerImpl` 的内容

	 @Override
    public int installPackage(String filepath, int flags) {
        //install plugin
        String apkfile = null;
        　
       　PackageManager pm = mContext.getPackageManager();
        PackageInfo info = pm.getPackageArchiveInfo(filepath, 0);
        if (info == null) {
                return PackageManagerCompat.INSTALL_FAILED_INVALID_APK;
        } 
        apkfile = PluginDirHelper.getPluginApkFile(mContext, info.packageName);

        ...
        new File(apkfile).delete();
        Utils.copyFile(filepath, apkfile);
        //在确保这个包是一个有效的APK后，用自己的PackageeParser去做解析
        PluginPackageParser parser = new PluginPackageParser(mContext, new 
							            File(apkfile)); 					            
        parser.collectCertificates(0);
        PackageInfo pkgInfo = parser.getPackageInfo(
		           PackageManager.GET_PERMISSIONS | 
				　　PackageManager.GET_SIGNATURES);
				　　
	    if (pkgInfo != null && pkgInfo.requestedPermissions != null && 　　　　         
		　　　　　            pkgInfo.requestedPermissions.length > 0) {		　　　　　            

	        for (String requestedPermission : pkgInfo.requestedPermissions) {
	            boolean b = false;　
	            b = pm.getPermissionInfo(requestedPermission, 0) != null;	                    
	            if (!mHostRequestedPermission.contains(requestedPermission)&&b) {
	               //为了插件化，会先在HOST里先申请一堆的权限，插件的权限需要看下是否在HOST有，
	               //不支持的就say goodbye咯。
	                new File(apkfile).delete();
	                return PluginManager.INSTALL_FAILED_NO_REQUESTEDPERMISSION;
                }
            }
        }
	            
       saveSignatures(pkgInfo); 
     　copyNativeLibs(mContext, apkfile, parser.getApplicationInfo(0));
       dexOpt(mContext, apkfile, parser);

       //缓存信息，后面各种判断是否为插件内容时候都要用到的。
       mPluginCache.put(parser.getPackageName(), parser);       
       //这函数什么也没做，不知道为何
       mActivityManagerService.onPkgInstalled(mPluginCache, parser, 
						            parser.getPackageName()); 						            
       //发送广播通知插件添加成功						            
       sendInstalledBroadcast(info.packageName);
       return PackageManagerCompat.INSTALL_SUCCEEDED;	
	   ...
	}


因为安装的内容最终很大部分是由我们自己来管理的，因此我们需要存储一些插件APK的信息，然后在用到的时候我们做一些额外的工作。例如在获取`PackageUid`，`getPackageInfo`的时候，如果传过来的`packageName`是我们插件包名，那么我们需要偷换成Host的包名。


在这个安装过程很依赖这个`PluginPackageParser` 类去解析插件APK，这个类非常值得一提，为了支持这个解析工作，作者手动适配了不同的版本，包括：15/16/20/21/22/22Preview1。
我们来简单的看下

	 public PluginPackageParser(Context hostContext, File pluginFile) throws Exception {
	 
        mHostContext = hostContext;
        mPluginFile = pluginFile;

		//解析工作承担者
        mParser = PackageParser.newPluginParser(hostContext);//<--主角
        mParser.parsePackage(pluginFile, 0);
        mPackageName = mParser.getPackageName();
        mHostPackageInfo = mHostContext.getPackageManager().
			        getPackageInfo(mHostContext.getPackageName(), 0);

        List datas = mParser.getActivities();
        for (Object data : datas) {
            ComponentName componentName = new ComponentName(mPackageName, 
							            mParser.readNameFromComponent(data));
            synchronized (mActivityObjCache) {
                mActivityObjCache.put(componentName, data);
            }
            synchronized (mActivityInfoCache) {
                ActivityInfo value = mParser.generateActivityInfo(data, 0);
                fixApplicationInfo(value.applicationInfo);
                if (TextUtils.isEmpty(value.processName)) {
                    value.processName = value.packageName;
                }
                mActivityInfoCache.put(componentName, value);
            }

            List<IntentFilter> filters = mParser.readIntentFilterFromComponent(data);
            synchronized (mActivityIntentFilterCache) {
                mActivityIntentFilterCache.remove(componentName);
                mActivityIntentFilterCache.put(componentName, new 
				                ArrayList<IntentFilter>(filters));
            }
        }

        datas = mParser.getServices();
         
          ...
          ...各种mParser.getXXX()的处理工作;

        List<String> requestedPermissions = mParser.getRequestedPermissions();
        if (requestedPermissions != null && requestedPermissions.size() > 0) {
            synchronized (mRequestedPermissionsCache) {
                mRequestedPermissionsCache.addAll(requestedPermissions);
            }
        }
    }


好了，我们再来看下这个解析的构造函数内容

	abstract class PackageParser {
	
	    protected Context mContext;	
	    protected Object mPackageParser;
	
	    PackageParser(Context context) {
	        mContext = context;
	    } 
	
	    public static PackageParser newPluginParser(Context context) throws Exception {
	        if (VERSION.SDK_INT >= VERSION_CODES.LOLLIPOP_MR1) {
	            if ("1".equals(SystemPropertiesCompat.get(
		            "ro.build.version.preview_sdk", ""))) {
		            	            
	                return new PackageParserApi22Preview1(context);
	            } else {
	                return new PackageParserApi22(context);//API 20
	            }
	        } else if (VERSION.SDK_INT >= VERSION_CODES.LOLLIPOP) {
	        
	            return new PackageParserApi21(context);//API 21
	        } else if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN_MR1 && 
		        VERSION.SDK_INT<= VERSION_CODES.KITKAT_WATCH) {
		        
	            return new PackageParserApi20(context);//API 17,18,19,20
	        } else if (VERSION.SDK_INT == VERSION_CODES.JELLY_BEAN) {
	            return new PackageParserApi16(context); //API 16
	        } else if (VERSION.SDK_INT >= VERSION_CODES.ICE_CREAM_SANDWICH && 	        
		        VERSION.SDK_INT <= VERSION_CODES.ICE_CREAM_SANDWICH_MR1) {
		        
	            return new PackageParserApi15(context); //API 14,15
	        } else if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB &&
			         VERSION.SDK_INT <= VERSION_CODES.HONEYCOMB_MR2) {
			         
	            return new PackageParserApi15(context); //API 11,12,13
	        } else if (VERSION.SDK_INT >= VERSION_CODES.GINGERBREAD && 
				       VERSION.SDK_INT <= VERSION_CODES.GINGERBREAD_MR1) {
				        
	            return new PackageParserApi15(context); //API 9，10
	            
	        } else {
	            return new PackageParserApi15(context); //API 9，10
	        }
	    }
	}
我感受到了满满的安卓坑，作者在写的时候估计调试了不少
**在这里推荐一个网站**：[http://grepcode.com/](http://grepcode.com/)，他可以在线查看代码，直接比较各版本的差异。我想这比自己下载到本地，然后用各类`compare`软件来比较便捷多了。想想下载那么多个版本光空间就足够挤爆你的SSD了！

# 额外的解释:

为何要这样做？

我们看下我们的整个Activity过程，启动过程前需要弄一个真的host的Activity去检测，然后HOOK在**h**变量的**callback**。最后处理LAUNCH_ACTIVITY消息，加载我们的`targetActivity`。

1. 我们得判断这个要启动的Activity是否为插件的
2. 在检测完，回Handle时也判断一下
3. 加载插件Activty。如果是插件的Activty，我们的启动需要一些额外的信息。具体如下（在ActivityThread的performLaunchActivity函数里面）：
		
		java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass()); 
        r.intent.setExtrasClassLoader(cl);
             
            
在上篇文章介绍Activity启动过程的时候我们的TragetActivity是处在同一个包下的，这是有意回避一些问题的，因为我们默认拿到的ClassLoader是没法加载插件的内容的。我们需要造一个自己的ClassLoader出来。
这样，根据这个启动函数，我们需要一个特定的ClassLoader，用来加载我们插件的类。

这样我们来看下DroidPlug的源码，在`PluginCallback`的里面，这个就是处理`h`的回调消息的。

	private boolean handleLaunchActivity(Message msg) {
	
		 Object obj = msg.obj;
	     Intent stubIntent = (Intent) FieldUtils.readField(obj, "intent"); 
	     stubIntent.setExtrasClassLoader(mHostContext.getClassLoader());
	     Intent targetIntent = stubIntent.getParcelableExtra(
		            Env.EXTRA_TARGET_INTENT);
	            
	     // 这里多加一个isNotShortcutProxyActivity的判断，因为ShortcutProxyActivity的很特殊，
	     //启动它的时候，也会带上一个EXTRA_TARGET_INTENT的数据，就会导致这里误以为是启动插件
	     //Activity，所以这里要先做一个判断。之前ShortcutProxyActivity错误复用了key，但是为了兼
	     //容，所以这里就先这么判断吧。
	     
	     /** 上面这段是作者原话，没删掉是想说，要做好一个项目，确实不容易**/
	     
	     if (targetIntent != null && !isShortcutProxyActivity(stubIntent)) {
	     
	 
	         ComponentName targetComponentName = targetIntent.resolveActivity(
								                mHostContext.getPackageManager());
		                
	         ActivityInfo targetActivityInfo = PluginManager.getInstance().
								               getActivityInfo(targetComponentName, 0);
	               
	         if (targetActivityInfo != null) {
	             if (targetComponentName != null && 
		           targetComponentName.getClassName().startsWith(".")) {
	           
	                 targetIntent.setClassName(targetComponentName.getPackageName(), 
	                 targetComponentName.getPackageName() + 
	                 targetComponentName.getClassName());
	             }
	
	             ResolveInfo resolveInfo = mHostContext.getPackageManager().
	                    resolveActivity(stubIntent, 0);
	             ActivityInfo stubActivityInfo = resolveInfo != null ? 
	                    resolveInfo.activityInfo : null;
	             if (stubActivityInfo != null) {
	                 PluginManager.getInstance().reportMyProcessName(
	                 stubActivityInfo.processName, targetActivityInfo.processName, 
	                 targetActivityInfo.packageName);
	             }
			      
			      //这句话背后会去生成一个classLoader。      
	             PluginProcessManager.preLoadApk(mHostContext, targetActivityInfo);
	             
	             //获取刚才生成的classLoader
	             ClassLoader pluginClassLoader =PluginProcessManager.
							getPluginClassLoader(targetComponentName.getPackageName());
	
	             setIntentClassLoader(targetIntent, pluginClassLoader);
	             setIntentClassLoader(stubIntent, pluginClassLoader);
	
	             boolean success = false;
	  
	            targetIntent.putExtra(Env.EXTRA_TARGET_INFO,targetActivityInfo);
	            if (stubActivityInfo != null) {
	                targetIntent.putExtra(Env.EXTRA_STUB_INFO, stubActivityInfo);
	            }
	            success = true;
	        }
		         ...
		}
		
		  //最后调用工作
	      if (mCallback != null) {
	           return mCallback.handleMessage(msg);
	      } else {
	           return false;
	      }
       
    }


我们看下preLoad

	 public static void preLoadApk(Context hostContext, ComponentInfo pluginInfo) {
 
        /*添加插件的LoadedApk对象到ActivityThread.mPackages*/
        boolean found = false;
        
        synchronized (sPluginLoadedApkCache) {
            Object object = ActivityThreadCompat.currentActivityThread();
            if (object != null) {
            
                Object mPackagesObj = FieldUtils.readField(object, "mPackages");
                Object containsKeyObj = MethodUtils.invokeMethod(mPackagesObj, 
						                "containsKey", pluginInfo.packageName);

                //判断是否有存在pluginInfo.packageName这个字段了，没有就重新构造，有就跳过
                if (containsKeyObj instanceof Boolean && !(Boolean) containsKeyObj) {
 
                    final Object loadedApk;
                    
					// 又有一个版本不一样的地方，其实，如果你不做这件事情，
					// 我觉得很难掌握完各个SDK版本的差异.做了也难以记得全，都是需要的时候查啊.					
                    if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB) {
                        loadedApk = MethodUtils.invokeMethod(object, 
			                  "getPackageInfoNoCheck", pluginInfo.applicationInfo,						                           
		                     CompatibilityInfoCompat.DEFAULT_COMPATIBILITY_INFO());						                           				                        
                    } else {
                        loadedApk = MethodUtils.invokeMethod(object, 
			                       "getPackageInfoNoCheck", pluginInfo.applicationInfo);
                    }
                    
                    sPluginLoadedApkCache.put(pluginInfo.packageName, loadedApk);

	                /*添加ClassLoader LoadedApk.mClassLoader*/
                    String optimizedDirectory = PluginDirHelper.getPluginDalvikCacheDir(
						                    hostContext, pluginInfo.packageName);
                    String libraryPath = PluginDirHelper.getPluginNativeLibraryDir(
					                    hostContext, pluginInfo.packageName);
					                    
                    String apk = pluginInfo.applicationInfo.publicSourceDir;
                    
                    if (TextUtils.isEmpty(apk)) {
                        pluginInfo.applicationInfo.publicSourceDir = 
		                        PluginDirHelper.getPluginApkFile(hostContext, 
		                        pluginInfo.packageName);
		                        
                        apk = pluginInfo.applicationInfo.publicSourceDir;
                    }
                    
                    if (apk != null) {
                        ClassLoader classloader = null;
                        classloader = new PluginClassLoader(apk, optimizedDirectory, 
		                             libraryPath, ClassLoader.getSystemClassLoader());		                              
                        
                        if(classloader==null){
                            PluginDirHelper.cleanOptimizedDirectory(optimizedDirectory);
                            classloader = new PluginClassLoader(apk, optimizedDirectory, 
                                      libraryPath, ClassLoader.getSystemClassLoader());
                        }
                        
                        synchronized (loadedApk) {
                            FieldUtils.writeDeclaredField(loadedApk, "mClassLoader", 
										                            classloader);
                        }
                        sPluginClassLoaderCache.put(pluginInfo.packageName, 
							                        classloader);
							                        
							                        
                        Thread.currentThread().setContextClassLoader(classloader);
                        found = true;
                    }
                    ProcessCompat.setArgV0(pluginInfo.processName);
                }
            }
        }
        
        if (found) {
            PluginProcessManager.preMakeApplication(hostContext, pluginInfo);
        }
    }

上面的内容可能看起来让你觉得奇怪，取mPackages和构造loadedApk是什么意思？
我们重新看下我们的启动Activity的那几句话

	java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
	activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
	StrictMode.incrementExpectedActivityCount(activity.getClass());
	r.intent.setExtrasClassLoader(cl);

系统通过`r.packageInfo`获得`cl`的。而这个`r.packageInfo()`返回的是一个`LoadedApk`对象。而这个`LoadedApk`对象就是APK文件在内存中的表示！如各种库JNI的，Res和DataDir都通过此对象获取的,当然包括classLoader
因此有比较对他做些操作。而这个packageInfo是在H的handleMessage赋值的

	
	case LAUNCH_ACTIVITY: {
 
      final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
      r.packageInfo = getPackageInfoNoCheck(
              r.activityInfo.applicationInfo, r.compatInfo);
      handleLaunchActivity(r, null);
  
	} break;
（其实我挺好奇，为何这个case有一对`{ }`，而别的case都没有。）
因此你看到上面对的`preLoadAPK`函数对`getPackageInfoNoCheck`做了处理，我们继续看下函数背后的内容

	public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
            CompatibilityInfo compatInfo) {
        return getPackageInfo(ai, compatInfo, null, false, true, false);
    }
  
    private LoadedApk getPackageInfo(ApplicationInfo aInfo,CompatibilityInfo compatInfo,
            ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
            boolean registerPackage) {
            
        final boolean differentUser = (UserHandle.myUserId() != 
								       UserHandle.getUserId(aInfo.uid));
								       
								       
        synchronized (mResourcesManager) {
            WeakReference<LoadedApk> ref;
            //检测是否为同个user的，显然我们是同个            
            if (differentUser) {
                // Caching not supported across users
                ref = null;
            } else if (includeCode) {//调用函数传过来的为true
                ref = mPackages.get(aInfo.packageName);
            } else {
                ref = mResourcePackages.get(aInfo.packageName);
            }
             //这样我们通过mPackages得到的缓存内容获取引用
            LoadedApk packageInfo = ref != null ? ref.get() : null;
            
            //如果没有，显然下面的内容估计就是新建一个，然后放到mPackages里去
            if (packageInfo == null || (packageInfo.mResources != null
                    && !packageInfo.mResources.getAssets().isUpToDate())) {
                    
                 packageInfo =new LoadedApk(this, aInfo, compatInfo, baseLoader,                    
                            securityViolation, includeCode &&
                            (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, 
                            registerPackage);
				....
				
                if (differentUser) {
                    // Caching not supported across users
                } else if (includeCode) {
	                //缓存新建的!如我们预测的那样
                    mPackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo)); 
                            
                } else {
                    mResourcePackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo));
                }
                
            }
            return packageInfo;
        }
    }
看到这部分内容，想必对于为何preLoadAPK的内容也明白个大概了吧？
花大力气去用自己的`PluginClassLoader`替代`loadApk`的`mClassLoader`变量！至于这个 PluginClassLoader啊，背后隐藏着一个和`奇酷手机青春版兼容`的故事。哈哈哈哈，有兴趣去看下作者的注释！

另外一件事就是在生产这个LoadedApk对象的时候，他不知直接调用构造函数，而是通过`getPackageInfoNoCheck`函数去生成一个回来的，而且分了两个版本！
我们传进去的参数有两个：`ApplicationInfo` 和`CompatibilityInfo` 。
前者`ApplicationInfo`表示我们的`AndroidManifes`的`application`这个**Tag**


	<?xml version="1.0" encoding="utf-8"?>
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.example.ApiTest"
	    android:versionCode="1"
	    android:versionName="1.0" >
	 
	    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
	    
	    <application   //<--就是这个
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme"
        android:persistent="true">
			 ...
		</application>
		
	</manifest>

因此我在开头有提到那个PackageParser类！他兼容很多版本，因为没办法，谷歌一直改来改去。关于这个类是如何生成这个信息我们就不深究了。背后也涉及不少内容，但现在不是我们关系的重点。


这样，一个基本的启动插件Activity的流程主要核心内容就大致结束了！


# 小坑
在上面过程，还有一个小细节没提到，就是我们还需对PMS做一些小手脚 ！
为啥呢？让我们再来看下`ActivityThread`的`performLaunchActivity`函数

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent){
 
        ...

        Activity activity = null;  
         java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
         activity = mInstrumentation.newActivity(
                 cl, component.getClassName(), r.intent);
         StrictMode.incrementExpectedActivityCount(activity.getClass());
         r.intent.setExtrasClassLoader(cl);
         r.intent.prepareToEnterProcess();
         if (r.state != null) {
             r.state.setClassLoader(cl);
         } 
         
         //重要的一句
         Application app = r.packageInfo.makeApplication(false, mInstrumentation);

          ...
          activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
                         
          mInstrumentation.callActivityOnCreate(activity, r.state);                 
          ....          
        return activity;
    }
    
在启动Activity前，会有一句调用我们的`loadApk`的`makeApplication()`方法！在这个方法最后
  
	private void initializeJavaContextClassLoader() {
	    IPackageManager pm = ActivityThread.getPackageManager();
	    android.content.pm.PackageInfo pi;
	    try {
	        pi = pm.getPackageInfo(mPackageName, 0, UserHandle.myUserId());
	    } catch (RemoteException e) {
	        throw new IllegalStateException("Unable to get package info for "
	                + mPackageName + "; is system dying?", e);
	    }
	    if (pi == null) {
	        throw new IllegalStateException("Unable to get package info for "
	                + mPackageName + "; is package not installed?");
	    }
    }

这个方法之后回去调用 PM的`pm.getPackageInfo(mPackageName, 0, UserHandle.myUserId());`方法，这个方法需要我们需HOOK掉，替换信息，因为我们APK是真的没有安装在系统上，是不可能获取到的！
导致了PI==null，然后就崩溃咯！也亏他们一开始就想到这个pi可能为空。他们自己在写系统的时候是遇到什么坑，才会加多这个判断呢？

所以我们看到在插件的`IPackageManagerHookHandle`类，里面有对`getPackageInfo`函数做处理。让他返回了我们parse后的内容

	private class getPackageInfo extends HookedMethodHandler { 

        @Override
        protected boolean beforeInvoke(Object receiver, Method method, Object[] args){
        
           final int index0 = 0, index1 = 1;
           String packageName = null;
           if (args.length > index0) {
               if (args[index0] != null && args[index0] instanceof String) {
                   packageName = (String) args[index0];
               }
           }

           int flags = 0;
           if (args.length > index1) {
               if (args[index1] != null && args[index1] instanceof Integer) {
                   flags = (Integer) args[index1];
               }
           }

           if (packageName != null) {
              PackageInfo packageInfo = null;
              packageInfo = PluginManager.getInstance().
              getPackageInfo(packageName, flags);//就是这句啦
              if (packageInfo != null) {
	             setFakedResult(packageInfo);
                 return true;
	          }
           } 
           
           return super.beforeInvoke(receiver, method, args);
       }
    }

到这里！我们成功的解析了大致的内容！这样这篇也可以收尾了！

# 小结

最近忙完手头一些重构工作，从五号起稿到今天终于完成这篇文章了！！！！可以开心去吃饭了！！
虽然都八点了，但感觉全身愉快了好多，中午写到现在，也只是把前进段后面的补上去。感觉也没什么内容就耗费了我八个多小时，不得说...真的挖来挖去真的不容易，熟悉了一遍快捷键...
 

看完这部分，星罗密布的各种版本差异的坑啊，再次觉得作者的功力深厚，确实得花不少时间才能做好。

LBE有开一个平行空间的概念，某酷的手机也是出类似的功能，对于系统底层的探索和性能的压榨真是不容易啊。得抽空回去修改自己的PMS解析文章才行，现在看下好多内容没在里面写到，解析不到位啊。

