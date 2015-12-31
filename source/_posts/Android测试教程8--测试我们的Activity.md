title: Android测试教程8--测试我们的Activity
date: 2015-11-27 11:31
tags: [android,测试]
categories: android
---

做安卓开发，怎能不测试下我们的Activity呢？
但实际的项目经验告诉我的直觉，这个测试真是因人而异啊。
为啥？

- 有些人把网络请求直接写在Activity里面的
- 有采用MVVM，MVP架构的
- 有的把数据库操作部分直接写在Activity里面的
- 还有各种乱七八糟的

所以理解的Activity测试内容就给人不一样的感觉。
这里讲下一个特殊的测试类`ActivityUnitTestCase` 

<!--more-->

## ActivityUnitTestCase

这个类通常用来测试单独Activity，和前面写的测试 ActivityInstrumentationTestCase2<Activity>是一个相反面。什么意思呢？
前者更多的是关注与这个Activity本事的行为，而不是和的别的组件或UI之类相关的操作，即被测试的Activity 不运行在一般应用运行的环境中也不和其它Activity产生交互，什么跳转到别的Activity的不是他该干的。调用了不能用的方法的话是会抛出异常的
后者就偏向于和Anroid系统进行交互。

废话这么多，我们来看下实际的测试情况
### 上代码

	public class MainActivityTest extends ActivityUnitTestCase<MainActivity> {
	
	    private Intent mStartIntent;
	    private Button mButton;
	    
	    public MainActivityTest(Class<MainActivity> activityClass) {
	        super(activityClass);
	    }
	
	    public MainActivityTest() {
	        super(MainActivity.class);
	    }
	 
	    @Override
	    protected void setUp() throws Exception {
	        super.setUp(); 
	        mStartIntent = new Intent(Intent.ACTION_MAIN);
	    }
	
	    @MediumTest
	    public void testPreconditions() {
	        startActivity(mStartIntent, null, null);
	        mButton = (Button) getActivity().findViewById(R.id.go);
	        assertNotNull(getActivity());
	        assertNotNull(mButton);
	    }
	
	    @MediumTest
	    public void testSubLaunch() {
	        MainActivity activity = startActivity(
	                mStartIntent, null, null);
	        mButton = (Button) activity.findViewById(R.id.go);  
	        mButton.performClick();
	        assertNotNull(getStartedActivityIntent());
	        assertTrue(isFinishCalled());
	    }
	
	
	    @MediumTest
	    public void testLifeCycleCreate() {
	        MainActivity activity = startActivity(
	                mStartIntent, null, null); 
	        getInstrumentation().callActivityOnStart(activity);
	        getInstrumentation().callActivityOnResume(activity); 
	        getInstrumentation().callActivityOnPause(activity); 
	        getInstrumentation().callActivityOnStop(activity);         
	        //onDestory（）可以不调用的，系统自动会调用这个操作
	    }
	}

 上面演示的是一些常见的方法调用，包括对整个的生命周期的调用。
 和对一个按钮Button的点击操作的演示，点击后执行的操作是跳转到另外一个Activity然后结束当前的。
 操作如下： 
 
	        startActivity(new Intent(this,test2Activity.class));
	        finish();
所以，在我们执行`mButton.performClick();`的后面跟着这么两句
 
	        assertNotNull(getStartedActivityIntent());
	        assertTrue(isFinishCalled());
获取新启动的Intent，同时判断自己是否结束。
这两个应该是很多界面都会遇到的操作。

# 进一步的测试

一般，我们的Activity还有很多操作在里面，例如去取一些数据库数据，从网络加点数据，展示到界面等。
那么我该怎么确定读取了什么数据，和我预期的是一致的呢？

那么现在需要进一步的测试，
来mock我们这个Activity需要的数据的时候啦。
怎么用，请看这一篇：

[Android测试教程11--Mock之mockito，异步测试](http://write.blog.csdn.net/mdeditor#!postId=50178215) 
 

#后记

另外需要提醒的一点是，
我们的Activity最好是继承于`extends Activity`
不要写成`MainActivity extends ActionBarActivity`之类的。
要不然会遇到下面的错误： 

	java.lang.RuntimeException: Unable to start activity ComponentInfo{com.example.test2/com.example.test2.MainActivity}: java.lang.IllegalStateException: You need to use a Theme.AppCompat theme (or descendant) with this activity.
 