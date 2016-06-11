title: 源码探索系列38---关于dalvik虚拟机
date: 2016-06-12 00:35
tags: [android,DVM,Dalvik]
categories: android

------

虚拟机是一个很大的话题，很多内容可以写，本篇先从类的加载到函数运行做一个简单的记录。关于这整个流程，一般的逻辑可能这样

1. 我们的DVM（Dalvik Virtual Machine）是怎么启动的
2. 启动后，JAVA类是怎样的加载和初始化
3. 初始后，我们的函数Method是怎么执行的

<!--more-->
 
涉及在整个过程的是，我们的申请的内存是怎样分配的，怎么个回收法，我们的类文件结构是怎样的等等问题。所以不得不说这是一个非常大的话题，每一点的深入都可以被深挖，然后写出一本大部头，叫《深入理解JAVA虚拟机---Dalvik高级特性与最佳实践》的书。
实际上我们并不能深入的去探索，篇幅所限，因此只可能都是适可而止的提到相关的内容即可。有需要的可以自己去看下相关材料，例如叫《深入Java虚拟机》之类的书籍，毕竟这些虚拟机都是根据标准做定制的一些虚拟机。
关于标准，可以看官方的这个文档材料
[The Java® Virtual Machine Specification---Java SE 7 Edition](https://docs.oracle.com/javase/specs/jvms/se7/html/)
 
 关于DVM的启动内容，我们在看系统启动过程的时候，我们有遇到对虚拟机的启动部分的内容，虽然不是很细致的去看下所有内容，不过也算有个大概的印象了。因此就不在这里罗列的，有兴趣也可以看下老罗哥的这篇文章 [Dalvik虚拟机的启动过程分析](http://blog.csdn.net/luoshengyang/article/details/8885792)。

 
 
 下面我们从我们的DVM加载我们想要调用的类开始说起。
 
## Class加载
API: Dalvik-kitkat版本

假设我们已经启动好我们的DVM了，现在是时候来加载我们写好的代码了。
 一般我们说类的加载，都会提到下面三个步骤：
 
 > 装载-> 连接( 验证->准备->解析 )->初始化

关于前面的几个步骤，我们都不过多细说，直接看最后一步的`初始化步骤`内容。
DVM和JVM一样在初始化某个类的时候，会调用其类初始化方法`<Clinit>`函数。这个函数是由我们的Java编译器把类的static成员变量或者static语句块等收集在一起，然后挪到这个函数里面去的。

我相信对于类的加载，你应该记得很有趣的一个案例：

	public class Father {

	    static {
	        System.out.println("father block 1");
	    }
	
	    Father() {
	        System.out.println("father block 2");
	    }
	}
	
	public class Child extends Father {

	    private String name;
	    private static String staticVar = "childStaticVar";
	    private static  Father father=new Father();

	    static {
	        System.out.println("child block1" + staticVar);
	    }
	
	    public Child(String name) {
	        this.name = name;
	        System.out.println("child block3");
	    }
	
	    public static void main(String[] args) {
	        Child child = new Child("child");
	    }
	}
然后问题就是问打印出来的是什么内容。我们知道是先父类的静态内容，然后是子类的静态块，结着是父类子类的构造函数顺序。
实际上他们就是被按照在类中写的位置顺序，挪到了`clinit`函数去了。
因此你会发现如果对调了staticVar和static{}块的顺序，那么会有一个`illegal forward reference`信息等着你。

> **TIPS:** 如果没有类变量（没有初始化或者用产量初始化也一样）或静态语句块，那么也是不会有这个函数。

这个`<Clinit>`方法只能由VM来调用，我们手动调用不了的。那么是什么时候被调用的呢？

有一个说法就是在**主动调用**的时候。当我们首次主动使用类型时候初始化他们，在《深入理解虚拟机》P162页，里面有把主动调用列了下面这六种：
1. 建新实例，如New一个，用newInstance()，Clone()，IO的getOjbect()
2. 调用静态方法
3. 操作类或接口中声明的非常量静态字段
4. 调用特定的反射方法
5. 初始化其子类
6. 指定其为虚拟机启动时的初始化类

对于一个类的加载，我们从主动加载中最常见的new一个对象做切入点来讨论这个问题，其余类似，不做累赘。


###  New Instance
在/dalvik/vm/mterp/out/InterpC-portable.cpp。

当我们去new一个新对象的时候，我们的DVM和JVM一样，会去调用对应的代码去给我们干活，这些内容在`InterpC-portable.cpp`中，在line413的地方，有下面一段代码


	*
	 * Instruction framing.  For a switch-oriented implementation this is
	 * case/break, for a threaded implementation it's a goto label and an
	 * instruction fetch/computed goto.
	 *
	 * Assumes the existence of "const u2* pc" and (for threaded operation)
	 * "u2 inst".
	 */
	# define H(_op)             &&op_##_op
	# define HANDLE_OPCODE(_op) op_##_op:
	# define FINISH(_offset) {                                                  \
	        ADJUST_PC(_offset);                                                 \
	        inst = FETCH(0);                                                    \
	        if (self->interpBreak.ctl.subMode) {                                \
	            dvmCheckBefore(pc, fp, self);                                   \
	        }                                                                   \
	        goto *handlerTable[INST_INST(inst)];                                \
	    }
 
 在经过编译器处理后，我们的new操作是有一个对应的操作码（OperationCode）的，当DVM看到这个时候，会跳转到对应的处理函数，去做对应的操作，涉及的代码就是上面一段（HANDLE_OPCODE，处理操作码的意思，很人如其名，不过是在portable模式下时候是这样），其中由`FINISH`宏就是取出当前的操作码，然后跳转（goto）到对应的处理逻辑去处理它。

那有个小问题，这个handlerTable是什么内容呢？
### dvmInterpretPortable()
我们去看下，在这个类的line1117处，有下面的内容：

	/* File: portable/entry.cpp */
	/*
	 * Main interpreter loop.
	 *
	 * This was written with an ARM implementation in mind.
	 */
	void dvmInterpretPortable(Thread* self){
		...
	
	    /* static computed goto table */
	    DEFINE_GOTO_TABLE(handlerTable);
	
	    ...
	
	    FINISH(0);    
	    /* fetch and execute first instruction */

		....下面是一堆opCode的内容，跳过
	}

对于这个 `dvmInterpretPortable`是porttable模式下Java字节码的**执行入口**！。也就是当执行Java字节码的时候（比如Child.class中的main函数时）都会调用这个函数。
然后那个**DEFINE_GOTO_TABLE**则定义了操作码的标记。

####  DEFINE_GOTO_TABLE
在dalvik-kitkat/libdex/DexOpCode.h文件里面，我们看到下面一长串的内容功能：

	/*
		 * Macro used to generate a computed goto table for use in 
		 * implementing an interpreter in C.
	 */
	 
	#define DEFINE_GOTO_TABLE(_name) \
	    static const void* _name[kNumPackedOpcodes] = {                      \
	        /* BEGIN(libdex-goto-table); GENERATED AUTOMATICALLY BY opcode-gen */ \
	        
	        H(OP_NOP),                                                            \
	        H(OP_MOVE),                                                           \
	        ...
	       H(OP_ARRAY_LENGTH),                                                   \
	        H(OP_NEW_INSTANCE),                                                   \
	        ....
	        H(OP_UNUSED_FF),                                                      \
	        /* END(libdex-goto-table) */                                          \
	    };


 例如我们的要去new一个对象的时候，遇到的就是OP_NEW_INSTANCE这个字段。
 然后回到我们的InterpC-portable.cpp里面，会看回应的处理代码

#### 	HANDLE_OPCODE(OP_NEW_INSTANCE)

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
	        newObj = dvmAllocObject(clazz, ALLOC_DONT_TRACK);
	        if (newObj == NULL)
	            GOTO_exceptionThrown();
	        SET_REGISTER(vdst, (u4) newObj);
	    }
	    FINISH(2);
	OP_END

