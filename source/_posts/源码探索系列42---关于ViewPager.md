title: 源码探索系列42---关于ViewPager
date: 2016-08-29 23:11
tags: [android,ViewPager]
categories: android
---

 
最近做内存优化的时候，想对首页做延迟加载的优化，在嵌套的情况下有点小问题，所以打算顺便就把这几个类的具体内容看下，屡下思路，然后试着自己改造封装一下，从而满足嵌套情况下的一些小问题。

#起航
SDK23--support-v4
 

	public class ViewPager extends ViewGroup 
	
 
我们先来看下一个view的三个绘制步骤onMeasure，onLayout，onDraw三个函数。
再在去看下几个常用的接口，设置adapter的内容，以及选中特定页面的逻辑。
最后再看下对于touch事件的处理。


<!--more-->

# 三个绘制流程

## onMeasure

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec){
    
        // For simple implementation, our internal size is always 0.
        // We depend on the container to specify the layout size of
        // our view.  We can't really know what it is since we will be
        // adding and removing different arbitrary views and do not
        // want the layout to change as this happens.
        setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
                getDefaultSize(0, heightMeasureSpec));

        final int measuredWidth = getMeasuredWidth();
        final int maxGutterSize = measuredWidth / 10;
        mGutterSize = Math.min(maxGutterSize, mDefaultGutterSize);

        // Children are just made to fill our space.
        int childWidthSize =measuredWidth-getPaddingLeft()-getPaddingRight();
        int childHeightSize = getMeasuredHeight() - getPaddingTop() 
										         - getPaddingBottom();



关于这个Decor Views，主要是目前谷歌抽出了PagerTitleStrip和PagerTabStrip两个类，
提供一个简单的方案来在界面上加多标题，不过目前还是挺少用的，
毕竟很多都要求挺高的定制效果。类似下面这种原生效果估计没几个人喜欢

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/title.PNG) 
 

	<android.support.v4.view.ViewPager  
	        android:id="@+id/viewpager"  
	        android:layout_width="wrap_content"  
	        android:layout_height="200dip"  
	        android:layout_gravity="center">  
	          
	        <android.support.v4.view.PagerTitleStrip  
	            android:id="@+id/pagertitle"    
	            android:layout_width="wrap_content"    
	            android:layout_height="wrap_content"    
	            android:layout_gravity="top"  
	            />  
	          
    </android.support.v4.view.ViewPager>  
像上面这段xml代码写的类似，
我们可以通过layout_gravity来设置这个title在pager的位置是顶部还是底部。


    /*
     * Make sure all children have been properly measured. Decor views first.
     * Right now we cheat and make this less complicated by assuming decor
     * views won't intersect. We will pin to edges based on gravity.
     */
    int size = getChildCount();
    for (int i = 0; i < size; ++i) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (lp != null && lp.isDecor) {
                final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
                int widthMode = MeasureSpec.AT_MOST;
                int heightMode = MeasureSpec.AT_MOST;
                boolean consumeVertical = vgrav == Gravity.TOP || 
			                  vgrav == Gravity.BOTTOM;
			                  //这个就是XML配置文件的gravity字段
			                  
                boolean consumeHorizontal = hgrav == Gravity.LEFT || 
		                       hgrav == Gravity.RIGHT;

                if (consumeVertical) {
                    widthMode = MeasureSpec.EXACTLY;
                } else if (consumeHorizontal) {
                    heightMode = MeasureSpec.EXACTLY;
                }

                int widthSize = childWidthSize;
                int heightSize = childHeightSize;
                if (lp.width != LayoutParams.WRAP_CONTENT) {
                    widthMode = MeasureSpec.EXACTLY;
                    if (lp.width != LayoutParams.FILL_PARENT) {
                        widthSize = lp.width;
                    }
                }
                if (lp.height != LayoutParams.WRAP_CONTENT) {
                    heightMode = MeasureSpec.EXACTLY;
                    if (lp.height != LayoutParams.FILL_PARENT) {
                        heightSize = lp.height;
                    }
                }
                final int widthSpec = MeasureSpec.makeMeasureSpec(
				                    widthSize, widthMode);
                final int heightSpec = MeasureSpec.makeMeasureSpec(
				                    heightSize, heightMode);
				                    
                child.measure(widthSpec, heightSpec);

                if (consumeVertical) {
                    childHeightSize -= child.getMeasuredHeight();
                } else if (consumeHorizontal) {
                    childWidthSize -= child.getMeasuredWidth();
                }
            }
        }
    }


    mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
				        childWidthSize, MeasureSpec.EXACTLY);
    mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
				        childHeightSize, MeasureSpec.EXACTLY);
				        

    // Make sure we have created all fragments that we need to have shown.
    mInLayout = true;
    populate(); //这个函数很长，主要是初始化界面等操作，后面会讲到
    mInLayout = false;

    // Page views next.
    size = getChildCount();
    for (int i = 0; i < size; ++i) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (lp == null || !lp.isDecor) {
                final int widthSpec = MeasureSpec.makeMeasureSpec(                    
                        (int) (childWidthSize * lp.widthFactor), 
                        MeasureSpec.EXACTLY);
                        
                child.measure(widthSpec, mChildHeightMeasureSpec);
            }
        }
    }
    }

