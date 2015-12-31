title: Android测试教程5--模拟键盘输入
date: 2015-11-24 11:35
tags: [android,测试]
categories: android
---

 开发过程中，如何向需要输入字符串的输入框输入数据呢？
 最重要的语句如下：
 
	 ulPhoneET.requestFocus();
	 sendKeys("1 8 8 1 2 3 4 5 6 7 8");

先让输入框获得焦点，然后调用`sendKeys`输入我们想要的内容。
例如这里输入一个电话号码`18812345678`（请各个字用空格空开）

<!--more-->

# 完整测试案例
具体的代码如下：

	public class ULoginActivityTest extends ActivityInstrumentationTestCase2<ULoginActivity> {
	
	    private EditText ulPasswordET; //密码输入框
	    private EditText ulPhoneET; //电话号码输入框
	    private View uLoginBtn; //登录按钮
	    private ULoginActivity uLoginActivity;
	
	    public ULoginActivityTest(Class<ULoginActivity> activityClass) {
	        super(activityClass);
	    }
	
	    public ULoginActivityTest() {
	        super(ULoginActivity.class);
	    }
	
	    public void setUp() throws Exception {
	        super.setUp();
	        uLoginActivity = getActivity();
	        findView();
	    }
	 
	    public void testLogin() throws Throwable {
	        runTestOnUiThread(new Runnable() {
	            @Override
	            public void run() {
	                ulPhoneET.requestFocus();
	            }
	        });
	        sendKeys("1 8 8 1 2 3 4 5 6 7 8");
	
	        runTestOnUiThread(new Runnable() {
	            @Override
	            public void run() {
	                ulPasswordET.requestFocus();
	            }
	        });
	
	        sendKeys("1 2 3 4 5 6");
	
	        runTestOnUiThread(new Runnable() {
	            @Override
	            public void run() {
	                uLoginBtn.performClick();
	            }
	        });
	    }
	
	    private void findView() {
	
	        ulPhoneET = (EditText) uLoginActivity.findViewById(R.id.u_l_phone_et);
	        ulPasswordET = (EditText) uLoginActivity.findViewById(R.id.u_l_password_et);
	        uLoginBtn = uLoginActivity.findViewById(R.id.u_login_bt);
	    }
	
	}
	 

# 解释
1. **主线程**
 可以看到，我们的test里面有两个`runTestOnUiThread`，这是为了使他运行在主线程。
如果所有操作都是在主线程运行的，可以加多这个注解`@UiThreadTest`
那么怎个函数的代码就运行在UI线程了。
但需要注意的，`sendKeys();`这玩意是不能在主线程跑得。

2. **输入的问题**
输入的样子不只有这样，还有别的模式，例如下面这个

		sendRepeatedKeys(1, KeyEvent.KEYCODE_H,
		3, KeyEvent.KEYCODE_E,
		1, KeyEvent.KEYCODE_Y,
		1, KeyEvent.KEYCODE_ALT_LEFT,
		1, KeyEvent.KEYCODE_1,
		1, KeyEvent.KEYCODE_DPAD_DOWN,
		1, KeyEvent.KEYCODE_ENTER);
		final String expected = "HEEEY!";

 这个是用来帮助我们有时需要输入较多重复内容的时候用的，输入的内容是：`HEEEY!`，前面数字代表重复的个数。
 也可以有简洁点的
	```
	 sendKeys("H 3*E Y ALT_LEFT 1 DPAD_DOWN ENTER");
	```
	还有很原始的版本的
	
		sendKeys(KeyEvent.KEYCODE_H, 
				KeyEvent.KEYCODE_E,
				KeyEvent.KEYCODE_E,
				KeyEvent.KEYCODE_E,
				KeyEvent.KEYCODE_Y,
				KeyEvent.KEYCODE_ALT_LEFT,
				KeyEvent.KEYCODE_1,
				KeyEvent.KEYCODE_DPAD_DOWN,
				KeyEvent.KEYCODE_ENTER);
另外还有这个，
`getInstrumentation().sendStringSync("Hello Android!");`

# 铺垫
我这里留下一个铺垫的内容：

	 uLoginBtn.performClick();
	 
我们的登录测试，因为执行的是异步的，结果是通过回调来弄的。
那么该如何测试呢？我们的登录结果是怎样？成功与否？
后面的章节将介绍如何做异步测试。

----
欢迎持续关注。
---