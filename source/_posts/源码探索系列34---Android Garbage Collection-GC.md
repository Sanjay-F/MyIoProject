title: 源码探索系列34---Android Garbage Collection/dalvik GC
date: 2016-04-25 15:54:46
tags: [android,源码,gc]
categories: android

------------------------------------------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/34-4519811.jpg)
when we saw the startup process of Zygote last time , we have come across a familiar words---**GC**
The Android Garbage Collection is quite strange to me,so i decided to record down something about it.
this is the reason why i write this page. 
 <!--more-->

# Garbage Collection Algorithm
API: 17/4.2 ---  
 

general , we have  `Reference Counting`、`Mark and Sweep GC`、`Copying GC`  And `Generational GC`。every Algorithm has it's strength and weakness  , so in order to make a better performance ,.In Android, it use `Mark and Sweep GC`、`Copying GC` instead of  Generational GC ,which is JVM used .

##  Mark and Sweep GC

 when the available memory is used out , system will suspend all threads and  the GC  thread is trigger   start to collection the garbage . the basic theory is simple. from the GC Roots , iterator all and preserver the objects which is refer directly or indirectly by the GC Roots and the rest of the object will be treat as garbage to be collection  ,so that we can release the memory.

Pseudocode:

	void GC(){

		SuspendAllThreads(); 
		List<Object> roots = GetRoots();		
		foreach ( Object root : roots ) {		
			Mark(root);		
		} 
		Sweep(); 
		ResumeAllThreads();		
	}
	

Basicly , the algorithm is divede into two step. Mark and Sweep

- Mark step , mark all elements which can be find from GC roots.

		void Mark(Object* pObj) {
		
			if ( !pObj->IsMarked() ) {
			     // 修改对象头的Marked标志
			     pObj->Mark();
			     // 深度优先遍历对象引用到的所有对象
			     List<Object *> fields = pObj->GetFields();
			     foreach ( Object* field : fields ) {
			     Make(field); // 递归处理引用到的对象
			     }
			 }
		}
