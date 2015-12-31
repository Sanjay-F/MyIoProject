title: Android测试教程2--简单小测试
date: 2015-11-19 00:41
tags: [android,测试]
categories: android

---
 
 

在教程1，我们知道了怎么运行起我们的测试，现在，我们是时候对我们的界面进行一些操作了，让我们来做一个小的简单测试吧。

<!--more-->

# MainActivity
我把代码都粘贴上来了，这个主要就做了两件事，绑定界面，然后点击了button后，就设置tvResult内容为result.

	public class MainActivity extends AppCompatActivity {
	
	
	    private static final String TAG = MainActivity.class.getSimpleName();
	    private TextView tvResult;
	
	    private int number = 0;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        Log.e(TAG, "onCreate() ");
	        tvResult = (TextView) findViewById(R.id.am_result_tv);
	
	        FloatingActionButton button = (FloatingActionButton) findViewById(R.id.fab);
	        button.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                tvResult.setText("result");
	            }
	        });
	        
	    }
	
	    @Override
	    protected void onDestroy() {
	        Log.e(TAG, "onDestroy() ");
	        super.onDestroy();
	    }
	
	    public void addNumber() {
	        number++;
	    }
	
	    public int getNumber() {
	        return number;
	    }
	
	}
那么我们该怎么测试，我们点击了按钮后，某些事情确实发生了？
例如我的textview的内容改变了呢？
看看下面的案例把
#MainActivityTest 

	public class MainActivityTest extends InstrumentationTestCase {
	
	
	    private String TAG = MainActivityTest.class.getSimpleName();
	
	    private Instrumentation mInstrumentation;
	    private MainActivity mMainActivity;
	    private TextView tvResult;
	    private FloatingActionButton mButton;
	
	
	    public void setUp() throws Exception {
	        super.setUp();
	        mInstrumentation = getInstrumentation();
	        Intent intent = new Intent();
	        intent.setClassName(MainActivity.class.getPackage().getName(), MainActivity.class.getName());
	        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	
	
	        mMainActivity = (MainActivity) mInstrumentation.startActivitySync(intent);
	        Log.e(TAG, " setUP");
	    }
	
		public void testPreconditions() {
		    assertNotNull(tvResult);
		    assertNotNull(mButton);
		}
	 
	    public void testOnCreate() throws Exception {
	        Log.e(TAG, " test on create");
	        tvResult = (TextView) mMainActivity.findViewById(R.id.am_result_tv);
	        mButton = (FloatingActionButton) mMainActivity.findViewById(R.id.fab);

            //在主线程执行，要不然报错！	
	        mInstrumentation.runOnMainSync(new Runnable() {
	            @Override
	            public void run() {
	                mButton.performClick();
	                assertTrue(tvResult.getText().toString().equals("result"));
	            }
	        });
	         //	案例2
	        mMainActivity.addNumber();
	        assertEquals(1, mMainActivity.getNumber());
	    }
	
	
	    public void tearDown() throws Exception {
	        Log.e(TAG, " tear down");
	        testOnDestroy();
	    }
	
	
	    public void testOnDestroy() throws Exception {
	        Log.e(TAG, " test on destroy");
	        mMainActivity.finish();
	    }
	}
上面的代码简单的展示了，如何测试结果的，用到的方法其实很简单粗暴！
绑定界面，执行方法，然后读取textview的内容。

这里关于这个InstrumentationTestCase，需要简单的描述下，
不知道你有没注意到在每个生命周期，都有一个打印的log。
根据这个，我们可以知道一个大概的调用顺序的前后。


## 一些简介：

1. `setUp（）`
 在每个测试案例前，程序都回自动调用`setUp（）`函数，用于做一些初始化等操作，确保每个测试是独立互不影响的，因为在一些测试案例中我们可能对一些变量进行一些操作，如弄成null等，导致在另外一个测试有问题啊。

2.  `tearDown()`
这个和setup是配对的，每次的测试案例跑完就会回调这个函数，用于删除一些垃圾，会收下内测的样子。

2. `test的约定`
   每个函数我们以test为开头，这个是默认的约定，只要是一test开头的，系统都回认为这是一个测试用例，运行的时候就会调用他。以前我们写测试看到最多是有`@test`这样的注解。
3. `testPreconditions()`
   很有意思的是，这个函数的名字叫“前提条件”，不知道你是否记得，我们以前写AsyncTask的时候，也会有一个`onPreExecute（）`的方法，他是运行在主线程，且在我们的`doInBackground（）`前执行的，这个也是类似的意思，在我们的`testXXX（）`函数执行前执行，但说有意思的地方，不在这，他有意思的地方是，**系统不保证他在这些测试被执行前执行**，**系统不保证他在这些测试被执行前执行**，**系统不保证他在这些测试被执行前执行**。呵呵，就是这样啦！


----





# 介绍InstrumentationTestCase

继承自JUnit TestCase类，并可以使用Instrumentation框架，用于测试Activity。
使用Instrumentation，Android可 以向程序发送事件来自动进行UI测试，并可以精确控制Activity的启动，监测Activity生命周期的状态。你可以看到他拥有一系列的callActivityOnXXX（）的函数。

	Instrumentation.callActivityOnCreate();
	Instrumentation.callActivityOnPause();
	Instrumentation.callActivityOnXXX();

基类是InstrumentationTestCase。
它的所有子类都能发送按键或触摸事件给UI。
子类还可以注入一个模拟的Intent。

子类有：

-  ActivityTestCase——Activity测试类的基类。

- SingleLaunchActivityTestCase——测试单个Activity的类。它能触发一次setup()和tearDown()，而不是每个方法调用时都触发。如果你的测试方法都是针对同一个Activity的话，那就使用它吧。
- SyncBaseInstrumentation——测试Content Provider同步性的类。它使用Instrumentation在启动测试同步性之前取消已经存在的同步对象。

- ActivityUnitTestCase——对单个Activity进行单一测试的类。使用它，你可以注入模拟的Context或 Application，或者两者。它用于对Activity进行单元测试。不同于其它的Instrumentation类，这个测试类不能注入模拟的 Intent。
 
- ActivityInstrumentationTestCase2——在正常的系统环境中测试单个Activity的类。你不能注入一 个模拟的Context，但你可以注入一个模拟的Intent。另外，你还可以在UI线程（应用程序的主线程）运行测试方法，并且可以给应用程序UI发送 按键及触摸事件。