整个OnMeasure函数都是一些常规的viewGroup的套路，
包括一些测量长宽值为父容器给与的最大长宽值，
测量DecorView，测量ContentView。

## OnLayout
接下来是布局的内容，我们猜测估计和在onMeasure里面有类似的内容，对DecorView和ContentView进行layout操作，毕竟都是模板操作。如果没有那些没什么用的child，估计整个代码会精简不少


	@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int count = getChildCount();
        int width = r - l;
        int height = b - t;
        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingRight = getPaddingRight();
        int paddingBottom = getPaddingBottom();
        final int scrollX = getScrollX();

        int decorCount = 0;

		//和猜想类似，开始了对于child的相关操作
        // First pass - decor views. We need to do this in two passes so that
        // we have the proper offsets for non-decor views later.
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                int childLeft = 0;
                int childTop = 0;
                if (lp.isDecor) {
                    final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
                    final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
                    switch (hgrav) {                       
                    
	                       ...分情况去获得childLeft,childTop,Padding信息                      
                           ...
                    } 
                    childLeft += scrollX;                    
                    //根据等到的布局信息，让child也去layout
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                    decorCount++;
                }
            }
        }

        final int childWidth = width - paddingLeft - paddingRight;
        // Page views. Do this once we have the right padding offsets from above.
        
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
            
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                ItemInfo ii;
                if (!lp.isDecor && (ii = infoForChild(child)) != null) {
                    int loff = (int) (childWidth * ii.offset);
                    int childLeft = paddingLeft + loff;
                    int childTop = paddingTop;
                    if (lp.needsMeasure) {
                        // This was added during layout and needs measurement.
                        // Do it now that we know what we're working with.
                        lp.needsMeasure = false;
                        final int widthSpec = MeasureSpec.makeMeasureSpec(
                                (int) (childWidth * lp.widthFactor),
                                MeasureSpec.EXACTLY);
                                
                        final int heightSpec = MeasureSpec.makeMeasureSpec(
                                (int) (height - paddingTop - paddingBottom),
                                MeasureSpec.EXACTLY);
                                
                        child.measure(widthSpec, heightSpec);
                        
                    }
					 //现在对我们的contentView进行处理
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                }
            }
        }
        mTopPageBounds = paddingTop;
        mBottomPageBounds = height - paddingBottom;
        mDecorChildCount = decorCount;
		//默认为ture，所以会执行
        if (mFirstLayout) {
            scrollToItem(mCurItem, false, 0, false);
        }
        mFirstLayout = false;
    }

 

### scrollToItem
这个就是让我们滚到默认的第一个页面去的作用


	private void scrollToItem(int item, boolean smoothScroll, int velocity,
            boolean dispatchSelected) {
        final ItemInfo curInfo = infoForPosition(item);
        int destX = 0;
        if (curInfo != null) {
            final int width = getClientWidth();
            destX = (int) (width * Math.max(mFirstOffset,
                    Math.min(curInfo.offset, mLastOffset)));
        }
        if (smoothScroll) {
            smoothScrollTo(destX, 0, velocity);
            if (dispatchSelected) {
                dispatchOnPageSelected(item);
            }
        } else {
            if (dispatchSelected) {
                dispatchOnPageSelected(item);
            }
            completeScroll(false);
            scrollTo(destX, 0);
            pageScrolled(destX);
        }
    }	

看我上面这两个，就是最后一个onDraw函数了，不过viewPager重写多了Draw函数，因为需要做多额外的绘制工作，那是什么呢？我们来看下

