title: Android测试教程12--模拟读取的文件/数据库
date: 2015-12-05 16:13 
tags: [android,测试]
categories: android
---


 有时我们需要对文件或数据库进行测试，但我们又不想破坏应用程序或设备原有的数据。
此时我们就需要一个Mock来替代他们，这里的Mock不是android.test.mock，
但也是android.test包下面的，`RenamingDelegatingContext`的类。

<!--more-->

首个先我们创建一个应用，功能很简单就是读取该应用目录下的一个txt文件内容，并展示到应用的上。
        
	
	public class MainActivity extends BaseActivity {
	
	    private TextView mTv;
	    public final static String FILE_NAME = "data.txt";
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        mTv = (TextView) findViewById(R.id.tv);
	
	    }
	
	
	    @Override
	    protected void onStart() {
	        super.onStart(); 
	        loadText();
	    }
	     
	    private void loadText() {	
	        try {
	            FileInputStream fis = openFileInput(FILE_NAME);
	            int size = fis.available();
	            byte[] buffer = new byte[size];
	            fis.read(buffer);             
	            mTv.setText(new String(buffer, 0, size));
	            fis.close();
	        } catch (Exception e) {
	            mTv.setText(e.toString());
	            mTv.setTextColor(Color.RED);
	        }
	    }
 
	   public String getText(){
	        return mTv.getText().toString();
	    }
	}

	 
	
运行一下，此时我们的包目录下没有`data.txt`，页面显示的可能是个错误。

OK，然后我们创建两个文件。

一个名为`data.txt`，一个 `mock.data.txt`。
请先按这个名字写，后面有解释!!
前者用于应用中显示的文件内容，后者是作为测试Mock调用的。

    echo "data" >data/data/com.example.test.mockeample/files/data.txt
    echo "mock_data">data/data/com.example.test.mockeample/files/mock.data.txt
    