我们看到在这段逻辑中，先做些检验的操作，然后才是真正分配内存的操作dvmAllocObject。

对于各个函数具体的内容，这里不做罗列，主要是掌握原理和大概的流程为主要目的，细节上以日后需要到再深挖。

#### 小结
以上是这对portable平台的，portable模式下，操作码是一条一条解释执行的。而具体CPU平台上，则是由相关汇编代码来处理。二者实际上大同小异。但是由CPU来执行，显然处理要快，比如对于+这种操作，用portable的解释执行当然比直接转换成机器指令来执行要慢很多。

别的如arm,MIPS,X86等的平台有类似处理过程。对应的代码都在dalvik/vm/mterp/out有对应的类内容。
 对于ARM平台，则有InterpAsm-armXXX.S和对应的InterpC-armXXX.cpp。其中.S文件是汇编文件，而.CPP文件是对应的C++文件。二者要结合起来使用。当CPU类型不属于ARM、x86或mips（也不采用纯解释方法），则通过InterpAsm-allstubs.S和interpAsm-allsubts.cpp来处理。

这样，我们对DVM是怎么执行Java字节码的流程有个大概的了解。
到此，我们了解了Java字节码到底是怎么执行的。

* 对于涉及的类的检验，和类的结构等内容，暂时不是我们关注的重心，我们先跳过，后面时间合适的时候再看。