## draw

	  @Override
    public void draw(Canvas canvas) {
        super.draw(canvas);
        boolean needsInvalidate = false;

        final int overScrollMode = ViewCompat.getOverScrollMode(this);
        if (overScrollMode == ViewCompat.OVER_SCROLL_ALWAYS ||
                (overScrollMode == ViewCompat.OVER_SCROLL_IF_CONTENT_SCROLLS &&
                        mAdapter != null && mAdapter.getCount() > 1)) {
            if (!mLeftEdge.isFinished()) {
                final int restoreCount = canvas.save();
                final int height = getHeight() - getPaddingTop()
							                   -getPaddingBottom();
							                   
                final int width = getWidth();

                canvas.rotate(270);
                canvas.translate(-height + getPaddingTop(), mFirstOffset * width);
                mLeftEdge.setSize(height, width);
                needsInvalidate |= mLeftEdge.draw(canvas);
                canvas.restoreToCount(restoreCount);
            }
            if (!mRightEdge.isFinished()) {
                final int restoreCount = canvas.save();
                final int width = getWidth();
                final int height = getHeight() - getPaddingTop() - getPaddingBottom();

                canvas.rotate(90);
                canvas.translate(-getPaddingTop(), -(mLastOffset + 1) * width);
                mRightEdge.setSize(height, width);
                needsInvalidate |= mRightEdge.draw(canvas);
                canvas.restoreToCount(restoreCount);
            }
        } else {
            mLeftEdge.finish();
            mRightEdge.finish();
        }

        if (needsInvalidate) {
            // Keep animating
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }

我们看我全部，发现它只是对左右边缘效果进行处理 。
我们在水平滑动到左边缘和右边缘的时候，我们可以看到一个半透明的东西，就像那个listview下拉的时候，顶部有一个东西表示到顶了。这个可以抽出来用在别的地方，哈。


## onDraw(Canvas canvas)

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        // Draw the margin drawable between pages if needed.
        if (mPageMargin > 0 && mMarginDrawable != null && mItems.size() > 0 && mAdapter != null) {
            final int scrollX = getScrollX();
            final int width = getWidth();

            final float marginOffset = (float) mPageMargin / width;
            int itemIndex = 0;
            ItemInfo ii = mItems.get(0);
            float offset = ii.offset;
            final int itemCount = mItems.size();
            final int firstPos = ii.position;
            final int lastPos = mItems.get(itemCount - 1).position;
            for (int pos = firstPos; pos < lastPos; pos++) {
                while (pos > ii.position && itemIndex < itemCount) {
                    ii = mItems.get(++itemIndex);
                }

                float drawAt;
                if (pos == ii.position) {
                    drawAt = (ii.offset + ii.widthFactor) * width;
                    offset = ii.offset + ii.widthFactor + marginOffset;
                } else {
                    float widthFactor = mAdapter.getPageWidth(pos);
                    drawAt = (offset + widthFactor) * width;
                    offset += widthFactor + marginOffset;
                }

                if (drawAt + mPageMargin > scrollX) {
                    mMarginDrawable.setBounds(Math.round(drawAt), mTopPageBounds,
                            Math.round(drawAt + mPageMargin), mBottomPageBounds);
                    mMarginDrawable.draw(canvas);
                }

                if (drawAt > scrollX + width) {
                    break; // No more visible, no sense in continuing
                }
            }
        }
    }
`onDraw`只是绘画了页面的间隔效果。



