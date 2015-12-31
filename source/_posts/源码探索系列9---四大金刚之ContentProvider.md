title: 源码探索系列9---四大金刚之ContentProvider
date: 2015-12-19 02:42
tags: [android,源码,ContentProvider]
categories: android
------------------------------------------

好了，终于到了最后一个啦，写到这里，真的觉得不容意啊，以前看这些组件也就那样了，现在还要记录下来，重点是这东西都被分析烂了，我们这些后人屌丝还在写，没点突破的。真没意思呢！就当写作业咯。啦啦啦啦，不管如何，让我们开始看下吧

# 起航 --- ContentResolver
API： 23

说真的，这个组件我还真的相对于Activity和Service用起来真的好少啊！
现在都不能记起来用他来干嘛了，虽然知道他能用来做跨进程通讯用，不过对于一般的app。
这玩意还真的用的次数少啊! 现在能想起来的一次使用这个就是要获取图片的时候。
以前开发的时候遇到的恶心的是，有些Rom把相册给阉割了，搞了个别的来代替，导致调不了图库！
所以搞到需要自己开发一个，真的很无语！ 

	ContentResolver mContentResolver = this.getContentResolver();
	Cursor mCursor = mContentResolver.query(mImageUri, null,
                MediaStore.Images.Media.MIME_TYPE + "=? or "
                        + MediaStore.Images.Media.MIME_TYPE + "=? or " + MediaStore.Images.Media.MIME_TYPE + "=?" + " or " + MediaStore.Images.Media.MIME_TYPE + "=?",
                new String[]{"image/jpeg", "image/png", "image/jpg", "image/jpe"},
                MediaStore.Images.Media.DATA);

从这里我们开始看起吧

<!--more-->

	public final Cursor query(Uri uri, String[] projection,
            String selection, String[] selectionArgs, String sortOrder) {
        return query(uri, projection, selection, selectionArgs, sortOrder, null);
    }


    public final Cursor query(final Uri uri, String[] projection,
            String selection, String[] selectionArgs, String sortOrder,
            CancellationSignal cancellationSignal) {
            
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try { 
            ...

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) { 
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);

            // Wrap the cursor object into CursorWrapperInner object.
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) { 
            return null;
        } finally {
           ...
        }
    }
	@Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }

开头调用`acquireUnstableProvider(uri)`，经过一轮跟踪，发现他跑到了`ActivityThread`里面的`acquireProvider`，内容如下

	public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
            
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        } 
        
        IActivityManager.ContentProviderHolder holder = null;
        try {
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
        }
        if (holder == null) {         
            return null;
        } 
        
        // Install provider will increment the reference count for us, and break
        // any ties in the race.
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
我们看到他先查找本地有没有，有就返回，没有就去找了AMS要`ContentProvider`，拿回来后就存本地，然后返回。
好了，让我们去看下AMS里面的实现吧

	 @Override
    public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String name, int userId, boolean stable) {
            
        enforceNotIsolatedCaller("getContentProvider");
        if (caller == null) {
            String msg = "null IApplicationThread when getting content provider "
                    + name;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        } 
        
        return getContentProviderImpl(caller, name, null, stable, userId);
    }
他跑去了他的实现函数`getContentProviderImpl(）`....真无语，同个类里面还搞抽象和实现啊，
像前面那样写成`realGetContentProviderImpl(）`不就好了嘛，哈哈，真 **·** 干活函数 
不过这个实现还不小，三百多行，这在AMS里面都是这样的，看着头疼，一堆的判断，截取关键的部分：

	private final ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
            String name, IBinder token, boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;

        ...
        
        proc = startProcessLocked(cpi.processName,
                                 cpr.appInfo, false, 0, "content provider",
	                             new ComponentName(cpi.applicationInfo.packageName,
	                             cpi.name), false, false, false);
	                             
         cpr.launchingApp = proc;
         mLaunchingProviders.add(cpr);
 

         mProviderMap.putProviderByName(name, cpr);
         conn = incProviderCountLocked(r, cpr, token, stable); 
         
        return cpr != null ? cpr.newHolder(conn) : null;
    }
这里需要补充的说明是，我们都知道那个ContentProvider的启动是伴随进程的启动，启动进程是有上面的startProcessLocked(）来完成的。我们来看下他里面写了什么。。。

	  private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
             ...
            
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
             ...
    }
    
这个函数也是一个巨无霸，我想那个AMS一定是靠几个人写完的，因为他的函数风格不在一样，呵呵，虽然我也经常写得不一样。只截取重要的几句..
这个函数最终通过调用Process的start方法来启动。我们继续深入看下

	 public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }
好，遇到了一个传说级别的名字了`Zygote`，不要问我这个名词代表的故事，百度下吧。
感觉写完这些组件后，可以开始写更底层的部分了...哎。说真的，挺无聊的，看完又没怎么样，但出来混，他们老觉得高级开发就得看底层代码，精通整个Android系统啊！这学习量真的不小，不过也就那点工资...
还不如学校门口一个摆摊的，好了，罗嗦了，继续吧。

结论就是通过Zygote,最后我们到了ActivityThread的main方法，还记得这个玩意嘛？
我们在很久前解析那个Handler的时候有提到，程序的入口方法是ActivityThread的`mian`方法！
是不是觉得很像以前学java时候！居然在安卓里面遇到人家！

	public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        ...
        
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

整个的main方法就初始主线程的地方，同时开动主Looper，	然后attach这个方法会最终把消息传递AMS，然后AMS就完成了ContentProvider的创建！

	private void attach(boolean system) {
          ...
          
           try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            } 
          ...
    }

	private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        ...
        
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Binding proc "
                    + processName + " with config " + mConfiguration);
            ApplicationInfo appInfo = app.instrumentationInfo != null
                    ? app.instrumentationInfo : app.info;
            app.compat = compatibilityInfoForPackageLocked(appInfo);
            if (profileFd != null) {
                profileFd = profileFd.dup();
            }
            ProfilerInfo profilerInfo = profileFile == null ? null
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
            
         ...
    }
每次到AMS里面的函数都是一大串！！！这里的thread就是那个`ApplicationThread`，我们很熟悉。
走了一圈，又跑回来了

	 public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

            ...

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            sendMessage(H.BIND_APPLICATION, data);
        }

最终这个变成了同我们的H先生发送BIND_APPLICATION消息！

	private void handleBindApplication(AppBindData data) {
        ...
        // 1.创建Context        
       ContextImpl instrContext = ContextImpl.createAppContext(this, pi);
              
	   java.lang.ClassLoader cl = instrContext.getClassLoader();
       mInstrumentation = (Instrumentation)
             cl.loadClass(data.instrumentationName.getClassName()).newInstance();
	     
      mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);

       ...
       // 2.创建app
      Application app = data.info.makeApplication(data.restrictedBackupMode, null);
      mInitialApplication = app;

       ...
       // 3. 启动ContentProvider，发送消息，调用onCreate函数
      // don't bring up providers in restricted mode; they may depend on the
      // app's custom Application class
      if (!data.restrictedBackupMode) {
          List<ProviderInfo> providers = data.providers;
          if (providers != null) {
              installContentProviders(app, providers);
              // For process that contains content providers, we want to
              // ensure that the JIT is enabled "at some point".
              mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
          }
      }

      ...
      //4. 启动我们Application的onCreate方法。
      // Do this after providers, since instrumentation tests generally start their
         // test thread at this point, and we don't want that racing.         
	   mInstrumentation.onCreate(data.instrumentationArgs);　
	    mInstrumentation.callApplicationOnCreate(app);   
	    StrictMode.setThreadPolicy(savedPolicy);
    　
    }