## 函数的运行
看我了前面的类的加载部分，那么既然我们的类都加回来了，我们可以看下当我们调用函数的时候，我们的函数是怎么个运行起来的。
在`dalvik/VM/Thread.cpp`参考`dvmAttachCurrentThread()`方法.

### 创建栈帧 allocThread

JVM在执行一个函数之前，它会首先分配一个栈帧（JVM中叫Frame），这个Frame其实就是一块**内存**，里边存储了**参数**，还预留空间用来**存储返回值**等。
当我们的函数执行时，就会从当前**栈帧**（每一个函数执行之前，JVM都会为它分配一个栈帧）获取参数等**信息**，然后执行，然后将返回值存储到当前栈帧。

当前正在执行的函数叫current Method（当前方法）
函数执行当然是在一个线程里运行的，栈帧则理所当然就会和某个线程相**关联**。函数返回后，VM回收当前栈帧。


我们先来看dalvik是怎么创建线程及对应栈的。 
**allocThread**用于创建代表一个线程的线程对象

	/*
	 * Alloc and initialize a Thread struct.
	 *
	 * Does not create any objects, just stuff on the system (malloc) heap.
	 */
	static Thread* allocThread(int interpStackSize)
	{   
	    Thread* thread;
	    u1* stackBottom;
	
	    thread = (Thread*) calloc(1, sizeof(Thread));
 	
		...忽略一些assert
	 
	    thread->status = THREAD_INITIALIZING;
	
	    /*
	     * Allocate and initialize the interpreted code stack.  We essentially
	     * "lose" the alloc pointer, which points at the bottom of the stack,
	     * but we can get it back later because we know how big the stack is.
	     * The stack must be aligned on a 4-byte boundary.
	     */     
		...忽略一些别的#ifdef
		
	    stackBottom = (u1*) mmap(NULL, interpStackSize, PROT_READ | PROT_WRITE,
	        MAP_PRIVATE | MAP_ANON, -1, 0);
	   ...
	

	    thread->interpStackSize = interpStackSize;
	    thread->interpStackStart = stackBottom + interpStackSize;
	    thread->interpStackEnd = stackBottom + STACK_OVERFLOW_RESERVE;
	
	#ifndef DVM_NO_ASM_INTERP
	    thread->mainHandlerTable = dvmAsmInstructionStart;
	    thread->altHandlerTable = dvmAsmAltInstructionStart;
	    thread->interpBreak.ctl.curHandlerTable = thread->mainHandlerTable;
	#endif
	
	    /* give the thread code a chance to set things up */
	    dvmInitInterpStack(thread, interpStackSize);
	
	    /* One-time setup for interpreter/JIT state */
	    dvmInitInterpreterState(thread);
	
	    return thread;
	}