# 两个常用操作
# setAdapter(PagerAdapter adapter) 
	/**
     * Set a PagerAdapter that will supply views for this pager as needed.
     *
     * @param adapter Adapter to use
     */
    public void setAdapter(PagerAdapter adapter) {
    
        if (mAdapter != null) {
            mAdapter.setViewPagerObserver(null);
            mAdapter.startUpdate(this);
            for (int i = 0; i < mItems.size(); i++) {
                final ItemInfo ii = mItems.get(i);
                mAdapter.destroyItem(this, ii.position, ii.object);
            }
            mAdapter.finishUpdate(this);
            mItems.clear();
            removeNonDecorViews();
            mCurItem = 0;
            scrollTo(0, 0);
        }
如果原来就有设置adapter的话，是会先清空旧的观察者，然后这个`startUpdate`函数，目前在FSPA和FPA里面都是没有用到，里面代码都为空，他的作用在官方的文档里面写的是`Called when a change in the shown pages is going to start being made.`。然后调用adapter的destroyItem，去销毁所有的item。对应的下面那个finishUpdate函数，我们看下官方的注释内容如下 
> Called when the a change in the shown pages has been completed.  At this
> point you must ensure that all of the pages have actually been added or
> removed from the container as appropriate.
> 所以这个是在告诉adaper一些添加/移除改动结束用的。

	    
做完上面的移除旧adapter工作后，后面对应的操作是否会合上面的有一个对应的类似操作呢？
毕竟销毁和创建总是搭配出现的。

        final PagerAdapter oldAdapter = mAdapter;
        mAdapter = adapter;
        mExpectedAdapterCount = 0;

        if (mAdapter != null) {
            if (mObserver == null) {
                mObserver = new PagerObserver();
            }
            mAdapter.setViewPagerObserver(mObserver);//
            mPopulatePending = false;
            final boolean wasFirstLayout = mFirstLayout;
            mFirstLayout = true;
            mExpectedAdapterCount = mAdapter.getCount();
            if (mRestoredCurItem >= 0) {
                mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
                setCurrentItemInternal(mRestoredCurItem, false, true);
                mRestoredCurItem = -1;
                mRestoredAdapterState = null;
                mRestoredClassLoader = null;
				//关于这个mResotredCurItem是在viewpager自己的onRestoreInstanceState记录的，
				//如果恢复有这个值就继续调到对应页面,当然，如果是第一次设置，这个值不是>=0，
				//所以不走这条路径
				
            } else if (!wasFirstLayout) {
                populate();
            } else {
                requestLayout();
            }
        }

关于这个wasFirstLayout，他的默认是true，然后在`onAttachedToWindow`里面也是再次设置为true。
只有在`onLayout`的时候设为false；
关于这个populaite很长，我们后面再慢慢看下
整个函数最后就是这个接口更新
 
        if (mAdapterChangeListener != null && oldAdapter != adapter) {
            mAdapterChangeListener.onAdapterChanged(oldAdapter, adapter);
        }
    }



 
## populate

这函数这么长，居然没在开头加注释，说下这函数是刚什么鬼的

	  void populate() {
        populate(mCurItem);//默认0
    }

    void populate(int newCurrentItem) {
        ItemInfo oldCurInfo = null;
        if (mCurItem != newCurrentItem) {//相等，跳过
            oldCurInfo = infoForPosition(mCurItem);
            mCurItem = newCurrentItem;
        }

        if (mAdapter == null) {   //不为空，跳过
            sortChildDrawingOrder();
            return;
        }

        // Bail now if we are waiting to populate.  This is to hold off
        // on creating views from the time the user releases their finger to
        // fling to a new position until we have finished the scroll to
        // that position, avoiding glitches from happening at that point.        
        if (mPopulatePending) {//在前面setAdapter里面mPopulatePending = false;
            if (DEBUG) Log.i(TAG, "populate is pending, skipping for now...");
            sortChildDrawingOrder();
            return;
        }

        // Also, don't populate until we are attached to a window.  This is to
        // avoid trying to populate before we have restored our view hierarchy
        // state and conflicting with what is restored.
        if (getWindowToken() == null) {
            return;
        }

        mAdapter.startUpdate(this);
        //看到熟悉的一句，开始做一些操作的标记


        final int pageLimit = mOffscreenPageLimit;
        final int startPos = Math.max(0, mCurItem - pageLimit);
        final int N = mAdapter.getCount();
        final int endPos = Math.min(N-1, mCurItem + pageLimit);

		//在继续之前他会做多判断，看数目和在setAdaper时候数目是否一样，变了就报错
        if (N != mExpectedAdapterCount) {
            String resName;
            try {
                resName = getResources().getResourceName(getId());
            } catch (Resources.NotFoundException e) {
                resName = Integer.toHexString(getId());
            }
            throw new IllegalStateException("The application's PagerAdapter changed the adapter's" +
                    " contents without calling PagerAdapter#notifyDataSetChanged!" +
                    " Expected adapter item count: " + mExpectedAdapterCount + ", found: " + N +
                    " Pager id: " + resName +
                    " Pager class: " + getClass() +
                    " Problematic adapter: " + mAdapter.getClass());
        }

        // Locate the currently focused item or add it if needed.
        int curIndex = -1;
        ItemInfo curItem = null;
        
		//前面并没往mItems添加过内容，所以循环不执行				
        for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
            final ItemInfo ii = mItems.get(curIndex);
            if (ii.position >= mCurItem) {
                if (ii.position == mCurItem) curItem = ii;
                break;
            }
        }        
 
        if (curItem == null && N > 0) {
            curItem = addNewItem(mCurItem, curIndex);
        }

