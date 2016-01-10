title:  源码探索系列20---安装APK的PackageManagerService
date: 2016-01-10 23:56:46
tags: [android,源码,PMS]
categories: android

------------------------------------------

 我们新新苦苦的写好了程序，打好包，给用户安装。
 那么问题来了，为何能够安装这个APK呢， 背后到底发生了什么事？
  
  ![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/20150105041330439.png)
  
<!--more-->
我们的安装程序是靠`PackageManagerService（应用程序管理服务，缩写PMS）`来完成的，他会对APK文件进行解析，得到应用程序的相关信息（其实就是解析析应用程序配置文件AndroidManifest.xml的过程，并从里面得到得到应用程序的相关信息，例如得到应用程序的组件Activity、Service、Broadcast Receiver和Content Provider等信息，有了这些信息后，通过ActivityManagerService这个服务，我们就可以在系统中正常地使用这些应用程序了），完成应用程序的安装过程。
 
  
# 起航
API:23

PackageManagerService是由SystemServer组件启动的(SystemServer由Zygote进程负责启动)。

在介绍前，需要说下我们的apk安装位置 

	 系统自带的保存在/system/app/ 。
	 我们安装的保存在/data/app/包名/

	
Package Manager创建数据目录/data/data/xxx/来保存数据库、shared preference、本地函数库和缓存数据。
  
好了，我们看下他的构造函数，真的很长！

	public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
             
		...

        File dataDir = Environment.getDataDirectory();
        mAppDataDir = new File(dataDir, "data");
        mAppInstallDir = new File(dataDir, "app");//我们的app路径
        mUserAppDataDir = new File(dataDir, "user");
        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");

	   ...

	    File frameworkDir = new File(Environment.getRootDirectory(), "framework"); 
        alreadyDexOpted.add(frameworkDir.getPath() + "/framework-res.apk"); 
        alreadyDexOpted.add(frameworkDir.getPath() + "/core-libart.jar");


		 ...
       // Collect ordinary system packages.
        final File systemAppDir = new File(Environment.getRootDirectory(), "app");
        scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

        // Collect all OEM packages.
        final File oemAppDir = new File(Environment.getOemDirectory(), "app");
        scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
                | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

        ...

            if (!mOnlyCore) {            
                scanDirLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
                scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);
                 ... 
            }            
            ... 
    }
不过缩小成上面主要的那几句，配置要扫码的目录，然后去扫描下，我们去看下那个扫描函数的内容

	private void scanDirLI(File dir, int parseFlags, int scanFlags, long currentTime) {
	
        final File[] files = dir.listFiles();
        ...
        for (File file : files) {
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }
            try {
                scanPackageLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK,
                        scanFlags, currentTime, null);
            } catch (PackageManagerException e) {
	            ...
             }
        }
    }
交给`scanPackageLI`去解析apk文件。

	private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
            long currentTime, UserHandle user) throws PackageManagerException {
 
        parseFlags |= mDefParseFlags;
        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setOnlyCoreApps(mOnlyCore);
        pp.setDisplayMetrics(mMetrics);

        if ((scanFlags & SCAN_TRUSTED_OVERLAY) != 0) {
            parseFlags |= PackageParser.PARSE_TRUSTED_OVERLAY;
        }

        final PackageParser.Package pkg;
        try {
            pkg = pp.parsePackage(scanFile, parseFlags);
        } catch (PackageParserException e) {
            throw PackageManagerException.from(e);
        }

		...
		
          // Note that we invoke the following method only if we are about to unpack an application
        PackageParser.Package scannedPkg = scanPackageLI(pkg, parseFlags, scanFlags
                | SCAN_UPDATE_SIGNATURE, currentTime, user);

         return scannedPkg;
    }
    
我们看到他主要是交给了PackageParser去做的解析，然后在通过另外一个同名的函数去做进一步的解析工作。
我们先看在解析的内容，再看下那个同名的

	public Package parsePackage(File packageFile, int flags) throws PackageParserException {
        if (packageFile.isDirectory()) {
            return parseClusterPackage(packageFile, flags);
        } else {
            return parseMonolithicPackage(packageFile, flags);
        }
    }