说实在的，看惯了java的那个花括号“{”的位置后，像C++这样的写法看着是别扭的！

我们大致看下上面的内容，DVM为每个线程都创建了一个线程栈。其中是调用`mmap`去创建一个内存块的，大小默认为16KB（参考Dvm.stackSize），并设置了相关的栈顶和栈底指针。

* interpStackStart为栈顶，位于内存高位值。
* interpStackEnd为栈底，位于内存地位。
* 整个栈的内存起始位置为stackBottom。
stackBottom和interpStackEnd还有一个768字节的保护区域**STACK_OVERFLOW_RESERVE**。如果栈内容下压到这块区域，就认为出错了。
  
另外这个申请的16KB的线程大小，是属于延迟加载或者所懒惰加载的模式，要申请了才分配多点，不申请就模式先不分配给你。既16KB只是告诉kernel，我最多用16KB，系统收到只是建立一个内存映射项。如果一直没有使用这块内存的话，那么内存并不会真正分配。所以，只有我们真正操作了这块内存，系统才会为它分配内存。 
 
### dvmCallMethod   
 dalvik/vm/interp/stack.cpp
 有了现在这个帧基础，我们调用方法时候，就可以开始做处理了

	/*
	 * Issue a method call.
	 *
	 * Pass in NULL for "obj" on calls to static methods.
	 *
	 * (Note this can't be inlined because it takes a variable number of args.)
	 */
	void dvmCallMethod(Thread* self, const Method* method, Object* obj,
	    JValue* pResult, ...)
	{
	    va_list args;
	    va_start(args, pResult);
	    dvmCallMethodV(self, method, obj, false, pResult, args);
	    va_end(args);
	}

这里的self就是当前线程	，method就是我们的方法，obj就是这个方法所属的对象，为空表示是静态方法，然后pResult就是返回结果，后面的几个点“...”就是方法的参数
 我们看这个就是一个壳，背后是dvmCallMethodV方法

### dvmCallMethodV
 
	void dvmCallMethodV(Thread* self, const Method* method, Object* obj,
	    bool fromJni, JValue* pResult, va_list args)
	{
	    const char* desc = &(method->shorty[1]); // [0] is the return type.
	    int verifyCount = 0;
	    ClassObject* clazz;
	    u4* ins;
	
		//  1.调用callPrep准备栈帧，这是函数调用的关键一步
	    clazz = callPrep(self, method, obj, false);
	    if (clazz == NULL)
	        return;
	        
	    // 2. 参数入栈       
	    /* "ins" for new frame start at frame pointer plus locals */
	    ins = ((u4*)self->interpSave.curFrame) +
	           (method->registersSize - method->insSize);
	
	    /* put "this" pointer into in0 if appropriate */
	    if (!dvmIsStaticMethod(method)) {
	#ifdef WITH_EXTRA_OBJECT_VALIDATION
	        assert(obj != NULL && dvmIsHeapAddress(obj));
	#endif
	        *ins++ = (u4) obj;
	        verifyCount++;
	    }
	
	    while (*desc != '\0') {
	        switch (*(desc++)) {
	            case 'D': case 'J': {
	                u8 val = va_arg(args, u8);
	                memcpy(ins, &val, 8);       // EABI prevents direct store
	                ins += 2;
	                verifyCount += 2;
	                break;
	            }
	            case 'F': {
	                /* floats were normalized to doubles; convert back */
	                float f = (float) va_arg(args, double);
	                *ins++ = dvmFloatToU4(f);
	                verifyCount++;
	                break;
	            }
	            case 'L': { 
	                /* 'shorty' descr uses L for all refs, incl array */
	                void* arg = va_arg(args, void*);
	                assert(obj == NULL || dvmIsHeapAddress(obj));
	                jobject argObj = reinterpret_cast<jobject>(arg);
	                if (fromJni)
	                    *ins++ = (u4) dvmDecodeIndirectRef(self, argObj);
	                else
	                    *ins++ = (u4) argObj;
	                verifyCount++;
	                break;
	            }
	            default: {
	                /* Z B C S I -- all passed as 32-bit integers */
	                *ins++ = va_arg(args, u4);
	                verifyCount++;
	                break;
	            }
	        }
	    }
	    
		//3. 分情况调用，Native函数由nativeFunc处理，JAVA函数由dvmInterprent处理	    
	    if (dvmIsNativeMethod(method)) {
	        TRACE_METHOD_ENTER(self, method);		        
	        //Because we leave no space for local variables,
	         // "curFrame" points directly at the method arguments.	        
	        (*method->nativeFunc)((u4*)self->interpSave.curFrame, pResult,
	                              method, self);
	        TRACE_METHOD_EXIT(self, method);
	        
	    } else {
	        dvmInterpret(self, method, pResult);
	    }
	    
	 	    dvmPopFrame(self);
	}

我们看，这个函数主要是三部分内容，一个是做些准备工作，然后入栈，最后才是实际的分情况处理工作。
现在我们看下准备内容的部分，结合看下处理java函数时候的情况

#### callPrep()

	/*
	 * Common code for dvmCallMethodV/A and dvmInvokeMethod.
	 *
	 * Pushes a call frame on, advancing self->interpSave.curFrame.
	 */
	static ClassObject* callPrep(Thread* self, const Method* method, Object* obj,
	    bool checkAccess)
	{
		...
	 
	    if (checkAccess) {    // 反射时候你遇到过这个问题吗？
	        /* needed for java.lang.reflect.Method.invoke */
	        if (!dvmCheckMethodAccess(dvmGetCaller2Class(self->interpSave.curFrame),
	                method)){
	            /* note this throws IAException, not IAError */
	            dvmThrowIllegalAccessException("access to method denied");
	            return NULL;
	        }
	    }
	
	    /*
	     * Push a call frame on.  If there isn't enough room for ins, locals,
	     * outs, and the saved state, it will throw an exception.
	     *
	     * This updates self->interpSave.curFrame.
	     */
	    if (dvmIsNativeMethod(method)) {
	        /* native code calling native code the hard way */
	        if (!dvmPushJNIFrame(self, method)) {
	            assert(dvmCheckException(self));
	            return NULL;
	        }
	    } else {
	        /* native code calling interpreted code */
	        if (!dvmPushInterpFrame(self, method)) {
	            assert(dvmCheckException(self));
	            return NULL;
	        }
	    }
	
	    return clazz;
	}
我们看这里主要是分环境处理，
对于JNI的用dvmPushJNIFrame，对于我们的java代码，用dvmPushInterpFarme去处理。

#### dvmPushInterpFrame（）
	
	*
	* Push a frame for an interpreted method onto the stack.  This is only
	 * used when calling into interpreted code from native code.  (The
	 * interpreter does its own stack frame manipulation for interp-->interp
	 * calls.)
	 *
	 * The size we need to reserve is the sum of parameters, local variables,
	 * saved goodies, and outbound parameters.
	 *
	 * We start by inserting a "break" frame, which ensures that the interpreter
	 * hands control back to us after the function we call returns or an
	 * uncaught exception is thrown.
	 */
	 
	static bool dvmPushInterpFrame(Thread* self, const Method* method)
	{
	    StackSaveArea* saveBlock;
	    StackSaveArea* breakSaveBlock;
	    int stackReq;
	    u1* stackPtr;
	
	    ...
	
	    stackReq = method->registersSize * 4        // params + locals
	                + sizeof(StackSaveArea) * 2     // break frame + regular frame
	                + method->outsSize * 4;         // args to other methods
	
	    if (self->interpSave.curFrame != NULL)
	        stackPtr = (u1*) SAVEAREA_FROM_FP(self->interpSave.curFrame);
	    else
	        stackPtr = self->interpStackStart;
	
	    if (stackPtr - stackReq  <   self->interpStackEnd) {
	        /* not enough space */
	        dvmHandleStackOverflow(self, method);
	        assert(dvmCheckException(self));
	        return false;
	    }
	
	    /*
	     * Shift the stack pointer down, leaving space for the function's
	     * args/registers and save area.
	     */
	    stackPtr -= sizeof(StackSaveArea);
	    breakSaveBlock = (StackSaveArea*)stackPtr;
	    stackPtr -= method->registersSize * 4 + sizeof(StackSaveArea);
	    saveBlock = (StackSaveArea*) stackPtr;
	
	#if !defined(NDEBUG) && !defined(PAD_SAVE_AREA)
	    /* debug -- memset the new stack, unless we want valgrind's help */
	    memset(stackPtr - (method->outsSize*4), 0xaf, stackReq);
	#endif
	#ifdef EASY_GDB
	    breakSaveBlock->prevSave =
	       (StackSaveArea*)FP_FROM_SAVEAREA(self->interpSave.curFrame);
	    saveBlock->prevSave = breakSaveBlock;
	#endif
	
	    breakSaveBlock->prevFrame = self->interpSave.curFrame;
	    breakSaveBlock->savedPc = NULL;             // not required
	    breakSaveBlock->xtra.localRefCookie = 0;    // not required
	    breakSaveBlock->method = NULL;
	    saveBlock->prevFrame = FP_FROM_SAVEAREA(breakSaveBlock);
	    saveBlock->savedPc = NULL;                  // not required
	    saveBlock->xtra.currentPc = NULL;           // not required?
	    saveBlock->method = method;
	
	    ...
	    self->interpSave.curFrame = FP_FROM_SAVEAREA(saveBlock);
	
	    return true;
	}
 
 当调用函数时，DVM需要为它弄一个新的栈帧，栈帧的大小stackReq包括2个`StackSaveArea`和输入参数及函数内部本地变量（大小为method->registersSize*4）所需的空间。在计算栈是否溢出的时候，会额外加上该函数内部调用其他函数时所传参数所占空间（大小为method->outsSize*4）
 
这2个StackSaveArea，一个叫`BreakSaveBlock`，另外一个叫`SaveBlock`。
self->interpSave.curFrame指向saveBlock的高地址。紧接其上的就是参数空间

1.  注意：registersSize包括函数输入参数和函数内部本地变量的个数
2.  dvmPushJNIFrame，这个函数是当Java要调用JNI函数时的压栈处理，该函数和dvmPushInterpFrame几乎一样，只是在计算所需栈空间时，没有加上outsSize*4，因为native函数所需栈是由Native自己控制的。此函数代码很简单，请童鞋们自己学习

好了，栈已经准备好了，我们会主线索，去看下JAVA函数怎么执行。

###  dvmInterpret

	void dvmInterpret(Thread* self, const Method* method, JValue* pResult)
	{
	    InterpSaveState interpSaveState;
	    ExecutionSubModes savedSubModes;
	
		...
		//1. 一些前期准备工作
	    /* Save interpreter state from previous activation, 
	              *linking new to last.
	     */
	    interpSaveState = self->interpSave;
	    self->interpSave.prev = &interpSaveState;
	    /*
	     * Strip out and save any flags that should not be inherited by
	     * nested interpreter activation.
	     */
	    savedSubModes = (ExecutionSubModes)(
	              self->interpBreak.ctl.subMode & LOCAL_SUBMODE);
	    if (savedSubModes != kSubModeNormal) {
	        dvmDisableSubMode(self, savedSubModes);
	    }
	    
		...
	    
	    /*
	     * Initialize working state.
	     *
	     * No need to initialize "retval".
	     */
	    self->interpSave.method = method;
	    self->interpSave.curFrame = (u4*) self->interpSave.curFrame;
	    self->interpSave.pc = method->insns;
		
		//2. 根据模式选择并执行	
	    /*
	     * Make sure the class is ready to go.  Shouldn't be possible to get
	     * here otherwise.
	     */
	    if (method->clazz->status < CLASS_INITIALIZING ||
	        method->clazz->status == CLASS_ERROR)
	    {
	            method->clazz->descriptor, method->clazz->status);
	        dvmDumpThread(self, false);
	        dvmAbort();
	    }
	    typedef void (*Interpreter)(Thread*);
	    Interpreter stdInterp;
	    if (gDvm.executionMode == kExecutionModeInterpFast)
	        stdInterp = dvmMterpStd;
		...
	    else
	        stdInterp = dvmInterpretPortable;
	
	    // Call the interpreter
	    (*stdInterp)(self);
	
	    *pResult = self->interpSave.retval;
	    //3. 得到返回结果,并做些后续处理
	
	    /* Restore interpreter state from previous activation */
	    self->interpSave = interpSaveState;
		...
	    if (savedSubModes != kSubModeNormal) {
	        dvmEnableSubMode(self, savedSubModes);
	    }
	    
	}

