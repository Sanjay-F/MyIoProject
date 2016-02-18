title: 源码探索系列23---那个做布局性能优化的ViewStub
date: 2016-02-18 23:25
tags: [android,viewstub,源码]
categories: android

------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/blog_%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-02-18%2023.22.53.png)

ViewStub在以开始学安卓的时候就有听说过了，但至今使用的次数都不多，只是在一些加载失败显示错误界面之类的情况用到。虽然他是布局优化的工具之一,不过不得不说，他的实际使用频率还是偏低，可能是我优化的不够多。逃..
他的使用形式是下面这样，在xml中

	<ViewStub
        android:id="@+id/am_demoStub_vs"
        android:layout="@layout/content_main"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

然后在我们的代码。

	if (demoStubView == null) {
	   ViewStub mViewStub = (ViewStub) view.findViewById(R.id.am_demoStub_vs);
       demoStubView = mViewStub.inflate();
	}
     
 

<!--more-->

# 简介

1. 内存耗费很小
必须小啊，要是大还用他干嘛，人家一开始的设计目的之一就是这个，为啥小呢？人家是麻雀嘛，小巧精致，相比较复杂的控件，因为本身简单，所占内存很小；不信你可以看下他源码，本身代码也很短，主要就是那个inflate()函数。但另外一个原因可能是，ViewStub的“延迟化加载”，因为在教多数情况下，程序无需显示ViewStub所指向的布局文件，除了特定条件，ViewStub所指向的布局文件才需要被inflate，然后显示出来。 
ViewStub直接继承于View的，所以可以理解成一个非常轻量级的View。

		@RemoteView
		public final class ViewStub extends View 
		
2. 占位特性
ViewStub主要是起一个“霸位”的性质，放置在view tree中，且本身是不可见的。当我们调用inflate()的时候，ViewStub本身被我们指定的布局代替。

3. 不可见
ViewStub本身是不可见的。在他的构造函数写着，直接就是设置为gone了。但如果你是第一次设置他的Visibility是visible或者invisible，那会起到和inflate类似，在下面的第二个函数写着。而再次使用则是相当于对其指向的布局文件设置可见性。

		public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
	        super(context);
	
	        final TypedArray a = context.obtainStyledAttributes(attrs,
	                R.styleable.ViewStub, defStyleAttr, defStyleRes);
	        mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
	        mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
	        mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
	        a.recycle();
	
	        setVisibility(GONE);
	        setWillNotDraw(true);
	    }
    
		@Override
	    @android.view.RemotableViewMethod
	    public void setVisibility(int visibility) {
	        if (mInflatedViewRef != null) {
	            View view = mInflatedViewRef.get();
	            if (view != null) {
	                view.setVisibility(visibility);
	            } else {
	                throw new IllegalStateException("setVisibility called on un-referenced view");
	            }
	        } else {
	            super.setVisibility(visibility);
	            if (visibility == VISIBLE || visibility == INVISIBLE) {
	                inflate();
	            }
	        }
	    }

4. 一次性买卖
 对ViewStub的inflate()调用只能调用一次，如果多次调用，会抛出个异常给你：ViewStub must have a non-null ViewGroup viewParent。
因为inflate的时候，它会将其指向的布局文件解析inflate并替换掉当前本身，一旦替换，原来的布局文件中就没有这个控件了。
	
		public View inflate() {
	        final ViewParent viewParent = getParent();
	
	        if (viewParent != null && viewParent instanceof ViewGroup) {
	            if (mLayoutResource != 0) {
	                final ViewGroup parent = (ViewGroup) viewParent;
	                final LayoutInflater factory;
	                if (mInflater != null) {
	                    factory = mInflater;
	                } else {
	                    factory = LayoutInflater.from(mContext);
	                }
	                final View view = factory.inflate(mLayoutResource, parent,
	                        false);
	
	                if (mInflatedId != NO_ID) {
	                    view.setId(mInflatedId);
	                }
	
	                final int index = parent.indexOfChild(this);
	                parent.removeViewInLayout(this);//移除自己的地方。
	
					//使用自己的parmars给我们指定的view。所以请以它为准。
	                final ViewGroup.LayoutParams layoutParams = getLayoutParams();
	                //很有舍己为人的精神，移除自己后就加了我们要的。
	                if (layoutParams != null) {
	                    parent.addView(view, index, layoutParams);
	                } else {
	                    parent.addView(view, index);
	                }
	
	                mInflatedViewRef = new WeakReference<View>(view);
					//居然有回调。哈哈，不看代码都不知道，真没使用过。
	                if (mInflateListener != null) {
	                    mInflateListener.onInflate(this, view);
	                }
	
	                return view;
	            } else {
	                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
	            }
	        } else {
	        //敢多次调用就抛出个异常给你
	            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
	        }
	    }
    这代码很好的说了他的占坑属性，infalte的时候把自己替换掉成指定的layout。

# 后记

今天闲聊，小伙伴申请离职了，又是一年跳槽时，也是，最近几个月加得多了点，连过年也没放过，任务都排上去了。年终奖居然只有半个月，过完年也没涨点工资......相比他上一家3个月，确实好像是在逼他走。

上次广州天气百年难得一遇的下雪天，应景的感了个冒，也被抓去会议室密闭开发。没办法项目赶，任务重的，理解，但boss和我说不推崇加班，特别是带病加班。我不知道这是安慰我还是怎么。

这....抓我们几个去那富含装修臭味的小黑屋加到那么晚的，醉了。。。
     
 