这个函数受AMS污染，也是贼长的一个函数。
在这个handleBindApplication函数里面，我截取保留了核心的信息，同时加了注释。
具体就是创建`context`和`Application`，然后通过installContentProviders(）来启动当前进程的`ContentProvider`，所以下次有人和你提问题，说你知道那个ContentProvider嘛？
你就反问，说你知道这个组件是在哪里初始化的嘛？
不知道你就说是在ActivityThread里面的H先生接受到了BindApplication消息，然后在处理消息对应的`handleBindApplication（）`方法里面的`installContentProviders(）`函数啦！
是不是很装逼的样子！！

为了彻底的装逼，我们来看下这个installContentProviders(）到底做了什么！
免得人家也看过代码，惨遭装逼失败的下场。

	private void installContentProviders(
            Context context, List<ProviderInfo> providers) {
            
        final ArrayList<IActivityManager.ContentProviderHolder> results =
            new ArrayList<IActivityManager.ContentProviderHolder>();

        for (ProviderInfo cpi : providers) {
          
            IActivityManager.ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }
        }

        try {
            ActivityManagerNative.getDefault().publishContentProviders(
                getApplicationThread(), results);
        } catch (RemoteException ex) {
        }
    }
这里面遍历了`providers`，然后`installProvider`去启动他们。最后找AMS把他们给注册保存了。
而且是保存在一个叫`mProviderMap`的map里面中，便于别的进程通过AMS来调用！
有代码有真相，有木有，就在下面，没骗人的啊！

	 public final void publishContentProviders(IApplicationThread caller,
            List<ContentProviderHolder> providers) {
            
         ...
           
        enforceNotIsolatedCaller("publishContentProviders");
        synchronized (this) {
            final ProcessRecord r = getRecordForAppLocked(caller);
             final long origId = Binder.clearCallingIdentity();
             

            final int N = providers.size();
            for (int i=0; i<N; i++) {
                ContentProviderHolder src = providers.get(i);
 
                ...
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                              
                if (dst != null) {
                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }
                    ...
                     updateOomAdjLocked(r);
                }
            }
            Binder.restoreCallingIdentity(origId);
        }
    }

 在彻底的装逼，我们需要来看下这个`installProvider`函数
	
	private IActivityManager.ContentProviderHolder installProvider(Context context,
            IActivityManager.ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
 
        ...
        
            try { //用ClassLoader去加载！
                final java.lang.ClassLoader cl = c.getClassLoader();
                localProvider = (ContentProvider)cl.
                    loadClass(info.name).newInstance();
                provider = localProvider.getIContentProvider();
                 ...
                // XXX Need to create the correct context for this provider.
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
               ...   
             }
                return null;
            }
        }  

        ...
    }
好了他的启动过程基本到这里结束，我们回溯到前面的步骤，回到我们处理Application消息的第四点

	 //4. 启动我们Application的onCreate方法。
      // Do this after providers, since instrumentation tests generally start their
         // test thread at this point, and we don't want that racing.         
	   mInstrumentation.onCreate(data.instrumentationArgs);　
	    mInstrumentation.callApplicationOnCreate(app);   
	    
	    StrictMode.setThreadPolicy(savedPolicy);

好了，整个过程基本这样了。

我们从一开始的`acquireUnstableProvider(uri);`函数跑到这里，都快忘记原本是要干嘛了。

# 续航 ==== Query

 
让我们继续启程，回到一开始query函数，在获得了我们IContentProvider后，就调用了他的query方法。

		qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);

这个`IContentProvider` 是一个`Binder`接口，具体的干活的人是谁呢？
`IContentProvider`的具体实现是`ContentProviderNative`，然后`Transport` 又继承了它。

	abstract public class ContentProviderNative extends Binder implements IContentProvider 
	
	class Transport extends ContentProviderNative 
	
这个类是躲在ContentProvider的里面的即ContentProvider.Transport 。好了，写了这么多，回主线

	@Override
        public Cursor query(String callingPkg, Uri uri, String[] projection,
                String selection, String[] selectionArgs, String sortOrder,
                ICancellationSignal cancellationSignal) {
            validateIncomingUri(uri);
            uri = getUriWithoutUserId(uri);
            if (enforceReadPermission(callingPkg, uri, null) != AppOpsManager.MODE_ALLOWED) {
                   ...
                 Cursor cursor = ContentProvider.this.query(uri, projection, selection,
                        selectionArgs, sortOrder, CancellationSignal.fromTransport(
                                cancellationSignal));
                if (cursor == null) {
                    return null;
                }

                // Return an empty cursor for all columns.
                return new MatrixCursor(cursor.getColumnNames(), 0);
            }
            
            final String original = setCallingPackage(callingPkg);
            try {
                return ContentProvider.this.query(
                        uri, projection, selection, selectionArgs, sortOrder,
                        CancellationSignal.fromTransport(cancellationSignal));
            } finally {
                setCallingPackage(original);
            }
        }
到这里去调用了ContentProvider的query方法。这样整个过程就结束了。其余的方法也在这个类里面，套路类似的，就不再写了，这篇文章已经写了很长了，而且现在有事，得去做点别了！

	

# 后记

今天写得好晚，看到都有点累，不段的翻滚。。。写到后面都没精神和注意力了。。
先写到这里，洗个澡睡觉，明天抽时间在继续做补充。。

但无论如何把四大金刚的源码都写了一遍了。

====
更新：
重新补充了部分内容，把整个流程更细化了下。
不过还是有些具体的内容不是很清楚，
下一次在游览别的时候，能有空再把这个过程写得更清楚的话，就再来更新吧

对于Zygote这个下次得抽时间再写一篇他的专辑。
谷歌起这个名字给它，这么有深度东西，很有故事。

下一篇就这么定是他了 ^_ ^