然后这个addNewItem函数的内容，就是构建一个结构，记录了对应的position和object信息与宽度。
最后加mItems里面。我们看到的是这个函数里面调用了`mAdapter.instantiateItem(）`去初始化了我们的第一个页面，因为传过来的position就是0；整个viewpagr里面就这里有去调用`instantiateItem(）`

###  addNewItem

	ItemInfo addNewItem(int position, int index) {
	
	        ItemInfo ii = new ItemInfo();
	        ii.position = position;
	        ii.object = mAdapter.instantiateItem(this, position);
	        ii.widthFactor = mAdapter.getPageWidth(position);
	        if (index < 0 || index >= mItems.size()) {
	            mItems.add(ii);
	        } else {
	            mItems.add(index, ii);
	        }
	        return ii;
	}

继续看下面的内容，这作者就不能把这堆代码分几个函数出来吗？这么长

        // Fill 3x the available width or up to the number of offscreen
        // pages requested to either side, whichever is larger.
        // If we have no current item we have no work to do.
        if (curItem != null) {
            float extraWidthLeft = 0.f;
            int itemIndex = curIndex - 1;
            ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            //我们的curIndex为0,所以ii为null，这个重命名成preNode才感觉妥当            
            final int clientWidth = getClientWidth();
            // clientWidth=getMeasuredWidth()-getPaddingLeft()-getPaddingRight();
            
            final float leftWidthNeeded = clientWidth <= 0 ? 0 :
                    2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
            //2.F一个magic number。。。不知道什么意思
            //这个widthFActor是1.f，在addNewItem(）里面返回的
                    
                    
            for (int pos = mCurItem - 1; pos >= 0; pos--) {
            //下面这一堆主要是根据当前的位置和 最多有效页面数 ，来销毁页面也新建页面  
                if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
                    if (ii == null) {
                        break;
                    }
                    if (pos == ii.position && !ii.scrolling) {
                        mItems.remove(itemIndex);
                        mAdapter.destroyItem(this, pos, ii.object);                         
                        itemIndex--;
                        curIndex--;
                        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                    }
                } else if (ii != null && pos == ii.position) {
                    extraWidthLeft += ii.widthFactor;
                    itemIndex--;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                } else {
                    ii = addNewItem(pos, itemIndex + 1);
                    extraWidthLeft += ii.widthFactor;
                    curIndex++;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                }
            }

            float extraWidthRight = curItem.widthFactor;
            itemIndex = curIndex + 1;
            if (extraWidthRight < 2.f) {
                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                final float rightWidthNeeded = clientWidth <= 0 ? 0 :
                        (float) getPaddingRight() / (float) clientWidth + 2.f;
                for (int pos = mCurItem + 1; pos < N; pos++) {
                    if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
                        if (ii == null) {
                            break;
                        }
                        if (pos == ii.position && !ii.scrolling) {
                            mItems.remove(itemIndex);
                            mAdapter.destroyItem(this, pos, ii.object);                            
                            ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                        }
                    } else if (ii != null && pos == ii.position) {
                        extraWidthRight += ii.widthFactor;
                        itemIndex++;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    } else {
                        ii = addNewItem(pos, itemIndex);
                        itemIndex++;
                        extraWidthRight += ii.widthFactor;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    }
                }
            }

            calculatePageOffsets(curItem, curIndex, oldCurInfo);
            // 计算页面的偏移量,主要是ItemInfo里面的信息，很长，就不贴上来了
        }

	    ...
	    //设置当前的可见item为哪一个
        mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
        //在做延迟加载功能时候，我们有需要用到Fragment的setUserVisibleHint函数的回调。
具体的代码想下面这样

### setPrimaryItem
        
	@Override
    public void setPrimaryItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment)object;
        if (fragment != mCurrentPrimaryItem) {
            if (mCurrentPrimaryItem != null) {
                mCurrentPrimaryItem.setMenuVisibility(false);
                mCurrentPrimaryItem.setUserVisibleHint(false);
            }
            if (fragment != null) {
                fragment.setMenuVisibility(true);
                fragment.setUserVisibleHint(true);
            }
            mCurrentPrimaryItem = fragment;
        }
    }

