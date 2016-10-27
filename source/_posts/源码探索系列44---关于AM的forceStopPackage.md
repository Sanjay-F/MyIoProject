title: 源码探索系列44---关于AM的forceStopPackage
date: 2016-09-27 23:25
tags: [android,AM,forceStopPackage]
categories: android

------

最近想寻找什么方法能够彻底的关闭别人的app！
这听起来就很流氓，居然是关闭别人的，而且还要是关闭 `前台进程（Foreground process)`，这想法真大胆。查了下目前的资料，

1. 类似各种xx大师，管家的一键加速功能，这种根据各个进程adj评分去关闭方案，似乎对于前台进程是鞭长莫及。
按照我们熟悉的

2. forceStopPackage  这个是AM的隐藏方法，实际上由于签名，uid等限制条件，并不能真的关闭第三方，别人的程序。另外AM还有别的方法听起来像可以关闭别人进程的，如`killBackgroundProcesses`，`removeTask`等。

3.  Process.killProcess()
在process这个类里面，由几个方法可以调用去关闭，其中有一个的注释很有意思，打算贴出来。

		 /**
	     * @hide
	     * Private impl for avoiding a log message...  DO NOT USE without doing
	     * your own log, or the Android Illuminati （安卓光明会）will find you some night and beat you up.
	     * 这段注释很调皮😝，特拿出来哈
		 */
	    public static final void killProcessQuiet(int pid) {
	        sendSignalQuiet(pid, SIGNAL_KILL);
	    }

	    public static final native void sendSignalQuiet(int pid, int signal);


对于上面的所有，这篇文章主要记录`forceStopPackage（）`背后执行的内容，别的方法之后再考虑写。

 <!--more-->

 
# 起航
api:23


## AM.forceStopPackage

	 /**
     * @see #forceStopPackageAsUser(String, int)
     * @hide
     */
    public void forceStopPackage(String packageName) {
        forceStopPackageAsUser(packageName, UserHandle.myUserId());
    }
	
我们调用这个方法，背后是去掉哟功能asUser这个。前者参数为我们想关闭的包名，后者是当前调用者的uid。
	
    /**
     * Have the system perform a force stop of everything associated with
     * the given application package.  All processes that share its uid
     * will be killed, all services it has running stopped, all activities
     * removed, etc.  In addition, a {@link Intent#ACTION_PACKAGE_RESTARTED}
     * broadcast will be sent, so that any of its registered alarms can
     * be stopped, notifications removed, etc.
     * 看注释，我们知道这个是非常强效的关闭第三方app，很彻底的，包括他的service，闹钟等都关闭。对应我们需要声明FORCE_STOP_PACKAGES这个权限。
     * 
     * <p>You must hold the permission
     * {@link android.Manifest.permission#FORCE_STOP_PACKAGES} to be able to
     * call this method.
     *
     * @param packageName The name of the package to be stopped.
     * @param userId The user for which the running package is to be stopped.
     *
     * @hide This is not available to third party applications due to
     * it allowing them to break other applications by stopping their
     * services, removing their alarms, etc.
     */
    public void forceStopPackageAsUser(String packageName, int userId) {
        try {
            ActivityManagerNative.getDefault().forceStopPackage(packageName, userId);
            //然后这个方法实际是跑到ams去调用的
        } catch (RemoteException e) {
        }
    }

