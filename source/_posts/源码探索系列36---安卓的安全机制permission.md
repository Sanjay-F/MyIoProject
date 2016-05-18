title: 源码探索系列36---安卓的安全机制permission
date: 2016-05-18 00:45:46
tags: [android,源码,permission]
categories: android

------------------------------------------
 
本文尝试从源码的角度来对permission等机制做一个了解。
另外从安装一个apk的整个流程来看与permission相关的内容。
 

Android 是一个**权限分离**的系统。利用 Linux 已有的权限管理机制，通过为每一个APP分配不同的`uid`和`gid`，从而使得不同的 App之间的私有数据和访问(native 以及 java 层通过这种 sandbox 机制，都可以)达到隔离的目的。

与此同时，安卓在此基础上还进行了扩展，提供了`permission机制`，它主要是用来对`App`可以执行的某些具体操作进行权限细分和访问控制，同时提供了`per-URI permission 机制`，用来提供对某些特定的数据块进行 ad-hoc 方式的访问。
 
<!--more-->

## Android 系统权限定义 uid 、 gid 、 gids

安卓的权限分离是在Linux已有的`uid`、`gid` 、`gids`基础上的UID。为了做到隔离，避免一些攻击等，当我们安装应用程序时，系统会为它分配一个uid(见`PackageManagerService`中`newUserLP`部分)。

参考配置的id如下:		
	 
	// Tools for managing OS processes.	 
	public class Process {	
	
	  //Defines the root UID. 
	  public static final int ROOT_UID = 0;	
	   
	  //Defines the UID/GID under which system code runs. 
	  public static final int SYSTEM_UID = 1000;
	
	  //Defines the UID/GID under which the telephony code runs.	  
	  public static final int PHONE_UID = 1001;	
	   
	  //Defines the UID/GID for the user shell.
	  public static final int SHELL_UID = 2000;	
	  
	  //Defines the UID/GID for the log group. 
	  public static final int LOG_UID = 1007; 
	    
	  //Defines the UID/GID for the Bluetooth service process. 
	  public static final int BLUETOOTH_UID = 1002; 
	  
	  //Defines the UID/GID for the shared RELRO file updater process. 
	  public static final int SHARED_RELRO_UID = 1037;	
	 
	  //Defines the start of a range of UIDs (and GIDs), going from this
	  //number to {@link #LAST_APPLICATION_UID} that are reserved for assigning
	  //to applications. 
	  public static final int FIRST_APPLICATION_UID = 10000;	
	  //我们的起始id
	  
	  ...
	  	
	}     
	 
1. **UID** :  Android在安装一个应用程序，会为它分配一个 uid，我们普通应用程序的`uid`是从 10000 开始分配 （见 Process.FIRST_APPLICATION_UID ）。在上面的定义代码块的最后一行有写到， 0-10000系统进程的 uid 。
2.  **GID :** 对于普通应用程序来说， gid 等于 uid 。由于每个应用程序的 uid 和 gid 都不相同， 因此不管是 native 层还是 java 层都能够达到保护私有数据的作用 。
3.  **GIDS:** 。 gids 是由框架在 Application 安装过程中生成，与 Application 申请的具体权限相关。 如果 Application 申请的相应的 permission 被 granted ，而且 中有对应的 gid s ， 那么 这个 Application 的 gids 中将 包含这个 gids
 
---


# Android permission 管理机制
API:23
##  检测apk

在系统源码的packages/apps/PackageInstaller目录，对应系统的安装程序,就是熟悉的这个
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/36_QQ%E6%88%AA%E5%9B%BE20160517154926.png)

程序会解析包，对在AndroidManifest里面写的权限要求显示出来
分析包的安全的流程是这样的

	onCreate()-->initiateInstall()-->startInstallConfirm()

### oncreate()
在该oncreate内容，里面包含不少的操作。

    @Override
    protected void onCreate(Bundle icicle) {
    
	    super.onCreate(icicle);

        mPm = getPackageManager();
        mInstaller = mPm.getPackageInstaller();
        mUserManager = (UserManager) getSystemService(Context.USER_SERVICE);

        ...

        final PackageUtil.AppSnippet as;
        if ("package".equals(mPackageURI.getScheme())) {
            mInstallFlowAnalytics.setFileUri(false); 
            mPkgInfo = mPm.getPackageInfo(mPackageURI.getSchemeSpecificPart(),
                        PackageManager.GET_PERMISSIONS | 
                        PackageManager.GET_UNINSTALLED_PACKAGES);
            ...
            
        } else {
             
            final File sourceFile = new File(mPackageURI.getPath());
            PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);
            ...
            mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
            mPkgDigest = parsed.manifestDigest;
            
            ...
        }
 

        //set view
        setContentView(R.layout.install_start);  
		...

        // Block the install attempt on the Unknown Sources setting if necessary.
        if (!requestFromUnknownSource) {
            initiateInstall();
            return;
        }


        //对没在设置那里允许Unknown Sources的APP，要弹框提示，这对于安卓用户来说不陌生。        
        // If the admin prohibits it, or we're running in a managed profile, 
        //just show error and exit. Otherwise show an option to take the user to 
        // Settings to change the setting.final boolean isManagedProfile = 
        // mUserManager.isManagedProfile();
        
        if (!unknownSourcesAllowedByAdmin
                || (!unknownSourcesAllowedByUser && isManagedProfile)) {
            showDialogInner(DLG_ADMIN_RESTRICTS_UNKNOWN_SOURCES);
			...
        } else if (!unknownSourcesAllowedByUser) {
            // Ask user to enable setting first
            showDialogInner(DLG_UNKNOWN_SOURCES);
            ...
        } else {        
            initiateInstall();
        }

    }

在onCreate会调用getPackageInfo()去解析我们的包，生成一个package类

	PackageParser.Package parsed = PackageUtil.getPackageInfo(sourceFile);

之后调用generatePackageInfo()去生成我们的PackageInfo类的实例，这个是在后面解析时候需要用到的。

	mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                    PackageManager.GET_PERMISSIONS, 0, 0, null,
                    new PackageUserState());
	
上面的两步解析和在看插件化库DroidPlugin的内容有不少相似的地方，主要就是解析我们的apk文件，提取出AndroidManifest的内容等操作。在这里就不贴出来了。
	
### initiateInstall() 	
	private void initiateInstall() {
	
        String pkgName = mPkgInfo.packageName;
        // Check if there is already a package on the device with this name
        // but it has been renamed to something else.
        String[] oldName = mPm.canonicalToCurrentPackageNames(new String[] { pkgName });
        if (oldName != null && oldName.length > 0 && oldName[0] != null) {
            pkgName = oldName[0];
            mPkgInfo.packageName = pkgName;
            mPkgInfo.applicationInfo.packageName = pkgName;
        }
        // Check if package is already installed. 
        //display confirmation dialog if replacing pkg
        try {
            // This is a little convoluted because we want to get all uninstalled
            // apps, but this may include apps with just data, and if it is just
            // data we still want to count it as "installed".
            mAppInfo = mPm.getApplicationInfo(pkgName,
                    PackageManager.GET_UNINSTALLED_PACKAGES);
            if ((mAppInfo.flags&ApplicationInfo.FLAG_INSTALLED) == 0) {
                mAppInfo = null;
            }
        } catch (NameNotFoundException e) {
            mAppInfo = null;
        }
		...
        startInstallConfirm();
    }

