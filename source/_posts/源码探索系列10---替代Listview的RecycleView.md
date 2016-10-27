title: 源码探索系列10---替代Listview的RecycleView
date: 2015-12-23 01:22
tags: [android,源码,RecycleView]
categories: android
------------------------------------------

 自从有了Recycleview，很多原本是我们的Listview业务都被替代了，关于两者的简单比较，[可以看这篇文章](http://blog.csdn.net/sanjay_f/article/details/48830311)。我们今天就去看看他背后故事，下次再写Listview，这名征战多年的老将。

一些不要搞懂的问题

1. 为何谷歌推荐用这个，背后的效率是高在哪里？
2. LayoutManager是怎么去弄不同布局的

<!--more-->
 
# 起航
API：23 ，这RecyclerView有一万多行，看起来真的亚历山大啊。

我们常用的方式就是下面这样：

	mRecycleView.setAdapter(mAdapter);

扔给他一个适配器，所以这个就当作我们的起航的第一个突破口吧，看下他背后都做了些什么事。

	public void setAdapter(Adapter adapter) {
        // bail out if layout is frozen
        setLayoutFrozen(false);
        setAdapterInternal(adapter, false, true);
        requestLayout();
    }

他先去调用`setLayoutFrozen(）`去停止移动，再更新适配器，最后调用`requestLayout()`去更新界面。这里补充说下，这个`RecyclerView`是直接继承`ViewGroup`的。
	
	public void setLayoutFrozen(boolean frozen) {
        if (frozen != mLayoutFrozen) { 
          ...
          final long now = SystemClock.uptimeMillis();
          MotionEvent cancelEvent = MotionEvent.obtain(now, now,
                  MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
          onTouchEvent(cancelEvent);
          mLayoutFrozen = frozen;
          mIgnoreMotionEventTillDown = true;
          stopScroll(); 　
        }
    }
我们看到他背后做的是发送一个cancelEvent同时调用了`stopScroll()`去停止滚动，背后是怎么停止滚动的呢？

	public void stopScroll() {
        setScrollState(SCROLL_STATE_IDLE);
        stopScrollersInternal();
    }

	private void setScrollState(int state) {
        if (state == mScrollState) {
            return;
        } 
        ...
        mScrollState = state; 
        dispatchOnScrollStateChanged(state);
    }

	void dispatchOnScrollStateChanged(int state) {
        // Let the LayoutManager go first; this allows it to bring any properties into
        // a consistent state before the RecyclerView subclass responds.
        if (mLayout != null) {
            mLayout.onScrollStateChanged(state);
        }

        // Let the RecyclerView subclass handle this event next; any LayoutManager property
        // changes will be reflected by this time.
        onScrollStateChanged(state);

        // Listeners go last. All other internal state is consistent by this point.
        if (mScrollListener != null) {
            mScrollListener.onScrollStateChanged(this, state);
        }
        if (mScrollListeners != null) {
            for (int i = mScrollListeners.size() - 1; i >= 0; i--) {
                mScrollListeners.get(i).onScrollStateChanged(this, state);
            }
        }
    }
    
    /**
     * Similar to {@link #stopScroll()} but does not set the state.
     */
    private void stopScrollersInternal() {
        mViewFlinger.stop();
        if (mLayout != null) {
            mLayout.stopSmoothScroller();
        }
    }
    
	void stopSmoothScroller() {
            if (mSmoothScroller != null) {
                mSmoothScroller.stop();
            }
        }
上面代码我们看到些有意思的东西，他先去调用我们的mLayout去设置状态是IDLE闲置状态，再不通知监听的接口更新状态。最后才是实际的调用mLayout的`stopSmoothScroller()`去停止，这个SmoothScroller是一个静态的抽象内部类，具体干活的是`LinearSmoothScroller` 
这个类最终是这mLayout是`LayoutManager`类，它是RecycleView的一个静态的抽象内部类，主要负责的是Measuring和Positioning我们的Item views 。
干活的有三个`StaggeredGridLayoutManager`，`LinearLayoutManager`，`GridLayoutManager`  。


	StaggeredGridLayoutManager mGridLayoutManager =
						new StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL);
	//两列竖直方向的瀑布流
	mRecyclerView.setLayoutManager(mStaggeredGridLayoutManager);

相信使用过RecyclerView的应该对这么名字不陌生，经典的案例就是拿来修改方向灯。这个类有个2K行的就不深挖了，点到即可，继续回主线。
	
	    /**
         * Stops running the SmoothScroller in each animation callback. Note that this does not
         * cancel any existing {@link Action} updated by
         * {@link #onTargetFound(android.view.View, RecyclerView.State, SmoothScroller.Action)} or
         * {@link #onSeekTargetStep(int, int, RecyclerView.State, SmoothScroller.Action)}.
         */
	final protected void stop() {
            if (!mRunning) {
                return;
            }
            onStop();
            mRecyclerView.mState.mTargetPosition = RecyclerView.NO_POSITION;
            mTargetView = null;
            mTargetPosition = RecyclerView.NO_POSITION;
            mPendingInitialRun = false;
            mRunning = false;
            // trigger a cleanup
            mLayoutManager.onSmoothScrollerStopped(this);
            // clear references to avoid any potential leak by a custom smooth scroller
            mLayoutManager = null;
            mRecyclerView = null;
        } 
我们到一个有意思的事情了，他在运行了得情况下并没有实际的去停止运行，就像我们的AsyncTask一样，是个假停止。如果没运行，才调用`SmoothScroller.onStop()`去实际的停止。

**继续回主线**，我们看完 `setLayoutFrozen(false)`的过程
现在继续下一步

	setAdapterInternal(adapter, false, true);

	private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious,
            boolean removeAndRecycleViews) {
         ...
         
        mAdapterHelper.reset();
        final Adapter oldAdapter = mAdapter;
        mAdapter = adapter;
        if (adapter != null) {
            adapter.registerAdapterDataObserver(mObserver);
            adapter.onAttachedToRecyclerView(this);
        }
        if (mLayout != null) {
            mLayout.onAdapterChanged(oldAdapter, mAdapter);
        }
        mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
        mState.mStructureChanged = true;
        markKnownViewsInvalid();
    }
    