再次运行下，应该可以看到Activity上显示的是**data**
如果失败，那就百度下吧。
或者用这个办法，桌面生存这两个文件，然后开`Android Device Monitor` ,找到你项目的目录下，然后拖过去
![这里写图片描述](http://img.blog.csdn.net/20151205154112587)


我们的目的是测试这个app能够正确读取文件。 

OK，现在我用另外一个文件mock.data.txt来代替data.txt会不会出错。

	public class MockContextExampleTest extends ActivityUnitTestCase<MainActivity> {
	
	    //注意这个PREFIX，他的值就是 "mock."
	    private static final String PREFIX = "mock.";
	    private RenamingDelegatingContext mMockContext;
	    
	    public MockContextExampleTest() {
	        super(MockContextExampleActivity.class);
	    }
	    
	    @Override
	    protected void setUp() throws Exception {
	        super.setUp();
	        mMockContext = new RenamingDelegatingContext(
					        getInstrumentation().getTargetContext(), PREFIX);
	        mMockContext.makeExistingFilesAndDbsAccessible();
	    } 
	    
	    @MediumTest
	    public void testSampleTextDisplayed(){
	    
	        setActivityContext(mMockContext);
	        MainActivity mActivity = startActivity(new Intent(), null, null);
            getInstrumentation().callActivityOnStart(mActivity);
            
	        assertNotNull(mActivity);
	        String text = mActivity.getText();
	        assertEquals("mock_data", text);
	        
	    }
	    
	    @Override
	    protected void tearDown() throws Exception {
	        super.tearDown();
	    }
	}
	

# 解释：

1. `PREFIX.`
String PREFIX = "mock."，是表示我们要mock的文件或数据库的`前缀`，是相对于实际调用的文件而言的，所以我们前面文件名字，一个是`data.txt`，一个是 `mock.data.txt`。就是通过这个前缀符号，系统返回了后面那个文件给了我们，如果错了，就会返回不了这个文件。

2. `getTargetContext`
另外这里为什么使用`getTargetContext`方法而不是`getContext`，后者我们经常在UiTest中使用到。
这里其实看下注释就明白了：
> getContext() :  The instrumentation's package context.
> 
> getTargetContext()： A Context in the target application.

3. `getInstrumentation().callActivityOnStart(mActivity);`
**不知道你有没注意到这一句！！！**
**不知道你有没注意到这一句！！！**
**不知道你有没注意到这一句！！！**
人家可是十分重要的一句！！！！！
我看了别的介绍的例子，都无法跑通，因为都遇到这么一个实际的BUG，后来急中生智，才来了这么句，才顺利通过测试啊！！！！！！！！！！！！！为了解决这个bug。都疯了了好嘛！
	
			E/ActionBarView: Activity component name not found!                                                android.content.pm.PackageManager$NameNotFoundException: ComponentInfo{com.example.sanjay.myapplication.activity/com.example.sanjay.myapplication.activity.MainActivity}
			at android.app.ApplicationPackageManager.getActivityInfo(ApplicationPackageManager.java:241)
			at android.app.ApplicationPackageManager.getActivityIcon(ApplicationPackageManager.java:682)
			at com.android.internal.widget.ActionBarView.<init>(ActionBarView.java:210)
			at java.lang.reflect.Constructor.constructNative(Native Method)
			at java.lang.reflect.Constructor.newInstance(Constructor.java:417)
			at android.view.LayoutInflater.createView(LayoutInflater.java:594)
			at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:696)
			at android.view.LayoutInflater.rInflate(LayoutInflater.java:755)
			at android.view.LayoutInflater.rInflate(LayoutInflater.java:758)
			at android.view.LayoutInflater.rInflate(LayoutInflater.java:758)
			at android.view.LayoutInflater.inflate(LayoutInflater.java:492)
			at android.view.LayoutInflater.inflate(LayoutInflater.java:397)
			at android.view.LayoutInflater.inflate(LayoutInflater.java:353)
			at com.android.internal.policy.impl.PhoneWindow.generateLayout(PhoneWindow.java:2823)
			at com.android.internal.policy.impl.PhoneWindow.installDecor(PhoneWindow.java:2886)
			at com.android.internal.policy.impl.PhoneWindow.setContentView(PhoneWindow.java:263)
			at android.app.Activity.setContentView(Activity.java:1895)
			at com.example.sanjay.myapplication.activity.MainActivity.onCreate(MainActivity.java:30)
			at android.app.Activity.performCreate(Activity.java:5133)
			at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1087)
			at android.test.ActivityUnitTestCase.startActivity(ActivityUnitTestCase.java:158)
			at com.example.sanjay.myapplication.activity.MainActivityTest.testSampleTextDisplayed(MainActivityTest.java:83)
			at java.lang.reflect.Method.invokeNative(Native Method)
			at java.lang.reflect.Method.invoke(Method.java:525)
			at android.test.InstrumentationTestCase.runMethod(InstrumentationTestCase.java:214)
			at android.test.InstrumentationTestCase.runTest(InstrumentationTestCase.java:199)
			at junit.framework.TestCase.runBare(TestCase.java:134)
			at junit.framework.TestResult$1.protect(TestResult.java:115)
			at junit.framework.TestResult.runProtected(TestResult.java:133)
			at junit.framework.TestResult.run(TestResult.java:118)
			at junit.framework.TestCase.run(TestCase.java:124)
			at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:191)
			at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:176)
			at android.test.InstrumentationTestRunner.onStart(InstrumentationTestRunner.java:554)
			at android.app.Instrumentation$InstrumentationThread.run(Instrumentation.java:1701)
			


以上就是RenamingDelegatingContext函数如何来mock文件的，数据库的使用也是一样的。



# 后记
这个测试真的是搞死人啊，实际的讲解资料又不多，搜索一圈，都是雷同的，而且很旧了。

这里有对`ActivityUnitTestCase`做进一步介绍的需要，他相对于ActivityInstrumentationTestCase2类有还很多不一样的地方，虽然两者的爸爸都是一样的。
如下图所示：

**ActivityUnitTestCase**
![这里写图片描述](http://img.blog.csdn.net/20151205160134988)

**ActivityInstrumentationTestCase2**

![这里写图片描述](http://img.blog.csdn.net/20151205160145540)

 ---
 
 **比较：**
 
 - `ActivityUnitTestCase`——对单个Activity进行单一测试的类。
 使用它，你可以 <font color=red size=5>注入</font>模拟的`Context`或 `Application`，或者两者。
 它用于对Activity进行单元测试。 不同于其它的Instrumentation类，这个测试类不能注入模拟的 Intent。
 
 - `ActivityInstrumentationTestCase2`——在 **正常** 的系统环境中测试单个Activity的类。
你<font color=red size=5>不能</font>注入一 个模拟的Context，但你可以注入一个模拟的Intent。 
另外，你还可以在UI线程（应用程序的主线程）运行测试方法，并且可以给应用程序UI发送 按键及触摸事件。  

所以，你看到了嘛？我们上面有注入`setActivityContext(mMockContext);`，把我们的mockContext给注入进去，所以我们可以读取另外一个`mock.data.txt`文件！




----
参考来源：
 [Testing和Instrumentation](http://blog.chinaunix.net/uid-20771867-id-121133.html) 
  