整个就做了三步重要的内容，我把涉及的JIT内容都剔掉了。

在前面的地方，有一句`self->interpSave.pc = method->insns;`，对于这里的`insns`就是我们函数对应的指令，后面的流程就是从第一条开始，一条一条的继续下去的。所以这个PC也是人如其名的。


 我们执行不再kExecutionModeInterpFast模式下，所以我们的stdInterp 被赋值的是下面的 `dvmInterpretPortable`，现在让我们去看下里面的内容，看他是如何执行的。

### dvmInterpretPortable 
这个函数的位置在`dalvik/vm/mterp/out/InterpC-portable.cpp`里面，前面我们在说主动加载的new instance时候就有提到过。现在我们重新去看下，同时把一些信息恢复出来，因为现在看得懂了。

	void dvmInterpretPortable(Thread* self)
	{
		...
	    DvmDex* methodClassDex;  
	     // curMethod->clazz->pDvmDex
	    JValue retval;
	    /* core state */
	    const Method* curMethod;   
	     // method we're interpreting
	    const u2*  pc; // program counter
	    u4* fp;  // frame pointer
	    u2 inst; // current instruction
	    
	    /* instruction decoding */
		    u4 ref;// 16 or 32-bit quantity fetched directly
	    u2 vsrc1, vsrc2, vdst;   // usually used for register indexes
	    
	    /* method call setup */
	    const Method* methodToCall;
	    bool methodCallRange;
        
        //1. 算index
	    /* static computed goto table */
	    DEFINE_GOTO_TABLE(handlerTable);

	    //2. 备份状态信息
	    curMethod = self->interpSave.method;
	    pc = self->interpSave.pc;
	    fp = self->interpSave.curFrame;
	    retval = self->interpSave.retval;   /* only need for kInterpEntryReturn? */
	
	    methodClassDex = curMethod->clazz->pDvmDex;		   
	
	    /*
	     * Handle any ongoing profiling and prep for debugging.
	     */
	    if (self->interpBreak.ctl.subMode != 0) {
	        TRACE_METHOD_ENTER(self, curMethod);
	        self->debugIsMethodEntry = true;   // Always true on startup
	    }
	     methodToCall = (const Method*) -1;
		...
		//3. 后续核心处理跳转地方	
 	    FINISH(0); 
 	     /* fetch and execute first instruction */
 	     
       //4. 处理关于操作码opCode的内容
	/*--- start of opcodes ---*/

	/* File: c/OP_NOP.cpp */
	HANDLE_OPCODE(OP_NOP)
	    FINISH(1);
	OP_END
	
	/* File: c/OP_MOVE.cpp */
	HANDLE_OPCODE(OP_MOVE /*vA, vB*/)
	    vdst = INST_A(inst);
	    vsrc1 = INST_B(inst);

		...一堆opCode内容


		//5. 关于返回的内容
	    /*
	     * General handling for return-void, return, and return-wide.  Put the
	     * return value in "retval" before jumping here.
	     */
	GOTO_TARGET(returnFromMethod)
	    {
	       ...// 后面谈到的时候再粘贴出来
	GOTO_TARGET_END

		
			。。。

	self->interpSave.retval = retval;
	
	}
	
前面我们知道的`DEFINE_GOTO_LABLE`的意思，
现在我们需要来看下前面没有细说的`FINISH(0)`内容

### FINISH()


	# define FINISH(_offset) {                      \
	        ADJUST_PC(_offset);                     \
	        inst = FETCH(0);                        \
	        if (self->interpBreak.ctl.subMode) {    \
	            dvmCheckBefore(pc, fp, self);       \
	        }                                       \
	        goto *handlerTable[INST_INST(inst)];    \
	}

