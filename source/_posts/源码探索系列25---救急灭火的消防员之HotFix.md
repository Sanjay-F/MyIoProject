
title: 源码探索系列25---救急灭火的消防员之插件&HotFix
date: 2016-03-05 23:33:46
tags: [android,源码,HotFix]
categories: android

------------------------------------------


![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/hotfix_201832-120i022461493.jpg)

在插件化和热修复方面，应该算是解决了不少燃眉问题了。

目前开源的有：
360的[DroidPlugin](https://github.com/Qihoo360/DroidPlugin)，
阿里的[AndFix](https://github.com/alibaba/AndFix)，[Dexposed](https://github.com/alibaba/dexposed)
主席的[dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)
点评的[Android Dynamic Loader](https://github.com/mmin18/AndroidDynamicLoader)， 
 
现在我们来看看它们的原理和各自的优缺点吧。

<!--more-->

# 方案介绍

> 原理无不是“移梁换柱，欺上瞒下“的工作。
 

## DroidPlugin 
 
版本：708df95

简述：360这个DroidPlugin使用的动态代理的方案，为了让插件中的方法能够正确的调用，把所有的方法都Hook了，然后自己写对应的Handle类来接管。另外为了兼容，我们看到作者是费劲了心机啊，里面包含的吐槽那是看着就明白一个兼容安卓机的程序员的心酸啊，哈。根据在PluginApplication 里面的写的创建日期为14年底，到八月份开源，应该背后也测了不少bug。

### 起航 
介绍：
整个项目还是挺大的，还有别项目要说，我们就来简单快速的看下。首先我们从官方介绍的安装apk插件的入口开始--installPackage。 

	PluginManager.getInstance().installPackage(String filepath, int flags);

对于这个接口的调用

	private IPluginManager mPluginManager;
	
	public int installPackage(String filepath, int flags) throws RemoteException {
 
            if (mPluginManager != null) {
                int result = mPluginManager.installPackage(filepath, flags);
                Log.w(TAG, String.format("%s install result %d", filepath, result));
                return result;
            } else {
                Log.w(TAG, "Plugin Package Manager Service not be connect");
            }
         ...
        return -1;
    }
 我们看到背后是交给了mPluginManger去干活的，他背后实现是IPluginManagerImpl
 
	 public interface IPluginManager extends android.os.IInterface
	 public class IPluginManagerImpl extends IPluginManager.Stub 
    
接着我们去看下真-干活着-内容   , 有点长，我们慢慢看下

	public int installPackage(String filepath, int flags) throws RemoteException {
        //install plugin
        String apkfile = null;
	     ...
           PackageManager pm = mContext.getPackageManager();
           PackageInfo info = pm.getPackageArchiveInfo(filepath, 0);
           if (info == null) {
               return PackageManagerCompat.INSTALL_FAILED_INVALID_APK;
           }

           apkfile = PluginDirHelper.getPluginApkFile(mContext, info.packageName);
			
           if ((flags & PackageManagerCompat.INSTALL_REPLACE_EXISTING) != 0) {
               //下面是安装插件的内容，先停止，有缓存就删除
               forceStopPackage(info.packageName);
               if (mPluginCache.containsKey(info.packageName)) {
                   deleteApplicationCacheFiles(info.packageName, null);
               }
               new File(apkfile).delete();
               Utils.copyFile(filepath, apkfile);
               PluginPackageParser parser = new PluginPackageParser(mContext, 
										               new File(apkfile));
               parser.collectCertificates(0);
               PackageInfo pkgInfo = parser.getPackageInfo(
							               PackageManager.GET_PERMISSIONS 											       
							               | PackageManager.GET_SIGNATURES);
				//对程序的权限请求做下判断，宿主如果没有的就报错。			               
               if (pkgInfo != null && pkgInfo.requestedPermissions != null && 
		          pkgInfo.requestedPermissions.length > 0) {
                   for (String requestedPermission : pkgInfo.requestedPermissions) {
                     boolean b = false;
                     b = pm.getPermissionInfo(requestedPermission, 0) != null; 
                     if (!mHostRequestedPermission.contains(requestedPermission) && b) {
                           Log.e(TAG, "No Permission %s", requestedPermission);
                           new File(apkfile).delete();
                           return PluginManager.INSTALL_FAILED_NO_REQUESTEDPERMISSION;
                       }
                   }
               }
                saveSignatures(pkgInfo);//保存签名，没什么好说的
                copyNativeLibs(mContext, apkfile, parser.getApplicationInfo(0));//复制插件中的native库
                //Dex文件，使用DexClassLoader动态的载入类代码
                dexOpt(mContext, apkfile, parser);
                
                mPluginCache.put(parser.getPackageName(), parser);
                //回调通知
                mActivityManagerService.onPkgInstalled(mPluginCache, parser, 
								                parser.getPackageName());
								                
                sendInstalledBroadcast(info.packageName);
                //ok，安装完
                return PackageManagerCompat.INSTALL_SUCCEEDED;
                
            } else {
            
                if (mPluginCache.containsKey(info.packageName)) {
                    return PackageManagerCompat.INSTALL_FAILED_ALREADY_EXISTS;
                } else {
                   //啊.....下面的内容....居然和上面的一样...
                   //...估计是直接CV大法....
                   //...就是安装更新...能否把上面的封装到一个函数去...
                   //...我的强迫症...
                    forceStopPackage(info.packageName);
                    new File(apkfile).delete();
                    Utils.copyFile(filepath, apkfile);
                    PluginPackageParser parser = new PluginPackageParser(mContext, new 
								                   File(apkfile));
                    parser.collectCertificates(0);
                    PackageInfo pkgInfo =parser.getPackageInfo(
					                    PackageManager.GET_PERMISSIONS
				                       | PackageManager.GET_SIGNATURES);
                    if (pkgInfo != null && pkgInfo.requestedPermissions != null && 
				      pkgInfo.requestedPermissions.length > 0) {
                       for (String requestedPermission : pkgInfo.requestedPermissions) {
                       
                        boolean b = false;  
                        b = pm.getPermissionInfo(requestedPermission, 0) != null;
                        if (!mHostRequestedPermission.contains(requestedPermission)
                            &&b) {
                            
                              Log.e(TAG, "No Permission %s", requestedPermission);
                              new File(apkfile).delete();
                              return PluginManager.
                              INSTALL_FAILED_NO_REQUESTEDPERMISSION;
                          }
                      }
                    }
                    
                   saveSignatures(pkgInfo);
                   copyNativeLibs(mContext, apkfile, parser.getApplicationInfo(0));
                   dexOpt(mContext, apkfile, parser);
                   mPluginCache.put(parser.getPackageName(), parser);
                   mActivityManagerService.onPkgInstalled(mPluginCache, parser, 
							                   parser.getPackageName());
                   sendInstalledBroadcast(info.packageName);
                   return PackageManagerCompat.INSTALL_SUCCEEDED;
               }
           }
       ...
    }
 
	
	private void dexOpt(Context hostContext, String apkfile, PluginPackageParser parser) 
	throws Exception {
			
	        String packageName = parser.getPackageName();
	        String optimizedDirectory = PluginDirHelper.
						        getPluginDalvikCacheDir(hostContext, packageName);
	        String libraryPath = PluginDirHelper.getPluginNativeLibraryDir(
										        hostContext, packageName);
										        
			//单独挑这个方法出来说，是想告诉在细心看的你一个彩蛋，就在这个类里面有一个叫
		    // 《适配安卓机的心酸故事》，有兴趣就自己去看下，这里不贴出来,哈哈 
	        ClassLoader classloader = new PluginClassLoader(apkfile, optimizedDirectory, 
							        libraryPath, ClassLoader.getSystemClassLoader());							         
			
			// 里面的注释还有很多有趣的，例如另外一个地方的一段 
		    //
			//    // 方式2
	        //    if (iam1 == iam2) {
	        //        //这段代码是废的，没啥用，写这里只是不想改而已。
	        //        FieldUtils.writeField(obj, "mInstance", object);
	        //   }							        
	        
			...							        
	 }

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/hotfix_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160305201106.jpg)    

结合作者的PPT，四大组件用的是先注册占坑方式。 
### Hook 的内容
 
前面有提到为了保证正确，作者hook了所有的方法，具体Hook内容在Hook包里面的`HookFactory`里面


	public final void installHook(Context context, ClassLoader classLoader)
	 throws Throwable {
	 
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

看到这里，我们对整体的方案应该有个大概的了解了。具体的细节就不再贴了。有兴趣可以看下他们的源码。
 
 
## AndFix
版本： c68d981  

简述： 同样是方法的hook，AndFix不像Dexposed从Method入手，而是以**Field**为切入点。把方法改为native的，再在底层Hook替换。 


### 起航
根据官方介绍的`Load patch`，我们下`patchManager.loadPatch()`的内容

	/**
	 * load patch,call when application start
	 * 
	 */
	public void loadPatch() {
		mLoaders.put("*", mContext.getClassLoader());// wildcard
		Set<String> patchNames;
		List<String> classes;
		for (Patch patch : mPatchs) {
			patchNames = patch.getPatchNames();
			for (String patchName : patchNames) {
				classes = patch.getClasses(patchName);
				mAndFixManager.fix(patch.getFile(), mContext.getClassLoader(),
						classes);
			}
		}
	}

看到最后干活的是那个`AndFixManger`的`fix（）`方法

    /**
	 * fix
	 *
	 * @param file        patch file
	 * @param classLoader  classloader of class that will be fixed
	 * @param classes     classes will be fixed
	 */
	public synchronized void fix(File file, ClassLoader classLoader, List<String> classes) {
	
		...
 
		File optfile = new File(mOptDir, file.getName());
		boolean saveFingerprint = true;
		if (optfile.exists()) {
			// need to verify fingerprint when the optimize file exist,
			// prevent someone attack on jailbreak device with
			// Vulnerability-Parasyte.
			// btw:exaggerated android Vulnerability-Parasyte
			// http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
			if (mSecurityChecker.verifyOpt(optfile)) {
				saveFingerprint = false;
			} else if (!optfile.delete()) {
				return;
			}
		}
           //读取补丁dex文件
		final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
				optfile.getAbsolutePath(), Context.MODE_PRIVATE);

		if (saveFingerprint) {
			mSecurityChecker.saveOptSig(optfile);
		}
		
 

		ClassLoader patchClassLoader = new ClassLoader(classLoader) {
			@Override
			protected Class<?> findClass(String className) {
				Class<?> clazz = dexFile.loadClass(className, this);
				if (clazz == null && 
					className.startsWith("com.alipay.euler.andfix")) {
					
					return Class.forName(className);
					// annotation’s class not found
				}
				if (clazz == null) {
					throw new ClassNotFoundException(className);
				}
				return clazz;
			}
		};
		
		Enumeration<String> entrys = dexFile.entries();
		Class<?> clazz = null;
		while (entrys.hasMoreElements()) {
			String entry = entrys.nextElement();
			if (classes != null && !classes.contains(entry)) {
				continue;// skip, not need fix
			} 
			clazz = dexFile.loadClass(entry, patchClassLoader);
			if (clazz != null) {
				fixClass(clazz, classLoader);    // 找到补丁，加载 
			}
		}
	}
	
我们去看下fixClass
 
	private void fixClass(Class<?> clazz, ClassLoader classLoader) {
	  Method[] methods = clazz.getDeclaredMethods();
	  MethodReplace methodReplace;
	  String clz;
	  String meth;
	  //替换所有方法，annotation是补丁包加上的
	  for (Method method : methods) {
	    methodReplace = method.getAnnotation(MethodReplace.class);
	    if (methodReplace == null)
	      continue;
	    clz = methodReplace.clazz();
	    meth = methodReplace.method();
	    if (!isEmpty(clz) && !isEmpty(meth)) {
	      replaceMethod(classLoader, clz, meth, method);
	    }
	  }
	}

替换方法的	 replaceMethod()

	private void replaceMethod(ClassLoader classLoader, 
	String clz,String meth, Method method) {
		 
		String key = clz + "@" + classLoader.toString();
		Class<?> clazz = mFixedClass.get(key);
		if (clazz == null) {// class not load
			// 要被替换的class
			Class<?> clzz = classLoader.loadClass(clz);
			// initialize target class
			clazz = AndFix.initTargetClass(clzz);
			//这个initTargetClass居然是去修改 access flag 为Public
		}
		if (clazz != null) {// initialize class OK
			mFixedClass.put(key, clazz);
    	      // 需要被替换的函数
			Method src = clazz.getDeclaredMethod(meth,
					method.getParameterTypes());
			AndFix.addReplaceMethod(src, method);
			// 替换逻辑，在cpp进行实现
		}
		...
	}


	/**
	 * initialize the target class, and modify access flag of class’ fields to
	 * public   <---修改access flag为Public，这有才！我们跟踪去看下
	 * 
	 * @param clazz
	 *            target class
	 * @return initialized class
	 */
	public static Class<?> initTargetClass(Class<?> clazz) {
		try {
			Class<?> targetClazz = Class.forName(clazz.getName(), true,
					clazz.getClassLoader());

			initFields(targetClazz);
			return targetClazz;
		} catch (Exception e) {
			Log.e(TAG, "initTargetClass", e);
		}
		return null;
	}
	
	private static void initFields(Class<?> clazz) {
		Field[] srcFields = clazz.getDeclaredFields();
		for (Field srcField : srcFields) {
			Log.d(TAG, "modify " + clazz.getName() + "." + srcField.getName()
					+ " flag:");
			setFieldFlag(srcField);
		}
	}
	
	private static native void setFieldFlag(Field field);
	Runtime.getRuntime().loadLibrary("andfix");    //  <---这个
	
是个native的内容，我们看下加载的是什么，看起来对art和dalvik做了区分对待

	static void setFieldFlag(JNIEnv* env, jclass clazz, jobject field) {
		if (isArt) {
			art_setFieldFlag(env, field);
		} else {
			dalvik_setFieldFlag(env, field);
		}
	}

我们去看下dalvik的，在`dalvik_method_replace.cpp`里面

	extern void dalvik_setFieldFlag(JNIEnv* env, jobject field) {
	
		Field* dalvikField = (Field*) env->FromReflectedField(field);
		dalvikField->accessFlags = dalvikField->accessFlags & (~ACC_PRIVATE)
								   | ACC_PUBLIC;
		LOGD("dalvik_setFieldFlag: %d ", dalvikField->accessFlags);
	}

// 真是大法厉害，这都可以。哈哈
好了，我们回主线，去看下那个`AndFix.replaceMethod()` 

 
	 public static void addReplaceMethod(Method src, Method dest) {
			try {
				replaceMethod(src, dest);
				initFields(dest.getDeclaringClass());
			} catch (Throwable e) {
				Log.e(TAG, "addReplaceMethod", e);
			}
		}
     //也是一个native的
	private static native void replaceMethod(Method dest, Method src);

在 andFix.cpp里面，也是区分了dalvik和art。

	static void replaceMethod(JNIEnv* env, jclass clazz, jobject src,
	jobject dest) {
		if (isArt) {
			art_replaceMethod(env, src, dest);
		} else {
			dalvik_replaceMethod(env, src, dest);
		}
	}

我们先挑个dalvik的来看下

	extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
        JNIEnv *env, jobject src, jobject dest) {
        
	    jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	    ClassObject *clz = (ClassObject *) dvmDecodeIndirectRef_fnPtr(
	            dvmThreadSelf_fnPtr(), clazz);
	            
	    clz->status = CLASS_INITIALIZED;
	    //我们要替换的有bug的方法。
	    Method *meth = (Method *) env->FromReflectedMethod(src);
	    
	    // 我们的新的解决方法
	    Method *target = (Method *) env->FromReflectedMethod(dest);
	    LOGD("dalvikMethod: %s", meth->name);
	
	    meth->jniArgInfo = 0x80000000; 
	    meth->accessFlags |= ACC_NATIVE;//修改方法为native方法！！
	
	    int argsSize = dvmComputeMethodArgsSize_fnPtr(meth);
	    if (!dvmIsStaticMethod(meth))
	        argsSize++;
	    meth->registersSize = meth->insSize = argsSize;
	    meth->insns = (void *) target;//保存新的方法到insnn
	    meth->nativeFunc = dalvik_dispatcher;
	    //绑定桥接函数，java方法的跳转函数
	}


	static void dalvik_dispatcher(const u4 *args, jvalue *pResult,
                              const Method *method, void *self) {
 
		  ...
	  
	    Method *meth = (Method *) method->insns;
	    meth->accessFlags = meth->accessFlags | ACC_PUBLIC;
 
		 ...	
		 
	    if (!dvmIsStaticMethod(meth)) {
	        Object *thisObj = (Object *) args[0];
	        ClassObject *tmp = thisObj->clazz;
	        thisObj->clazz = meth->clazz;
	        argArray = boxMethodArgs(meth, args + 1);
	        if (dvmCheckException_fnPtr(self))
	            goto bail;  //<-- 咦，我们看到了goto！
	
	        dvmCallMethod_fnPtr(self, (Method *) jInvokeMethod,	        
	                            dvmCreateReflectMethodObject_fnPtr(meth), &result,
	                             thisObj,argArray);
	
	        thisObj->clazz = tmp;
	    } else {
	        argArray = boxMethodArgs(meth, args);
	        if (dvmCheckException_fnPtr(self))
	            goto bail;
	
	        dvmCallMethod_fnPtr(self, (Method *) jInvokeMethod,
	                            dvmCreateReflectMethodObject_fnPtr(meth), &result, NULL,
	                            argArray);
	    }
	    
	    if (dvmCheckException_fnPtr(self)) {
	        Object *excep = dvmGetException_fnPtr(self);
	        jni_env->Throw((jthrowable) excep);
	        goto bail;
	    }
	
	    ...
	
	    bail:
	    dvmReleaseTrackedAlloc_fnPtr((Object *) argArray, self);
	}

 
	extern jboolean __attribute__ ((visibility ("hidden"))) dalvik_setup(
        JNIEnv *env, int apilevel) {
    jni_env = env;
    void *dvm_hand = dlopen("libdvm.so", RTLD_NOW);
    if (dvm_hand) {
        ...
        dvmCallMethod_fnPtr = dvm_dlsym(dvm_hand,apilevel>10?
				"_Z13dvmCallMethodP6ThreadPK6MethodP6ObjectP6JValuez" :
				"dvmCallMethod");
        if (!dvmCallMethod_fnPtr) {
            throwNPE(env, "dvmCallMethod_fnPtr");
            return JNI_FALSE;
        }
      ...


我们看到，它通过`dvmCallMethod_fnPtr`来调用`libdvm.so`中的`dvmCallMethod()`来加载替换后的新方法，从而替换方法。 
至此，通过dalvik_dispatcher这个跳转函数完成最后的替换工作，到这里就完成了两个方法的替换。

	
对于art，他还根据系统的版本的不同，分了6.0，5.1版和5.0版。
	
	extern void __attribute__ ((visibility ("hidden"))) art_replaceMethod(
			JNIEnv* env, jobject src, jobject dest) {
		if (apilevel > 22) {
			replace_6_0(env, src, dest);
		} else if (apilevel > 21) {
			replace_5_1(env, src, dest);
		} else {
			replace_5_0(env, src, dest);
		}
	}
我们挑个5.0的吧，反正大同小异的。art_method_replace_5_0.cpp

	void replace_5_0(JNIEnv* env, jobject src, jobject dest) {
	
		art::mirror::ArtMethod* smeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(src);
	
		art::mirror::ArtMethod* dmeth =
				(art::mirror::ArtMethod*) env->FromReflectedMethod(dest);
	
		dmeth->declaring_class_->class_loader_ =
				smeth->declaring_class_->class_loader_; //for plugin classloader
		dmeth->declaring_class_->clinit_thread_id_ =
				smeth->declaring_class_->clinit_thread_id_;
		dmeth->declaring_class_->status_ = smeth->declaring_class_->status_-1;
	
		smeth->declaring_class_ = dmeth->declaring_class_;
		smeth->access_flags_ = dmeth->access_flags_;
		smeth->frame_size_in_bytes_ = dmeth->frame_size_in_bytes_;
		smeth->dex_cache_initialized_static_storage_ =
				dmeth->dex_cache_initialized_static_storage_;
		smeth->dex_cache_resolved_types_ = dmeth->dex_cache_resolved_types_;
		smeth->dex_cache_resolved_methods_ = dmeth->dex_cache_resolved_methods_;
		smeth->vmap_table_ = dmeth->vmap_table_;
		smeth->core_spill_mask_ = dmeth->core_spill_mask_;
		smeth->fp_spill_mask_ = dmeth->fp_spill_mask_;
		smeth->mapping_table_ = dmeth->mapping_table_;
		smeth->code_item_offset_ = dmeth->code_item_offset_;
		smeth->entry_point_from_compiled_code_ =
				dmeth->entry_point_from_compiled_code_;
	
		smeth->entry_point_from_interpreter_ = dmeth->entry_point_from_interpreter_;
		smeth->native_method_ = dmeth->native_method_;
		smeth->method_index_ = dmeth->method_index_;
		smeth->method_dex_index_ = dmeth->method_dex_index_;
	
		LOGD("replace_5_0: %d , %d", smeth->entry_point_from_compiled_code_,
				dmeth->entry_point_from_compiled_code_);

	}


	void setFieldFlag_5_0(JNIEnv* env, jobject field) {
		art::mirror::ArtField* artField =
				(art::mirror::ArtField*) env->FromReflectedField(field);
		artField->access_flags_ = artField->access_flags_ & (~0x0002) | 0x0001;
		LOGD("setFieldFlag_5_0: %d ", artField->access_flags_);
	}

我们看到，主要是把原方法的各种属性都改成补丁方法的，同时实现的指针也替换为新的。
 

真是写Android程序，虽说是用java入门，不过写到后面变成了C/C++去了。整个项目还是挺小巧精悍的，赞

## Dexposed
版本：d108256

简述：基于Xposed的AOP框架，取Dexposed也是向他致敬的意思，目前做到了方法级粒度（可以调用方法前，方法后，替代方法，这三种都可以做到，AOP，写过spring的应该对这个再熟悉不过了，针对java方法做拦截，但不支持C的方法。），另外可以插桩，做热补丁和SDK hook等等的功能，很好很强大。就是不支持5.0+。  我们知道，Xposed是需要Root权限的，因为它劫持 `zygote`，并使用`Xposed Bridge`来hook方法并注入自己的代码，实现非侵入式的runtime修改。但Dexposed不需要，因为他劫持的是自己，通过劫持 `java method`，将java方法改为native，并且将这个方法的实现链接到一个通用的`Native Dispatch`方法上。已经半年没更新了，不知道他们对ART支持是不是已经搞定，不过没更新。

具体流程如图:

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/hotfix_015a873b31a0044c65a5a1c4501b9bb9_b.png)

 具体到方法，可参见XposedBridge:
	 
	/**
	 * Intercept every call to the specified method and call a handler function instead.
	 * @param method The method to intercept
	 */
	 
	private native synchronized static void hookMethodNative(
									Member method, Class<?> declaringClass, int slot, 
									Object additionalInfo);
									

其具体native实现则在Xposed的`libxposed_common.cpp`里面有注册，根据系统版本分发到
`libxposed_dalvik`和`libxposed_art`里面，以`dalvik`为例大致来说就是记录下原来的方法信息，并把方法指针指向我们的`hookedMethodCallback`，从而实现拦截的目的。

至于代码的解析，应为不能修art的我们先不深究 (  -_ -" )
其余几个库，我们下次再写。

# 比较

## DroidPlugin

### 限制和缺陷:

1. 无法在插件中发送具有自定义资源的Notification，例如： a. 带自定义RemoteLayout的Notification b. 图标通过R.drawable.XXX指定的通知（插件系统会自动将其转化为Bitmap）
2. 无法在插件中注册一些具有特殊Intent Filter的Service、Activity、BroadcastReceiver、ContentProvider等组件以供Android系统、已经安装的其他APP调用。
3. 缺乏对Native层的Hook，对某些带native代码的apk支持不好，可能无法运行。比如一部分游戏无法当作插件运行。

### 特点：

1. 支持Androd 2.3以上系统，在host中集成Droid Plugin项目非常简单
2. 插件APK完全不需做任何修改，可以独立安装运行、也可以做插件运行。要以插件模式运行某个APK，你无需重新编译、无需知道其源码。
3. 插件的四大组件完全不需要在Host程序中注册，支持Service、Activity、BroadcastReceiver、ContentProvider四大组件
4. 插件之间、Host程序与插件之间会互相认为对方已经"安装"在系统上了。
5. API低侵入性：极少的API。HOST程序只是需要一行代码即可集成Droid Plugin
6. 超强隔离：插件之间、插件与Host之间完全的代码级别的隔离：不能互相调用对方的代码。通讯只能使用Android系统级别的通讯方法。
7. 支持所有系统API
8. 资源完全隔离：插件之间、与Host之间实现了资源完全隔离，不会出现资源窜用的情况。
9. 实现了进程管理，插件的空进程会被及时回收，占用内存低。
10. 插件的静态广播会被当作动态处理，如果插件没有运行（即没有插件进程运行），其静态广播也永远不会被触发。
使


   
## AndFix
 
AndFix是全版本支持，但是，但是，以我国内的繁荣的各种定制ROM前，适配这种坑啊，说多都是泪。
虽然从实现方式上来说，类似Dexposed，通过jni来替换，但更简洁直接，应用**patch不需要重启**。
	但由于跳过了类初始化，所以像静态xx和构造函数都会出问题，复杂点的类Class.forname可能直接挂掉。我觉得还是得重启好。反正一般需要紧急修复的这种Bug都可以**非常的成功，非常成功的**导致程序闪退。

## ClassLoader
这个没在上面贴，是在一篇QQ空间的微信号发的文章看到的，具体我介绍我就不复制了，在结尾的参考文章有提到，里面有详细的介绍。ClassLoader是全版本支持，虽对启动速度略有影响（微信启动那时间久得啊，不想说），且只能在重启时生效（个人感觉这还是可以接受的，参考上面AndFix那句话，而且我记得QQ空间的一篇文章，有做多次崩溃就重置app的操作的。这还是挺细心的！），不过已在扣扣空间有了较长时间的实际测试了，还是个不错的选择的。 开源实现有Nuwa, HotFix, DroidFix。


## Dexposed

Dexposed不支持**Art模式**，写补丁的话有点小困难，需要反射写混淆后的代码，粒度细，如果你要替换的方法多的话，这工作量会比较大。热补丁和增量都是一个应对紧急情况的救火方法，做好测试与写好代码还是不能丢的。

下面是官方贴的支持情况：

> Follow is support status. 
Runtime | Android Version | Support
------  | --------------- | --------
Dalvik  | 2.2             | Not Test
Dalvik  | 2.3             | Yes
Dalvik  | 3.0             | No
Dalvik  | 4.0-4.4         | Yes
ART     | 5.0             | Testing
ART     | 5.1             | No
ART     | M               | No

> 另外，据闻facebook也有自己的一个叫[Nish的热修复项目](https://www.zhihu.com/question/29093652)，不过目前还在解决这个ART问题，所以推迟了。

 不过又是据闻，对ART的支持是有解决方案的，具体它们的是什么，目前不得而知。
  
  
小结
---

  插件化给我们带来的一些好处是明显的，相信你也看了很多介绍了，但下面两点想提下 
  
1. **A/B testing**
以前写过A/B test的代码都知道，一堆根据服务器传来的参数做的判断条件代码镶嵌在程序里面。看起来还是挺难受的，用了插件化，就可以直接的直接分成A,B两个包，这样就分割开了。
2. **解耦项目**
 当你的项目到了一定程序，要接入的内容很多的时候，这时候从MultiDex转到插件化是在所难免的。它让我们避免了协同开发过程中可能遇到的问题，例如bug的回归测试，编译速度的提高等。负责特定模块的人只需要负责好自己的程序就好咯。
 
至于带来的麻烦嘛，也是有的，就看你项目的复杂度到什么级别。
   

 
# 参考资料

1. [阿里技术沙龙第十六期《android插件化及动态部署—ATLAS》](http://v.youku.com/v_show/id_XNTMzMjYzMzM2.html#paction)，查相关内容时候在知乎看到关于Atlas的一段事，传送地址 －－ 》》 [你怎么看待携程DynamicApk插件化框架的抄袭现象?](https://www.zhihu.com/question/38083715)
 
2. [DroidPlugin插件机制介绍.pptx](https://github.com/Qihoo360/DroidPlugin/blob/master/DOC/%E6%8F%92%E4%BB%B6%E6%9C%BA%E5%88%B6%E4%BB%8B%E7%BB%8D.pptx) ，作者写的一个介绍ppt

3. [custom-class-loading-in-dalvik](http://android-developers.blogspot.com/2011/07/custom-class-loading-in-dalvik.html)，动态加载class的Google官方教程，插件化原理基础

4. [安卓App热补丁动态修复技术介绍](http://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=0#wechat_redirect) ， QQ空间团队的热修复方案，灵感来源于谷歌的MultiDex

5. [手机淘宝Hotpatch技术介绍](http://www.infoq.com/cn/presentations/mobile-phone-taobao-hotpatch-technology-introduction)，qcon大会上手淘hotpatch技术介绍

6. [Android动态加载技术三个关键问题详解](http://www.infoq.com/cn/articles/android-dynamic-loading?utm_campaign=rightbar_v2&utm_source=infoq&utm_medium=articles_link&utm_content=link_text)，任主席的一篇文章
7.  [携程Android App插件化和动态加载实践](http://www.infoq.com/cn/articles/ctrip-android-dynamic-loading?email=947091870@qq.com)



# 后记

插件化虽然在自己现在的项目中用不到，不过学习下他们怎么实现的也挺好的，在这里做下总结，以后要是有需要也可以快速的用上，虽然到以后可能都有别的解决方案了，或者安卓没落，被VR/AR给替代了...