这个更改适配器 的界面，主要就更换了原来的适配器，然后注册新的数据观察者等操作
重要一句是调用Recycler的onAdapterChanged(）方法。这个Recycler主要的工作是负责我们在RecyclerView上的各自小itemView的重用功能，所以我们更新了适配器需要告诉下人家。

	void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,
                boolean compatibleWithPrevious) {
            clear();
            getRecycledViewPool().onAdapterChanged(oldAdapter, newAdapter, compatibleWithPrevious);
        }
        
 这样他就先去调用clear函数去清空原有的。再去调用RecycledViewPool的更新。
需要补充下，这个RecycledViewPool是RecyclerViews的静态内部类，他可以让你做到在不同的RecyclerViews内共享Views，**这确实对我们的第一个问题有一定的解答作用**，因为这是一个静态内部类啊，而且我们的View都是继承自`ViewHolder`的，就像我们java的`object`给人的感觉一样。这样用一个内部的ViewPool的做法，就像线程池，我们可以达到了更高的复用，提高滚动的效率。
  
	private SparseArray<ArrayList<ViewHolder>> mScrap;

这个是RecycledViewPool内部使用稀疏数组来存储我们的ViewHolder。嗯，稀疏，直觉好像觉得不对啊，后面看完再看下是怎么回事.

	void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,
                boolean compatibleWithPrevious) {
		if (oldAdapter != null) {
	        detach();
	    }
        if (!compatibleWithPrevious && mAttachCount == 0) {
             clear();
         }
	    if (newAdapter != null) {
	       attach(newAdapter);
	    }
     } 

    void detach() {
         mAttachCount--;
    }
        
	void attach(Adapter adapter) {
          mAttachCount++;//啊...这句让我有点意外，传的参数留着以后用？那就以后再加嘛.. 
    }
    
    public void clear() {
            mScrap.clear();
        }
    
