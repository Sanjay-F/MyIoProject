title: Android测试教程10--monkey Test
date: 2015-12-03 23:48 
tags: [android,测试]
categories: android
---


 
#1，Monkey Test简介

什么是monkey test？
如其名，像猴子一样，虽然不懂什么，但是可以乱点一通。
是的，他的一大特色就是可以乱点一通！！！！！！！
就在规定的次数范围内做任何随机的操作，随机操作包括点击、滑动、Application切换、横竖屏、应用关闭等等，用户能做的操作统统可以模拟；！！
所以简单说就是 **“压力测试”**

[官方介绍文档地址，点这里](http://developer.android.com/tools/help/monkey.html)

那要准备点什么吗？

<!--more-->


Monkey的tools是一个命令行工具，当连接Android设备时，只要在命令行里输入相应命令就能运行tools；
像下面这样：
```
adb shell monkey -v 10
```
  
Monkey test跑出来crash的bug等级永远为1，版本release前，Monkey跑出的结果中crash要为0。

业内标准：final release前，Monkey跑完的总次数应为25W次，其结果里不允许有nullPointException出现.


# 2，Monkey test的使用流程

你可以直接在Android Studio的底部的`Terminal`里面输入指令的。
如果现在手机连接着的话。
如果显示没有ADB这个命令，那么你就配置下环境，在path里面加多  `; 你的SDK路径/platform-tools`
因为`adb.exe`就在这个文件夹里面

![这里写图片描述](http://www.testwo.com/attachment/201211/15/10203_1352975858ZMmb.jpg)
#3，简单的Monkey脚本示例

![这里写图片描述](http://www.testwo.com/attachment/201211/15/10203_13529765616p8w.jpg)



```
adb shell monkey -v 10
```

其中10代表运行脚本的操作次数为10次，若seed不指定值默认为0；

后面两行为intent的描述，运行了Android基本的LAUNCHER主进程，LAUNCHER主进程之后运行了MONKEY进程;

Event percentages,因为命令中只加了一个限制条件，次数为10，当其余参数没有添加时，就如上图百分比出现；

//Money finished 完成。

# 4，Monkey test实例1
![这里写图片描述](http://www.testwo.com/attachment/201211/15/10203_1352981102ks1g.jpg)

> 第一个-s emulator-5554 设备的序列号；
> -p com.lovebizhi.wallpaper-1 要测试对象的package name(adb shell data/data)， 	若测试多个对象，则应为-p package_name1 -p package_name2；
> --pct-xxx 用来设定每个事件在测试中的百分比，百分比总和不能超过100%；
> --pct-touch 调整触摸事件的百分比
> --pct-motion 调整动作事件的百分比
> --pct-trackball 调整轨迹事件的百分比(轨迹事件由一个或几个随机的移动组成，有时还伴随着点击)
> --pct-nav 调整“基本”导航事件的百分比(导航事件由来自方向输入设备的up/down/left/right组成) 
> --pct-majormav 调整“主要”导航事件的百分比(这些导航事件通常引发图形界面中的动作) 
		       第二个-s 同样的seed值(seed值由自己定义，相当于一个文件的文件名,monkey的操作顺序
		     相当于文件)有同样的随机序列，复现问题时，需要monkey用同样的操作步骤重新跑一遍，
		    可以在日志的第一行看到seed值；
		    
> --throttle 设定事件发生的间隔，不设置时，在android系统极限内操作，若手机性能较低，容易出现系统无响应,最佳的时间间隔在300或者500，单位是毫秒；
> 
> -v 指定Log的详细程度，共有三个级别：
> -v 日志级别为level 0
> -v -v 日志级别为level 1
> -v -v -v 日志级别为level 2     日志的详细程度越来越详细

 
Monkey tools在Android内只能针对Activity做测试，**不能对service做测试。**

**tips:**

		 adb devices 可以获取设备id
		
		 adb shell data/data获取应用包的名称


Monkey test中只能指定activity属性的应用包进行测试，当出现指定的应用程序不是activity的时候，monkey会出现以下log，并终止运行：
![这里写图片描述](http://www.testwo.com/attachment/201211/15/10203_1352983461eSu4.jpg)

 

# 5，Monkey test实例2

想要跑完脚本后再去总结过程中出现的crash或者系统无响应，
需要指定参数`--ignore-crashes --ignore-timeouts`，
若不指定，遇到问题就会停止运行，效率会降低。

当且仅当设备有滚轮时需要设定--pct-trackball，如果没有滚轮需设置为0。

monkey可以不指定-p后的应用，若为了有针对的跑需要指定。

# 6，检查结果

![这里写图片描述](http://www.testwo.com/attachment/201211/15/10203_1352984702F1Zz.jpg)

查找关键字crash

//sending event 表示目前已经执行的测试次数

查看有效crash，注意crash:后的进程(pid)，及其后的package name是被测对象；java.lang.NullPointerException下会给出错误在开发工程中的第几行。

**tips:**

	指定要保存log的路径（> d:\test.txt）可以进入D盘下的test.txt进行crash关键字的筛选 

# 注意事项：

在进行monkey的测试时，最好不进行adb的操作；
跑monkey时需要记录`3个log`：
monkey的log（重新指向到某个txt中），
dump system的log(查看系统占有)，
android本身的log logcat。


#后记

 你可能觉得这种乱点乱好的测试不好，希望可以控制下点那里，输入什么等等，那就有了下面这个
 `MonkeyRunner`，也是Android SDK提供的测试工具。

严格意义上来说MonkeyRunner其实是一个Api工具包，比Monkey强大，可以编写测试脚本来自定义数据、事件。

缺点是脚本用`Python`来写，对测试人员来说要求较高，有比较大的学习成本。

或者你也可以用[Robotium来生自动生成操作](http://blog.csdn.net/sanjay_f/article/details/49925821)
 这个就是记录你所有的操作的一个记录软件！
 
---

来源:
[Monkey test——Mr.Monkey 移动测试培训课后总结(三)](http://www.testwo.com/blog/6188)
 