继续回主线

        mAdapter.finishUpdate(this);//我们看下面这句表示了结束操作

        // Check width measurement of current pages and drawing sort order.
        // Update LayoutParams as needed.
        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            lp.childIndex = i;
            if (!lp.isDecor && lp.widthFactor == 0.f) {
                // 0 means requery the adapter for this, it doesn't have a valid width.
                final ItemInfo ii = infoForChild(child);
                if (ii != null) {
                    lp.widthFactor = ii.widthFactor;
                    lp.position = ii.position;
                }
            }
        }
        sortChildDrawingOrder();

        if (hasFocus()) {
            View currentFocused = findFocus();
            ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
            if (ii == null || ii.position != mCurItem) {
                for (int i=0; i<getChildCount(); i++) {
                    View child = getChildAt(i);
                    ii = infoForChild(child);
                    if (ii != null && ii.position == mCurItem){
                        if (child.requestFocus(View.FOCUS_FORWARD)) {
                            break;
                        }
                    }
                }
            }
        }
    }


###  sortChildDrawingOrder()

	  private void sortChildDrawingOrder() {
		//我们并没有设置pageTranstramer，所以默认为default，判断为false，不执行。
        if (mDrawingOrder != DRAW_ORDER_DEFAULT) {
            if (mDrawingOrderedChildren == null) {
                mDrawingOrderedChildren = new ArrayList<View>();
            } else {
                mDrawingOrderedChildren.clear();
            }
            final int childCount = getChildCount();
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                mDrawingOrderedChildren.add(child);
            }
            Collections.sort(mDrawingOrderedChildren, sPositionComparator);
        }
    }
 

以上我们简单的快速的看了下这个设置adapter的内容，我们现在来看下我们手动设置具体页面的时候，都经过什么样的逻辑代码
 
#  setCurrentItem(int item) 

	public void setCurrentItem(int item) {
	        mPopulatePending = false;
	        setCurrentItemInternal(item, !mFirstLayout, false);
	    }
	
	void setCurrentItemInternal(int item, boolean smoothScroll, boolean always) {
	        setCurrentItemInternal(item, smoothScroll, always, 0);
	    }

		 void setCurrentItemInternal(int item, boolean smoothScroll, boolean always, int velocity) {
        if (mAdapter == null || mAdapter.getCount() <= 0) {
            setScrollingCacheEnabled(false);
            return;
        }
        if (!always && mCurItem == item && mItems.size() != 0) {
            setScrollingCacheEnabled(false);
            return;
        }

        if (item < 0) {
            item = 0;
        } else if (item >= mAdapter.getCount()) {
            item = mAdapter.getCount() - 1;
        }
        final int pageLimit = mOffscreenPageLimit;

这个mOffscreenPageLimit默认为1；我们可以通过setOffscreenPageLimit（）函数来另外的设置这个值的大小，主要就是缓存的页面大小
        
        if (item > (mCurItem + pageLimit) || item < (mCurItem - pageLimit)) {
            // We are doing a jump by more than one page.  To avoid
            // glitches, we want to keep all current pages in the view
            // until the scroll ends.
            for (int i=0; i<mItems.size(); i++) {
                mItems.get(i).scrolling = true;
            }
        }
        final boolean dispatchSelected = mCurItem != item;

        if (mFirstLayout) {
            // We don't have any idea how big we are yet and shouldn't have any pages either.
            // Just set things up and let the pending layout handle things.
            mCurItem = item;
            if (dispatchSelected) {
                dispatchOnPageSelected(item);
            }
            requestLayout();
        } else {
            populate(item);
            scrollToItem(item, smoothScroll, velocity, dispatchSelected);
        }
    }

看我这部分，我们必要去看下他对于touch事件的处理。

# 事件分发
事件分发都是三步走的套路，一个`dispatchTouchEvent()` ，`onInterceptTouchEvent()`和一个`onTouchEvent()`；不过这里他没重写这个dispatchTouchEvent，因为他是直接继承与ViewGroup的，只有后面两个，所以我们来看下后面两个的吧


## onInterceptTouchEvent() 
关于拦截，一个点就是根据判断，看要不要处理这个点，对这个点做拦截，所以我们逻辑应该是看下对这个的判断逻辑是什么。


	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * This method JUST determines whether we want to intercept the motion.
         * If we return true, onMotionEvent will be called and we do the actual
         * scrolling there.
         */
        final int action = ev.getAction() & MotionEvent.ACTION_MASK;
        