看这个函数，大致的意思，我猜是先调整偏移，接着fetch回下一条操作码，然后调用我们的handlerTable去执行对应的OpCode内容。

为验证想法，我们来看下这个ADJUST_PC的内容，估计就是移动我们的PC位置吧
	
	# define ADJUST_PC(_offset) do {                                            \
	        pc += _offset;                                                      \
	        EXPORT_EXTRA_PC();                                                  \
	    } while (false) 

然后就是我们的FETCH，应该是拿什么数据回来

	/*
	 * Get 16 bits from the specified offset of the program counter.  We always
	 * want to load 16 bits at a time from the instruction stream -- it's more
	 * efficient than 8 and won't have the alignment problems that 32 might.
	 *
	 * Assumes existence of "const u2* pc".
	 */
	#define FETCH(_offset)     (pc[(_offset)])

这就是获取下一条去执行。

这样让我记起以前在大学的时候，学习《计算机组成原理》的时间，开课的老师是国家的百人计划，他的博导拿诺贝尔奖的，在国外呆的职位太高，过不了所谓的国家安全因素，因为是中国人嘛，所以回来了。那时候听他课真是如醍醐灌顶，学得也很有乐趣！！！好想回去上课，听他吹水！！！说远了，继续。


看完这两个宏，我们确定FINISH(0)的内容为： 移动PC，然后获取对应指令的操作码到ins。再根据ins获取该指令的操作码（注意，一条指令包含操作码和操作数，就例如我们的一般A+B一样，我们的操作码是“+”，然后操作数是A和B两个，完成两个数的相加工作！），然后goto到该操作码对应的处理label处。

