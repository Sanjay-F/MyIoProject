title: Android测试教程4--定点测试
date: 2015-11-21 21:39
tags: [android,测试]
categories: android
---



 
 

我们在测试的时候，很多时候难以避免的问题就是，我写了很多测试案例。
但只是想对其中一部分重要的测试进行测试，或者只是希望对数据库部分进行测试。
但那些测试有分割在不同的包，不同的类里面，那么，到底该怎么解决呢？

能不能**“钓鱼测试”**

执行特定的一类的测试案例呢？

显然是可以啦。
下面是其中解决方案之一。

<!--more-->

# 自定义Annotation


我们可以自定义一个我们的特定的Annotation，对于重要的测试案例，我们加多这个标签。

	/**
	* Annotation for very important tests.
	* 
	*
	*/
	public @interface VeryImportantTest {
	
	}
	
然后，在我们的特定测试上加多这个标签。

	public class MyTest extend TestCase{
	
		@VeryImportantTest
		public void testOtherStuff() {
		
		   fail("Important ge  bird! ");
		   
		}
	}		

最后在执行的时候，配合上这一个命令

> 	$： adb shell am instrument -w -e **annotation VeryImportantTest** \
> 	com.example.aatg.MyTest.test/android.test.
> 	InstrumentationTestRunner
> 

这里的一个参数就是`annotation VeryImportantTest`，对我们特定的标签执行测试。