initiateInstall()主要是做些初始化和检测的内容。
### startInstallConfirm()

	private void startInstallConfirm() {
	
        TabHost tabHost = (TabHost)findViewById(android.R.id.tabhost);
        tabHost.setup();
        ViewPager viewPager = (ViewPager)findViewById(R.id.pager);
        TabsAdapter adapter = new TabsAdapter(this, tabHost, viewPager);
		...
			 
        // If the app supports runtime permissions the new permissions will
        // be requested at runtime, hence we do not show them at install.
        boolean supportsRuntimePermissions = mPkgInfo.applicationInfo.targetSdkVersion
                >= Build.VERSION_CODES.M; 
                
                ...        

        AppSecurityPermissions perms = new AppSecurityPermissions(this, mPkgInfo);
        final int N = perms.getPermissionCount(AppSecurityPermissions.WHICH_ALL);

        ....

         if (!supportsRuntimePermissions) {
             newPermissionsFound =
                     (perms.getPermissionCount(AppSecurityPermissions.WHICH_NEW) > 0);
             mInstallFlowAnalytics.setNewPermissionsFound(newPermissionsFound);
             if (newPermissionsFound) {
                 permVisible = true;
                 mScrollView.addView(perms.getPermissionsView(
                         AppSecurityPermissions.WHICH_NEW));
             }
         }
         ...
         adapter.addTab(tabHost.newTabSpec(TAB_ID_NEW).setIndicator(
                 getText(R.string.newPerms)), mScrollView);       
        ...

        if (!supportsRuntimePermissions && N > 0) {
            ...            
            ((ViewGroup)root.findViewById(R.id.permission_list)).addView(
                        perms.getPermissionsView(AppSecurityPermissions.WHICH_ALL));
            adapter.addTab(tabHost.newTabSpec(TAB_ID_ALL).setIndicator(
                    getText(R.string.allPerms)), root);
        }
        
         ...
	}


在startInstallConfirm函数里面，会调用`AppSecurityPermissions`类去解析安装包，然后调用它的`getPermissionsView()`获得view，把解析到的内容显示在界面上。显示用的非常原始的`TabHost` ，这个类好久没用过了，只在一开始学安卓时候用过。

不知道你有没注意到，在开头的图片，是有分`new`和`ALL`两栏的，对于新安装包相对于现在安装在手机上的安装包，新的权限需求就在`New`栏。

#### AppSecurityPermissions()

	 public AppSecurityPermissions(Context context, PackageInfo info) {
        this(context);
        Set<MyPermissionInfo> permSet = new HashSet<MyPermissionInfo>();

        mPackageName = info.packageName;
        
        // Convert to a PackageInfo
        PackageInfo installedPkgInfo = null;
        // Get requested permissions
        if (info.requestedPermissions != null) {
         
           installedPkgInfo = mPm.getPackageInfo(info.packageName,
                    PackageManager.GET_PERMISSIONS);
            extractPerms(info, permSet, installedPkgInfo);
        }
        
        // Get permissions related to shared user if any
        if (info.sharedUserId != null) {
            int sharedUid;         
            sharedUid = mPm.getUidForSharedUser(info.sharedUserId);
            getAllUsedPermissions(sharedUid, permSet);          
        }
        
        // 这部分mPermsList就是所有的权限列表内容
        //我们在getPermissionsView()函数调用回来的内容就是根据这个list内容去构造出view。 
        mPermsList.addAll(permSet);
        setPermissions(mPermsList);
    }

首先，通过extractPerms函数来获取我们的权限内容，然后放到permSet里面去。在调用的时候传多了一个installedPkgInfo,这个参数是通过传我们要安装的报名去获取的，作用是查看是否已经有安装过这个包，有就返回这个包的信息回来。
之所以要这个参数，意思是查看下这个包以前是否安装过，对于安装过的程序，在展示权限信息的时候，会只展示这个新包要求的新的权限，相对于的原有的上一个版本已经出现过的就不再做展示啦。用户已经知道过了。

如果我们在配置文件还有关于shareUserId部分的内容，那么就查下相关的权限，关于sharedUerId，目前个人的开发还真没用过，因为不像大公司的全家桶套餐，有多个app，而且app间有一定的通讯需求。现在以一个独立的为主。


####  extractPerms()

	private void extractPerms(PackageInfo info, Set<MyPermissionInfo> permSet,
            PackageInfo installedPkgInfo) {
            
         //权限信息来自packageInfo类，这个类有所有的AndroidManifest的信息   
        String[] strList = info.requestedPermissions;
        int[] flagsList = info.requestedPermissionsFlags;
        if ((strList == null) || (strList.length == 0)) {
            return;
        }
        
        for (int i=0; i<strList.length; i++) {
            String permName = strList[i];
            try {
                PermissionInfo tmpPermInfo = mPm.getPermissionInfo(permName, 0);
                if (tmpPermInfo == null) {
                    continue;
                }
				
                int existingIndex = -1;
                if (installedPkgInfo != null
                        && installedPkgInfo.requestedPermissions != null) {
                        //判断手机安装了的包，相对于现在要安装的这个包，是否已有这条权限了。
                    for (int j=0; j<installedPkgInfo.requestedPermissions.length; j++) {
                        if (permName.equals(installedPkgInfo.requestedPermissions[j])) {
                            existingIndex = j;
                            break;
                        }
                    }
                }
                
                final int existingFlags = existingIndex >= 0 ?
                        installedPkgInfo.requestedPermissionsFlags[existingIndex] : 0;
                        
 
               if (!isDisplayablePermission(tmpPermInfo, flagsList[i], existingFlags)){
                     //过滤用户不需要知道的权限
                    // This is not a permission that is interesting for the user
                    // to see, so skip it.
                    continue;
                }
                
                //我们的权限是有分组的
                final String origGroupName = tmpPermInfo.group;
                String groupName = origGroupName;
                if (groupName == null) {
                    groupName = tmpPermInfo.packageName;
                    tmpPermInfo.group = groupName;
                }
                MyPermissionGroupInfo group = mPermGroups.get(groupName);
                if (group == null) {
                    PermissionGroupInfo grp = null;
                    if (origGroupName != null) {
                        grp = mPm.getPermissionGroupInfo(origGroupName, 0);
                    }
                    if (grp != null) {
                        group = new MyPermissionGroupInfo(grp);
                    } else {
                        // We could be here either because the permission
                        // didn't originally specify a group or the group it
                        // gave couldn't be found.  In either case, we consider
                        // its group to be the permission's package name.
                        tmpPermInfo.group = tmpPermInfo.packageName;
                        group = mPermGroups.get(tmpPermInfo.group);
                        if (group == null) {
                            group = new MyPermissionGroupInfo(tmpPermInfo);
                        }
                        group = new MyPermissionGroupInfo(tmpPermInfo);
                    }
                    mPermGroups.put(tmpPermInfo.group, group);
                }
                
                final boolean newPerm = installedPkgInfo != null
                   && (existingFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED)==0;
                   
                MyPermissionInfo myPerm = new MyPermissionInfo(tmpPermInfo);
                myPerm.mNewReqFlags = flagsList[i];
                myPerm.mExistingReqFlags = existingFlags;
                // This is a new permission if the app is already installed and
                // doesn't currently hold this permission.
                myPerm.mNew = newPerm;
                permSet.add(myPerm);
            } catch (NameNotFoundException e) {
                Log.i(TAG, "Ignoring unknown permission:"+permName);
            }
        }
    }