原本也打算写下这个MotionEvent的介绍，不过前段时间都没太大的动力，
配套的一个demo案例[PullToZoomInListView](https://github.com/Sanjay-F/PullToZoomInListView)都上传好了，就是没开动，看哪天有想法再写吧。
 

        // Always take care of the touch gesture being complete.
        if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
            // Release the drag.
            if (DEBUG) Log.v(TAG, "Intercept done!");
            mIsBeingDragged = false;
            mIsUnableToDrag = false;
            mActivePointerId = INVALID_POINTER;
            if (mVelocityTracker != null) {
                mVelocityTracker.recycle();
                mVelocityTracker = null;
            }
            return false;
        }

        // Nothing more to do here if we have decided whether or not we
        // are dragging.
        if (action != MotionEvent.ACTION_DOWN) {
            if (mIsBeingDragged) { 
                return true;
            }
            if (mIsUnableToDrag) { 
                return false;
            }
        }

        switch (action) {
        
			 case MotionEvent.ACTION_DOWN: {
                /*
                 * Remember location of down touch.
                 * ACTION_DOWN always refers to pointer index 0.
                 */
                mLastMotionX = mInitialMotionX = ev.getX();
                mLastMotionY = mInitialMotionY = ev.getY();
                mActivePointerId = ev.getPointerId(0);
                mIsUnableToDrag = false;

                mScroller.computeScrollOffset();
                if (mScrollState == SCROLL_STATE_SETTLING &&                
                    Math.abs(mScroller.getFinalX()-mScroller.getCurrX())>mCloseEnough){

默认的初始化时idle状态，private int mScrollState = SCROLL_STATE_IDLE;

然后这个mCloseEnough就是2个dp的距离
private static final int CLOSE_ENOUGH = 2; // dp
final float density = context.getResources().getDisplayMetrics().density;
mCloseEnough = (int) (CLOSE_ENOUGH * density);
                   
                    // Let the user 'catch' the pager as it animates.
                    mScroller.abortAnimation();
                    mPopulatePending = false;
                    populate();
                    //这个函数我们在前面已经知道了，就是new一个页面出来。
                    
                    mIsBeingDragged = true;//通过这个标记，在开头的判断条件就帮助我们提前返回了
                    
                    //要求parent不要拦截事件
                    requestParentDisallowInterceptTouchEvent(true);
                    //更新状态为Dragging
                    setScrollState(SCROLL_STATE_DRAGGING);
                   
看完这部分，主要就是在滑动这个ViewPager的时候，我们把手指放上去，这时候能够定住页面，不要让他再滚动下去了。
                    
                } else {
                    completeScroll(false);
                    mIsBeingDragged = false;
                }
                break;
            }


            case MotionEvent.ACTION_MOVE: {
           
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    break;
                }

                final int pointerIndex = ev.findPointerIndex(activePointerId);
                final float x = ev.getX(pointerIndex);
                final float dx = x - mLastMotionX;
                final float xDiff = Math.abs(dx);
                final float y = ev.getY(pointerIndex);
                final float yDiff = Math.abs(y - mInitialMotionY);
             
                if (dx != 0 && !isGutterDrag(mLastMotionX, dx) &&
                        canScroll(this, false, (int) dx, (int) x, (int) y)) {
				 //由于嵌套的View在这个位置是可以滑动的，所以我们不拦截他，
				 //例如有一个水平滑动的listview之类结构
                     mLastMotionX = x;
                    mLastMotionY = y;
                    mIsUnableToDrag = true;
                    return false;
                }
                
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
	                //这个就是我们的拦截判断逻辑条件，
	                //根据滑动的时候水平方向和垂直方向的滑动大小
	                
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0 ? mInitialMotionX + mTouchSlop :
                            mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {
                    mIsUnableToDrag = true;
                }
                
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    if (performDrag(x)) {
	                    //这个函数主要就是根据当前的x去处理下那个边缘效果
	                    
                        postInvalidateOnAnimation();
                    }
                }
                break;
            }

            

            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                break;
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);
 
        return mIsBeingDragged;
    }


