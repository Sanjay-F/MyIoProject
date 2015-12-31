title: Android测试教程6--测试我们的Application
date: 2015-11-26 13:27
tags: [android,测试]
categories: android
---

有时候我们会继承我们的Application，然后在里面加多一些全局的设置，常量啊等等。

那么该如何测试呢？
这里举一个简单的例子
来测试一下用SharedPreferences保存和读取数据。

<!--more-->

# 测试代码
	public class MyApplicationTest extends ApplicationTestCase<MyApplication> {
	
	    private MyApplication mApplication; 
	    
	    public MyApplicationTest() {
	        this(MyApplicationTest.class.getSimpleName());
	    }
	
	    public MyApplicationTest(String name) {
	        super(MyApplication.class);
	        setName(name);
	    }
	
	    @Override
	    protected void setUp() throws Exception {
	        super.setUp();
	        //不调用的话就getApplication返回null
	        createApplication();
	        mApplication = getApplication();
	    }
	
	    public final void testPreconditions() {
	        assertNotNull(mApplication);
	    }
	    
	    public void testSetDec() {
	        int dec = 3;
	        mApplication.setDec(dec);
	        assertEquals(dec, mApplication.getDec());
	    }
	}

在这类，我们继承于ApplicationTestCase，要测试的类是`MyApplication`。

# MyApplication代码

	public class MyApplication extends Application {
	
	    SharedPreferences mSharePrferenc;
	    private String KEY = "KEY";
	
	    @Override
	    public void onCreate() {
	        super.onCreate();
	        mSharePrferenc = PreferenceManager.getDefaultSharedPreferences(this);
	    }
	
	    public void setDec(int dec) {
	        mSharePrferenc.edit().putInt(KEY, dec).apply();
	    }
	    public int getDec() {
	        return mSharePrferenc.getInt(KEY,-1);
	    }
	}


# 后记
在测试过程，遇到了下面的问题。
 
	junit.framework.AssertionFailedError
	at android.test.ActivityUnitTestCase.startActivity(ActivityUnitTestCase.java:147)
	at com.example.administrator.testapplication.activity.MainActivityTest.testLifeCycleCreate(MainActivityTest.java:63)
	at java.lang.reflect.Method.invokeNative(Native Method)
	at android.test.InstrumentationTestCase.runMethod(InstrumentationTestCase.java:214)
	at android.test.InstrumentationTestCase.runTest(InstrumentationTestCase.java:199)
	at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:191)
	at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:176)
	at android.test.InstrumentationTestRunner.onStart(InstrumentationTestRunner.java:554)
	at android.app.Instrumentation$InstrumentationThread.run(Instrumentation.java:1701)

后来加多了一句，就好了。 

	@Override
	    protected void setUp() throws Exception {
	        super.setUp();
	
	        renamingMockContext = new RenamingMockContext(getInstrumentation().getTargetContext());
	        renamingMockContext.makeExistingFilesAndDbsAccessible();
	        setActivityContext(renamingMockContext);
	        mStartIntent = new Intent(Intent.ACTION_MAIN);
	    }
 就是加多这么一句` renamingMockContext.makeExistingFilesAndDbsAccessible();`  
就可以像书本说的那样，遇到`UnsupportedOperationException`

	java.lang.UnsupportedOperationException
	at android.test.mock.MockContext.databaseList(MockContext.java:215)
	at android.test.RenamingDelegatingContext.makeExistingFilesAndDbsAccessible(RenamingDelegatingContext.java:76)
	at com.example.administrator.testapplication.activity.MainActivityTest.setUp(MainActivityTest.java:39)

可能你觉得奇怪，因为我的测试的案例里面的setUP函数没有把mock加上去，因为一直遇到`AssertionFailedError`,所以就没在代码加上去，只是在这里备注给自己看。