在开头的那张安装包的图片可以看到，左边有一排的声音，同步，设置等等，右边有对应的申请的权限具体内容。这个结构就是分组，每个组有对应的一些权限内容。权限信息保存用`PermissionInfo`，如权限的一些`protectionLevel` ，`group` ，`descriptionRes` 等属性。对于用户自己编的权限是跳过的，因为我们看到他会对返回的tmpPermInfo做判断是否为空.



#### isDisplayablePermission()

对于我们写在配置文件声明的权限，不全部都会展示出来的，是有一定的判断标准的，会过滤一些不需要用户知道的权限，具体的判断条件在这个函数里面。

	private boolean isDisplayablePermission(PermissionInfo pInfo, int newReqFlags,
            int existingReqFlags) {
        final int base = pInfo.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE;
        final boolean isNormal = (base == PermissionInfo.PROTECTION_NORMAL);

        // We do not show normal permissions in the UI.
        if (isNormal) {
            return false;
        }

        final boolean isDangerous = (base == PermissionInfo.PROTECTION_DANGEROUS)
                || ((pInfo.protectionLevel&PermissionInfo.PROTECTION_FLAG_PRE23) != 0);
        final boolean isRequired =
                ((newReqFlags&PackageInfo.REQUESTED_PERMISSION_REQUIRED) != 0);
        final boolean isDevelopment =
                ((pInfo.protectionLevel&PermissionInfo.PROTECTION_FLAG_DEVELOPMENT) != 0);
        final boolean wasGranted =
                ((existingReqFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0);
        final boolean isGranted =
                ((newReqFlags&PackageInfo.REQUESTED_PERMISSION_GRANTED) != 0);

        // Dangerous and normal permissions are always shown to the user if the permission
        // is required, or it was previously granted
        if (isDangerous && (isRequired || wasGranted || isGranted)) {
            return true;
        }

        // Development permissions are only shown to the user if they are already
        // granted to the app -- if we are installing an app and they are not
        // already granted, they will not be granted as part of the install.
        if (isDevelopment && wasGranted) {
            if (localLOGV) Log.i(TAG, "Special perm " + pInfo.name
                    + ": protlevel=0x" + Integer.toHexString(pInfo.protectionLevel));
            return true;
        }
        return false;
    }

对于权限等级为NORMAL，过滤。
对于Dangerous，是被要求展示的或者前面被授权过的都要显示。
对于是开发同时被授权过的，也要显示。（**这条还真没配置过！**） 
### 小结

通过上面的流程，我们知道关于权限的基本信息。

一个权限主要包含三个方面的信息：权限的名称，属于的权限组，保护级别。

1. **权限的名称**： 就是写在各个permission表情里面的名字，我们想申请的权限内容
1. **权限组：** 一个权限组是指把权限按照功能分成的不同的集合。 每一个权限组包含若干具体权限，
 如在SMS组中包含 `android.permission.SEND_SMS`，`android.permission.READ_SMS`等和短信相关权限（参考：frameworks/base/core/res /AndroidManifest.xml ）。
我们看到安卓对权限的分组是重构过的，V23版本是依据功能来划分，因为旧版吧发短信和打电话归到了COSTS_MONEY 组。这个6.0版本确实更新不少，对底层以前的很多工作都做了调整，背后一定有不少故事，毕竟整个项目那么大。
 
	    <permission-group android:name="android.permission-group.SMS"
        android:icon="@drawable/perm_group_sms"
        android:label="@string/permgrouplab_sms"
        android:description="@string/permgroupdesc_sms"
        android:priority="300" />

	    <!-- Allows an application to send SMS messages.
         <p>Protection level: dangerous
	    -->
	    <permission android:name="android.permission.SEND_SMS"
	        android:permissionGroup="android.permission-group.SMS"
	        android:label="@string/permlab_sendSms"
	        android:description="@string/permdesc_sendSms"
	        android:permissionFlags="costsMoney"
	        android:protectionLevel="dangerous" />
	     
	    <!-- Allows an application to read SMS messages.
	         <p>Protection level: dangerous
	    -->
	    <permission android:name="android.permission.READ_SMS"
	        android:permissionGroup="android.permission-group.SMS"
	        android:label="@string/permlab_readSms"
	        android:description="@string/permdesc_readSms"
	        android:protectionLevel="dangerous" />


2. **保护等级：**每个权限通过 protectionLevel 来标识保护级别：`normal`，`dangerous`，`signature` ，和额外的如 `system`，`development`，`appop` 等等(详情见android/content/pm/permissionInfo)。不同的保护级别代表了程序要使用此权限时的认证方式。对于`normal` 的权限只要申请了就可以用，而`dangerous` 的权限在安装时需用户确认才可以用； `signature`权限需要使用者的app和系统使用同一个数字证书。
 
Package 的权限信息主要通过在 AndroidManifest.xml 中通过一些标签来指定。如 `<permission>` 标签， `<permission-group>` 标签 `<permission-tree>` 等标签。
如果 package 需要申请使用某个权限，那么需要使用 `<uses-permission>` 标签来指定。通过上面的一段XML代码片段，想说下的是`permission-group`只是我们的`permission`的一个tag而已，用于逻辑分组，通过向 `<permission>` 元素的 permissionGroup 属性分配组名，将权限放入组中。而且并不是所有的Permission都有Permission-group，有的可能没有分组。这些没有分组的Permission也会自己指定label，以便告知用户这个权限的作用。`<permission-tree>` 元素为一组将在代码中定义的权限声明命名空间。

例如我们在使用JPush的时候，用到了permission表情，

	<permission
	        android:name="${applicationId}.permission.JPUSH_MESSAGE"
	        android:protectionLevel="signature" />
	        
他用到的level是signature的，所以会对用到的证书做验证，这也是我们需要申请认证注册得到KEY的原因。	        
另外当我们需要联网的时候，会加入intentn的权限，用到的是`uses-permission`标签

	<uses-permission android:name="android.permission.INTERNET" />

## 安装APK
 
通过上面的一圈内容，我们得到我们需要的界面内容，现在我们对我们得而安装后续内容做个了解。 

	public void onClick(View v) {
        if (v == mOk) {
            ...
            startInstall();
                
         }else if(v == mCancel) {
            // Cancel and finish
            setResult(RESULT_CANCELED);
            if (mSessionId != -1) {
                mInstaller.setPermissionsResult(mSessionId, false);
            }
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_CANCELLED_BY_USER);
            finish();
        } 
          ...
    }
 当我们点击右下角的确定安装按钮的时候，他会去调用`startInstall();`函数去开始安装我们的APK

### startInstall()

	private void startInstall() {
	
        // Start subactivity to actually install the application
        Intent newIntent = new Intent();
        newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO,
                mPkgInfo.applicationInfo);
        newIntent.setData(mPackageURI);
        newIntent.setClass(this, InstallAppProgress.class);
        newIntent.putExtra(InstallAppProgress.EXTRA_MANIFEST_DIGEST, mPkgDigest);
        newIntent.putExtra(
                InstallAppProgress.EXTRA_INSTALL_FLOW_ANALYTICS, mInstallFlowAnalytics);
        String installerPackageName = getIntent().getStringExtra(
                Intent.EXTRA_INSTALLER_PACKAGE_NAME);
        if (mOriginatingURI != null) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_URI, mOriginatingURI);
        }
        if (mReferrerURI != null) {
            newIntent.putExtra(Intent.EXTRA_REFERRER, mReferrerURI);
        }
        if (mOriginatingUid != VerificationParams.NO_UID) {
            newIntent.putExtra(Intent.EXTRA_ORIGINATING_UID, mOriginatingUid);
        }
        if (installerPackageName != null) {
            newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME,
                    installerPackageName);
        }
        if (getIntent().getBooleanExtra(Intent.EXTRA_RETURN_RESULT, false)) {
            newIntent.putExtra(Intent.EXTRA_RETURN_RESULT, true);
            newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
        }
        
        if(localLOGV) Log.i(TAG, "downloaded app uri="+mPackageURI);
        startActivity(newIntent);
        finish();
    }
    
 这一长串我们看到是交给InstallAppProgress这个类去处理的，我们去看下