before GC, the whole graphic look like that ,
![回收内存垃圾之前的对象引用关系](http://images.cnitblog.com/blog/49788/201306/12111951-a62f397ce02443d3ad6c7d79d28f5a0a.png)
after `mark()` the highlight blue one  means marked, and the white one means garbage,which isn't refer by the GC Roots.
![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111951-4184d3205adb4f7ea7e1abb83344cfd4.png)

- Sweep step. executor the garbage collect process. only the useful objects can survival.

		void Sweep() {
		    Object *pIter = GetHeapBegin();
		    
		    while ( pIter < GetHeapEnd() ) {
		     if ( !pIter->IsMarked() )
			     Free(pIter);
		     else
			     pIter->UnMark();
			     		 
			  pIter = MoveNext(pIter);
	        }
	    }
![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111951-a791b1ba05b04ed4a6dd6e8fd816f654.png)
when the memory is enough , this algorithm is very memory friendly , only malloc very tiny space，but the most fatal features is that it will **suspend all threads**!


##  Copying GC Algorithms
in fact ,this algorithms is other king of  Mark algorithms , use two heap call ping & pong。at the beginning ,all the memory malloc is assign to ping, he will manage `the Next Object Pointer`，all we need to do just to move the position of the pointer . when the ping memory is nearly used out. we  use Mark algorithms to identify all the available objects . which will be copy to pong heap. and the following memory malloc job will continue in the pong heap.
![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111952-2441543b673242fe870b7d4ef37bd393.png)

![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111952-68b161d6d30a479aad702181b907382a.png)`enter code here`
in verse , when the pong space is out .change the process  
 
 this algorithm is very fast on memory allocation and lower interruption . because during  the gc procession , when we copy one available objects to another head, we can also meet the need to allocation of the memory at the same time，but as we see, it takes double space at all. 

 
# Android Garbage Collection 

In Android ，it use  Mark and Sweep  and Copying GC，but indeed,use which one is  decide on compile stage . it can't change dynamic when the system is running 。

on the source code of dalvik  machine's .(dalvik/vm/Dvm.mk ) , there is some piece of code of the Dvm.mk file.

	WITH_COPYING_GC := $(strip $(WITH_COPYING_GC))

	ifeq ($(WITH_COPYING_GC),true)
	  LOCAL_CFLAGS += -DWITH_COPYING_GC
	  LOCAL_SRC_FILES += \
		alloc/Copying.cpp.arm
	else
	  LOCAL_SRC_FILES += \
		alloc/DlMalloc.cpp \
		alloc/HeapSource.cpp \
		alloc/MarkSweep.cpp.arm
	endif

if we have setup  `WITH_COPYING_GC=true`,android will use copying GC algorithm or mark&Sweep algorithm.

## dvmStartup
we all know,object are assign on Java Heap,when  

	/*
	 * VM initialization.  Pass in any options provided on the command line.
	 * Do not pass in the class name or the options for the class.
	 *
	 * Returns 0 on success.
	 */
	 
	std::string dvmStartup(int argc, const char* const argv[],
	        bool ignoreUnrecognized, JNIEnv* pEnv){
 
	    ... 	    
	    // init the default config 
		setCommandLineDefaults(); 
	    // Initialize components.
	    
	    dvmQuasiAtomicsStartup();
	    if (!dvmAllocTrackerStartup()) {
	        return "dvmAllocTrackerStartup failed";
	    }
	    if (!dvmGcStartup()) {
	        return "dvmGcStartup failed";
	    }
	    if (!dvmThreadStartup()) {
	        return "dvmThreadStartup failed";
	    }
	     
	    if (!dvmInstanceofStartup()) {
	        return "dvmInstanceofStartup failed";
	    }         
		 ...
	}
this function has do many job, one of the most important thing is call dvmGcStartUP()(/dalvik/vm/alloc/Alloc.cpp) to init the gc componnets

###  setCommandLineDefaults()

	static void setCommandLineDefaults(){
	    const char* envStr = getenv("CLASSPATH");
	    if (envStr != NULL) {
	        gDvm.classPathStr = strdup(envStr);
	    } else {
	        gDvm.classPathStr = strdup(".");
	    }
	    envStr = getenv("BOOTCLASSPATH");
	    if (envStr != NULL) {
	        gDvm.bootClassPathStr = strdup(envStr);
	    } else {
	        gDvm.bootClassPathStr = strdup(".");
	    }
	
	    gDvm.properties = new std::vector<std::string>();
	
	    /* Defaults overridden by -Xms and -Xmx.
	     * TODO: base these on a system or application-specific default
	     */
	    gDvm.heapStartingSize = 2 * 1024 * 1024;  // Spec says 16MB; too big for us.
	    gDvm.heapMaximumSize = 16 * 1024 * 1024;  // Spec says 75% physical mem
	    gDvm.heapGrowthLimit = 0;  // 0 means no growth limit
	    gDvm.stackSize = kDefaultStackSize;
	    gDvm.mainThreadStackSize = kDefaultStackSize;
	    
	    ...
	}


## dvmGcStartup()


    /* Initialize the GC universe. 
	 * We're currently using a memory-mapped arena to keep things off of the
	 * main heap.  This needs to be replaced with something real.
	 */
	 bool dvmGcStartup(){
	    dvmInitMutex(&gDvm.gcHeapLock);
	    pthread_cond_init(&gDvm.gcHeapCond, NULL);
	    return dvmHeapStartup();
	}

## dvmHeapStartup()
	
	/* Initialize the GC heap.	 
	*Returns true if successful, false otherwise.
	*/
	bool dvmHeapStartup(){
	
	    GcHeap *gcHeap;
	
	    if (gDvm.heapGrowthLimit == 0) {
	        gDvm.heapGrowthLimit = gDvm.heapMaximumSize;
	    }
	    //1. allocate a contiguous region of virtual memory
	    gcHeap = dvmHeapSourceStartup(gDvm.heapStartingSize,
	                                  gDvm.heapMaximumSize,
	                                  gDvm.heapGrowthLimit);
	    if (gcHeap == NULL) {
	        return false;
	    }
	    gcHeap->ddmHpifWhen = 0;
	    gcHeap->ddmHpsgWhen = 0;
	    gcHeap->ddmHpsgWhat = 0;
	    gcHeap->ddmNhsgWhen = 0;
	    gcHeap->ddmNhsgWhat = 0;
	    gDvm.gcHeap = gcHeap;
	
	    //2. SetUp the lists we'll use for cleared reference objects.
	    // this will save  the Finalizable objects & execute it on another thread !
	      
	    gcHeap->clearedReferences = NULL;
	
	    //3. init the card table!dalvik/vm/alloc/Heap.cpp:100
	    if (!dvmCardTableStartup(gDvm.heapMaximumSize, gDvm.heapGrowthLimit)) {
	        LOGE_HEAP("card table startup failed.");
	        return false;
	    }	
	    return true;
	}
	
Allocate a contiguous region of virtual memory to subdivided among the heaps managed by the garbage collector.
when this function call dvmHeapSourceStartup(), it'll  allocate a contiguous region of virtual memory  which will auto growth when needed. the `heapGrowthLimit` mean the max available memory. ( 0 mean there is no limit ,we can allocate as we want )heapStartSize=16MB mean current available memory space 。
![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111953-9a444864854a4cd09775770d0316beec.png)
in `setCommandLineDefaults()`， we have see the argus value,
gDvm.heapStartingSize  is 16MB, if spend out this 16MB memory space , cause OOM.

## dvmHeapSourceStartup()

		/*
	 * Initializes the heap source; must be called before any other
	 * dvmHeapSource*() functions.  Returns a GcHeap structure
	 * allocated from the heap source.
	 */
	GcHeap* dvmHeapSourceStartup(size_t startSize, size_t maximumSize,
	                             size_t growthLimit)
	{
	    GcHeap *gcHeap;
	    HeapSource *hs;
	    mspace msp;
	    size_t length;
	    void *base;
		...
		
	    /*  
	     * 1. first Step!, Allocate a contiguous region of virtual memory to 
	     * subdivided among the heaps managed by the garbage collector.
	     *  dvmAllocRegion() on /dalvik/vm/Misc.cpp 
	     */
	    length = ALIGN_UP_TO_PAGE_SIZE(maximumSize); 
	    base = dvmAllocRegion(length, PROT_NONE, "dalvik-heap");
	    if (base == NULL) {
	        return NULL;
	    }
	
	   ...
	
	    hs->targetUtilization = gDvm.heapTargetUtilization * HEAP_UTILIZATION_MAX;
	    hs->minFree = gDvm.heapMinFree;
	    hs->maxFree = gDvm.heapMaxFree;
	    hs->startSize = startSize;
	    hs->maximumSize = maximumSize;
	    hs->growthLimit = growthLimit;
	    hs->idealSize = startSize;
	    hs->softLimit = SIZE_MAX;    // no soft limit at first
	    hs->numHeaps = 0;
	    hs->sawZygote = gDvm.zygote;
	    hs->hasGcThread = false;
	    hs->heapBase = (char *)base;
	    hs->heapLength = length;
	
	    if (hs->maxFree > hs->maximumSize) {
	      hs->maxFree = hs->maximumSize;
	    }
	    if (hs->minFree < CONCURRENT_START) {
	      hs->minFree = CONCURRENT_START;
	    } else if (hs->minFree > hs->maxFree) {
	      hs->minFree = hs->maxFree;
	    }
	    
	   //2. int three additional  Heap : livebits /markbits /Mark Stack
	    if (!addInitialHeap(hs, msp, growthLimit)) {
	        LOGE_HEAP("Can't add initial heap");
	        goto fail;
	    }
	    if (!dvmHeapBitmapInit(&hs->liveBits, base, length, "dalvik-bitmap-1")) {
	        LOGE_HEAP("Can't create liveBits");
	        goto fail;
	    }
	    if (!dvmHeapBitmapInit(&hs->markBits, base, length, "dalvik-bitmap-2")) {
	        LOGE_HEAP("Can't create markBits");
	        dvmHeapBitmapDelete(&hs->liveBits);
	        goto fail;
	    }
	    if (!allocMarkStack(&gcHeap->markContext.stack, hs->maximumSize)) {
	        ALOGE("Can't create markStack");
	        dvmHeapBitmapDelete(&hs->markBits);
	        dvmHeapBitmapDelete(&hs->liveBits);
	        goto fail;
	    }
	    gcHeap->markContext.bitmap = &hs->markBits;
	    gcHeap->heapSource = hs;
	
	    gHs = hs;
	    return gcHeap;
	
	fail:
	    munmap(base, length);
	    return NULL;
	}
 
 
 
### dvmAllocRegion()
 
	void *dvmAllocRegion(size_t byteCount, int prot, const char *name) {
	    void *base;
	    int fd, ret;
	
	    byteCount = ALIGN_UP_TO_PAGE_SIZE(byteCount);
	    //1 . allocate a contiguous region by ashmem
	    fd = ashmem_create_region(name, byteCount);
	    if (fd == -1) {
	        return NULL;
	    }
  	    //2 . allocate a contiguous region by mmap
	    base = mmap(NULL, byteCount, prot, MAP_PRIVATE, fd, 0);
	    ret = close(fd);
	    if (base == MAP_FAILED) {
	        return NULL;
	    }
	    if (ret == -1) {
	        return NULL;
	    }
	    return base;
	}
As we have see,Allocates a memory region using ashmem( i remember facebook's glide use ashmem memory) and mmap, initialized to zero.  Actual allocation rounded up to page multiple.  Returns  NULL on failure. 

just like this picture ,when we finish  start up dvm heap source. whole big picture like this one!

![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111953-031642565a5e470d89eaf3435e6676db.png)	 
	 


# start to collect garbage

## dvmCollectGarbageInternal()
 
 	void dvmCollectGarbageInternal(const GcSpec* spec){
 	
	    ... 
	    //1. suspend all threads	    
	    dvmSuspendAllThreads(SUSPEND_FOR_GC);
	
 
	    // 2. raise the priority of gc thread.
	    / /If we are not marking concurrently raise the priority 
	     // of the thread performing the garbage collection.	   	     
	    if (!spec->isConcurrent) {
	        oldThreadPriority = os_raiseThreadPriority();
	    }
	    ...
	    
	     //Begin GC
	    dvmMethodTraceGCBegin();
	
	    /* Set up the marking context.
	     */
	    if (!dvmHeapBeginMarkStep(spec->isPartial)) {
	        LOGE_HEAP("dvmHeapBeginMarkStep failed; aborting");
	        dvmAbort();
	    }
	
	    //3. start Marking
	    //Mark the set of objects that are strongly reachable 
	    //from the roots.  
	    dvmHeapMarkRootSet();
	
	   ... 
	    /*
	     * All strongly-reachable objects have now been marked.  Process
	     * weakly-reachable objects discovered while tracing.
	     */
	    dvmHeapProcessReferences(&gcHeap->softReferences,
	                             spec->doPreserve == false,
	                             &gcHeap->weakReferences,
	                             &gcHeap->finalizerReferences,
	                             &gcHeap->phantomReferences);
	
	
	    //  4. start sweeping.. 
	    dvmHeapSweepSystemWeaks();
	
	    /*
	     * Live objects have a bit set in the mark bitmap, swap the mark
	     * and live bitmaps.  The sweep can proceed concurrently viewing
	     * the new live bitmap as the old mark bitmap, and vice versa.
	     */
	    dvmHeapSourceSwapBitmaps();
	
		....
		//5.  resume all thread 
	   if (spec->isConcurrent) {
	        dvmUnlockHeap();
	        dvmResumeAllThreads(SUSPEND_FOR_GC);
	        dirtyEnd = dvmGetRelativeTimeMsec();
	    }
	    dvmHeapSweepUnmarkedObjects(spec->isPartial, spec->isConcurrent,
	                                &numObjectsFreed, &numBytesFreed);

        //6. Cleaning up... 
	    dvmHeapFinishMarkStep();
	     
	    /* Now's a good time to adjust the heap size, since
	     * we know what our utilization is.
	     *
	     * This doesn't actually resize any memory;
	     * it just lets the heap grow more when necessary.
	     */	     
	    dvmHeapSourceGrowForUtilization();	
	    currAllocated = dvmHeapSourceGetValue(HS_BYTES_ALLOCATED, NULL, 0);
	    currFootprint = dvmHeapSourceGetValue(HS_FOOTPRINT, NULL, 0);
	    ...	
	    
	    //GC finished
	    dvmMethodTraceGCEnd();
 
	    //7. Move queue of pending references back into Java. 
	    dvmEnqueueClearedReferences(&gDvm.gcHeap->clearedReferences);	
    	  ...
	}
	
at firts ,  hold gDvm.threadListLock, if not , it's possible for a thread to be added to the thread list while we work.  The thread should NOT start executing, so this is only interesting when we start chasing thread stacks.  (Before we do so, grab the lock.)	 
We are not allowed to GC when the debugger has suspended the VM, which is awkward because debugger requests can cause allocations.  The easiest way to enforce this is to refuse to GC on an allocation made by the
 JDWP thread -- we have to expand the heap or fail.
[ after mark ]
![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111954-00a3795a8d7e43db84108313c4a64f23.png)
 
 [ after clean ]
 ![enter image description here](http://images.cnitblog.com/blog/49788/201306/12111954-34d7a5cd727a4021a3c60d7203f00c1c.png)


#summary 

谷歌对这个dalvik的改动还是挺大的，从4.+-现在的6+版本,取代为art。
整个目录树都变化不小，一时间都让我找不到北,这篇文章主要看的就是dalvik，后面再学习下art的内容。 不得不说整个过程前面的原理部分相对好理解，看源码部分就看得很吃力，全部是没看过的内容。查了些资料补充了不少知识点，感觉有收获，虽然短时间内没办法在实战中运用起来的感觉，不过会好的，就像看Activity和service的启动过程一样，在后面的看插件化内容帮助不小，最少技多不压身嘛。

关于标记和清晰的部分还是没去详细的贴上来，对于finalizable对象的处理也没细看。但现在有大局，对整体有个先对好点的把握了,后面有空可以细看

# Ref
1. [npc11-improve-Android-UX-with-regional-GC.pdf](https://people.apache.org/~xli/papers/npc11-improve-Android-UX-with-regional-GC.pdf)

2. [Android内存管理原理](http://www.cnblogs.com/killmyday/archive/2013/06/12/3132518.html)

3. [Android DVM memory management research and analysis](http://www.phonesdevelopers.com/1708168/)
 
4.  [Android GC 那点事--QQ空间终端开发团队](http://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400021278&idx=1&sn=0e971807eb0e9dcc1a81853189a092f3&scene=5&srcid=1019lhTQRdJKb9HCqUHPntbT#rd&utm_source=tuicool&utm_medium=referral) 
 
5.  [Android 内存剖析 – 发现潜在问题](http://www.importnew.com/2433.html)
6. [Google I/O 2011: Memory management for Android Apps](https://www.youtube.com/watch?v=_CruQY55HOk) 
 
7. 用到的两个在线源码查看网站 :
	http://androidxref.com/source/xref/
   https://android.googlesource.com/platform/dalvik/+/android-4.2.1_r1.1