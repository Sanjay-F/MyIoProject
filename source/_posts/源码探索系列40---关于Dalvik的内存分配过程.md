title:  源码探索系列40---关于Dalvik的内存分配过程
date: 2016-06-19 12:55:46
tags: [android,dvm,Dalvik]
categories: android

------------------------------------------

我们新建一个对象后，到底我们的虚拟机是怎么分配空间的呢？
我们以前都听说是一些基本数据类型会被放到栈。如果是对象的话，会放到堆里面去，对这个对象的引用是放到栈里面去的。类似下面这张图一样，我们的data这个变量是放在左边的栈，然后实际的内存空间即data引用的这部分，是放在右边的堆里面的。

那么实际是这样吗？真的是这样吗？

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dalvik_%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%A0%86%E4%B8%8E%E6%A0%88%20%281%29.png)

我们带着疑问，来看下dalvik为对象分配内存过程是怎样的，最后来回复这个问题。

<!--more-->

# 起航
API:23，Dalvik：kitkat

在上一篇[源码探索系列38---关于dalvik虚拟机](http://sanjay-f.github.io/2016/06/12/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9738---%E5%85%B3%E4%BA%8Edalvik%E8%99%9A%E6%8B%9F%E6%9C%BA/)有提及到，我们的VM加载类是当遇到下面这么一堆情况时候 ：

1. 建新实例，如New一个，用newInstance()，Clone()，IO的getOjbect()
2. 调用静态方法
3. 操作类或接口中声明的非常量静态字段
4. 调用特定的反射方法
5. 初始化其子类
6. 指定其为虚拟机启动时的初始化类

然后针对的提了遇到new 时候的情况，他会有个操作码`OP_NEW_INSTANCE`，然后去生成。不过在这个过程，我们没针对的去探讨关于内存的分配过程，而是理顺了整个流程。现在我们可以就内存部分开始来说点内容了。

我们知道了遇到new一个对象对应的函数是下面这个`HANDLE_OPCODE(OP_NEW_INSTANCE)`，所以我们来看下其中的内存部分吧。

## HANDLE_OPCODE(OP_NEW_INSTANCE)

处理我们的new对象的代码如下，line1649
	
	/* File: c/OP_NEW_INSTANCE.cpp */
	HANDLE_OPCODE(OP_NEW_INSTANCE /*vAA, class@BBBB*/)
	    {
	        ClassObject* clazz;
	        Object* newObj;
	
	        EXPORT_PC();
	
	        vdst = INST_AA(inst);
	        ref = FETCH(1);
	        ILOGV("|new-instance v%d,class@0x%04x", vdst, ref);
	        clazz = dvmDexGetResolvedClass(methodClassDex, ref);
	        if (clazz == NULL) {
	            clazz = dvmResolveClass(curMethod->clazz, ref, false);
	            if (clazz == NULL)
	                GOTO_exceptionThrown();
	        }
	
	        if (!dvmIsClassInitialized(clazz) && 
	            !dvmInitClass(clazz))                
	            GOTO_exceptionThrown();
	
	#if defined(WITH_JIT)
	 ...不看JIT的内容
	#endif
	...
	        //我们关注的一句话
	        newObj = dvmAllocObject(clazz, ALLOC_DONT_TRACK);
	        if (newObj == NULL)
	            GOTO_exceptionThrown();
	        SET_REGISTER(vdst, (u4) newObj);
	    }
	    FINISH(2);
	OP_END

在这里，真正分配内存的操作dvmAllocObject，我们看下参数，左边的`clazz`，就是我们想创建的对象类，前面的一大段代码做的内容，就是帮我们初始化这个变量。【嗯，下一次或许得看下是否应该写下关于class结构的内容，不过感觉也还不是很有用处，除了面试时候.....】

另外一个参数`ALLOC_DONT_TRACK`是告诉Dalvik虚拟机的堆管理器，要分配的对象是一个根集对象，不需要对它进行跟踪。因为根集对象在GC时是会自动被追踪处理的。一般我们的根对象有这么几种

1. 栈中引用的对象，引用是在栈帧中的本地变量表中的，真正的对象在堆中

2. 方法区perm中的类静态属性引用的对象，以及常量引用的对象

3. 本地方法栈中JNI（Native方法）的引用的对象

然后，我们继续看下我们的关注内容dvmAllcoObject

##  Alloc.dvmAllcoObject()
文件路径：dalvik/vm/alloc/Alloc.cpp

	 /*
		 * Create an instance of the specified class.
		 *
		 * Returns NULL and throws an exception on failure.
		 */
	Object* dvmAllocObject(ClassObject* clazz, int flags)
	{
	    Object* newObj;
	
	    assert(clazz != NULL);
	    assert(dvmIsClassInitialized(clazz) || dvmIsClassInitializing(clazz));
	
	    /* allocate on GC heap; memory is zeroed out */
	    newObj = (Object*)dvmMalloc(clazz->objectSize, flags);
	    if (newObj != NULL) {
	        DVM_OBJECT_INIT(newObj, clazz);
	        dvmTrackAllocation(clazz, clazz->objectSize);   /* notify DDMS */
	        // Add barrier to force all metadata writes to main memory to complete
	        ANDROID_MEMBAR_FULL();
	    }
	
	    return newObj;
	}

在这里，我们看到了句官方的注释，在GC heap上给我们申请clazz->objectSize大小的内存空间给我们的这个newObj使用。如果创建成功，就调用DVM_OBJECT_INIT()来初始化我们的变量，把它和clazz类关联起来。因为这个
dvmMalloc返回的实际就是一块我们制定大小（clazz->objectSize）的内存地址而已，至于要怎么用这块空间，就由我们决定了。所以我们把它这我们要new的那个clazz关联在一起。

。我们知道，一个DVM加载类会有三个步骤：

> 装载-> 连接( 验证->准备->解析 )->初始化

装载就是上一个函数做了，这个函数看起来不就是做了连接的工作嘛？背后这 dvmTrackAllocation做了什么，我们后面再看。
接着是通知DDMS，用dvmTrackAllocation()记录我们的这个新的内存分配信息。 最后是加barrier从而强制所有元数据写到主内存去完成。
 
### dvmMalloc()

vm/alloc/heap.cpp


	void* dvmMalloc(size_t size, int flags)
	{
	    void *ptr;
	
	    dvmLockHeap();
	   	 /* Try as hard as possible to allocate some memory.
	   	 */
	    ptr = tryMalloc(size);
	    if (ptr != NULL) {   
	        /* We've got the memory.
	         */
	        if (gDvm.allocProf.enabled) {
	            Thread* self = dvmThreadSelf();
	            gDvm.allocProf.allocCount++;
	            gDvm.allocProf.allocSize += size;
	            if (self != NULL) {
	                self->allocProf.allocCount++;
	                self->allocProf.allocSize += size;
	            }
	        }
	    } else {
	        /* The allocation failed.
	         */
	
	        if (gDvm.allocProf.enabled) {
	            Thread* self = dvmThreadSelf();
	            gDvm.allocProf.failedAllocCount++;
	            gDvm.allocProf.failedAllocSize += size;
	            if (self != NULL) {
	                self->allocProf.failedAllocCount++;
	                self->allocProf.failedAllocSize += size;
	            }
	        }
	    }
	
	    dvmUnlockHeap();
	
	    if (ptr != NULL) {
	        /*
	         * If caller hasn't asked us not to track it, add it to the
	         * internal tracking list.
	         */
	        if ((flags & ALLOC_DONT_TRACK) == 0) {
	            dvmAddTrackedAlloc((Object*)ptr, NULL);
	        }
	    } else {
	        /*
	         * The allocation failed; throw an OutOfMemoryError.
	         */
	        throwOOME();
	    }
	
	    return ptr;
	}

来，我们来看下这个函数，开头先去锁住堆，然后就去申请size大小的内存，看起来这个申请过程很不容易啊，人家都说恶劣try as hard as possible。确实很不容易，背后有很多的逻辑，后面我们再提及。继续看下流程。
在去malloc空间回来后，如果成功，不为空的话，就记录这次信息，失败也记录下。
接着解锁.因我们在`HANDLE_OPCODE()`传的flag是`ALLOC_DONT_TRACK`，所以我们这里不会跑去加到内部Tracking list.
倒是如果前面申请失败，会抛给我们一个OOME的bug。呵，这个我们遇到比较多点，传闻中的OOM啊！大名鼎鼎的内存溢出bug。

好了，看完流程，我们来看下这个  
####  tryMalloc(size)
	
	static void *tryMalloc(size_t size)
	{
		...
	    char* hprof_file = NULL;
	    void *ptr;
	    int result = -1;
	    int debug_oom = 0;
		
		//step 1
	    ptr = dvmHeapSourceAlloc(size);
	    if (ptr != NULL) {
	        return ptr;
	    }
	    
       //step 2	    
	    if (gDvm.gcHeap->gcRunning) {	         
	        dvmWaitForConcurrentGcToComplete();
	    } else {	       
	      gcForMalloc(false);
	    }	
	    ptr = dvmHeapSourceAlloc(size);
	    if (ptr != NULL) {
	        return ptr;
	    }
	 
		//STEP 3
	    ptr = dvmHeapSourceAllocAndGrow(size);
	    if (ptr != NULL) {
	        size_t newHeapSize;	
	        newHeapSize = dvmHeapSourceGetIdealFootprint();
	        ...
 	        return ptr;
	    }
	    
	   //STEP 4	    
	    /* Most allocations should have succeeded by now, so the heap
	     * is really full, really fragmented, or the requested size is
	     * really big.  Do another GC, collecting SoftReferences this
	     * time.  The VM spec requires that all SoftReferences have
	     * been collected and cleared before throwing an OOME.
	     */	     
 	    LOGI_HEAP("Forcing collection of SoftReferences for %zu-byte allocation",
	            size);
	    gcForMalloc(true);
	    ptr = dvmHeapSourceAllocAndGrow(size);
	    if (ptr != NULL) {
	        return ptr;
	    }
	    
	     dvmDumpThread(dvmThreadSelf(), false);	
		...	
	    if(debug_oom == 1) {
	       ...
	    }
	    
	    return NULL;
	}

我们来看下，这个调用流程，主要分了四步走。

1. 先直接调用dvmHeapSourceAlloc去申请空间，如果成功，就返回，失败就继续step2
2.  前面一步没申请到，就进行GC，如果当前GC正在进行就等下，没有就触发，同时GC的参数为`false`，表示对**软引用**的对象**还不要回收**。然后再调用一次`dvmHeapSourceAlloc()`去申请，成功则返回。失败再走第三部
3.  如果GC完还失败，那就调用`dvmHeapSourceAllocAndGrow(）`，尝试来增长下我们的heap空间，然后再返回一个给我们。基本上到这一步理论都成功了。失败了再走第四步咯
4.  到了这步，系统会回收软引用`SoftRef`的变量空间，因为我们看到GC参数为true，在GC完后，就再次调用`dvmHeapSourceAllocAndGrow（）`去申请下空间。到这里如果还失败，那没办法，只能给你个NULL了，这回导致上层`dvmMalloc()`抛出OOM咯！

#### dvmHeapSourceAlloc
dalvik/vm/alloc/HeapSource.cpp

我们来看下这个dvmHeapSourceAlloc是怎么给我们申请内存空间的

	  

	void* dvmHeapSourceAlloc(size_t n)
	{
	    HS_BOILERPLATE();
	
	    HeapSource *hs = gHs;
	    Heap* heap = hs2heap(hs);//这是一个宏,长下面这样，用来获得头位置的
	    //#define hs2heap(hs_) (&((hs_)->heaps[0]))
	    
	    if (heap->bytesAllocated + n > hs->softLimit) {
	        /*
	         * This allocation would push us over the soft limit; 
	         * act as if the heap is full.
	         */
	        LOGV_HEAP("softLimit of %zd.%03zdMB hit for %zd-byte allocation",
	                  FRACTIONAL_MB(hs->softLimit), n);
	        return NULL;
	    }
	    
开头地方我们看到了gHs，它是一个全局变量，指向一个HeapSource结构。
开头判断下，如果我们已调用的大小（heap->bytesAllocated）加多新调用的大小N，是否超过了软限制（softLimit），超了就返回NULL。

关于这个HeapSocure的几个字段含义，可以看下他的结构定义，长下面这样：
 

		struct HeapSource {
		
	    // Target ideal heap utilization ratio; range 1..HEAP_UTILIZATION_MAX 
	    size_t targetUtilization;
	      
	    size_t startSize;	    // The starting heap size.
	
	    // The largest that the heap source as a whole is allowed to grow.	      
	    size_t maximumSize;
	
	    /*
	     * The largest size we permit the heap to grow.  This value allows
	     * the user to limit the heap growth below the maximum size.  This
	     * is a work around until we can dynamically set the maximum size.
	     * This value can range between the starting size and the maximum
	     * size but should never be set below the current footprint of the
	     * heap.
	     */
	    size_t growthLimit;
	
	    // The desired max size of the heap source as a whole.	     
	    size_t idealSize;
	
	    /* The maximum number of bytes allowed to be allocated from the
	     * active heap before a GC is forced.  This is used to "shrink" the
	     * heap in lieu of actual compaction.
	     */
	    size_t softLimit;
	
	    /* Minimum number of free bytes. Used with the target utilization when
	     * setting the softLimit. Never allows less bytes than this to be free
	     * when the heap size is below the maximum size or growth limit.
	     */
	    size_t minFree;
	
	    /* Maximum number of free bytes. Used with the target utilization when
	     * setting the softLimit. Never allows more bytes than this to be free
	     * when the heap size is below the maximum size or growth limit.
	     */
	    size_t maxFree;
	
	
	    // The heaps; heaps[0] is always the active heap,
	    //which new objects should be allocated from.	       
	    Heap heaps[HEAP_SOURCE_MAX_HEAP_COUNT];
	
	    // The current number of heaps.	      
	    size_t numHeaps;
	
	    // True if zygote mode was active when the HeapSource was created.	      
	    bool sawZygote;
	
	 
	    // The base address of the virtual memory reservation. 
	    char *heapBase;
	
 
	     // The length in bytes of the virtual memory reservation. 
	    size_t heapLength;
  
	     //The live object bitmap. 
	    HeapBitmap liveBits;
	
 
	     // The mark bitmap. 
	    HeapBitmap markBits;
 
 	     //Native allocations. 
	    int32_t nativeBytesAllocated;
	    size_t nativeFootprintGCWatermark;
	    size_t nativeFootprintLimit;
	    bool nativeNeedToRunFinalization;
		   
	     //State for the GC daemon. 
	    bool hasGcThread;
	    pthread_t gcThread;
	    bool gcThreadShutdown;
	    pthread_mutex_t gcThreadMutex;
	    pthread_cond_t gcThreadCond;
	    bool gcThreadTrimNeeded;
	};


然后我们看到我们的softLimit是写着：The maximum number of bytes allowed to be allocated from the  active heap before a GC is forced。根据这个注释，我们知道，这是我们的`active堆`在GC前允许被调用的大小。

这个值得意义就是自己说的那样，是我们在GC前允许被调用的大小，这样我们的程序在上面那个函数得到NULL后，就会**触发GC**，然后再去申请空间。
即利用这个`softLimit`来帮助我们合理的管理内存空间，避免内存碎片化问题过于严重，把内存利用率提高些，避免我们无限制的内存调用。
所以，在每次GC后，DVM会根据`Active堆`已经分配的内存字节数、设定的堆目标利用率和Zygote堆的大小，重新计算`Soft Limit`。


我们再回顾下这个判断

	 if (heap->bytesAllocated + n > hs->softLimit) {	 
	        return NULL;
	    }

之所有如果超了就返回NULL，不允许进行增加内存，我想这大概就是这套内存管理逻辑中的一环，它更倾向于保持内存合理的使用。
另外的dvmHeapSourceAllocAndGrow函数才是负责grow的。

继续看后面的。在还有空间的时候需要分是否为lowMemoryMode来做处理。

	 void* ptr;
	    if (gDvm.lowMemoryMode) {
	        /* This is only necessary because mspace_calloc always memsets the
	         * allocated memory to 0. This is bad for memory usage since it leads
	         * to dirty zero pages. If low memory mode is enabled, we use
	         * mspace_malloc which doesn't memset the allocated memory and madvise
	         * the page aligned region back to the kernel.
	         */
	        ptr = mspace_malloc(heap->msp, n);
	        if (ptr == NULL) {
	            return NULL;
	        }
	        uintptr_t zero_begin = (uintptr_t)ptr;
	        uintptr_t zero_end = (uintptr_t)ptr + n;
	        
	        // Calculate the page aligned region.	         
	        uintptr_t begin = ALIGN_UP_TO_PAGE_SIZE(zero_begin);
	        uintptr_t end = zero_end & ~(uintptr_t)(SYSTEM_PAGE_SIZE - 1);
	        
	        /* If our allocation spans more than one page, we attempt to madvise.
	         */
	        if (begin < end) {
	        
	            //重要的一句调用
	            // madvise the page aligned region to kernel. 
	            madvise((void*)begin, end - begin, MADV_DONTNEED);
				
	            // Zero the region after the page aligned region. 
	            memset((void*)end, 0, zero_end - end);
				//调用系统接口madvice时，指定的内存地址以及内存大小都必须以
				//页大小为边界的，但是函数mspace_malloc分配出来的内存的地址
				//只能保证对齐到8个字节，因此，我们是有可能不能将所有分配出
				//来的内存都通过系统接口madvice标记为MADV_DONTNEED的。
				//这时候对于不能标记为MADV_DONTNEED的内存，
				//因此需调用memset来将它们初始化为0。
	            
	            // Zero out the region before the page aligned region.	            
	            zero_end = begin;
	        }
	        
	        //开头的注释有说到，因为这`mspace_malloc`分配的内存是不会把
	        // 内存初始化为0的，而DVM又要求分配给对象的内存初始化为0,
	        //所以我们得手动调用下这个
	        memset((void*)zero_begin, 0, zero_end - zero_begin);
	    } else {
	        ptr = mspace_calloc(heap->msp, 1, n);
	        if (ptr == NULL) {
	            return NULL;
	        }
	    }
	
我们看到，在低内存模式中，如果我们调用的内存对象大小是超过一个page大小的，DVM假设对象不会马上就使用分配到的内存，因此，它就通过系统接口`madvice（）`和`MADV_DONTNEED标志`，告诉内核刚刚分配出去的内存在**近期内不会使用**，内核可以对该内存做对应的物理页回收。如果分配出去的内存被使用，则内核会重新给它映射物理页，这样就可以做到按需分配物理内存。达到适合在内存小的设备上运行的特点。

对于非低内存模式，就相对简单了，直接调用`mspace_calloc（）`，然后不为空就返回。因为他相对`mspace_malloc（）`好处就是还会自动帮我们置0；

然后我们继续看下后面的内容：
	
	    countAllocation(heap, ptr); 
        //有在gc就返回，没有就触发一次GC然后再返回
	    if (gDvm.gcHeap->gcRunning || !hs->hasGcThread) { 
	        return ptr;
	    }
	    if (heap->bytesAllocated > heap->concurrentStartBytes){	 
		    //通知GC线程执行一次Concurrent GC
	        dvmSignalCond(&gHs->gcThreadCond);
	    }
	    return ptr;
	} 
   
	/*
	 * Functions to update heapSource->bytesAllocated when an object
	 * is allocated or freed.  mspace_usable_size() will give
	 * us a much more accurate picture of heap utilization than
	 * the requested byte sizes would.
	 *
	 * These aren't exact, and should not be treated as such.
	 */
	static void countAllocation(Heap *heap, const void *ptr)
	{
	    assert(heap->bytesAllocated < mspace_footprint(heap->msp));
	
	    heap->bytesAllocated += mspace_usable_size(ptr) +
	            HEAP_SOURCE_CHUNK_OVERHEAD;
	    heap->objectsAllocated++;
	    HeapSource* hs = gDvm.gcHeap->heapSource;
	    dvmHeapBitmapSetObjectBit(&hs->liveBits, ptr);
	
	    assert(heap->bytesAllocated < mspace_footprint(heap->msp));
	}
	
这个countAllocation主要是记录了`Active堆`已分配的字节数和对象数。
然后调用`dvmHeapBitmapSetObjectBit（）`函数将新分配的对象在`Live Heap Bitmap`上对应的位设置为**1**，也就是说将新创建的对象标记为是**存活的**。关于Live Heap Bitmap注释可以看前面的结构提到的。


到这里，这个函数基本就到这里了，我们来看下另外一个倾向于grow的函数

#### dvmHeapSourceAllocAndGrow（）
dalvik/vm/alloc/HeapSource.cpp

	/*
	 * Allocates <n> bytes of zeroed data, growing as much as possible
	 * if necessary.
	 */
	void* dvmHeapSourceAllocAndGrow(size_t n)
	{
	    HS_BOILERPLATE();
	
	    HeapSource *hs = gHs;
	    Heap* heap = hs2heap(hs);

		  // STEP 1	    
	    //复用他来负责申请空间，失败了才进行grow；
	    void* ptr = dvmHeapSourceAlloc(n);
	    if (ptr != NULL) {
	        return ptr;
	    }
	
		  //STEP 2
	    size_t oldIdealSize = hs->idealSize;
	    if (isSoftLimited(hs)) {   
	    
	        /* We're soft-limited.  Try removing the soft limit to
	         * see if we can allocate without actually growing.
	         */
	        hs->softLimit = SIZE_MAX;
	        ptr = dvmHeapSourceAlloc(n);
	        if (ptr != NULL) {
	            /* Removing the soft limit worked;  fix things up to
	             * reflect the new effective ideal size.
	             */
	            snapIdealFootprint();
	            return ptr;
	        }
	        // softLimit intentionally left at SIZE_MAX.
	    }
	
	  //STEP 3
	    /* We're not soft-limited.  Grow the heap to satisfy the request.
	     * If this call fails, no footprints will have changed.
	     */
	    ptr = heapAllocAndGrow(hs, heap, n);
	    if (ptr != NULL) {
	        /* The allocation succeeded.  Fix up the ideal size to
	         * reflect any footprint modifications that had to happen.
	         */
	        snapIdealFootprint();
	    } else {
	        /* We just couldn't do it.  Restore the original ideal size,
	         * fixing up softLimit if necessary.
	         */
	        setIdealFootprint(oldIdealSize);
	    }
	    return ptr;
	}

	 
我们看上面主要分了三步

1. 直接调用dvmHeapSourceAlloc去申请，因为这个之前有GC过，有可能有足够的空间。如果失败，再继续第二部。
2. 记录下oldIdealSize大小，用于后面恢复用。尝试移去软限制，来看下是否可以，如果行就做些恢复操作，然后返回。不行再进行第三步
3. 这时候还不行就只能真的涨空间了，因此调用了`heapAllocAndGrow`来做最后的处理。

知道个流程，我们看下其中几个细节点的意思


##### isSoftLimited()

		/*
	 * Returns true iff a soft limit is in effect for the active heap.
	 */
	static bool isSoftLimited(const HeapSource *hs)
	{
	    /* softLimit will be either SIZE_MAX or the limit for the
	     * active mspace.  idealSize can be greater than softLimit
	     * if there is more than one heap.  If there is only one
	     * heap, a non-SIZE_MAX softLimit should always be the same
	     * as idealSize.
	     */
	    return hs->softLimit <= hs->idealSize;
	}

在前面heapSoucre结构里有这两个的含义，`hs->softLimit`指`Active堆`的大小，而`hs->idealSize`指的是**Zygote堆**和**Active堆**的大小之和。
之所以有这么个判断，是因为存在可能没有zygote堆，然后两者一直相等的情况。

1. 当只有一个堆时，即只有Active堆时，如果设置了Soft Limit，那么它的大小总是等于Active堆的大小，即这时候hs->softLimit总是等于hs->idealSize。如果没有设置Soft Limit，那么它的值会被设置为SIZE_MAX值，这会就会保证hs->softLimit大于hs->idealSize。也就是说，当只有一个堆时，函数isSoftLimited能正确的反映Active堆是否设置有Soft Limit。

2. 当有两个堆时，即Zygote堆和Active堆同时存在，那么如果设置有Soft Limit，那么它的值就总是等于Active堆的大小。
由于hs->idealSize描述的是Zygote堆和Active堆的大小之和，因此就一定可以保证hs->softLimit小于等于hs->idealSize。如果没有设置Soft Limit，即hs->softLimit的值等于SIZE_MAX，那么就一定可以保证hs->softLimit的值大于hs->idealSize的值。也就是说，当有两个堆时，函数isSoftLimited也能正确的反映Active堆是否设置有Soft Limit。

##### heapAllocAndGrow
	 
	/* Remove any hard limits, try to allocate, and shrink back down.
	 * Last resort when trying to allocate an object.
	 */
	static void* heapAllocAndGrow(HeapSource *hs, Heap *heap, size_t n)
	{
	    /* Grow as much as possible, but don't let the real footprint
	     * go over the absolute max.
	     */
	    size_t max = heap->maximumSize;
	
	    mspace_set_footprint_limit(heap->msp, max);
	    void* ptr = dvmHeapSourceAlloc(n);
	
	    /* Shrink back down as small as possible.  Our caller may
	     * readjust max_allowed to a more appropriate value.
	     */
	    mspace_set_footprint_limit(heap->msp,
	                               mspace_footprint(heap->msp));
	    return ptr;
	} 

我们看这个函数先将`Active堆`（heap->msp）的大小设置为允许的最大值（heap->maximumSize），接着再调`dvmHeapSourceAlloc`去分配大小为n的内存。然后用`mspace_footprint（）`获得分配了n个字节之后Active堆的大小，并且将该值设置为Active堆的当前大小限制。
这就相当于是将Active堆的当前大小限制值从允许设置的最大值减少为一个刚刚合适的值。

至此，我们把整个的内存申请过程都看了下了！！！


### DVM_OBJECT_INIT()

 VM/oo/object.h

稍微回过头来看下开头那里的这个做初始化的宏

		/*
	 * Properly initialize an Object.
	 * void DVM_OBJECT_INIT(Object *obj, ClassObject *clazz_)
	 */
	#define DVM_OBJECT_INIT(obj, clazz_) \
	    dvmSetFieldObject(obj, OFFSETOF_MEMBER(Object, clazz), clazz_)

这里我们稍微提下obj这个对象，这个结构也在这个文件里面。

		/*
	 * There are three types of objects:
	 *  Class objects - an instance of java.lang.Class
	 *  Array objects - an object created with a "new array" instruction
	 *  Data objects - an object that is neither of the above
	 *
	 * We also define String objects.  At present they're equivalent to
	 * DataObject, but that may change.  (Either way, they make some of the
	 * code more obvious.)
	 *
	 * All objects have an Object header followed by type-specific data.
	 */
	struct Object {
	    /* ptr to class object */
	    ClassObject*    clazz;
	
	    /*
	     * A word containing either a "thin" lock or a "fat" monitor.  See
	     * the comments in Sync.c for a description of its layout.
	     */
	    u4              lock;
	};

我们看Object类只有两个成员变量：clazz和lock。

1.  clazz的类型为ClassObject，它对应于Java层的`java.lang.Class`类，用来描述对象所属的类。
2. lock是一个锁，正是因为有了这个成员变量，在Java层中，每一个对象都可以当锁使用。

 然后这个对应的函数
 
#### dvmSetFieldObject()
vm/oo/objectInlines.h

	INLINE void dvmSetFieldObject(Object* obj, int offset, Object* val) {
    JValue* lhs = (JValue*)BYTE_OFFSET(obj, offset);
    lhs->l = val;
    if (val != NULL) {
        dvmWriteBarrierField(obj, &lhs->l);
    }
} 

嗯，背后的内容就很多，我们暂时点到为止。
先留一条线索在这里，下次写DVM关于类的的加载部分时候，再来细入的谈。
   
### dvmTrackAllocation()
VM\AllocTracker.cpp
 
 这个宏的定义在AllocTracker.H文件中。然后我们看下对应的实际实现
 
	#define dvmTrackAllocation(_clazz, _size)                                   \
    {                                                                       \
        if (gDvm.allocRecords != NULL)                                      \
            dvmDoTrackAllocation(_clazz, _size);                            \
    }
	void dvmDoTrackAllocation(ClassObject* clazz, size_t size);


#### dvmDoTrackAllocation()

	/*
	 * Add a new allocation to the set.
	 */
	void dvmDoTrackAllocation(ClassObject* clazz, size_t size)
	{
	    Thread* self = dvmThreadSelf();
	    if (self == NULL) {
	        ALOGW("alloc tracker: no thread");
	        return;
	    }
	
	    dvmLockMutex(&gDvm.allocTrackerLock);
	    if (gDvm.allocRecords == NULL) {
	        dvmUnlockMutex(&gDvm.allocTrackerLock);
	        return;
	    }
	
	    /* advance and clip */
	    if (++gDvm.allocRecordHead == gDvm.allocRecordMax)
	        gDvm.allocRecordHead = 0;
	
	    AllocRecord* pRec = &gDvm.allocRecords[gDvm.allocRecordHead];
	
	    pRec->clazz = clazz;
	    pRec->size = size;
	    pRec->threadId = self->threadId;
	    getStackFrames(self, pRec);
	
	    if (gDvm.allocRecordCount < gDvm.allocRecordMax)
	        gDvm.allocRecordCount++;
	
	    dvmUnlockMutex(&gDvm.allocTrackerLock);
	}

关于这部分，暂时也不做深入，留一个线索在这里，以后再做对应的深入记录。


# 小结

看完这部分内容，我们可以确定的是，我们的调用的对象，他的实际内存空间确实在堆里面，而且是放在Active堆里面的。

不过关于这个堆的创建过程和划分等内容，想必有些人看了怪怪的，所以下面可以考录来写下这个HeapStruct的内容。

# ref

1. http://blog.csdn.net/luoshengyang/article/details/41688319

============再次更新

2016/10/27
最近看到微信开发分享的一篇文章，觉得不错，就贴在这
[Android内存申请分析]{http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286327&idx=1&sn=b69513e3dfd1de848daefe03ab6719c2&scene=26#wechat_redirect}