在InstallAppProgress类的onCreate函数,对于scheme为`file`和`package`的是支持的。

	@Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        Intent intent = getIntent();
        mAppInfo = intent.getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mInstallFlowAnalytics = intent.getParcelableExtra(EXTRA_INSTALL_FLOW_ANALYTICS);
        mInstallFlowAnalytics.setContext(this);
        mPackageURI = intent.getData();

        final String scheme = mPackageURI.getScheme();
        if (scheme != null && !"file".equals(scheme) && !"package".equals(scheme)) {
            mInstallFlowAnalytics.setFlowFinished(
                    InstallFlowAnalytics.RESULT_FAILED_UNSUPPORTED_SCHEME);
            throw new IllegalArgumentException("unexpected scheme " + scheme);
        }

        initView();
    }

###  initView()

	 public void initView() {
	               
        PackageManager pm = getPackageManager();      
	    PackageInfo pi = pm.getPackageInfo(mAppInfo.packageName, 
	                 PackageManager.GET_UNINSTALLED_PACKAGES);
         if(pi != null) {
             installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;
         }
     
        ...//省略一堆界面find和设置操作
        
        PackageInstallObserver observer = new PackageInstallObserver();
		...    
        pm.installPackageWithVerificationAndEncryption(mPackageURI, observer, 
            installFlags,installerPackageName, verificationParams, null);        
        
    }

这里的PackageInstallObserver就是安装的观察者，用来回调更新界面的，也是一个binder

	class PackageInstallObserver extends IPackageInstallObserver.Stub {
	        public void packageInstalled(String packageName, int returnCode) {
	            Message msg = mHandler.obtainMessage(INSTALL_COMPLETE);
	            msg.arg1 = returnCode;
	            mHandler.sendMessage(msg);
	        }
	    }
然后pm是通过getPackageManger()获取，我们知道具体的实现是`ApplicationPackageManager`类。

### installPackageWithVerificationAndEncryption()

	@Override
    public void installPackageWithVerificationAndEncryption(Uri packageURI,
            PackageInstallObserver observer, int flags, String installerPackageName,
            VerificationParams verificationParams,
            ContainerEncryptionParams encryptionParams) {
            
        installCommon(packageURI, observer, flags, installerPackageName, 
			        verificationParams,encryptionParams);
    }
    
### installCommon()
	private void installCommon(Uri packageURI,PackageInstallObserver observer, 
				int flags, String installerPackageName,
	            VerificationParams verificationParams,
	             ContainerEncryptionParams encryptionParams) {
	             
			...
	
	        final String originPath = packageURI.getPath();
	        try {
	            mPM.installPackage(originPath, observer.getBinder(), flags, 
	            installerPackageName,verificationParams, null);
	            
	        } catch (RemoteException ignored) {
	        }
	    }	

饶了一个大圈，之后是靠PMS去安装包.

