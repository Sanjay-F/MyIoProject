title:  源码探索系列41---关于DVM的Heap堆创建过程
date: 2016-06-22 12:54:46
tags: [android,dvm,Dalvik,heap]
categories: android

------------------------------------------

在上一篇的[关于Dalvik的内存分配过程](http://sanjay-f.github.io/2016/06/19/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9740---%E5%85%B3%E4%BA%8EDalvik%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E8%BF%87%E7%A8%8B/)，我们简单的看了下`new`一个对象时候的内存分配情况，期间我们多次的看到了HeapStruce的东西，他是负责存我们的Heap信息的，其中一些内容还是挺一笔带过的，这里来做更进一步的了解。

<!--more-->
# 起航
API:23, dalvik:kitkat

我们的手机程序初始于Zygote，它创建并启动了第一个虚拟机。 

这一切都是从AndroidRuntime::startVm开始说去的...

经历从startVm->Jni.JNI_CreateJavaVM()---->Init.dvmStartup()-->Alloc.dvmGcStartup.

这样到了我们今天的第一个要开始讨论的函数dvmGcStartUp()，我们的堆就是从这里开始创建的，因为我们的堆很大程度也是为了GC做服务的，所以这个函数有着GcStartUp的名字。

## Alloc.dvmGcStartup
vm/alloc/Alloc.cpp

	/*
	 * Initialize the GC universe.
	 *
	 * We're currently using a memory-mapped arena to keep things off of the
	 * main heap.  This needs to be replaced with something real.
	 */
	bool dvmGcStartup()
	{
	    dvmInitMutex(&gDvm.gcHeapLock);
	    pthread_cond_init(&gDvm.gcHeapCond, NULL);
	    return dvmHeapStartup();
	}

我们看开头就是初始化一个mutex锁和一个条件变量。我们后面的一些竞争条件会用到他们来做些处理。然后是dvmHeapStartup来做进一步操作

## Heap.dvmHeapStartup
dalvik/vm/alloc/Heap.cpp

 
	bool dvmHeapStartup(){
	
	    GcHeap *gcHeap;
	
	    if (gDvm.heapGrowthLimit == 0) {
	        gDvm.heapGrowthLimit = gDvm.heapMaximumSize;
	    }	
	    gcHeap = dvmHeapSourceStartup(gDvm.heapStartingSize,
	                                  gDvm.heapMaximumSize,
	                                  gDvm.heapGrowthLimit);
	    if (gcHeap == NULL) {
	        return false;
	    }
	    
	    ...
	    gDvm.gcHeap = gcHeap;
	
	 //Set up the lists we'll use for cleared reference objects.	    　
	    gcHeap->clearedReferences = NULL;
	
	    if (!dvmCardTableStartup(gDvm.heapMaximumSize, gDvm.heapGrowthLimit)) {
	        LOGE_HEAP("card table startup failed.");
	        return false;
	    }
	
	    return true;
	}

这函数开头去用`dvmHeapSourceStartup（）`来生成一个heap，然后赋值给我们的gDvm变量。这个在前面也有接触过了，一个全局变量，记录了Dalvik虚拟机的各种信息。我们看下这三个参数的含义：

1. heapStartingSize，即堆启动时候的默认物理内存大小，通过-Xms选项可以做设置，默认为4M。当我们需要更多的时候，才继续增大对的大小，是按需分派的策略，最多一直可以增大到默认的16M（HeapMaximumSize）大小，当然现在很多手机厂商都会做不一样的调整，特别是在有4GB内存的现在。
2. HeapMaximumSize，这个就是堆能够增大到的最大大小，通过-Xmx选项可以做设置，默认为16M。
3. heapGrowthLimit，辅助用于动态调整Java堆的可用最大值，通过-XX:HeapGrowthLimit选项可以做设置。作用类似于上一篇提到的softLimit的作用。上面那个最大值是物理内存可以达到的上限，这个是每次从起始的默认值开始涨的每次增加的内存大小限制值。


然后再通过dvmCardTableStartup来为我们刚才这个heap堆创建一个Card Table。

### HeapSource.dvmHeapSourceStartup

/vm/alloc/HeapSource.cpp


	GcHeap* dvmHeapSourceStartup(size_t startSize, size_t maximumSize,size_t growthLimit){
	
	    GcHeap *gcHeap;
	    HeapSource *hs;
	    mspace msp;
	    size_t length;
	    void *base;
	
	    assert(gHs == NULL);
	
		//这个初始配置还不可以随意配置哈，有大小关系的
	    if (!(startSize <= growthLimit && growthLimit <= maximumSize)) {
	        ALOGE("Bad heap size parameters (start=%zd, max=%zd, limit=%zd)",
	             startSize, maximumSize, growthLimit);
	        return NULL;
	    }
	
	//STEP 1.
	    /*
	     * Allocate a contiguous region of virtual memory to subdivided
	     * among the heaps managed by the garbage collector.
	     */
	    length = ALIGN_UP_TO_PAGE_SIZE(maximumSize);
	    base = dvmAllocRegion(length, PROT_NONE, gDvm.zygote ? "dalvik-zygote" : "dalvik-heap");
	    if (base == NULL) {
	        return NULL;
	    }
	
我们看他先把配置的maxinumSize给对其到页边界的大小，然后才是通过函数dvmAllocRegion来分配一块length大小的ASM内存块。这个就是我们的DVM的堆，在后面的赋值到hs->heapBase可以看出来。

		//STEP 2		
	    /* Create an unlocked dlmalloc mspace to use as
	     * a heap source.
	     */
	    msp = createMspace(base, kInitialMorecoreStart, startSize);
	    if (msp == NULL) {
	        goto fail;
	    }
	
用createMspace来将前面的分配的那块ASM封装为一个mspace，他的大小就是其实的JAVA堆的起始大小（startSize），在开头的判断条件已经告诉我们这大小是小于等于最大大小（maximumSize），因此能确保奉陪成功。
	
	    gcHeap = (GcHeap *)calloc(1, sizeof(*gcHeap));
	    if (gcHeap == NULL) {
	        LOGE_HEAP("Can't allocate heap descriptor");
	        goto fail;
	    }
	
	    hs = (HeapSource *)calloc(1, sizeof(*hs));
	    if (hs == NULL) {
	        LOGE_HEAP("Can't allocate heap source");
	        free(gcHeap);
	        goto fail;
	    }
	    
gcHeap和hs是用来维护堆信息的，包括Java堆的目标利用率、最小空闲值、最大空闲值、起始大小、最大值、增长上限值、堆个数、起始地址和大小等等内容
	
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
	    hs->nativeBytesAllocated = 0;
	    hs->nativeFootprintGCWatermark = startSize;
	    hs->nativeFootprintLimit = startSize * 2;
	    hs->nativeNeedToRunFinalization = false;
	    hs->hasGcThread = false;
	    hs->heapBase = (char *)base; //在这里赋值了堆内存
	    hs->heapLength = length;
	
	    if (hs->maxFree > hs->maximumSize) {
	      hs->maxFree = hs->maximumSize;
	    }
	    if (hs->minFree < concurrentStart) {
	      hs->minFree = concurrentStart;
	    } else if (hs->minFree > hs->maxFree) {
	      hs->minFree = hs->maxFree;
	    }
	
		//STEP 3
	    if (!addInitialHeap(hs, msp, growthLimit)) {
	        LOGE_HEAP("Can't add initial heap");
	        goto fail;
	    }
用`addInitialHeap`在前面分配的`msp`上创建一个`Active堆`。这个Active堆的最大值被设置为Java堆的起始大小。


	    if (!dvmHeapBitmapInit(&hs->liveBits, base, length, "dalvik-bitmap-1")) {
	        LOGE_HEAP("Can't create liveBits");
	        goto fail;
	    }
	    if (!dvmHeapBitmapInit(&hs->markBits, base, length, "dalvik-bitmap-2")) {
	        LOGE_HEAP("Can't create markBits");
	        dvmHeapBitmapDelete(&hs->liveBits);
	        goto fail;
	    }
调用函数dvmHeapBitmapInit创建和初始化一个`Live Bitmap（liveBits）`和一个`Mark Bitmap(markBits)`，它们在GC时会用得到。而且大有乾坤，通过两者对应位置的0与1来确定那个对象是可以被回收的。

	    if (!allocMarkStack(&gcHeap->markContext.stack, hs->maximumSize)) {
	        ALOGE("Can't create markStack");
	        dvmHeapBitmapDelete(&hs->markBits);
	        dvmHeapBitmapDelete(&hs->liveBits);
	        goto fail;
	    }
调用函数allockMarkStack创建和初始化一个`Mark Stack`，也是在GC时会用到的。如果失败就跳转到`fail`这个label去。当时课本建议不要用的`goto`又再次看到了，哈哈。


	    gcHeap->markContext.bitmap = &hs->markBits;
	    gcHeap->heapSource = hs;
	
	    gHs = hs;
	    return gcHeap;
	
	fail:
	    munmap(base, length);
	    return NULL;
	}	


我们先来看下STEP 1 这个内存空间怎么通过dvmAllocRegion来申请的。

#### Misc.dvmAllocRegion
vm/Misc.cpp
调用语句
	base = dvmAllocRegion(length, PROT_NONE, gDvm.zygote ? "dalvik-zygote" : "dalvik-heap");
	
	
	void *dvmAllocRegion(size_t byteCount, int prot, const char *name) {  
	    void *base;  
	    int fd, ret;  
	  
	    byteCount = ALIGN_UP_TO_PAGE_SIZE(byteCount);  
	    fd = ashmem_create_region(name, byteCount);  
	    if (fd == -1) {  
	        return NULL;  
	    }  
	    base = mmap(NULL, byteCount, prot, MAP_PRIVATE, fd, 0);  
	    ret = close(fd);  
	    if (base == MAP_FAILED) {  
	        return NULL;  
	    }  
	    if (ret == -1) {  
	        munmap(base, byteCount);  
	        return NULL;  
	    }  
	    return base;  
	}  
首先，通过`ashmem_create_region()`去建一块ASM的内存块。然后用`mmap()`建立一个映射关系，用于返回给上层调用。
倒是这个对齐函数ALIGN_UP_TO_PAGE_SIZE(）给我一种多余的感觉，前面已经对其一次，这里再调用一次。


#### HeapSource.createMspace()
vm/alloc/HeapSource.cpp
调用语句 `msp = createMspace(base, kInitialMorecoreStart, startSize);`

	
	static mspace createMspace(void* begin, size_t morecoreStart, size_t startingSize)
	{
	    // Clear errno to allow strerror on error.
	    errno = 0;
	    
	    // Allow access to inital pages that will hold mspace.
	    mprotect(begin, morecoreStart, PROT_READ | PROT_WRITE);
	    
     //Create mspace using our backing storage starting at begin and with a footprint of
    // morecoreStart. Don't use an internal dlmalloc lock. When morecoreStart bytes of 
    // memory are  exhausted morecore will be called.
	    
	    mspace msp = create_mspace_with_base(begin, morecoreStart, false /*locked*/);
	    if (msp != NULL) {
	        // Do not allow morecore requests to succeed beyond the starting size of the heap.
	        mspace_set_footprint_limit(msp, startingSize);
	    } else {
	        ALOGE("create_mspace_with_base failed %s", strerror(errno));
	    }
	    return msp;
	}

我们看到，这次他把刚才创建的ASM内存块，通过`create_mspace_with_base`将该块封装成一个mspace，再通过`mspace_set_footprint_limit`设置该`mspace`的大小为起始大小(startingSize)。	


#### addInitialHeap
dalvik/vm/alloc/HeapSource.cpp
在Step3 ，调用语句 `if (!addInitialHeap(hs, msp, growthLimit))`  
 	
	/*
	 * Add the initial heap.  Returns false if the initial heap was 
	 *  already added to the heap source.
	 */
	static bool addInitialHeap(HeapSource *hs, mspace msp, size_t maximumSize)
	{
	    assert(hs != NULL);
	    assert(msp != NULL);
	    if (hs->numHeaps != 0) {
	        return false;
	    }
	    hs->heaps[0].msp = msp;
	    hs->heaps[0].maximumSize = maximumSize;
	    hs->heaps[0].concurrentStartBytes = SIZE_MAX;
	    hs->heaps[0].base = hs->heapBase;
	    hs->heaps[0].limit = hs->heapBase + maximumSize;   //这个是growthLimit
	    hs->heaps[0].brk = hs->heapBase + kInitialMorecoreStart;
	    hs->numHeaps = 1;
	    return true;
	}

用于把前面分配的`msp`上创建一个`Active堆`。这个Active堆的最大值被设置为Java堆的起始大小。

这个解释可能听起来怪怪的，什么创建Active堆？没看到啊，那是因为没对HeapSoucre的结构做说明，在上一篇文章也只是简单的提及了一下而已，所以没明白过来，下面我们针对的说一下涉及到的变量。

	/* The largest number of separate heaps we can handle. 
	 */  
	#define HEAP_SOURCE_MAX_HEAP_COUNT 2  //想多没有，最多两个。一个Active，一个Zygote堆

	struct HeapSource {
	
	    ...
	
	　　/* The heaps; heaps[0] is always the active heap,
		* which new objects should be allocated from.
		*/
	    Heap heaps[HEAP_SOURCE_MAX_HEAP_COUNT];
	
		// The current number of heaps. 
	    size_t numHeaps; 
	
	  ...
	};

我们在代码设置了**hs->numHeaps=1**代表当前堆有且只有一个，而且就是我们前面申请的那个。
然后我们设置了整个堆数组的第一个heaps[0]为我们刚才的那个**hs->heaps[0].base 和 msp 两个属性**，通过这样的赋值给这个**hs**，把我们前面申请的空间赋值到它手上去，刚才才说了上面那句话。
但还有一点可能没提到的是，这个最大堆的参数值为2，那么我们的haep[1]是什么鬼？这个是zygote堆，在
我们的Zygote进程会调用的另外一个函数forkAndSpecializeCommon去做这部分内容。虽然和这个密切相关，不过暂时先不提这个话题的内容。

#### HeapBitmap.dvmHeapBitmapInit
vm/alloc/HeapBitmap.cpp
调用语句：`if (!dvmHeapBitmapInit(&hs->liveBits, base, length, "dalvik-bitmap-1"))` 


	/*
	 * Initialize a HeapBitmap so that it points to a bitmap large
	 * enough to cover a heap at <base> of <maxSize> bytes, where
	 * objects are guaranteed to be HB_OBJECT_ALIGNMENT-aligned.
	 */
	bool dvmHeapBitmapInit(HeapBitmap *hb, const void *base, size_t maxSize,
	                       const char *name)
	{
	    void *bits;
	    size_t bitsLen;
		...
		
	    bitsLen = HB_OFFSET_TO_INDEX(maxSize) * sizeof(*hb->bits);
	    bits = dvmAllocRegion(bitsLen, PROT_READ | PROT_WRITE, name);
	    if (bits == NULL) {
	        ALOGE("Could not mmap %zd-byte ashmem region '%s'", bitsLen, name);
	        return false;
	    }
	    hb->bits = (unsigned long *)bits;
	    hb->bitsLen = hb->allocLen = bitsLen;
	    hb->base = (uintptr_t)base;
	    hb->max = hb->base - 1;
	    return true;
	}	
我们看这个函数人如其名，就是对HeapBitmap（参数 hb）进行init初始化工作的。
根据送过来的maxSize去得到一个新的大小，然后调用dvmAllocRegion()申请空间，然后给传递过来的hb做初始化操作。

需要说的是这个bitmap不是我们安卓开发关于图片的那个Bitmap，这里会说的是关于数组，每个元素是bit，所以记作bitmap...虽然和图片那个类似的概念...但这里的每个bit是用来标记某个对象是否还存活的，是作为GC时候的一个是否对象的依据。

稍微来看下这个 HeapBitmap结构，路径在vm/alloc/HeapBitmap.h

	struct HeapBitmap {
	    /* The bitmap data, which points to an mmap()ed area of zeroed
	     * anonymous memory.
	     */
	    unsigned long *bits;
	    //我们初始化他的时候，就是用dvmAllocRegion分配的bitsLen大小ASM内存块
	    //这个对象的每个bit都是拿来标记一个对象的死活的。
	    //所以他的大小需要和liveBits的一样大才行，免得有些对象没有对应bit啊
	
	    /* The size of the used memory pointed to by bits, in bytes.  This
	     * value changes when the bitmap is shrunk.
	     */
	    size_t bitsLen;
	
	    /* The real size of the memory pointed to by bits.  This is the
	     * number of bytes we requested from the allocator and does not
	     * change.
	     */
	    size_t allocLen;
	
	    /* The base address, which corresponds to the first bit in
	     * the bitmap.
	     */
	    uintptr_t base;
	
	    /* The highest pointer value ever returned by an allocation
	     * from this heap.  I.e., the highest address that may correspond
	     * to a set bit.  If there are no bits set, (max < base).
	     */
	    uintptr_t max;
	};

我们看每个变量成员都有很好的注释，不用怎么多解释的。



####  allocMarkStack
vm/alloc/HeapSource.cpp
调用语句`if (!allocMarkStack(&gcHeap->markContext.stack, hs->maximumSize))` 

	/*
	 * Create a stack big enough for the worst possible case, where the
	 * heap is perfectly full of the smallest object.
	 * TODO: be better about memory usage; use a smaller stack with
	 *       overflow detection and recovery.
	 */
	static bool allocMarkStack(GcMarkStack *stack, size_t maximumSize)
	{
	    const char *name = "dalvik-mark-stack";
	    void *addr;
	
	    assert(stack != NULL);
	    stack->length = maximumSize * sizeof(Object*) /
	        (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD);	        
	    addr = dvmAllocRegion(stack->length, PROT_READ | PROT_WRITE, name);
	    if (addr == NULL) {
	        return false;
	    }
	    
	    stack->base = (const Object **)addr;
	    stack->limit = (const Object **)((char *)addr + stack->length);
	    stack->top = NULL;
	    madvise(stack->base, stack->length, MADV_DONTNEED);
	    return true;
	}
这个函数看起来好像是在初始化stack多一点啊，应该起个和上面的函数dvmHeapBitmapInit类似的名字感觉才对啊，什么dvmStackInit的，哈。

我们先按照套路的看下这个结构先。在vm\alloc\MarkSweep.h

	struct GcMarkStack {
	
	    /* Highest address (exclusive)
	     */
	    const Object **limit;
	
	    /* Current top of the stack (exclusive)
	     */
	    const Object **top;
	
	    /* Lowest address (inclusive)
	     */
	    const Object **base;
	
	    /* Maximum stack size, in bytes.
	     */
	    size_t length;
	};
我们看这个就是一个栈的模型，一个范围【base--top】，和当前的位置【top】，一个栈大小的【length】。


回看下几个计算公式	    

		stack->length = maximumSize * sizeof(Object*) /
	        (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD);	        

理论上，我们每个对象的大小，最小就是我们的基类祖宗Object的大小siezeof(object)，但是对于每个对象，我们需要配一些额外的空间做一些处理，因此每个要加多HEAP_SOURCE_CHUNK_OVERHEAD这个东西，因此最多的情况下，系统有着这么多的个对象：（用字节为单位）
		
		maximumSize * sizeof(Object*) /
	        (sizeof(Object) + HEAP_SOURCE_CHUNK_OVERHEAD)


这样整个申请完成对的空间的堆操作和对应的栈我们就看到了。
我们知道DVM用的垃圾回收算法是mark-sweep算法来处理的。这里我们就看到了对应的markStack内容咯！
 
在了解了这部分内容后，对dvmHeapSourceStartup就知道内容了，我们回到主线，看下一步。接下来调用到的dvmCardTableStartup，调用它来创建一个Card Table。这个Card Table在执行Concurrent GC时要使用到。

###  CardTable.dvmCardTableStartup
vm/alloc/CardTable.cpp
调用语句：dvmCardTableStartup(gDvm.heapMaximumSize, gDvm.heapGrowthLimit)

   //一大串的官方关于这个类的注释	
	/*
	 * Maintain a card table from the the write barrier. All writes of
	 * non-NULL values to heap addresses should go through an entry in
	 * WriteBarrier, and from there to here.
	 *
	 * The heap is divided into "cards" of GC_CARD_SIZE bytes, as
	 * determined by GC_CARD_SHIFT. The card table contains one byte of
	 * data per card, to be used by the GC. The value of the byte will be
	 * one of GC_CARD_CLEAN or GC_CARD_DIRTY.
	 *
	 * After any store of a non-NULL object pointer into a heap object,
	 * code is obliged to mark the card dirty. The setters in
	 * ObjectInlines.h [such as dvmSetFieldObject] do this for you. The
	 * JIT and fast interpreters also contain code to mark cards as dirty.
	 *
	 * The card table's base [the "biased card table"] gets set to a
	 * rather strange value.  In order to keep the JIT from having to
	 * fabricate or load GC_DIRTY_CARD to store into the card table,
	 * biased base is within the mmap allocation at a point where it's low
	 * byte is equal to GC_DIRTY_CARD. See dvmCardTableStartup for details.
	 */


  //函数实际在这里
	/*
	 * Initializes the card table; must be called before any other
	 * dvmCardTable*() functions.
	 */
	bool dvmCardTableStartup(size_t heapMaximumSize, size_t growthLimit)
	{
	    size_t length;
	    void *allocBase;
	    u1 *biasedBase;
	    GcHeap *gcHeap = gDvm.gcHeap;
	    int offset;
	    
	    void *heapBase = dvmHeapSourceGetBase();
	    ...
	    /* All zeros is the correct initial value; all clean. */
	    assert(GC_CARD_CLEAN == 0);
	
	    /* Set up the card table */
	    length = heapMaximumSize / GC_CARD_SIZE;	    
	    /* Allocate an extra 256 bytes to allow fixed low-byte of base */
	    allocBase = dvmAllocRegion(length + 0x100, PROT_READ | PROT_WRITE,
	                            "dalvik-card-table");	                            
	    if (allocBase == NULL) {
	        return false;
	    }	    
	    gcHeap->cardTableBase = (u1*)allocBase;
	    
	    gcHeap->cardTableLength = growthLimit / GC_CARD_SIZE;
	    gcHeap->cardTableMaxLength = length;
	    
	    biasedBase = (u1 *)((uintptr_t)allocBase -
	                       ((uintptr_t)heapBase >> GC_CARD_SHIFT));
	    offset = GC_CARD_DIRTY - ((uintptr_t)biasedBase & 0xff);
	    gcHeap->cardTableOffset = offset + (offset < 0 ? 0x100 : 0);
	    
	    biasedBase += gcHeap->cardTableOffset;
	    assert(((uintptr_t)biasedBase & 0xff) == GC_CARD_DIRTY);
	    gDvm.biasedCardTableBase = biasedBase;
	
	    return true;
	}	

感觉以后起到一个新的类，或者需要说下这个类是在负责什么内容的，从整体的把握应该可以帮助我们更好的理解各个函数到底是在想些什么。

这个 catdTable和 HeapBitmap是起到类似的作用，帮助我们后面在GC的时候协助回收垃圾的。不一样的地方在于这个不是用bit来描述，而是用byte来描述GC_CARD_SIZE个对象了！
同时来描述Concurrent Gc过程被修改的对象，这些对象是需要我们做进一步特殊处理的。

因为一开始的GC是Stop the World的，把所有线程都暂停掉，然后一个个处理，这会很忙啊，每次GC都回引起卡顿，所以后来一直改进，到了现在的并发GC，这样速度会快，但也是成本的，在并发过程，有可能出现一些对象的引用改变等等操作。所以需要额外的工具来记录这些内容信息，后面做进一步的处理。所以这个是使用cardTable的一个理由。

我们继续回到函数本身来看下里面做了什么。
里面主要的内容是初始化gDvm中的gcHeap变量。
我们来稍微看下这个结构关键(在vm\alloc\HeapInternal.h)字段的含义：


	struct GcHeap {
	    HeapSource *heapSource;
	
	    /* Linked lists of subclass instances of java/lang/ref/Reference
	     * that we find while recursing.  The "next" pointers are hidden
	     * in the Reference objects' pendingNext fields.  These lists are
	     * cleared and rebuilt each time the GC runs.
	     */
	    Object *softReferences;
	    Object *weakReferences;
	    Object *finalizerReferences;
	    Object *phantomReferences;
	    //我们的四种引用
	
	    ...
	
	    /* GC's card table */
	    u1* cardTableBase;
	    size_t cardTableLength;
	    size_t cardTableMaxLength;
	    size_t cardTableOffset;
	
	    /* Is the GC running?  Used to avoid recursive calls to GC.
	     */
	    bool gcRunning;
	
		...
	};

居然没有详细的注释，不过看过前面一些结构后，我们在这里可以稍微的知道这些名字对应的意思：   base-----[length]------offset[当前位置]-----[MaxLength]
base就是基址，然后MaxLength就是整个的大小。然后length描述的是当前用了多少，另外这个offset是用来描述从基于base地址后的偏移地址，这个地址才是实际的使用的开始位置，不是说像数组那样，用来说从base开始，用到了那个位置的意思。 之所以这么做，在前面开头的那大长串的类的注释那里就有提到了，要这样做是为了避免脏数据；
 

关于这个CardTable的使用，还是很大的内容的，涉及到了GC内容，暂时先放一边，下次记录.

# 小结

在写着这文章的过程，看起来不长9个函数，而且写得很水，都是流水记账的样子，但就断断续续写了两天多时间，本来看着就不容易，中间穿插了一些事情，写得都有点难过到变形了。哈哈哈.....憋下气，然后再继续坑这玩意。
当然主要还是想去看下ART的内容了，毕竟现在很多手机都开始是5.0系统了，很多APP也开始从4.3/4.4作为最低支持版本了，不像以前还在支持2.3，适配器来那真的是说多都是泪的故事啊...



# ref

1. [罗哥的文章](http://blog.csdn.net/luoshengyang/article/details/8885792)
2. [关于mmap的详解](http://kenby.iteye.com/blog/1164700) 