以上就是在portable模式下的套路，一条条的子指令。

看我这部分，我们需要回到主线，看下前面第4步的内容。截取在下面
	
	  //4. 处理关于操作码opCode的内容
		/*--- start of opcodes ---*/
	
		/* File: c/OP_NOP.cpp */
		HANDLE_OPCODE(OP_NOP)
		    FINISH(1);
		OP_END
		
		/* File: c/OP_MOVE.cpp */
		HANDLE_OPCODE(OP_MOVE /*vA, vB*/)
		    vdst = INST_A(inst);
		    vsrc1 = INST_B(inst);
	   
	   ....
    }

    //粘贴几个会涉及到的宏
	#define INST_A(_inst)       (((_inst) >> 8) & 0x0f)
	#define INST_B(_inst)       ((_inst) >> 12)
	#define INST_AA(_inst)      ((_inst) >> 8)

经过前面的处理，我们的inst已经初始化过了。我们看这个第四部的内容，主要就是到对应的lable去，做对应的处理，然后移动PC，来指向下一条指令。
我们看这个`INST_A`宏实际就是向下移动PC，来指向新的指令。
 


这样，我们知道了大致的一个流程，在portable模式下dalvik如何运行java指令的，没太复杂，和学计算机组成原理时候学到的理论类似。
我们编译器把代码处理成一堆的指令，然后我们的DVM根据碰到的对应的opCode做对应的处理。