### PMS.installPackage()
	@Override
    public void installPackage(String originPath, IPackageInstallObserver2 observer,
            int installFlags, String installerPackageName, 
            VerificationParams verificationParams, String packageAbiOverride) {
            
        installPackageAsUser(originPath, observer, installFlags, installerPackageName,
                verificationParams, packageAbiOverride, UserHandle.getCallingUserId());
                
    }	
    
	@Override
    public void installPackageAsUser(String originPath, IPackageInstallObserver2 
		    observer,int installFlags, String installerPackageName, 
		    VerificationParams verificationParams,
            String packageAbiOverride, int userId) {
	        
        mContext.enforceCallingOrSelfPermission(
	       android.Manifest.permission.INSTALL_PACKAGES, 
	       null);

        final int callingUid = Binder.getCallingUid();
        enforceCrossUserPermission(callingUid, userId, true, true, 
						        "installPackageAsUser");

        if (isUserRestricted(userId, UserManager.DISALLOW_INSTALL_APPS)) {            
	         if (observer != null) {
	             observer.onPackageInstalled("", INSTALL_FAILED_USER_RESTRICTED, 
	             null, null);
	         }
             return;
        }

        if ((callingUid == Process.SHELL_UID) || (callingUid == Process.ROOT_UID)) {
            installFlags |= PackageManager.INSTALL_FROM_ADB;

        } else {
            // Caller holds INSTALL_PACKAGES permission, so we're less strict
            // about installerPackageName.

            installFlags &= ~PackageManager.INSTALL_FROM_ADB;
            installFlags &= ~PackageManager.INSTALL_ALL_USERS;
        }

        UserHandle user;
        if ((installFlags & PackageManager.INSTALL_ALL_USERS) != 0) {
            user = UserHandle.ALL;
        } else {
            user = new UserHandle(userId);
        }

        // Only system components can circumvent runtime permissions when installing.
        if ((installFlags & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0
                && mContext.checkCallingOrSelfPermission(Manifest.permission
                .INSTALL_GRANT_RUNTIME_PERMISSIONS)==PackageManager.PERMISSION_DENIED){
                
            throw new SecurityException("You need the "
                    + "android.permission.INSTALL_GRANT_RUNTIME_PERMISSIONS permission "
                    + "to use the PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS 
                    + flag");
                    
        }

        verificationParams.setInstallerUid(callingUid);

        final File originFile = new File(originPath);
        final OriginInfo origin = OriginInfo.fromUntrustedFile(originFile);

        final Message msg = mHandler.obtainMessage(INIT_COPY);
        msg.obj = new InstallParams(origin, null, observer, installFlags, 
							        installerPackageName,null, verificationParams, 
							        user, packageAbiOverride, null);
											        
        mHandler.sendMessage(msg);
    }

这里首先做一些权限的检查，并判断当前安装APK的user是否具有相应的权限。在安装APK的时候分为程序开发人员通过ADB安装和user通过网上下载安装，当通过ADB安装时，往往不需要对程序做验证，这就是INSTALL_FROM_ADB这个flag的作用。最后构造一个INIT_COPY的cmd，并带有InstallParams的message发给`PackageHandler`处理。

 然后我们去看下关于这消息的处理是怎样的


    void doHandleMessage(Message msg) {
	      switch (msg.what) {
	            case INIT_COPY: {
	                HandlerParams params = (HandlerParams) msg.obj;
	                int idx = mPendingInstalls.size();
	                ...
	                mPendingInstalls.add(idx, params);
	                 // Already bound to the service. Just make
	                 // sure we trigger off processing the first request.
	                 if (idx == 0) {
	                     mHandler.sendEmptyMessage(MCS_BOUND);
	                 }
	                 ...
	                 
		        case MCS_BOUND: {		                    
	                  HandlerParams params = mPendingInstalls.get(0);
	                  if (params != null) {
	                      if (params.startCopy()) { 
	                         ...
	                  }           

### startCopy()
跑了一圈消息后，通过调用的HandlerParams的startCopy开始复制程序。

	final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);

                if (++mRetries > MAX_RETRIES) {                     
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    handleServiceError();
                    return false;
                } else {
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }
            handleReturnCode();
            return res;
        }
这段代码看到，程序会尝试最多MAX_RETRIES即4次的copy尝试，超过就发送错误消息，放弃安装
现在让我们去看下那个handleStartCopy内容

### installParams.handleStartCopy()

	/*
         * Invoke remote method to get package information and install
         * location values. Override install location based on default
         * policy if needed and then create install arguments based
         * on the install location.
         */
        public void handleStartCopy() throws RemoteException {
            int ret = PackageManager.INSTALL_SUCCEEDED;

            // If we're already staged, we've firmly committed to an install location
            if (origin.staged) {
                if (origin.file != null) {
                    installFlags |= PackageManager.INSTALL_INTERNAL;
                    installFlags &= ~PackageManager.INSTALL_EXTERNAL;
                } else if (origin.cid != null) {
                    installFlags |= PackageManager.INSTALL_EXTERNAL;
                    installFlags &= ~PackageManager.INSTALL_INTERNAL;
                } else {
                    throw new IllegalStateException("Invalid stage location");
                }
            }

            final boolean onSd = (installFlags & PackageManager.INSTALL_EXTERNAL) != 0;
            final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;

            PackageInfoLite pkgLite = null;
			//STEP1,看安装位置标记是否要把apk同时装到内部存储和sd卡
			//这是什么鬼意思呢？如果都要就报错，难道还有这情况？
            if (onInt && onSd) {
                // Check if both bits are set.
                Slog.w(TAG, "Conflicting flags specified for installing on both internal and external");
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else {
            
            //STEP2. 调用MCS的getMinimalPackageInfo来得到apk的推荐安装位置，并检查是否能装。例如空间够不够，不够就清点空间，无效的apk文件等             
                pkgLite = mContainerService.getMinimalPackageInfo(origin.resolvedPath, installFlags,
                        packageAbiOverride);

                ...                    
                    if (pkgLite.recommendedInstallLocation
                            == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                        pkgLite.recommendedInstallLocation
                            = PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
                            //不够空间
                    }
                }
            }

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                int loc = pkgLite.recommendedInstallLocation;
                if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                    ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                    ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_APK;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_URI;
                } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                    ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
                } else {
                
                  //STEP3.如果前面过了，就开始调用这个安装
                    loc = installLocationPolicy(pkgLite);
                    if (loc == PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE) {
                        ret = PackageManager.INSTALL_FAILED_VERSION_DOWNGRADE;
                    }
                   ...
                }
            }

            final InstallArgs args = createInstallArgs(this);
            mArgs = args;

            ...
            //STEP4.  调用copyApk方法来完成apk的复制过程
            ret = args.copyApk(mContainerService, true);
            ...
        }

###  createInstallArgs()

	private InstallArgs createInstallArgs(InstallParams params) {
       if (params.move != null) {
           return new MoveInstallArgs(params);
       }else if(installOnExternalAsec(params.installFlags) || params.isForwardLocked()){
           return new AsecInstallArgs(params);
       } else {
           return new FileInstallArgs(params);
       }
    }
根据安装路径的不同会建不同的InstallArgs，AsecInstallArgs就是指安装在外部存储空间上；FileInstallArgs是指安装在内部存储空间。而这个MoveInstallArgs看起来像是挪到data目录的。

在第四步的时候，他调用了args.copyApk方法！我们分类看下

#### MoveInstallArgs.copyApk()

	int copyApk(IMediaContainerService imcs, boolean temp) {
            if (DEBUG_INSTALL) Slog.d(TAG, "Moving " + move.packageName + " from "
                    + move.fromUuid + " to " + move.toUuid);
            synchronized (mInstaller) {
                if (mInstaller.copyCompleteApp(move.fromUuid, move.toUuid, move.packageName,
                        move.dataAppName, move.appId, move.seinfo) != 0) {
                    return PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
                }
            }

            codeFile = new File(Environment.getDataAppDirectory(move.toUuid), move.dataAppName);
            resourceFile = codeFile;
            if (DEBUG_INSTALL) Slog.d(TAG, "codeFile after move is " + codeFile);

            return PackageManager.INSTALL_SUCCEEDED;
        }

#### AsecInstallArgs.copyApk 

	int copyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
            ...

            if (temp) {
                createCopyFile();
            }
             ...

            final String newMountPath = imcs.copyPackageToContainer(
                    origin.file.getAbsolutePath(), cid, getEncryptKey(), isExternalAsec(),
                    isFwdLocked(), deriveAbiOverride(abiOverride, null /* settings */));

            if (newMountPath != null) {
                setMountPath(newMountPath);
                return PackageManager.INSTALL_SUCCEEDED;
            } else {
                return PackageManager.INSTALL_FAILED_CONTAINER_ERROR;
            }
        }

		void createCopyFile() {
            cid = mInstallerService.allocateExternalStageCidLegacy();
        }

 createCopyFile()找到一个尚未被使用的目录名赋予给cid。
在copyApk方法中调用MCS的copyPackageToContainer方法完成真正的创建目录以及拷贝文件的操作。


####  FileInstallArgs.copyApk 
	int copyApk(IMediaContainerService imcs, boolean temp) throws RemoteException {
	    ...
         try {
             final File tempDir = mInstallerService.
                allocateStageDirLegacy(volumeUuid);
             //最后靠Environment.getDataAppDirectory(volumeUuid);
             //这句创建到data/app
             
             codeFile = tempDir;
             resourceFile = tempDir;
         } catch (IOException e) {
             Slog.w(TAG, "Failed to create copy file: " + e);
             return PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
         }

         final IParcelFileDescriptorFactory target = 
	           new IParcelFileDescriptorFactory.Stub() {		           
	           
             @Override
             public ParcelFileDescriptor open(String name, int mode){
             
                 if (!FileUtils.isValidExtFilename(name)) {
                     throw new IllegalArgumentException("Invalid filename: " + name);
                 }
                 try {
                     final File file = new File(codeFile, name);
                     final FileDescriptor fd = Os.open(file.getAbsolutePath(),
                             O_RDWR | O_CREAT, 0644);
                     Os.chmod(file.getAbsolutePath(), 0644);
                     return new ParcelFileDescriptor(fd);
                 } catch (ErrnoException e) {
                     throw new RemoteException("Failed to open: " + e.getMessage());
                 }
             }
         };

		//下面是一些复制操作，包括些lib等内容
         int ret = PackageManager.INSTALL_SUCCEEDED;
         ret = imcs.copyPackage(origin.file.getAbsolutePath(), target);
         if (ret != PackageManager.INSTALL_SUCCEEDED) {
             Slog.e(TAG, "Failed to copy package");
             return ret;
         }

         final File libraryRoot = new File(codeFile, LIB_DIR_NAME);
         NativeLibraryHelper.Handle handle = null;
         try {
             handle = NativeLibraryHelper.Handle.create(codeFile);
             ret = NativeLibraryHelper.copyNativeBinariesWithOverride(
             handle, libraryRoot,abiOverride);
             
         } catch (IOException e) {
             Slog.e(TAG, "Copying native libraries failed", e);
             ret = PackageManager.INSTALL_FAILED_INTERNAL_ERROR;
         } finally {
             IoUtils.closeQuietly(handle);
         }
         return ret;
     }
经过上面一堆的操作，我们的apk就成功的复制好了

现在我们需要会主线，回到InstallParams的startCopy方法，它中会调用handleReturnCode来处理拷贝的结果

 
 
### handleReturnCode()


	@Override
    void handleReturnCode() { 
        if (mArgs != null) {
            processPendingInstall(mArgs, mRet);
        }
    }
    
### processPendingInstall()

	private void processPendingInstall(final InstallArgs args, final int currentStatus){
	
        // Queue up an async operation since the package installation may take a little while.
        mHandler.post(new Runnable() {
            public void run() {
                mHandler.removeCallbacks(this);
                 // Result object to be returned
                PackageInstalledInfo res = new PackageInstalledInfo();
                res.returnCode = currentStatus;
                res.uid = -1;
                res.pkg = null;
                res.removedInfo = new PackageRemovedInfo();
                if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                    args.doPreInstall(res.returnCode);
                    synchronized (mInstallLock) {
                        installPackageLI(args, res);
                    }
                    args.doPostInstall(res.returnCode, res.uid);
                }
                ... 
        });
    }
    
我们看到他会先调用doPreInstall函数，这个名字让我想起了在DroidPlugi的时候，他的代码在安装HOOK的时候，都有类似的preInstall和doPostInstall的调用抽象方法。这里的doPreInstall背后是去清了下缓存。

然后这个installPackageLi则贼长！PMS里面的代码就是这样和AMS一个鬼，虽然重构了那么多次，还是这样，更何况还经常改
 
              
###  PMS.installPackageLI()

	private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
        final int installFlags = args.installFlags;
        final String installerPackageName = args.installerPackageName;
        final String volumeUuid = args.volumeUuid;
        final File tmpPackageFile = new File(args.getCodePath());
        ...
  
		// Retrieve PackageSettings and parse package
        final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
                | (forwardLocked ? PackageParser.PARSE_FORWARD_LOCK : 0)
                | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0);   
                   
        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);
        final PackageParser.Package pkg;
		//1. 开始解析我们的包
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);
  
        // Mark that we have an install time CPU ABI override.
        pkg.cpuAbiOverride = args.abiOverride;

		...
		//2. 收集签字信息和Manifest里面的内容
        try {
            pp.collectCertificates(pkg, parseFlags);
            pp.collectManifestDigest(pkg);
        } catch (PackageParserException e) {
            res.setError("Failed collect during installPackageLI", e);
            return;
        }

		 ...
		 
        // Get rid of all references to package scan path via parser.
        pp = null;
        String oldCodePath = null;
        boolean systemApp = false;
        synchronized (mPackages) {
            // Check if installing already existing package
	         ...//调过一些和如果已安装过的一些相关操作

            PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps != null) {
                if (DEBUG_INSTALL) Slog.d(TAG, "Existing package: " + ps);
				//3.对签名信息做判断，早期很多盗版应用进程改下签名加广告的
                // Quick sanity check that we're signed correctly if updating;
                // we'll check this again later when scanning, but we want to
                // bail early here before tripping over redefined permissions.                
                if (shouldCheckUpgradeKeySetLP(ps, scanFlags)) {
                    if (!checkUpgradeKeySetLP(ps, pkg)) {
                        res.setError(INSTALL_FAILED_UPDATE_INCOMPATIBLE, "Package "
                                +pkg.packageName + " upgrade keys do not match the "
                                +"previously installed version");
                                
                        return;
                    }
                } else {
                    try {
                        verifySignaturesLP(ps, pkg);
                    } catch (PackageManagerException e) {
                        res.setError(e.error, e.getMessage());
                        return;
                    }
                }

                oldCodePath = mSettings.mPackages.get(pkgName).codePathString;
                if (ps.pkg != null && ps.pkg.applicationInfo != null) {
                    systemApp = (ps.pkg.applicationInfo.flags &
                            ApplicationInfo.FLAG_SYSTEM) != 0;
                }
                res.origUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            }
            
			//4. 权限处理部分，为何这哥们不分开成多个函数呢？看着好累	
		     //Check whether the newly-scanned package
		     // wants to define an already-defined perm
            
            int N = pkg.permissions.size();
            for (int i = N-1; i >= 0; i--) {
                PackageParser.Permission perm = pkg.permissions.get(i);
                BasePermission bp = mSettings.mPermissions.get(perm.info.name);
                if (bp != null) {
                    // If the defining package is signed with our cert, it's okay.  This
                    // also includes the "updating the same package" case, of course.
                    // "updating same package" could also involve key-rotation.
                    final boolean sigsOk;
                    if (bp.sourcePackage.equals(pkg.packageName)
                            && (bp.packageSetting instanceof PackageSetting)
                            && (shouldCheckUpgradeKeySetLP((PackageSetting)
                             bp.packageSetting,scanFlags))) {
                        sigsOk = checkUpgradeKeySetLP((PackageSetting)
		                         bp.packageSetting, pkg);
                    } else {
                        sigsOk = compareSignatures(
		                        bp.packageSetting.signatures.mSignatures,
                                pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
                    }
                    if (!sigsOk) {
                        // If the owning package is the system itself, we log but allow
                        // install to proceed; we fail the install on all other                         
                        //permission redefinitions.
                        
                        if (!bp.sourcePackage.equals("android")) {
                            res.setError(INSTALL_FAILED_DUPLICATE_PERMISSION, "Package "
                              +pkg.packageName + " attempting to redeclare permission "
                           +perm.info.name + " already owned by " + bp.sourcePackage);
                           
                            res.origPermission = perm.info.name;
                            res.origPackage = bp.sourcePackage;
                            return;
                        } else {
                           ...      
                            pkg.permissions.remove(i);
                        }
                    }
                }
            }

        }
		//不允许升级的系统app安装到外部存储
        if (systemApp && onExternal) {
            // Disable updates to system apps on sdcard
            res.setError(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                    "Cannot install updates to system apps on sdcard");
            return;
        }
  
		//5.对包进行opt操作，调用performDexOpt，最终调用的还是Install的dexopt函数
	    ...
        else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
            // Enable SCAN_NO_DEX flag to skip dexopt at a later stage
            scanFlags |= SCAN_NO_DEX;
            try {
                derivePackageAbi(pkg, new File(pkg.codePath), args.abiOverride,
                        true /* extract libs */);
            } catch (PackageManagerException pme) {
                Slog.e(TAG, "Error deriving application ABI", pme);
                res.setError(INSTALL_FAILED_INTERNAL_ERROR, "Error deriving application 
			                ABI");
                return;
            }			 
            // Run dexopt before old package gets removed, 
            //to minimize time when app is unavailable
            int result = mPackageDexOptimizer
                    .performDexOpt(pkg, null /* instruction sets */, false /* forceDex */,
                            false /* defer */, false /* inclDependencies */);
                            
            if (result == PackageDexOptimizer.DEX_OPT_FAILED) {
                res.setError(INSTALL_FAILED_DEXOPT, "Dexopt failed for " + pkg.codePath);
                return;
            }
        }
        
        //6.Rename package into final resting place. All paths on the given
		//scanned package should be updated to reflect the rename.		
        if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
            res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
            return;
        }

        startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);

		//7.由于我们是安装，不是升级旧包。所以走下面那条路径
        if (replace) {
            replacePackageLI(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                    installerPackageName, volumeUuid, res);
        } else {
            installNewPackageLI(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                    args.user, installerPackageName, volumeUuid, res);
        }
        
        synchronized (mPackages) {
            final PackageSetting ps = mSettings.mPackages.get(pkgName);
            if (ps != null) {
                res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
            }
        }
    }              