他根据是否为文件夹，再分开的处理，我们看下单个文件的。
	 
    public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
       ...
        final AssetManager assets = new AssetManager();
        try {
            final Package pkg = parseBaseApk(apkFile, assets, flags);
            pkg.codePath = apkFile.getAbsolutePath();
            return pkg;
        } finally {
            IoUtils.closeQuietly(assets);
        }
    }
 又被打包到下面`parseBaseApk`里面去了。

	private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        final String apkPath = apkFile.getAbsolutePath();
		... 
		
        mArchiveSourcePath = apkFile.getAbsolutePath(); 
        final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

        Resources res = null;
        XmlResourceParser parser = null;
        try {
            res = new Resources(assets, mMetrics, null);
            assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                    Build.VERSION.RESOURCES_SDK_INT);
                    
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

            final String[] outError = new String[1];
            final Package pkg = parseBaseApk(res, parser, flags, outError);
			... 
            return pkg;

        }  
        ...
    }

根据给的apk路径去把资源加到`AssetManger`里面，然后构造一个解析`AndroidManifest`文件的解析器，再交给`parseBaseApk()`去生成我们需要的pkg。我们看下这个函数的内容，挺长的

	private Package parseBaseApk(Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
        final boolean trustedOverlay = (flags & PARSE_TRUSTED_OVERLAY) != 0;

        AttributeSet attrs = parser;
		//保存写在AndroidManifest里面的类
        mParseInstrumentationArgs = null;
        mParseActivityArgs = null;
        mParseServiceArgs = null;
        mParseProviderArgs = null;

        final String pkgName;
        final String splitName;
        try {
            Pair<String, String> packageSplit = parsePackageSplitNames(parser, attrs, flags);
            pkgName = packageSplit.first;//我们的包名
            splitName = packageSplit.second;
        } catch (PackageParserException e) {
            mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME;
            return null;
        }

        int type;

		...

        final Package pkg = new Package(pkgName);
        boolean foundApp = false;
		//下面是一堆配置信息
        TypedArray sa = res.obtainAttributes(attrs,
                com.android.internal.R.styleable.AndroidManifest);
        pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
        pkg.baseRevisionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
        pkg.mVersionName = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_versionName, 0);
        if (pkg.mVersionName != null) {
            pkg.mVersionName = pkg.mVersionName.intern();
        }
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_sharedUserId, 0);
        if (str != null && str.length() > 0) {
            String nameError = validateName(str, true, false);
            if (nameError != null && !"android".equals(pkgName)) {
                outError[0] = "<manifest> specifies bad sharedUserId name \""
                    + str + "\": " + nameError;
                mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID;
                return null;
            }
            
            //我们在共享时候需要到的mSharedUserId，重要重要的
            pkg.mSharedUserId = str.intern();
            pkg.mSharedUserLabel = sa.getResourceId(
                    com.android.internal.R.styleable.AndroidManifest_sharedUserLabel, 0);
        }

        pkg.installLocation = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_installLocation,
                PARSE_DEFAULT_INSTALL_LOCATION);
        pkg.applicationInfo.installLocation = pkg.installLocation;
        pkg.coreApp = attrs.getAttributeBooleanValue(null, "coreApp", false);
        sa.recycle();
		//上面是一对配置信息的内容

		 ...
 
 
       //下面开始根据各种标签做解析工作，有些标签还是第一次看多。
       //例如合并标签的eat-comment和protected-broadcast等
       //我就不删除所有的内容，保留下标签的名字吧。看下你认识几个
        int outerDepth = parser.getDepth();
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("application")) {
                if (foundApp) {
                    if (RIGID_PARSER) {
                        outError[0] = "<manifest> has more than one <application>";
                        mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                        return null;
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        XmlUtils.skipCurrentTag(parser);
                        continue;
                    }
                }

                foundApp = true;
                if (!parseBaseApplication(pkg, res, parser, attrs, flags, outError)) {
                    return null;
                }
            } else if (tagName.equals("overlay")) {
	            ... 
            } else if (tagName.equals("key-sets")) {
                 ...
            } else if (tagName.equals("permission-group")) {
                ...
            } else if (tagName.equals("permission")) {
                ...
            } else if (tagName.equals("permission-tree")) {
                 ...
            } else if (tagName.equals("uses-permission")) {
                if (!parseUsesPermission(pkg, res, parser, attrs)) {
                    return null;
                }
            } else if (tagName.equals("uses-permission-sdk-m")
               ...
            } else if (tagName.equals("uses-configuration")) {
               ...
            } else if (tagName.equals("uses-feature")) {
                ...
            } else if (tagName.equals("feature-group")) {
                ...
            } else if (tagName.equals("uses-sdk")) {
                ...
            } else if (tagName.equals("supports-screens")) {
                ...                
            } else if (tagName.equals("protected-broadcast")) {
                 ...                
            } else if (tagName.equals("instrumentation")) {
                ...
            } else if (tagName.equals("original-package")) {
                ...
            } else if (tagName.equals("adopt-permissions")) {
                 ...
            } else if (tagName.equals("compatible-screens")) {
              ...
            } else if (tagName.equals("supports-input")) {
              ...  
            } else if (tagName.equals("eat-comment")) {
               ...                
            } else if (RIGID_PARSER) {
                ...
            } else {
                ...
            }
        }
         ...
        return pkg;
    }
