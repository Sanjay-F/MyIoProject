title: Android测试教程1--跑起来
date: 2015-11-18 23:11
tags: [android,测试]
categories: android
---



# 1 选择Android Instrument Tests
确认你选中的Test Artifact 是Android Instrument Tests；
![这里写图片描述](http://img.blog.csdn.net/20151118174128573)
就在AS的左下角，自己看吧。有些人选择的是Junit，导致这个类的前面都是红色小标记，没办法运行。

<!--more-->

# 2 自动生成测试
其实我们的AS已经有快捷的帮我们弄生成测试的工具了。在我们需要测试的类的界面，右键->GO TO->test。

 ![这里写图片描述](http://img.blog.csdn.net/20151118174352963)

  请注意在**SuperClass**里面选择这个`InstrumentationTestCase`.
   
![这里写图片描述](http://img.blog.csdn.net/20151118224150022)

 
 
 **点击后，点确定下一步，自动在我们对应的目录下生成好了。**
 具体内容如下

	public class MainActivityTest extends InstrumentationTestCase {
	
	    public void setUp() throws Exception {
	        super.setUp();
	        //我们在这里故意加多这一句！！
	        assertEquals(6, 8 + 9);
	
	    }
	
	    public void tearDown() throws Exception {
	
	    }
	
	    public void testOnCreate() throws Exception {
	
	    }
	
	    public void testOnDestroy() throws Exception {
	
	    }
	}

好了，做好准备了，是时候运行了，
# 3. 添加配置
按着箭头到Edit configuration里面，添加一个新的Android Tests  。

![这里写图片描述](http://img.blog.csdn.net/20151118224633666)
这样我们就可以运行了。
具体测试的内容，我们下个教程再说。
这个只是告诉怎么跑起来的问题。