title: Android测试教程11--Mock之mockito，异步测试
date: 2015-12-04 18:17
tags: [android,测试]
categories: android
---

 

#1. mockito是干什么的？

Mock框架之一，其余的还有EasyMock，PowerMock等。

> Mock说白了就是打桩（Stub）或则模拟，当你调用一个不好在测试中创建的对象时，
> Mock框架为你**模拟**一个和真实对象类似的替身来完成相应的行为

就是利用他，我们可以创建一个傀儡，然后被mock的类要返回的数据我们都可以**指定**！
就像下面这样 :

	User user = mock(User.class);
    when(user.getName(3)).thenReturn("张三");
    Log.e(TAG, "name=" + user.getName(3));
    assertEquals("张三", user.getName(3));
我们mock了User这个类，要求他当被请求返回名字`getName()`的时候，就返回`张三`

很好懂吧？下面说点实际的运用

<!--more-->

#2. 异步回调测试

我们做异步加载之类的，如发送网络请求等，很多时候都需要回调，在回调接口中返回特定的数据！
如果服务器还没开发好，我们就很被动，无法测试接口，进行下一步的工作！
当然，什么**模拟超时**，**服务器崩溃没响应的情况**，**返回特定的临界数据等特殊**的要求，
这个要求后台配合都是不容易做的，更何况还要求我们修改一些内容后能够再全部模拟一遍！

但有了这个mockito，这个问题就有了解决方案

假设我们的接口像这样的，调用userApi注册一个叫jack的账号，回调返回注册结果

	  mUserApi.register("jack", new Callback<Result>() {
	  	
	    @Override
	    public void success(Result result, Response response) {
	           Log.e(tag, "result ="+result);
	           activity.setListAdapter(result);
 	    }
	
	    @Override
	    public void failure(RetrofitError retrofitError) {
	        displayErrorMessage();
	    }
	});

好啦，那么我们怎么测试这个接口呢？
我们需要一个东西，叫**ArgumentCaptor**
关于他，我们后面再说，只需要记住，他人如其名，是参数捕抓能手，可以"保存参数的数据"。
	
	@Captor
	private ArgumentCaptor<Callback<Result> cb;
	@Mock
    UserApi mockUserApi
    
	@MediumTest
    public void testLifeCycleCreate() {
    
         // 我们的待测Activity里面有这么个接口，来让我们设置api，
         // 这是一种侵入式的方式，没办法，只能要求暴露个接口，或者把量变弄成 public ,protected级别
         // 要不然无法建立这个测试和我们实际要测的Activity之间的联系。
         // 不要以为我们在这里弄了个把UserApi设置成mock了的，
         // 那么整个app里面有用到这个的都会被mock，这只对现在的mockUserApi有效...
         // 我一开始也是天真的这么觉得的。。。。。
        getActivity().setUserApi(mockUserApi);
         
		Mockito.verify(mockUserApi).register(Mockito.anyString(), cb.capture());
		            
		List<Result> noRepos = new ArrayList<Repository>();
        //  给我们的回调函数返回指定内容 !		
		cb.getValue().success(noRepos, null);
		 
		assertThat(activity.getListAdapter()).isEmpty();
	}

这里有几点需要注意：

1. **Mockito.anyString()**    
他可以匹配任何参数数据，因为这个verify是有参数匹配要求的。

2. **cb.capture()**                  
  我们并不是直接人cb，而是函数cb.capture();
3. 	**@Captor** 
     这个是需要我们在setup（）函数加多这么句，`MockitoAnnotations.initMocks(this);`
     然后系统就会自动帮我们注入，不用像前面的 `User user = mock(User.class);` 需要手动写

然后对应的Actiivty里面

	
		@Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        	
			mUserApi.register("jack", mCallbackListener);	
	    }
	
        setApi(UserApi userApi){
        
               this.mUserApi=userApi;
		}

那么问题来了，如果我们要求实匹配特定的请求的呢，例如就判断jack的注册？
这里就有小插曲要说啦，一般我们mock是这么写的：

	when(mockUserApi).register("jack", callback) ;

但如果这么写，就会有bug

	org.mockito.exceptions.misusing.InvalidUseOfMatchersException:
	Invalid use of argument matchers!
	2 matchers expected, 0 recorded: 
	
	This exception may occur if matchers are combined with raw values:
	//incorrect:
	someMethod(anyObject(), "raw String");
	When using matchers, all arguments have to be provided by matchers.
	For example:
	//correct:
	someMethod(anyObject(), eq("String by matcher"));
	
	For more info see javadoc for Matchers class.
	
这是我见过最好的错误信息之一，简单易懂！！！！！
另外好的就是当年大学用Actel做硬件开发的时候，他们也都有这样的示范答案在旁边的！！
根据信息，我们需要修改成这样 ,加多`eq()`，同时把我们的回调改成`cb.capture();`

	when(mockUserApi).register(eq("jack"), cb.capture()) ;
同时我们的verify也要修改下

	Mockito.verify(mockUserApi).register(eq("jack"), cb.capture());


 好啦，基本就是这些！！
有了这个mockito，以后我们测试一些特殊的情况，例如超时，服务器崩溃了，一些特殊的值等就可以自动搞了！不再需要要求后台配合了。

# 铺垫
说到这里，你可能觉得可以了，但是有一些问题我们其实没测试到的，
那就我们的接口背后可能还做了很多内容，调用了很多别的接口，例如对返回的数据用gson做了解析，不想我们这样是直接返回的，或者在不同的情况下，调用了一些如Gzip来压缩数据等，才返回了这个接口的数据

虽然我们也可以针对各个接口写出测试，但这个成本显然是不低的！！

所以我们期望的是，能够**“模拟一个服务器”**来响应我们的请求。
这个就需要到别的工具啦，这个就是`MockServer`

`MockServer`是一个能够模仿任何通过HTTP连接的服务器或服务，如REST 或 RPC service。提供Java和JavaScript两种API。

关于这个进一步介绍，请保持关注！下次写.

---


 当然你可能允许的时候会遇到一些bug
 可以看下这篇配置文章!
 [Android测试教程9--聊聊配置测试环境的一些问题](http://blog.csdn.net/sanjay_f/article/details/50164425)
 

----
来源：

[Reliable API testing for Android with Retrofit and Mockito](http://mdswanson.com/blog/2013/12/16/reliable-android-http-testing-with-retrofit-and-mockito.html)
 
 [Mockito – @Mock, @Spy, @Captor and @InjectMocks](http://www.baeldung.com/mockito-annotations)
 
 
