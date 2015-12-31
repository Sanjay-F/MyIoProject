title:  Android测试教程9--聊聊配置测试环境的一些问题
date: 2015-12-03 18:09
tags: [android,测试]
categories: android
---

最近学测试的时候，遇到一些配置上的问题，在这里都写下来

<!--more-->

## testCompile , androidTestCompile 与 compile

写测试的时候遇到这两个配置，一时间没明白，查了下，是这样的
```
testCompile 'org.mockito:mockito-all:1.10.19'
androidTestCompile 'com.google.dexmaker:dexmaker:1.1'
compile 'com.google.dexmaker:dexmaker-mockito:1.1'
```
 
testCompile这个配置项是用来给我们的单元测试的，对应于目录的`src/test` 
androidTestCompile 这个是用来给我们测试api的，对应于目录是`src/androidTest` 

![这里写图片描述](http://img.blog.csdn.net/20151203180757070)


**这两者的主要区别是：**
前者是允许在一般的Java  JVM的，可以做脱离设备的测试
后者是运行在我们的安卓设备或者虚拟机上的情况



---

## 另外还有一些的编译配置

![这里写图片描述](http://images.cnitblog.com/blog2015/54939/201504/231129521259725.png)

**Compile**
这个最常见，再github看到的那些都是这样的形式
compile是对所有的build type以及favlors都会参与编译并且打包到最终的apk文件中。


**Provided**
Provided是对所有的build type以及favlors只在编译时使用，类似eclipse中的external-libs,只参与编译，不打包到最终apk。

**APK**
只会打包到apk文件中，而不参与编译，所以不能再代码中直接调用jar中的类或方法，否则在编译时会报错

**Test compile**
Test compile 仅仅是针对单元测试代码的编译编译以及最终打包测试apk时有效，而对正常的debug或者release apk包不起作用。

**Debug compile**
Debug compile 仅仅针对debug模式的编译和最终的debug apk打包。

**Release compile**
Release compile 仅仅针对Release 模式的编译和最终的Release apk打包。

 ===============================================

# mockito
在使用这个来做mock的时候，遇到一个更加难受的问题
如果编译依赖只有这个的话

	   androidTestCompile 'org.mockito:mockito-all:1.10.19'

会遇到下面这个问题

	java.lang.VerifyError: org/mockito/cglib/core/ReflectUtils
	at org.mockito.cglib.core.KeyFactory$Generator.generateClass(KeyFactory.java:167)
	at org.mockito.cglib.core.DefaultGeneratorStrategy.generate(DefaultGeneratorStrategy.java:25)
	at org.mockito.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:217)
	at org.mockito.cglib.core.KeyFactory$Generator.create(KeyFactory.java:145)
	at org.mockito.cglib.core.KeyFactory.create(KeyFactory.java:117)
	at org.mockito.cglib.core.KeyFactory.create(KeyFactory.java:109)
	at org.mockito.cglib.core.KeyFactory.create(KeyFactory.java:105)
	at org.mockito.cglib.proxy.Enhancer.<clinit>(Enhancer.java:70)
	at org.mockito.internal.creation.cglib.ClassImposterizer.createProxyClass(ClassImposterizer.java:95)
	at org.mockito.internal.creation.cglib.ClassImposterizer.imposterise(ClassImposterizer.java:57)
	at org.mockito.internal.creation.cglib.ClassImposterizer.imposterise(ClassImposterizer.java:49)
	at org.mockito.internal.creation.cglib.CglibMockMaker.createMock(CglibMockMaker.java:24)
	at org.mockito.internal.util.MockUtil.createMock(MockUtil.java:33)
	at org.mockito.internal.MockitoCore.mock(MockitoCore.java:59)
	at org.mockito.Mockito.mock(Mockito.java:1285)
	at org.mockito.Mockito.mock(Mockito.java:1163)
	at com.example.sanjay.myapplication.activity.MainActivityTest.setUp(MainActivityTest.java:56) 


上[stackoverflow](http://stackoverflow.com/questions/12267572/mockito-dexmaker-on-android)说是要加多这个

	androidTestCompile 'com.google.dexmaker:dexmaker:1.1'
	androidTestCompile 'com.google.dexmaker:dexmaker-mockito:1.1'

实际加了之后，还是不行，编译都过，呵呵呵呵。

	Error:Execution failed for task ':app:transformClassesWithDexForDebugAndroidTest'.
	> com.android.build.api.transform.TransformException: com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command 'C:\Program Files\Java\jdk1.8.0_60_64bit\bin\java.exe'' finished with non-zero exit value 2

索性只是保留这两个，又有下面的问题

	java.lang.IllegalArgumentException: dexcache == null (and no default could be found; consider setting the 'dexmaker.dexcache' system property)
 
真的给跪了。

[又是StackOverFlow的一个答案](http://stackoverflow.com/questions/12267572/mockito-dexmaker-on-android)，在SetUp()里面加多这个。

	System.setProperty(
	    "dexmaker.dexcache",
	    getInstrumentation().getTargetContext().getCacheDir().getPath());


成功通过
成功通过
成功通过
成功通过


参考：
[Confused about testCompile and androidTestCompile in android gradle](http://stackoverflow.com/questions/29021331/confused-about-testcompile-and-androidtestcompile-in-android-gradle)
[Android Studio中有六种依赖](http://www.cnblogs.com/kangyi/p/4449857.html)
 
 