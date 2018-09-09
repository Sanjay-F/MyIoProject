title: JetPack源码探索系列4---WorkManager

date: 2018-09-05 20:30:41

tags: [android,JetPack,WorkManager]

categories: android
 

------------------------------------------
 

# WorkManager

The WorkManager API makes it easy to specify deferrable, asynchronous tasks and when they should run. These APIs let you create a task and hand it off to WorkManager to run immediately or at an appropriate time.

在早期，为了做一些延迟定时的任务，做了个基于闹钟的定时任务队列SchedulerQueue。现在谷歌官方帮我们把一切都封装好了。后续你不用再自己造轮子了.像这样的轮子在后台等都有很完全的库了，只是安卓系统支持不太好，搞到现在才出一个吧。

整体原理如下
 ![](https://i.imgur.com/jGx5ihc.png)

使用流程如下：![](https://i.imgur.com/4G2rqGk.png)

<!--more-->

我们来看下源码