### 函数返回  	GOTO_TARGET(returnFromMethod)

处理完，当然需要返回下啊

	 /*
	     * General handling for return-void, return, and return-wide.  Put the
	     * return value in "retval" before jumping here.
	     */
	GOTO_TARGET(returnFromMethod)
	    {
	        StackSaveArea* saveArea;	 
	        PERIODIC_CHECKS(0);
		    ...
	        saveArea = SAVEAREA_FROM_FP(fp);
	
	        /* back up to previous frame and see if we hit a break */
	        fp = (u4*)saveArea->prevFrame;
	        assert(fp != NULL);
	
			 ....
	        //恢复现场
	        /* update thread FP, and reset local variables */
	        self->interpSave.curFrame = fp;
	        curMethod = SAVEAREA_FROM_FP(fp)->method;
	        self->interpSave.method = curMethod;
	        //methodClass = curMethod->clazz;
	        methodClassDex = curMethod->clazz->pDvmDex;
	        pc = saveArea->savedPc;
	       
	        /* use FINISH on the caller's invoke instruction */
	        //u2 invokeInstr = INST_INST(FETCH(0));
	        if (true /*invokeInstr >= OP_INVOKE_VIRTUAL &&
	            invokeInstr <= OP_INVOKE_INTERFACE*/)
	        {
	            FINISH(3);
	        } else {
	            //ALOGE("Unknown invoke instr %02x at %d",
	            //    invokeInstr, (int) (pc - curMethod->insns));
	            assert(false);
	        }
	    }
	GOTO_TARGET_END



### 小结
通过上面的一个大概流程的游览，我们有了一个函数调用的印象。
基本套路如下：

1.  建立栈帧，参数入栈。
2. 跳转到对应函数的位置做处理，Java是goto_label
2. 函数返回，恢复现场，pop栈帧。

# Summary

写VM这类文章，如果有深厚的功底，当然会更可能写出更好的文章，虽然现在对于我还有点吃力，不过会对于我成长更快！

每一步都算数，这是李宗盛在给NewBalance做的一个广告的名字。
确实每一步都重要，一步一个脚印，记录自己成长。

如果不是看了四大金刚的**启动流程**，对于我后面理解**插件化**，找**HOOK点**会很吃力。如果不是看了整个view的**绘制流程**，界面显示的Window内容和Handler的源码，那么对于我去看怎么做**启动优化**就很吃力 .....

如果不是看了这个DVM的内容，那么以后会对我做什么有压力呢？
会知道的，这是我的一个脚印。

虚拟机是一个对我一向很陌生的一个东西，做了3年多的Android开发，也不会说没事对Dalvik的源码内容翻个便，看下里面的具体内容，正如以前在写后台时候，不会去看JVM一样，毕竟还没到需要深入理解的时候。
 
今天先写加载和允许的大概内容，后面在针对类的结构和内存管理部分单独的写点内容记录下。

# REF
1.  [邓凡平大牛写的深入理解Android之Java虚拟机Dalvik](http://blog.csdn.net/innost/article/details/50377905)，读过他写的深入理解几本书，不过那时候对这些都是走马观花的看，毕竟有些内容，就算背下来，以后不用到，也是还给人家。哈
3.  [老罗哥关于Dalvik的学习计划](http://blog.csdn.net/luoshengyang/article/details/8852432)

2.  深入Java虚拟机，By Bill Venners
4. [dalvik的源码](https://github.com/android/platform_dalvik)

