title: 源码探索系列33---安卓系统的结合子Zygote
date: 2016-04-22 19:56:46
tags: [android,源码,Zygote]
categories: android

------------------------------------------
 
Last time，we hava talk about `SystemServer`,which is fork by `zygote`.
Today, we will do some research on  `Zygote`.The reason why Google use the name "Zygote" to name it Maybe it's because of the job Zygote handled. All the Application Process & the most important class `SystemServer` are all fork by him.This paper will introduction the hole startUp process of zygote.class

 
``` sequence

Title: StartUp Zygote 
init->app_process: 1.main
app_process->AndroidRuntime: 2.start
AndroidRuntime->ZygoteInit:3. main
ZygoteInit-> ZygoteInit: 4.registerZygoteSocket
ZygoteInit-> ZygoteInit: 5.preLoad
ZygoteInit-> ZygoteInit: 6.startSystemServer 
ZygoteInit-> RuntimeInit: 7.ZygoteInit
RuntimeInit->RuntimeInit: 8.ZygoteInitNative
RuntimeInit->SystemServer: 9.main
ZygoteInit-> ZygoteInit: 10.runSelectLoop

```



<!--more-->

# start
we all know in Linux,any kind of  process is fork by  `The Init Process`. android is based on Linux,so the zygote must be fork by init directly or indirectly.In face ,the process is fork use init process to  decode `init.zygote.rc` (\system\core\rootdir\init.zygote64.rc)

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote  --start-system-server

	class main
	socket zygote stream 660 root system
	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
	onrestart restart netd