接着我们去看下重要的一个，就是解析`Application`里面的


	private boolean parseBaseApplication(Package owner, Resources res,
            XmlPullParser parser, AttributeSet attrs, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;

		//又开始获取一些配置的信息内容啦
        TypedArray sa = res.obtainAttributes(attrs,
                com.android.internal.R.styleable.AndroidManifestApplication);
        String name = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifestApplication_name, 0);         
         ...
         ai.icon = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_icon, 0);
        ai.logo = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_logo, 0);
        ai.banner = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_banner, 0);
        ai.theme = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_theme, 0);
        ai.descriptionRes = sa.getResourceId(
                com.android.internal.R.styleable.AndroidManifestApplication_description, 0);

		 ....省略了一堆见过和没见过的flag设置
		 
		//下面是最重要的内容啦
        final int innerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, attrs, flags, outError, 
						                false,owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError=ackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                } 
                owner.activities.add(a);

            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, attrs, flags, outError, 
							               true, false);
                if (a == null) {
                    mParseError=PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.receivers.add(a);

            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, attrs, flags, outError);
                if (s == null) {
                    mParseError=PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }
                owner.services.add(s);

            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, attrs, flags, outError);
                if (p == null) {
                    mParseError =PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;                    
                    return false;
                }

                owner.providers.add(p);

            } else if (tagName.equals("activity-alias")) {
               ...
            } else if (parser.getName().equals("meta-data")) {
				...
            }  
        }
		...
         return true;
    }
好！看到这里，基本我们开头说的，一堆的Activity，service的信息就都看到了！
接下去的具体的各个的解析就不说了，我们就那一个解析Activity的例子来看下就好了。

	private Activity parseActivity(Package owner, Resources res,
            XmlPullParser parser, AttributeSet attrs, int flags, String[] outError,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {

        TypedArray sa = res.obtainAttributes(attrs, R.styleable.AndroidManifestActivity);

        if (mParseActivityArgs == null) {
            mParseActivityArgs = new ParseComponentArgs(owner, outError,
                    R.styleable.AndroidManifestActivity_name,
                    R.styleable.AndroidManifestActivity_label,
                    R.styleable.AndroidManifestActivity_icon,
                    R.styleable.AndroidManifestActivity_logo,
                    R.styleable.AndroidManifestActivity_banner,
                    mSeparateProcesses,
                    R.styleable.AndroidManifestActivity_process,
                    R.styleable.AndroidManifestActivity_description,
                    R.styleable.AndroidManifestActivity_enabled);
        }        
        mParseActivityArgs.tag = receiver ? "<receiver>" : "<activity>";
        mParseActivityArgs.sa = sa;
        mParseActivityArgs.flags = flags;
        
        Activity a = new Activity(mParseActivityArgs, new ActivityInfo());

		... 省略一堆配置内容
         

        if (!receiver) {
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_hardwareAccelerated,
                    hardwareAccelerated)) {
                a.info.flags |= ActivityInfo.FLAG_HARDWARE_ACCELERATED;
            }

            a.info.launchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_launchMode, ActivityInfo.LAUNCH_MULTIPLE); //默认的启动模式是LAUNCH_MULTIPLE
                    
            a.info.documentLaunchMode = sa.getInt(R.styleable.
                    AndroidManifestActivity_documentLaunchMode,
                    ActivityInfo.DOCUMENT_LAUNCH_NONE);
                    
            a.info.maxRecents = sa.getInt(R.styleable.
                    AndroidManifestActivity_maxRecents,
                    ActivityManager.getDefaultAppRecentsLimitStatic()); 

            a.info.persistableMode = sa.getInteger(
                    R.styleable.AndroidManifestActivity_persistableMode,
                    ActivityInfo.PERSIST_ROOT_ONLY);
			...
        } else {
            a.info.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
            a.info.configChanges = 0;

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_singleUser, false)) {
                a.info.flags |= ActivityInfo.FLAG_SINGLE_USER;
                if (a.info.exported && (flags & PARSE_IS_PRIVILEGED) == 0) {
                     a.info.exported = false;
                    setExported = true;
                }
            }
        }
	
		...
		//和前面一样的套路，根据标签做进一步处理		
         while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals("intent-filter")) {
		          ...
            } else if (!receiver && parser.getName().equals("preferred")) {
	            ...
             } else if (parser.getName().equals("meta-data")) {
     
            } else {
		      ...
            }            
        } 
        return a;
    }