## ams.forceStopPackage()
	
	@Override
    public void forceStopPackage(final String packageName, int userId) {
	   //开头会检验下你是否声明了FORCE_STOP_PACKAGES权限，没有就送你一个se的异常
	   //整个检验不是我们这次关注的核心就不细说了。
	   //结论就是跑去pms.checkUidPermission(permName,uid)
	   
        if (checkCallingPermission(
        android.Manifest.permission.FORCE_STOP_PACKAGES)
                != PackageManager.PERMISSION_GRANTED) {
                
            String msg = "Permission Denial: forceStopPackage() from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " requires " + android.Manifest.permission.FORCE_STOP_PACKAGES;
	            throw new SecurityException(msg);
        }

		//准备参数
        final int callingPid = Binder.getCallingPid();
        userId = handleIncomingUser(callingPid, Binder.getCallingUid(),
                userId, true, ALLOW_FULL_ONLY, "forceStopPackage", null);
        long callingId = Binder.clearCallingIdentity();
        
        try {
        
            IPackageManager pm = AppGlobals.getPackageManager();
            synchronized(this) {
            
                int[] users = userId == UserHandle.USER_ALL
                        ? getUsersLocked() : new int[] { userId };
                 //   /** @hide A user id to indicate all users on the device */
                //    public static final int USER_ALL = -1;
                //这句看起来让人觉得云里雾里，作为固定正数值-1,好像这个三元判断优点不对劲啊。

                for (int user : users) {
                    int pkgUid = -1;
                    pkgUid = pm.getPackageUid(packageName, user);                    
                    if (pkgUid == -1) {
                        Slog.w(TAG, "Invalid packageName: " + packageName);
                        continue;
                    }                    
                    //然后就是告诉pms去表记包为停止状态，
                    pm.setPackageStoppedState(packageName, true, user);                                        
                    if (isUserRunningLocked(user, false)) {
                        forceStopPackageLocked(packageName, pkgUid, "from pid " + callingPid);
                    }
                    
                }
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }

### pms.getPackageUid

	@Override
    public int getPackageUid(String packageName, int userId) {
        if (!sUserManager.exists(userId)) return -1;
        enforceCrossUserPermission(Binder.getCallingUid(), userId, false, false, "get package uid");

        // reader
        synchronized (mPackages) {
            
            PackageParser.Package p = mPackages.get(packageName);
            if(p != null) {
                return UserHandle.getUid(userId, p.applicationInfo.uid);
                //如果包存在，拿包的uid和当前用户的uid出来去
            }
            
            PackageSetting ps = mSettings.mPackages.get(packageName);
            if((ps == null) || (ps.pkg == null)
	             || (ps.pkg.applicationInfo == null)) {
	             
                return -1;
            }
            p = ps.pkg;
            return p != null ? 
		            UserHandle.getUid(userId, p.applicationInfo.uid) 
		            : -1;
        }
    }    


### getUid
		
		 /**
     * Returns the uid that is composed from the userId and the appId.
     * @hide
     */
    public static final int getUid(int userId, int appId) {
        if (MU_ENABLED) {
            return userId * PER_USER_RANGE + (appId % PER_USER_RANGE);
        } else {
            return appId;
        }
    }


## pms.setPackageStoppedState

	@Override
    public void setPackageStoppedState(String packageName, boolean stopped, int userId) {
    
        if (!sUserManager.exists(userId)) return;
        final int uid = Binder.getCallingUid();
        final int permission = mContext.checkCallingOrSelfPermission(
                android.Manifest.permission.CHANGE_COMPONENT_ENABLED_STATE);
                
        final boolean allowedByPermission = (permission == PackageManager.PERMISSION_GRANTED);
        
        enforceCrossUserPermission(uid, userId, true, true, "stop package");
        
        // writer
        synchronized (mPackages) {
            if (mSettings.setPackageStoppedStateLPw(this, packageName, stopped,
                    allowedByPermission, uid, userId)) {
                    
                scheduleWritePackageRestrictionsLocked(userId);
            }
        }
    }

关于这个uid需要说明下的是，一般的非系统级别的，除了uid一样外，两个包还需要一样的签名。

> The name of a Linux user ID that will be shared with other
> applications. By default, Android assigns each application its own
> unique user ID. However, if this attribute is set to the same value
> for two or more applications, they will all share the same ID —
> provided that they are also `signed by the same certificate`.
> Application with the same user ID can access each other's data and, if
> desired, run in the same process.



对于上面那个函数，我们不再深究下去，看下它调用的另外一个函数

## forceStopPackageLocked

	 private void forceStopPackageLocked(final String packageName, int uid, String reason) {
	 
	        forceStopPackageLocked(packageName, UserHandle.getAppId(uid), false,	        
			     false, true, false, false, UserHandle.getUserId(uid), reason);
			     ／／参数真多
	                
	        Intent intent = new Intent(Intent.ACTION_PACKAGE_RESTARTED,
	                Uri.fromParts("package", packageName, null));
	        if (!mProcessesReady) {
	            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
	                    | Intent.FLAG_RECEIVER_FOREGROUND);
	        }
	        intent.putExtra(Intent.EXTRA_UID, uid);
	        intent.putExtra(Intent.EXTRA_USER_HANDLE, UserHandle.getUserId(uid));
	        broadcastIntentLocked(null, null, intent,
	                null, null, 0, null, null, null, AppOpsManager.OP_NONE,
	                null, false, false, MY_PID, Process.SYSTEM_UID, 
	                UserHandle.getUserId(uid));	                
	    }




	private final boolean forceStopPackageLocked(String packageName, int appId,
            boolean callerWillRestart, boolean purgeCache, boolean doit,
            boolean evenPersistent, boolean uninstalling, int userId, String reason) {
        int i;

	   ...

		if (doit) {//doit为真
		
            final ArrayMap<String, SparseArray<Long>> pmap = mProcessCrashTimes.getMap();
            
            for (int ip = pmap.size() - 1; ip >= 0; ip--) {
                SparseArray<Long> ba = pmap.valueAt(ip);
                for (i = ba.size() - 1; i >= 0; i--) {
                    boolean remove = false;
                    final int entUid = ba.keyAt(i);
                    if (packageName != null) {
                        if (userId == UserHandle.USER_ALL) {
                            if (UserHandle.getAppId(entUid) == appId) {
                                remove = true;
                            }
                        } else {
                            if (entUid == UserHandle.getUid(userId, appId)) {
                                remove = true;
                            }
                        }
                        
                    } else if (UserHandle.getUserId(entUid) == userId) {
                        remove = true;
                    }
                    if (remove) {//找到记录就移除
                        ba.removeAt(i);
                    }
                }
                if (ba.size() == 0) {
                    pmap.removeAt(ip);
                }
            }
        }

        boolean didSomething = killPackageProcessesLocked(packageName, appId, 
        userId,-100, callerWillRestart, true, doit, evenPersistent,
        packageName == null ? ("stop user " + userId) : ("stop " + packageName));
         //这哥们真实节约啊，还弄个三元操作到参数里面去
 
        if (mStackSupervisor.finishDisabledPackageActivitiesLocked(
                packageName, null, doit, evenPersistent, userId)) {
            if (!doit) {
                return true;
            }
            didSomething = true;
        }

		//关闭service
        if (mServices.bringDownDisabledPackageServicesLocked(
                packageName, null, userId, evenPersistent, doit)) {
            if (!doit) {
                return true;
            }
            didSomething = true;
        }

        if (packageName == null) {
            // Remove all sticky broadcasts from this user.
            mStickyBroadcasts.remove(userId);
        }

		//清空contentProvider
        ArrayList<ContentProviderRecord> providers = new ArrayList<>();
        if (mProviderMap.collectPackageProvidersLocked(packageName, null, doit, evenPersistent,
                userId, providers)) {
            if (!doit) {
                return true;
            }
            didSomething = true;
        }
        for (i = providers.size() - 1; i >= 0; i--) {
            removeDyingProviderLocked(null, providers.get(i), true);
        }

        // Remove transient permissions granted from/to this package/user
        removeUriPermissionsForPackageLocked(packageName, userId, false);

		//关广播接收
        if (doit) {
            for (i = mBroadcastQueues.length - 1; i >= 0; i--) {
                didSomething |= mBroadcastQueues[i].cleanupDisabledPackageReceiversLocked(
                        packageName, null, userId, doit);
            }
        }
        
		//删除pending的intent
        if (packageName == null || uninstalling) {
            if (mIntentSenderRecords.size() > 0) {
                Iterator<WeakReference<PendingIntentRecord>> it
                        = mIntentSenderRecords.values().iterator();
                while (it.hasNext()) {
                    WeakReference<PendingIntentRecord> wpir = it.next();
                    if (wpir == null) {
                        it.remove();
                        continue;
                    }
                    PendingIntentRecord pir = wpir.get();
                    if (pir == null) {
                        it.remove();
                        continue;
                    }
                    if (packageName == null) {
                        // Stopping user, remove all objects for the user.
                        if (pir.key.userId != userId) {
                            // Not the same user, skip it.
                            continue;
                        }
                    } else {
                        if (UserHandle.getAppId(pir.uid) != appId) {
                            // Different app id, skip it.
                            continue;
                        }
                        if (userId != UserHandle.USER_ALL && 
	                        pir.key.userId != userId) {
                            // Different user, skip it.
                            continue;
                        }
                        
                        if (!pir.key.packageName.equals(packageName)) {
                            // Different package, skip it.
                            continue;
                        }
                    }
                    if (!doit) {
                        return true;
                    }
                    didSomething = true;
                    it.remove();
                    pir.canceled = true;
                    if (pir.key.activity != null && 
	                    pir.key.activity.pendingResults != null) {
	                    
                        pir.key.activity.pendingResults.remove(pir.ref);
                    }
                }
            }
        }

        if (doit) {
            if (purgeCache && packageName != null) {
                AttributeCache ac = AttributeCache.instance();
                if (ac != null) {
                    ac.removePackage(packageName);
                }
            }
            if (mBooted) {
                mStackSupervisor.resumeTopActivitiesLocked();
                mStackSupervisor.scheduleIdleLocked();
            }
        }

        return didSomething;
    }


未完待续。。。


最近在android weekly 看到一篇类似的文章，贴下地址，写得挺好的
[Android 进程绝杀技--forceStop]{http://www.diycode.cc/topics/374}
