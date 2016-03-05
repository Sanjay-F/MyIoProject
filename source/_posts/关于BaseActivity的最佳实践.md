title: 关于BaseActivity的最佳实践
date: 2016-01-17 22:21
tags: [android,最佳实践,baseActivity]
categories: android

------
![与友人共躺琶洲塔江堤边畅谈人生](http://7xl9zd.com1.z0.glb.clouddn.com/P60117-133225.jpg)

最近公司开新项目，一直加班，任务量很满，基本都排到过年后回来的二月底了。。。所以最近没什么时间写文章。
今天周日，终于有点时间缓冲，现在写点项目中遇到的内容，做点分享。

今天说的是关于`BaseActivity`和`ActionBar`个人觉得的最佳实践。

相信这个类名字对于很多人来说应该再熟悉不过了，毕竟开发就避免不了这两者，每个Activity顶部基本都会有一条`ActionBar`，当然我指的不是这个类本身，而是指代那个标题栏。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/131734139647542.jpg)

我相信我们基本每个界面都有这个东西存在，而且有时候看到一些项目的这标题栏用的不是系统的**Actionbar**，

而在`theme`里面写的是`noActionbar`的主题，
接着自己多写一个定制的bar的样式的`xml`文件，
最后再用`include`之类的来做。

可能因为有时候那个`PM`的需求太不按常规Android风格出牌了，只能自己做一个，不知你们有没遇到过，但我是遇到不少了，例如要求title是居中,**加粗**的。
后来谷歌出了个`ToolBar`的东西，他确实带来了不少惊喜，让我们可以有更高的定制程度，毕竟人家就是一个viewGroup类来的，但即使有了一个ToolBar，有一样东西是少不了的，那就是

**我们每个界面都需要单独的去添加这个view，同时各种的设置，那么问题来了，我们能否简化这部分工作呢？**

<!--more-->

---

 
#  我们的银弹

今天会写这文章，当然是有一个可以参考的方案，但这个是否就是最好的银弹呢，那就不知道了，但最少满足了我基本的需要，现在就来介绍下。

我们到底该怎么避免重复呢，一个简单的方案就是我们的各种layout都继承于一个`BaseLayout`。
哈哈，可是我们的layout不像各种java类，可以继承啊。
 

看下下面的案例
	
	public class Main2Activity extends BaseActivity {
	 
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main2);
	    }
	｝
这是一个再普通不过的Activity的样子吧？很没有侵入的痕迹，唯一不一样的就是基础了我们写的BaseActivity类。
再看下它的布局文件。

	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout
	xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:fitsSystemWindows="true"
	    tools:context="com.example.sanjay.myapplication.Main2Activity">
	
	
	    <Button
	        android:id="@+id/btn2"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        tools:text="btn" />
	
	
	</LinearLayout>

里面也没什么特别的内容，但我们还是可以做到他继承出一个Actionbar效果。问题出在哪里呢？

答案在`setContentView（）`里面
	
	public void setContentView(@LayoutRes int layoutResID) {
	｝
就是这个函数，我们可以稍微改造下他，让他实现我们想要的`继承关系`，怎么做呢？ 

我们去看下这个BaseActivity做了什么。

# BaseActivity

	public abstract class BaseActivity extends AppCompatActivity {
	
	    protected String TAG = this.getClass().getSimpleName();
	    private LinearLayout contentView = null;
	    protected Context mContext = this;
	    private Toolbar mToolbar;
 
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_base);
	    }
	
	    @Override
	    public void setContentView(@LayoutRes int layoutResID) {
	
	        if ( R.layout.activity_base == layoutResID) {
	            super.setContentView(R.layout.activity_base);
	            contentView = (LinearLayout) findViewById(R.id.layout_center);
	             contentView.removeAllViews();
	
	        } else if (layoutResID != R.layout.activity_base) {
	            View addView = LayoutInflater.from(this).inflate(layoutResID, null);
	            contentView.addView(addView);
	            setActionBar(); 
	        }
	    }
	    
	    public abstract void setActionBar();

     }

好了，最重要的一个内容写出来了，我们重写了`setContentView`函数，
当传过来的布局文件是我们的base的时候，我们才去实际的设置布局文件
`super.setContentView(R.layout.activity_base);`，同时获取里面的一个容器`ContentView`，在我们的子类调用这个方法的时候，我们就通过LayoutInflater去找到这个`view`，然后加到我们的`base`里面去，通过这种方式，我们`拐弯`的达到了继承`BaseLayout`的效果。
之后我们就可以把一些基础的各种设置`ActionBar`的工作从各个Activity里面抽出来，放到这个`BaseActivity`里面去。

我们来看下这个base布局的基本内容

	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:orientation="vertical"
	    tools:context=".BaseActivity">
	
	    <android.support.v7.widget.Toolbar
	        android:id="@+id/toolbar"
	        android:layout_width="match_parent"
	        android:layout_height="?android:actionBarSize"
	        android:background="@color/main_blue">
	
	     </android.support.v7.widget.Toolbar>
	
	    <LinearLayout
	        android:id="@+id/layout_center"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:orientation="vertical" />
	
	</LinearLayout>
一个`Toolbar`和一个装我们之类的容器的`LinearLayout`。
当然可以根据自己的实际需要定制。这都是基本的介绍，实际的使用不止这些，现在只是提供思路。

在我的demo里面写了一些实际可能会用到的内容出来供大家参考。
基本可以快速的移植到实际的项目开发中去了。

## 关于监听各个Activity的生命周期

在开发中，还看到不少程序写的时候，在这个`BaseActivity`里面加多一个注册本Activity为活跃的状态的信息的工作。就像下面这样的模板


	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState); 
        setContentView(R.layout.activity_base);
       	MyApplication.getApp().addActivity(this);

    } 
    
	@Override
    protected void onDestroy() {
        super.onDestroy();
        LifesenseApplication.getApp().removeActivity(this);
    }
	    
 
	
对这部分，我是建议分开出来到另外一个类来做，去实现`Application.ActivityLifecycleCallbacks`
接口的方式，估计可能对一些人来说都不知道这个玩意的存在，我在我的demo里面的M有Application里面有样式，可以参考下，当然也可以直接实际的拿去使用的。

	public class Foreground implements Application.ActivityLifecycleCallbacks {

	}
 

# Demo
写了一个简单的demo，下载的地址在[BaseActivityBestPractices](https://github.com/Sanjay-F/BaseActivityBestPractices)

# 后记
前段时间买了个高桌，放自己的笔记本，然后站着敲完这篇文章，感觉效率是没坐着的时候写的速度快。不过还是得多站立下，避免久坐！