```

this command tell  the init to create a process call zygote  and execute the program  `app_process64` with name:`-Xzygote`  pathname:`/system/bin`   arguments:`--zygote --start-system-server`. 

`socket zygote stream 660 root system` means the Zygote process wil create a socket call "zygote". so that we will have a file on  `"/dev/socket/zygote"` 

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/33-%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160422144521.png)

this socket is very important. `ActivityManagerService` use this socket to communication with zygote  for fork a new application process.

	onrestart write /sys/android_power/request_state wake
	onrestart write /sys/power/state on
	onrestart restart media
	onrestart restart netd
means what to restart when zygote service process is restarts.
(more info about those command -> `system/core/init/readme.txt`)


according to the command ,we know the face that the service path is 
system/bin/app_process/app_process/app_main.cpp .ok,let's have a look!

#1- app_main



	int main(int argc, char* const argv[]){	 
	
	    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
	        // Older kernels don't understand PR_SET_NO_NEW_PRIVS and return
	        // EINVAL. Don't die on such kernels.
	        if (errno != EINVAL) {
	            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
	            return 12; //magic number!!!.
	        }
	    }
	    
	 
	    //we have send arguments:`--zygote --start-system-server`
	        
	    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
	    // Process command line arguments
	    // ignore argv[0]
	    argc--;
	    argv++;
	
	    // Everything up to '--' or first non '-' arg goes to the vm.
	    //
	    // The first argument after the VM args is the "parent dir", which
	    // is currently unused.
	    //
	    // After the parent dir, we expect one or more the following internal
	    // arguments :
	    //
	    
	    // --zygote : Start in zygote mode  《== one of the argus
	    // --start-system-server : Start the system server.⇐ The second args
	    
	    // --application : Start in application (stand alone, non zygote) mode.
	    // --nice-name : The nice name for this process.
	    //
	    // For non zygote starts, these arguments will be followed by
	    // the main class name. All remaining arguments are passed to
	    // the main method of this class.
	    //
	    // For zygote starts, all remaining arguments are passed to the zygote.
	    // main function.
	    //
	    // Note that we must copy argument string values since we will rewrite the
	    // entire argument block when we apply the nice name to argv0.
	
	    int i;
	    for (i = 0; i < argc; i++) {
	        if (argv[i][0] != '-') {
	            break;
	        }
	        if (argv[i][1] == '-' && argv[i][2] == 0) {
	            ++i; // Skip --.
	            break;
	        }
	        runtime.addOption(strdup(argv[i]));
	    }
	    //主要为把存储保存到runtiem里去
	
	    // Parse runtime arguments.  Stop at first unrecognized option.
	    bool zygote = false;
	    bool startSystemServer = false;
	    bool application = false;
	    String8 niceName;
	    String8 className;
	
	    ++i;  // Skip unused "parent dir" argument.
	    while (i < argc) {
	        const char* arg = argv[i++];
	        if (strcmp(arg, "--zygote") == 0) {
	            zygote = true;
	            niceName = ZYGOTE_NICE_NAME;
	        } else if (strcmp(arg, "--start-system-server") == 0) {
	            startSystemServer = true;
	        } else if (strcmp(arg, "--application") == 0) {
	            application = true;
	        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
	            niceName.setTo(arg + 12);
	        } else if (strncmp(arg, "--", 2) != 0) {
	            className.setTo(arg);
	            break;
	        } else {
	            --i;
	            break;
	        }
	    }
	
	    Vector<String8> args;
	    if (!className.isEmpty()) {
	        // We're not in zygote mode, the only argument we need to pass
	        // to RuntimeInit is the application argument.
	        //
	        // The Remainder of args get passed to startup class main(). Make
	        // copies of them before we overwrite them with the process name.
	        args.add(application ? String8("application") : String8("tool"));
	        runtime.setClassNameAndArgs(className, argc - i, argv + i);
	    } else {
	        // We're in zygote mode.
	        maybeCreateDalvikCache();
	
	        if (startSystemServer) {
	            args.add(String8("start-system-server"));
	        }
	
	        char prop[PROP_VALUE_MAX];
	        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
	            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
	                ABI_LIST_PROPERTY);
	            return 11;
	        }
	
	        String8 abiFlag("--abi-list=");
	        abiFlag.append(prop);
	        args.add(abiFlag);
	
	        // In zygote mode, pass all remaining arguments to the zygote
	        // main() method.
	        for (; i < argc; ++i) {
	            args.add(String8(argv[i]));
	        }
	    }
	
	    if (!niceName.isEmpty()) {
	        runtime.setArgv0(niceName.string());
	        set_process_name(niceName.string());
	    }
	
	    if (zygote) {
	        runtime.start("com.android.internal.os.ZygoteInit", args, zygote); //<-- invoke this func
	    } else if (className) {
	        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
	    } else {
	        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
	        app_usage();
	        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
	        return 10;
	    }
	}

the whole process just collect the info and  call `runtime.start()` function.the RunTime is the subclass of AndroidRuntime.
 	
	class AppRuntime : public AndroidRuntime{
	
	public:
	    AppRuntime(char* argBlockStart, const size_t argBlockLength)
	        : AndroidRuntime(argBlockStart, argBlockLength)
	        , mClass(NULL){
	        
	    }
	
	  void setClassNameAndArgs(const String8& className, int argc, char * const *argv) {
	        mClassName = className;
	        for (int i = 0; i < argc; ++i) {
	             mArgs.add(String8(argv[i]));
	        }
	    }
	
	    ...  
	};

# 2. AndroidRuntime.start

	/*
	 * Start the Android runtime.  This involves starting the virtual machine
	 * and calling the "static void main(String[] args)" method in the class
	 * named by "className".
	 *
	 * Passes the main function two arguments, the class name and the specified
	 * options string.
	 */
	void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote){
	
		 ...			 
	 
	    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
	    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);
	
 
	    JniInvocation jni_invocation;
	    jni_invocation.Init(NULL);
	    JNIEnv* env;
  	    // 1. start the virtual machine ,this func is pretty llll...long,neary 415 line!
	    if (startVm(&mJavaVM, &env, zygote) != 0) {
	        return;
	    }		     
	    onVmCreated(env);
	
	
	    
	     //2.  Register android functions.		     
	    if (startReg(env) < 0) {
	        ALOGE("Unable to register all android natives\n");
	        return;
	    }
	
	    /*
	     * We want to call main() with a String array with arguments in it.
	     * At present we have two arguments, the class name and an option 
	     * string.Create an array to hold them.
	     */
	    jclass stringClass;
	    jobjectArray strArray;
	    jstring classNameStr;
	
	    stringClass = env->FindClass("java/lang/String");
	    assert(stringClass != NULL);
	    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
	    assert(strArray != NULL);
	    classNameStr = env->NewStringUTF(className);
	    assert(classNameStr != NULL);
	    env->SetObjectArrayElement(strArray, 0, classNameStr);
	
	    for (size_t i = 0; i < options.size(); ++i) {
	        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
	        assert(optionsStr != NULL);
	        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
	    }
	
	
	    /* 3. call ZygoteInit.main()
	     *
		*  args: className=com.android.internal.os.ZygoteInit.  
	     * 这个是前面调用时候就传过来的参数
	     * Start VM.  This thread becomes the main thread of the VM, and will
	     * not return until the VM exits.
	     * 
	     *  
	     */
	    char* slashClassName = toSlashClassName(className);
	    jclass startClass = env->FindClass(slashClassName);
	    if (startClass == NULL) {
	        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
	        /* keep going */
	    } else {
	        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
	            "([Ljava/lang/String;)V");
	        if (startMeth == NULL) {
	            ALOGE("JavaVM unable to find main() in '%s'\n", className);
	            /* keep going */
	        } else {
	            env->CallStaticVoidMethod(startClass, startMeth, strArray);
	
	#if 0
	            if (env->ExceptionCheck())
	                threadExitUncaughtException(env);
	#endif
	        }
	    }
	    free(slashClassName);
	
		...
	}
  
 the start() contain 3 steps. 
  - 1. startVm ,start up virtual machine
  - 2. startReg ,register jni function
  - 3. Invoke ZygoteInit.main()
 
##2.1 startVm

	int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote){
	
	   ... 
	   
	  /*
	    * The default starting and maximum size of the heap.  Larger
	    * values should be specified in a product property override.
	    */	    
	   parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
	   parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");
	   parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
	   parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
	   parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
	   parseRuntimeOption("dalvik.vm.heaptargetutilization",
	                      heaptargetutilizationOptsBuf, "-XX:HeapTargetUtilization=");
	
	   /*
	    * JIT related options.
	    */
	   parseRuntimeOption("dalvik.vm.usejit", usejitOptsBuf, "-Xusejit:");
	   parseRuntimeOption("dalvik.vm.jitcodecachesize", jitcodecachesizeOptsBuf, "-Xjitcodecachesize:");
	   parseRuntimeOption("dalvik.vm.jitthreshold", jitthresholdOptsBuf, "-Xjitthreshold:");
	
	   property_get("ro.config.low_ram", propBuf, "");
	   if (strcmp(propBuf, "true") == 0) {
	     addOption("-XX:LowMemoryMode");
	   }
	    ...
    
	    //craete jvm!!!
	   if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
	       ALOGE("JNI_CreateJavaVM failed\n");
	       return -1;
	   }
	   
	}

well, sometimes in our gradle ,we'll use  like  `javaMaxHeapSize "2g"` to opt the performence!

	android {  
	   dexOptions {
	       javaMaxHeapSize "2g"
	   }
	}

# 3. ZygoteInit.main

wow.  we have see this main function before when talk about the  System Server. so .make long story short 

	public static void main(String argv[]) {
	   
	   ...		
       boolean startSystemServer = false;
       String socketName = "zygote";
       String abiList = null;
       for (int i = 1; i < argv.length; i++) {
           if ("start-system-server".equals(argv[i])) {
               startSystemServer = true;
           } else if (argv[i].startsWith(ABI_LIST_ARG)) {
               abiList = argv[i].substring(ABI_LIST_ARG.length());
           }  ...
       }
       
	   if (abiList == null) {
		   //刚才好像没注意到哪里有加多这个abiList的内容,但应该不为空
           throw new RuntimeException("No ABI list supplied.");
        }
            
	   //1. register socket, commutation with AMS
       registerZygoteSocket(socketName); 
       //2.pre load resource
       preload();
       
       //3. start system server  
       if (startSystemServer) {
           startSystemServer(abiList, socketName);
 
       } 
       //4.  loop ,waiting for the msg from AMS
       runSelectLoop(abiList);

       closeServerSocket();
		...
    }
    
## 3.1 registerZygoteSocket

	private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                
                // create local Socket service
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
## 3.2 preload()

	static void preload() {
	       Log.d(TAG, "begin preload");
	       //load the class on /system/etc/preloaded-classes
	       preloadClasses();
	       // pre load resource drawable & color
	       //R.array.preloaded_drawables
	       //R.array.preloaded_color_state_lists
	       preloadResources();
	       
	       preloadOpenGL();
	       preloadSharedLibraries();
	       preloadTextResources();
	       // Ask the WebViewFactory to do any initialization that must run in the zygote process,
	       // for memory sharing purposes.
	       WebViewFactory.prepareWebViewInZygote();
	       Log.d(TAG, "end preload");
	   }

## 3.3 startSystemServer
create system server process ,uid=1000,gid=1000

	private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
            
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_BLOCK_SUSPEND,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);//<-- IMPORTANT
        }

        return true;
    }

link:  [源码探索系列32---万物初生System Servers](http://sanjay-f.github.io/2016/04/21/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9732---%E4%B8%87%E7%89%A9%E5%88%9D%E7%94%9F/)

         
           
## 3.4-runSelectLoop

	/**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     *
     * @throws MethodAndArgsCaller in a child process when a main() should
     * be executed.
     */
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    boolean done = peers.get(i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
    
 selected, poll, epoll。The scalable I/O event notification mechanism
 [more detail](https://en.wikipedia.org/wiki/Epoll) 
 


# Summary

强行用英文写了一篇，感觉实在是丢了太多英语知识了.看能继续坚持写多久。
查看整个启动过程，还是发现不少知识上的不足的地方，顺便也学习了下！
感觉又有新的突破感！继续加油
 

# Ref
information from many places to put this article together.
 

[Android Zygote Startup](http://elinux.org/Android_Zygote_Startup)  

[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)
 

**detail about services command in the init.rc file**

    Services
    --------
    Services are programs which init launches and (optionally) restarts
    when they exit.  Services take the form of:
    
    service <name> <pathname> [ <argument> ]*
       <option>
       <option>
       ...
    
    
    Options
    -------
    Options are modifiers to services.  They affect how and when init
    runs the service.
    
    class <name>
      Specify a class name for the service.  All services in a
      named class may be started or stopped together.  A service
      is in the class "default" if one is not specified via the
      class option.
      
    socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ]
      Create a unix domain socket named /dev/socket/<name> and pass
      its fd to the launched process.  <type> must be "dgram", "stream" or "seqpacket".
      User and group default to 0.
      'seclabel' is the SELinux security context for the socket.
      It defaults to the service security context, as specified by seclabel or
      computed based on the service executable file security context.
      
    onrestart
      Execute a Command (see below) when service restarts.



 