#### PackageParse.parsePackage()

	public Package parsePackage(File packageFile, int flags){
        if (packageFile.isDirectory()) {
            return parseClusterPackage(packageFile, flags);
        } else {
            return parseMonolithicPackage(packageFile, flags);
        }
    }

居然还分了一个安装包和一堆安装包的情况。还真没想过有一堆cluster的情况

##### parseMonolithicPackage

	public Package parseMonolithicPackage(File apkFile, int flags){
        if (mOnlyCoreApps) {
            final PackageLite lite = parseMonolithicPackageLite(apkFile, flags);
            if (!lite.coreApp) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                        "Not a coreApp: " + apkFile);
            }
        }

        final AssetManager assets = new AssetManager();
        try {
            final Package pkg = parseBaseApk(apkFile, assets, flags);
            pkg.codePath = apkFile.getAbsolutePath();
            return pkg;
        } finally {
            IoUtils.closeQuietly(assets);
        }
    }

##### parseMonolithicPackageLite(

	private static PackageLite parseMonolithicPackageLite(File packageFile, int flags)
	            throws PackageParserException {
		        //人如其名，很lite，轻量，只搜集一些基本信息
		        //如包名，versionCode，安装位置等 
	        final ApkLite baseApk = parseApkLite(packageFile, flags);
	        
	        final String packagePath = packageFile.getAbsolutePath();
	        return new PackageLite(packagePath, baseApk, null, null, null);
	    }	