## onTouchEvent()

	@Override
    public boolean onTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN && ev.getEdgeFlags() != 0) {
            // Don't handle edge touches immediately -- 
            //they may actually belong to one of our descendants.
            //注释说的是对于边缘的点，有可能是自己的child的 ，所以自己不立马处理，返回false
            return false;
        }

        if (mAdapter == null || mAdapter.getCount() == 0) {
            // Nothing to present or scroll; nothing to touch.
            return false;
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        final int action = ev.getAction();
        boolean needsInvalidate = false;

		//接下来我们看下对于不同action的处理
		
        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_DOWN: {
            //对于down事件，也是常规的处理，暂停滚动，标记坐标信息
                mScroller.abortAnimation();
                mPopulatePending = false;
                populate();

                // Remember where the motion event started
                mLastMotionX = mInitialMotionX = ev.getX();
                mLastMotionY = mInitialMotionY = ev.getY();
                mActivePointerId = ev.getPointerId(0);
                break;
            }
			
			//对于move时间，它根据Drag分成两部分
			
            case MotionEvent.ACTION_MOVE:
                if (!mIsBeingDragged) {
                    final int pointerIndex = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(pointerIndex);
                    final float xDiff = Math.abs(x - mLastMotionX);
                    final float y = ev.getY(pointerIndex);
                    final float yDiff = Math.abs(y - mLastMotionY);
                    
                    if (xDiff > mTouchSlop && xDiff > yDiff) { 
						//和在拦截的时候类似的套路，没什么好说的
						
                        mIsBeingDragged = true;
                        requestParentDisallowInterceptTouchEvent(true);
                        mLastMotionX = x - mInitialMotionX > 0 ? 
				                       mInitialMotionX + mTouchSlop :					                       
		                                mInitialMotionX - mTouchSlop;
		                                
                        mLastMotionY = y;
                        setScrollState(SCROLL_STATE_DRAGGING);
                        setScrollingCacheEnabled(true);

						//这个预防万一，感觉也是怪怪的，估计是曾经出过什么错误，没找出来，
						//发现在这里加多一句妥当
						
                        // Disallow Parent Intercept, just in case
                        ViewParent parent = getParent();
                        if (parent != null) {
                            parent.requestDisallowInterceptTouchEvent(true);
                        }
                    }
                }
 
				 //这个没写成else状态，因为上面代码会存在标记为true的可能！
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    final int activePointerIndex = ev.findPointerIndex(
				                    mActivePointerId);
                    
                    final float x = ev.getX(activePointerIndex);
                    needsInvalidate |= performDrag(x);
                }                
                break;


			关于up事件，也是一些常规操作，计算当前页面，恢复状态，释放那个边缘效果，刷新界面
            case MotionEvent.ACTION_UP:
                if (mIsBeingDragged) {
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                    final int initialVelocity = (int) velocityTracker.getXVelocity(
							                    mActivePointerId);
                    mPopulatePending = true;

                    final float scrollStart = getScrollStart();
                    final float scrolledPages = scrollStart / getPaddedWidth();
                    final ItemInfo ii = infoForFirstVisiblePage();
                    final int currentPage = ii.position;
                    final float nextPageOffset;
                    if (isLayoutRtl()) {
                        nextPageOffset = (ii.offset - scrolledPages) / ii.widthFactor;
                    }  else {
                        nextPageOffset = (scrolledPages - ii.offset) / ii.widthFactor;
                    }

                    final int activePointerIndex = ev.findPointerIndex(
								                    mActivePointerId);
								                    
                    final float x = ev.getX(activePointerIndex);
                    final int totalDelta = (int) (x - mInitialMotionX);
                    //计算当前应该显示那个页面，例如我们滑动超过40%时候，认为要滑倒下一页
                    final int nextPage = determineTargetPage(
                            currentPage, nextPageOffset, initialVelocity, totalDelta);
                            
                    setCurrentItemInternal(nextPage, true, true, initialVelocity);

                    mActivePointerId = INVALID_POINTER;
                    endDrag();
                    mLeftEdge.onRelease();
                    mRightEdge.onRelease();
                    needsInvalidate = true;
                }
                break;


            case MotionEvent.ACTION_CANCEL:
                if (mIsBeingDragged) {
                    scrollToItem(mCurItem, true, 0, false);
                    mActivePointerId = INVALID_POINTER;
                    endDrag();
                    mLeftEdge.onRelease();
                    mRightEdge.onRelease();
                    needsInvalidate = true;
                }
                break;

            case MotionEvent.ACTION_POINTER_DOWN: {
                final int index = ev.getActionIndex();
                final float x = ev.getX(index);
                mLastMotionX = x;
                mActivePointerId = ev.getPointerId(index);
                break;
            }
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionX = ev.getX(ev.findPointerIndex(mActivePointerId));
                break;
        }
        if (needsInvalidate) {
            postInvalidateOnAnimation();
        }
        return true;
    }

# 小结

这篇文章已经写得很长了，看了下有2万5的字数，对ViewPager的大部分内容我们是有一个大概的了解了。
原本也想写多关于FSPA和FPA的内容，不过好吃了还是算了把，单独起一篇来说