---

# 前进
好了，看了前面那么一大串，我们回到主线上，到一开始的另外一个同名函数里面的工作。


	private PackageParser.Package scanPackageLI(PackageParser.Package pkg, int 
			parseFlags, int scanFlags, long currentTime, UserHandle user) throws 
			PackageManagerException {
            
        boolean success = false;
        try {
            final PackageParser.Package res = scanPackageDirtyLI(pkg, parseFlags, 
		            scanFlags, currentTime, user);
            success = true;
            return res;
        } finally {
            if (!success && (scanFlags & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
                removeDataDirsLI(pkg.volumeUuid, pkg.packageName);
            }
        }
    }
    
又扔给别人工作。这个函数超过了1K行，我服，看得要哭啊。

	private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg, int parseFlags,
            int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
        final File scanFile = new File(pkg.codePath);
        
		 ...
		  
        // writer
        synchronized (mPackages) {
	         ...
	         //添加各种信息，基本模式雷同,为何责任不分割成各个小函数，唉
             
            N = pkg.services.size();             
            ...
             
            N = pkg.receivers.size();
 			...
			
            N = pkg.activities.size();
            r = null;
            for (i=0; i<N; i++) {
                PackageParser.Activity a = pkg.activities.get(i);
                a.info.processName = fixProcessName(pkg.applicationInfo.processName,
                        a.info.processName, pkg.applicationInfo.uid);
                mActivities.addActivity(a, "activity");
                if ((parseFlags&PackageParser.PARSE_CHATTY) != 0) {
                    if (r == null) {
                        r = new StringBuilder(256);
                    } else {
                        r.append(' ');
                    }
                    r.append(a.info.name);
                }
            } 

            N = pkg.permissionGroups.size();
             ...
              
            N = pkg.permissions.size();
             ...
             
            N = pkg.instrumentation.size();
             ...

        return pkg;
    }
 这个函数主要就是把前面一步解析出来的各种信息再处理下，添加个各种mActivities的变量去。
	
	public class PackageManagerService extends IPackageManager.Stub {
	
		final ActivityIntentResolver mActivities =
	            new ActivityIntentResolver();
	
	    // All available receivers, for your resolving pleasure.
	    final ActivityIntentResolver mReceivers =
	            new ActivityIntentResolver();
	
	    // All available services, for your resolving pleasure.
	    final ServiceIntentResolver mServices = new ServiceIntentResolver();
	
	    // All available providers, for your resolving pleasure.
	    final ProviderIntentResolver mProviders = new ProviderIntentResolver();
	    
		...
    }
    
通过这些操作，我们整个APK的信息就建立了。
最后我们的代码就可以通过各种`显示`和`隐示`的Intent等来做各种事情了。

	
 
# 后记

关于这个PMS还有很多没说的，例如开头图片的那个安装界面等，下次再补充吧，现在又是12点了，我要早点睡，不熬夜！

---

很久前在做app的时候，想做的一件事情是检测app被卸载这件事，然后弹一个问卷页面，想收集问题，不过查了下解决方案，都没什么漂亮的，有的是靠监听data文件被删除的，这个很耗费电量。不像Windows上卸载是靠自神提供的UnInstall.exe去执行。所以不好做到，看完这整个过程，似乎也没看到些什么特别好的信息。


参考资料：

[Android应用程序安装过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6766010) 

[PackageManagerService启动及初始化流程](http://blog.csdn.net/new_abc/article/details/12966503)
 