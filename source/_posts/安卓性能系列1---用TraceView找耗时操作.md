title:  安卓性能系列1---用TraceView找耗时操作
date: 2016-05-26 00:45:46
tags: [android]
categories: android

------------------------------------------

软件性能优化是一个很大的概念，这里从自带的一些工具开始，利用工具来协助我们对性能做优化。
当然据闻牛逼的人纯看代码直接看出来问题，哈。

解决系统性能问题的几个主要步骤是：**找->定->调**
	
<!--more-->    
1. **找**：要优化肯定要先找下那部分有性能问题啊，例如有用户投诉所APP卡，你和他沟通后，他说了在主界面滑动过程卡顿。这样我们就大致找到一个需要优化的范围点，接下来需要定位啊。当然这是游击，碎片化的方式，最好还是对程序进行大量的**有针对性**的测试，得到测试数据，然后根据需要定调整目标，要求达到一个怎样的优化效果，例如启动时间由2310ms编程1320ms等等。

2. **定**：找到了问题的大致方向，需要更具体的分析系统瓶颈，分析测试数据，找到其中的bottleneck。
3. **调**：找到瓶颈所在，当然需要对bottleneck的代码做优化。

关于瓶颈，一般而言主要就这样三类：
 

> 1. **低频高耗**：函数被调用次数不多，但每次调用却花费很长时间。 
> 2. **高频低耗**：指那些函数自身占用时间不长，但调用却非常频繁的函数。
> 3. **高频高耗**：那被调用频繁，且函数本身耗费长的，这绝对重点关注对象啊。

简单的理解就是这样的公式 

>  总用时 = 调用次数 * 函数的运行时间

# 如何找到这样的瓶颈呢？

找到这样的两点，我们可以用TraceView来做

在详细介绍如何找到这样的瓶颈前，我们先来操作一遍，看下整个流程是怎样的

## 操作流程

打开我们的ADM，选中我们想分析的程序，然后点击下图中指示的按钮（Start Method Profile），然后程序就会开始帮我们记录从这实践开始，我们程序的所有“工作”

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526162608.png)

接着在你觉得合适的时候，再点下这个按钮，就结束记录操作，生成一个结果出来。这个工具的使用就是这么简单。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526160639.png)

在开始用这个工具做分析前，我们先来写点代码，作为我们后面分析的demo。

## 实战演练

假设我们的程序非常简单，只有两个按钮，第一个点击后会调用下面这个函数。
内容为开个线程，跑4秒钟时间。
这个就是我们的**低频高耗**类型，一次操作就消费你很多CPU时间。

	void doSomething() {
         for (int i = 0; i < 66666; i++) {
            Log.e(TAG, "onTrackClick: " + i);
        }
    }

另外一个按钮点击后，会调用下面这个onRepeatclick函数。这个函数内容也很简单，调用666次的`simplyFunc`去给我们打印`hello World`出来

这个就是前面提到的第2类**高频低耗**类型。本身不特别耗时，不过被调用的次数多。

	public void onRepeatclick(View view) {
        for (int i = 0; i < 666; i++) {
            simplyFunc();
        }
    }

    private void simplyFunc() {
        String a = "Hello ";
        String b = "World !";
        Log.e(TAG, "simplyFunc() print= " + a + b);

    }

好了，有这样的背景，我们来开始实验吧，看怎么找出来这两个家伙！

套路如前面一样，先点击`Start Method Profile`按钮，然后我们分别点这两个函数对应的按钮，让程序帮我们记录下载先。运行后的效果如下图

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526163713.png)

[点击查看大图](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526163713.png)
## TraceView介绍
现在就需要来介绍下这个界面的内容了

整个界面主要分两个大块，顶部由A和C构成的时间线面板（Timeline Panel）和下面的分析面板（Profile Panel）。 

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526164749.png)
 

> **Tips：**
>  双击上面的函数信息那一栏可以缩小，鼠标选中线程的颜色部分轻微水平拉动可以放大图
>  颜色脉冲bar的高度表示cpu的利用率，高度越高表示cpu利用率越高 
> 白色gap空白块表示该线程目前没有占CPU，被其他线程占用 
> 黑块表示系统空闲(system idle)

 我们细看下顶部的时间板，在放大后，当我们鼠标停在某个刻柱的时候，顶部会显示对应时刻，系统在运行的程序信息。左边表示对应的线程信息。如果你点击某个刻度的柱子，对应的柱子也会动一下！

由图可得，在2509.181msec的时候，我们的CPU在运行着我们的`simplyfunc()`函数 下面一行的 excl cpu msec， incl cpu msec， excl readl msec等的时间信息会在后面细说。

  
 ![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526165532.png)

下面的模板信息就很多啦。我们看下顶栏，现在对主要的各个属性的意思如下面表格

	
|列名|描述|
| -----|----|  
|Name|该线程运行过程中所调用的函数名|
| Incl Cpu Time |某函数占用的CPU时间，包含内部调用其它函数的CPU时间|
|Excl Cpu Time|某函数占用的CPU时间，但不含内部调用其它函数所占用的CPU时间|
|Incl Real Time|某函数运行的真实时间（以毫秒为单位），内含调用其它函数所占用的真实时间|
|Excl Real Time|某函数运行的真实时间（以毫秒为单位），不含调用其它函数所占用的真实时间|
|Call+Recur Calls/Total|某函数被调用次数以及递归调用占总调用次数的百分比|
|Cpu Time/Call|某函数调用CPU时间与调用次数的比。相当于该函数平均执行时间|
|Real Time/Call|同CPU Time/Call类似，只不过统计单位换成了真实时间|
|Incl Cpu Time %| 表示以时间百分比来统计的Incl Cpu Time|
   