这里记录有多少个适配器，同时保存我们的ViewHolder，当我们的适配器都移除了，那就清空缓存的ViewHolder。
我们看下他存的方式

	public void putRecycledView(ViewHolder scrap) {
         final int viewType = scrap.getItemViewType();
         final ArrayList scrapHeap = getScrapHeapForType(viewType);
         if (mMaxScrap.get(viewType) <= scrapHeap.size()) {
             return;
         }
         if (DEBUG && scrapHeap.contains(scrap)) {
             throw new IllegalArgumentException("this scrap item already exists");
         }
         scrap.resetInternal();
         scrapHeap.add(scrap);
     }

	private ArrayList<ViewHolder> getScrapHeapForType(int viewType) {
        ArrayList<ViewHolder> scrap = mScrap.get(viewType);
          if (scrap == null) {
              scrap = new ArrayList<ViewHolder>();
              mScrap.put(viewType, scrap);
              if (mMaxScrap.indexOfKey(viewType) < 0) {
                  mMaxScrap.put(viewType, DEFAULT_MAX_SCRAP);
              }
          }
          return scrap;
     }
他的存储是用viewType来做key从而存储对应的ViewHolder列表。
目前在我的开发项目中，这个ViewType存在感有点弱啊。
查看整个过程，发现这个itemViewType最后就是调用的是`getItemViewType(int position)`，默认为0；

	final int type = mAdapter.getItemViewType(offsetPosition);
这个补充一点，在前面的一篇比较RecyclerView和Listview的文章有提到，如果要给我们的RecyclerView添加头和尾，不想Listview那样可以 简单的加，实际会负责一点，其中就需要用到这个函数。具体的看 [Listview和RecycleView的简单比较](http://blog.csdn.net/sanjay_f/article/details/48830311) 这篇文章里面的缺点第一条。

看完大致的设置适配器部分内容，我们继续回主线。
到了最后的一个函数

	requestLayout();
	
因为我们的RecyclerView是直接继承ViewGroup 的，那这句就会导致重画等步骤，我们继续看下去吧。
说道这里感觉也可以再开个贴，介绍下View的绘制流程和事件的传递流程，下次有空再写吧，虽然现在介绍这个已是烂大街的了，但自己来写应该有什么感觉呢？写了才知道  ^_^
继续：

我们看下实际的绘制界面的部分吧
 
----

今天时间有限，下次继续写。。。   


#后记

那个layoutManager可以做很多文章啊，上次就看到一个有意思的项目叫伦敦眼的

[**LondonEyeLayoutManager**](https://github.com/Danylo2006/LondonEyeLayoutManager)

他的效果就像摩天轮一样绕着转动！
 ![这里写图片描述](http://img.blog.csdn.net/20151223205013314)



==============2016.10.27====再次更新===
 最近看到一篇 微信开发团队对于 两者的对比，可以查看这里

 [Android ListView与RecyclerView对比浅析--缓存机制](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286405&idx=1&sn=414e2d2eb577884ccee5c9076e8b8357&chksm=8334c387b4434a9124f5acd93f331968a44256b8374eeafb4b1857671072b3b6364e5ec38485&scene=0#rd)

这里直接贴下结论内容：


> 三.结论
> 1. 在一些场景下，如界面初始化，滑动等，ListView和RecyclerView都能很好地工作，两者并没有很大的差异：
> 
> 文章的开头便抛出了这样一个问题，微信Android客户端卡券模块，大部分UI都是以列表页的形式展示，实现方式为ListView，是否有必要将其替换成RecyclerView呢？
> 
> 答案是否定的，从性能上看，RecyclerView并没有带来显著的提升，不需要频繁更新，暂不支持用动画，意味着RecyclerView优势也不太明显，没有太大的吸引力，ListView已经能很好地满足业务需求。
> 
> 2. 数据源频繁更新的场景，如弹幕：http://www.jianshu.com/p/2232a63442d6等RecyclerView的优势会非常明显；
> 
> 进一步来讲，结论是：
> 列表页展示界面，需要支持动画，或者频繁更新，局部刷新，建议使用RecyclerView，更加强大完善，易扩展；其它情况(如微信卡包列表页)两者都OK，但ListView在使用上会更加方便，快捷。
> 
> Ps：仅从一个角度做了对比，盲人摸象，有误跪求指正。