##### parseBaseApk()

	 private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
            
        final String apkPath = apkFile.getAbsolutePath();
		...

        final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

        Resources res = null;
        XmlResourceParser parser = null;
 
        res = new Resources(assets, mMetrics, null);
         assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                 Build.VERSION.RESOURCES_SDK_INT);
         parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

         final String[] outError = new String[1];
         
         final Package pkg = parseBaseApk(res, parser, flags, outError);
         ...
          pkg.volumeUuid = volumeUuid;
         pkg.baseCodePath = apkPath;
         pkg.mSignatures = null;

         return pkg;
		 ...
    }

这里最重要的一句就是parseBaseApk（）函数，他会对我们的AndroidManifest文件进行解析！
就不贴上来了，六百号，看着类，而且还有配套的函数做辅助，有个两千行了。
有不少标签是从没用过，也没见过的。

至今我对`DroidPlugin`里面包含的解析内容蛮有印象，作者为了兼容写了不少解析类!
在文章[源码探索系列30---插件包PackageManagerService/PMS](http://sanjay-f.github.io/2016/04/14/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9730---%E6%8F%92%E4%BB%B6%E5%8C%85PackageManagerService-PMS/)的PackageParser 类里面，就为了不同版本写了不同解析类。

  
  
#### installNewPackageLI()

	private void installNewPackageLI(PackageParser.Package pkg, int parseFlags,
	 int scanFlags,UserHandle user, String installerPackageName, String volumeUuid,
	 PackageInstalledInfo res) {
	 
        // Remember this for later, in case we need to rollback this install
        String pkgName = pkg.packageName;
                
        ...
        try {
            PackageParser.Package newPackage = scanPackageLI(pkg, parseFlags, scanFlags,
                    System.currentTimeMillis(), user);
            updateSettingsLI(newPackage, installerPackageName, volumeUuid, 
				            null, null, res, user); 
            if (res.returnCode != PackageManager.INSTALL_SUCCEEDED) {
				 //安装不成功就删除
                deletePackageLI(pkgName, UserHandle.ALL, false, null, null,
                        dataDirExists ? PackageManager.DELETE_KEEP_DATA : 0,
                                res.removedInfo, true);
            }
        } catch (PackageManagerException e) {
            res.setError("Package couldn't be installed in " + pkg.codePath, e);
        }
    }
    
1. scanPackageLI ,赫赫，整个scanPackageLI的长度一千两百行，看得我傻傻的，反正我看完大部分忘了。
主要内容为，调用该方法把新package的资源归入到`PMS`中，并创建一个`PackageSettings`对象，加入到Settings中的mPackages这个map中。
 
2. updateSettingsLI , 这个就短了不少，先调用Settings的writeLPr方法更新`packages.xml`文件，将新安装的package信息写到这个xml文件。`updatePermissionsLPw`，它用于给当前安装的APK分配权限，并把相应的gid号保存在PackageSetting或者SharedUserSetting的gids数组中。


 这一部分终于和我们文章的主题有关系了，看了前面那么多的内容！！！！
 终于看到与权限的内容啦！
 

####  updatePermissionsLPw

	private void updatePermissionsLPw(String changingPkg,
            PackageParser.Package pkgInfo, int flags) {
            
        // Make sure there are no dangling permission trees.
        Iterator<BasePermission> it = mSettings.mPermissionTrees.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            if (bp.packageSetting == null) {
                // We may not yet have parsed the package, so just see if
                // we still know about its settings.
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }
            if (bp.packageSetting == null) {
                Slog.w(TAG, "Removing dangling permission tree: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();
            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) {
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission tree: " + bp.name
                            + " from package " + bp.sourcePackage);
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        // Make sure all dynamic permissions have been assigned to a package,
        // and make sure there are no dangling permissions.
        it = mSettings.mPermissions.values().iterator();
        while (it.hasNext()) {
            final BasePermission bp = it.next();
            if (bp.type == BasePermission.TYPE_DYNAMIC) {
                if (DEBUG_SETTINGS) Log.v(TAG, "Dynamic permission: name="
                        + bp.name + " pkg=" + bp.sourcePackage
                        + " info=" + bp.pendingInfo);
                if (bp.packageSetting == null && bp.pendingInfo != null) {
                    final BasePermission tree = findPermissionTreeLP(bp.name);
                    if (tree != null && tree.perm != null) {
                        bp.packageSetting = tree.packageSetting;
                        bp.perm = new PackageParser.Permission(tree.perm.owner,
                                new PermissionInfo(bp.pendingInfo));
                        bp.perm.info.packageName = tree.perm.info.packageName;
                        bp.perm.info.name = bp.name;
                        bp.uid = tree.uid;
                    }
                }
            }
            if (bp.packageSetting == null) {
                // We may not yet have parsed the package, so just see if
                // we still know about its settings.
                bp.packageSetting = mSettings.mPackages.get(bp.sourcePackage);
            }
            if (bp.packageSetting == null) {
                Slog.w(TAG, "Removing dangling permission: " + bp.name
                        + " from package " + bp.sourcePackage);
                it.remove();
            } else if (changingPkg != null && changingPkg.equals(bp.sourcePackage)) {
                if (pkgInfo == null || !hasPermission(pkgInfo, bp.name)) {
                    Slog.i(TAG, "Removing old permission: " + bp.name
                            + " from package " + bp.sourcePackage);
                    flags |= UPDATE_PERMISSIONS_ALL;
                    it.remove();
                }
            }
        }

        // Now update the permissions for all packages, in particular
        // replace the granted permissions of the system packages.
        if ((flags&UPDATE_PERMISSIONS_ALL) != 0) {
            for (PackageParser.Package pkg : mPackages.values()) {
                if (pkg != pkgInfo) {
                    grantPermissionsLPw(pkg, (flags&UPDATE_PERMISSIONS_REPLACE_ALL) != 0,
                            changingPkg);
                }
            }
        }

        if (pkgInfo != null) {
            grantPermissionsLPw(pkgInfo, (flags&UPDATE_PERMISSIONS_REPLACE_PKG) != 0, changingPkg);
        }
    }



## apk的解析过程

在上面的安装完成后，系统接着会对apk进行dex提取和解析，在PMS内部有一个`AppDirObserver`类，它的作用是应用目录观察者，它时刻观察着应用目录/data/app/，当目录内部结构改变的时候（创建文件和删除文件）它会做出相应行为。

### 思考：监听app的卸载
以前有找过一些方案，如何监听自己的app被卸载，然后跳出一张问卷出来，以收集用户的反馈。
那时查资料时候，一个方案就是监听data目录，还有去监听系统log的方案.。参考：[360 手机卫士Android 版是如何做到在卸载完成后弹出一个网页的？](https://www.zhihu.com/question/20773194)
读log的方案似乎后来被安卓堵了，现在没有这需求，就没再跟进过这个问题了。


 


# 检验


权限验证，一些同能可能涉及到对权限的要求，例如写文件和联网等。
在进行请求前，我们可以自己先检验下，对此安卓提供的一个接口是：
	 context.checkCallingOrSelfPermission(String permission) ;
利用这接口，我们可以判断自己是否拥有这个权限，避免有时候被用户禁止了权限导致的一些不必要的bug问题。

顺着这个入口，我们看下背后是怎么做检测的。
当我们再Activity直接调用这个接口的时候，背后是调用了ContextWrapper类去处理，然后对这个类我们也很熟悉了，这个mBase就是ContextImpl类。

	@Override
    public int checkCallingOrSelfPermission(String permission) {
        return mBase.checkCallingOrSelfPermission(permission);
    }

## ContextImpl.checkCallingOrSelfPermission()

	@Override
    public int checkCallingOrSelfPermission(String permission) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        return checkPermission(permission, Binder.getCallingPid(),
                Binder.getCallingUid());
    }

## checkPermission()

	@Override
    public int checkPermission(String permission, int pid, int uid) {
        if (permission == null) {
            throw new IllegalArgumentException("permission is null");
        }

        try {
            return ActivityManagerNative.getDefault().checkPermission(
                    permission, pid, uid);
        } catch (RemoteException e) {
            return PackageManager.PERMISSION_DENIED;
        }
    }

在这里我们看到，他最后是通过AMS去做检测的。对于ActivityManagerNative这句话我们也是很熟悉了。
通过传递的pid，uid和我们想知道的permission来做判断，估计就是查下表，看对应的是否有这个权限。
  

## AMS.checkPermission()

	@Override
    public int checkPermission(String permission, int pid, int uid) {
        if (permission == null) {
            return PackageManager.PERMISSION_DENIED;
        }
        return checkComponentPermission(permission, pid, uid, -1, true);
    }
## checkComponentPermission()

	 int checkComponentPermission(String permission, int pid, int uid,
	            int owningUid, boolean exported) {
	            
        if (pid == MY_PID) {
            return PackageManager.PERMISSION_GRANTED;
        }
        return ActivityManager.checkComponentPermission(permission, uid,
                owningUid, exported);
    }

对于PID是自身的，都直接是通过的，非本人则需要再看看。

## ActivityManager.checkComponentPermission(

	public static int checkComponentPermission(String permission, int uid,
            int owningUid, boolean exported) {
            
        // Root, system server get to do everything.
        final int appId = UserHandle.getAppId(uid);
        if (appId == Process.ROOT_UID || appId == Process.SYSTEM_UID) {
            return PackageManager.PERMISSION_GRANTED;
        }
        
         //对于隔离的进程，都没有任何权限的，这要求PID为99000--99999范围的
        // Isolated processes don't get any permissions.
        if (UserHandle.isIsolated(uid)) {
            return PackageManager.PERMISSION_DENIED;
        }
        
        // If there is a uid that owns whatever is being accessed, it has
        // blanket access to it regardless of the permissions it requires.
        if (owningUid >= 0 && UserHandle.isSameApp(uid, owningUid)) {
            return PackageManager.PERMISSION_GRANTED;
        }

		...        
  
        return AppGlobals.getPackageManager()
                .checkUidPermission(permission, uid);
                
		... 
    }

我们看开头官方的注视提到的，对于是root和systemServer的都直接通过的，所以我们常说获取系统最高权限就是那个root，至于那个systemServer，他是一个系统开机自动的进程，**我们是否可以把自己的伪装成systemServer进程从而提高自己的权限,又或者通过hook等方式来修改自己传出去的uid，从而提高权限** 

做了一些检测后，就是去PMS调用对应的检测
 
## PMS.checkUidPermission()

    @Override
    public int checkUidPermission(String permName, int uid) {
        final int userId = UserHandle.getUserId(uid);

        if (!sUserManager.exists(userId)) {
            return PackageManager.PERMISSION_DENIED;
        }

        synchronized (mPackages) {
        
            //STEP 1
            Object obj = mSettings.getUserIdLPr(UserHandle.getAppId(uid));
            if (obj != null) {
                final SettingBase ps = (SettingBase) obj;
                final PermissionsState permissionsState = ps.getPermissionsState();
                if (permissionsState.hasPermission(permName, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
                // Special case: ACCESS_FINE_LOCATION permission includes ACCESS_COARSE_LOCATION
                if (Manifest.permission.ACCESS_COARSE_LOCATION.equals(permName) && permissionsState
                        .hasPermission(Manifest.permission.ACCESS_FINE_LOCATION, userId)) {
                    return PackageManager.PERMISSION_GRANTED;
                }
            } else {
            
                //STEP 2
                ArraySet<String> perms = mSystemPermissions.get(uid);
                if (perms != null) {
                    if (perms.contains(permName)) {
                        return PackageManager.PERMISSION_GRANTED;
                    }
                    if (Manifest.permission.ACCESS_COARSE_LOCATION.equals(permName) && perms
                            .contains(Manifest.permission.ACCESS_FINE_LOCATION)) {
                        return PackageManager.PERMISSION_GRANTED;
                    }
                }
            }
        }

        return PackageManager.PERMISSION_DENIED;
    }

1. STEP1 , 先调用getUserIdLPr，同PMS的Setting.mUserIds数组中根据`uid` 查找权限列表。找到则表示有相应的权限，接着再根据传过来的参数坐下判断，看是否有对应的，
2. STEP2 , 如没有找到，则去PMS的mSystemPermissions 中找。
这些信息是启动时从 `/system/etc/permissions/platform.xml` 中读取的。
这里记录了一些系统级的应用的 uid 对应的 permission 。 

### Setting.getUserIdLPr()

	public Object getUserIdLPr(int uid) {
	
       if (uid >= Process.FIRST_APPLICATION_UID) {
           final int N = mUserIds.size();
           final int index = uid - Process.FIRST_APPLICATION_UID;
           return index < N ? mUserIds.get(index) : null;
       } else {
           return mOtherUserIds.get(uid);
       }
       
    }

# 后记

不知不觉又是凌晨，每次写到深夜，都有一种感觉。
那就是写完很开心，但旁边空无一人。

# REF:

1. [Android 安全架构及权限控制机制剖析](http://www.ibm.com/developerworks/cn/opensource/os-cn-android-sec/) 
2. [Android的权限机制总结](http://dengzhangtao.iteye.com/blog/1990138)
3. [谷歌官方文档关于permission等标签的说明](https://developer.android.com/guide/topics/manifest/permission-element.html)
4. [Android内核解读-应用的安装过程](http://blog.csdn.net/singwhatiwanna/article/details/19578947?utm_source=tuicool&utm_medium=referral) 
http://blog.csdn.net/lilian0118/article/details/25792601