在明白了这些名字的意思后，现在我们可以用Traceview来查找我们的瓶颈了！

## 找瓶颈

###  高频低耗
先来看下我们的高频低耗类的，根据定义，他就是调用得多的那个家伙。
在我们前面关于列明的介绍中，有一列就是**Call+Recur Calls/Total**，他描述被调用的次数。
因此我们先按照这个属性来排个序，从高到低的。
排序结果如下图：
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526171932.png)

1. **次数** ，我们看到，我们的simplyFunc被用了666次，很正确。很好，这样
看下他的上面几个667次的，我们的SimplyFunc内容为拼接字符串然后打印日志，所以与String相关的几个函数也被调用了很多次。 

		private void simplyFunc() {
	        String a = "hello ";
	        String b = "world";
	        Log.e(TAG, "simplyFunc() print= " + a + b);
	    }
直觉告诉我们，编译器有可能把第三行代码用一个StringBuilder来处理了。
2. **时间**，我们是确定这个函数确实调用次数多很高频，但这不代表人家耗时，所以我们看下右边的关于霸占的CPU时间，我们看到他的Incl Cpu Time为37.183，占了总数的75.9%。很好，这样我们就确认这个函数是一个瓶颈，是我们需要花时间来处理的。
 
到这里基本我们确定了，完成了**找的步骤**。现在需要完成的是**定的步骤**，确定下瓶颈实际在哪里。
我们再看下图，在我们的函数展开的下面，有些内容，我们需要看下。

1. **Parents** ： 这个表示调用这个函数的父方法，就是指是`onRepeatClick`调用了这个函数。
2. **Children**：相对于Parents，这个就是他调用的。

我们看它耗费的时间多的是去调用StringBuilder去了。嗯，我们明明没调用这个类，为何他耗费时间去和Stringbuilder玩去了呢？因为这是编译后的运行效果，编译器会做些手脚嘛。

根据**Children**的信息，告诉我们SimplyFunc主要耗费的时间都去做了什么去了，这样我们可以根据这些信息，对我们的函数做针对性的优化内容。你也可以一直点`children`里面最耗费时间的，一直跟进去，直到你觉得找到答案为主。


#### 细看函数

上面的整流程是相对粗糙的，因为整个下面的面板还有很多我们不关心的内容，有时函数多起来就找的麻烦，为了细化范围，这里提供另外的方案

用`Debug.startMethodTracing()`和`Debug.stopMethodTracing()`方法，当我们再函数加多这对函数，运行完这段代码后，在`/sdcard`路径就会有一个trace文件生成。另外也可以调用`startMethodTracing(String traceName)` 设置trace文件的文件名，最后你可以用`adb pull /sdcard/test.trace /tmp` 命令将trace文件**复制**到你的电脑中，然后像刚才那样，用DDMS工具打开。

用这种更精确的方法，虽然罗嗦些，但非常适合检测某一个方法的性能，颗粒度更细。


    public void onRepeatClick(View view) {

        Debug.startMethodTracing("simplyFunc");
        for (int i = 0; i < 666; i++) {
            simplyFunc();
        }
        Debug.stopMethodTracing();
    }
    
不过作为一名非键盘侠，我选择用DDMS。
靠他来给我们导出文件，点击右上角的那个软盘ICON。这真是复古的一个ICON啊。

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/trace_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160526182335.png)

导出来后，再用ADM来打开这个文件，这样我们就可以细看了。

###  低频高耗

在	看了上面的找高频低耗流程后，我们需要找的就是这个低频高耗的函数了。
被调用次数少，不过一次就要搞好久的家伙。这需要我们关注的是耗费的时间是多少，这里就不细说了。
就是根据另外几个列属性，不断的排查.


###  查看部分GC原因和位置
 
因为安卓2.3以后GC并不会每次都停止其他线程因此只能跟踪到部分停止所有线程的GC情况。一般出现GC的时候时间线上会有比较大块的同颜色的区域点击后就可以定位到函数面板区域的GC函数一步一步向parent函数追踪就可以定位到GC的起因了。如下图的绿色部分主线程在加载资源图的时候发生了GC。


![enter image description here](http://img4.tbcdn.cn/L1/461/1/a9c179be255c18e3a46e244a328e23798c7e7c39)
#  小结

靠现在的方式，找瓶颈是一个精致的工作，需要用TraceView工具一行行的排查。如果不是需要，估计不会没事就像这样搞。另外的方案就是核心路径的内容不断的打点，然后上传到后台，根据大规模的使用情况来做调整，这会好些，先对一劳永逸。
是否有更好的呢？肯定有呀，后面再介绍


# REF 

1. [谷歌关于Systrace的文档](https://developer.android.com/studio/profile/systrace.html#options-4.3)
2. [谷歌关于trace的文档介绍](https://developer.android.com/studio/profile/traceview.html)
3. [关于优化的建议](https://developer.android.com/training/articles/perf-tips.html)
4. [Android系统性能调优工具介绍](http://blog.csdn.net/innost/article/details/9008691/)
5. http://androidperformance.com/2015/11/18/Android-app-lunch-optimize-delay-load.html
6. [TraceView的详细介绍](http://myeyeofjava.iteye.com/blog/2250801)
7. [正确使用Android性能分析工具——TraceView](http://bxbxbai.github.io/2014/10/25/use-trace-view/)