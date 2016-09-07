关于ViewPager，FragmentStatePagerAdapter和FragmentPagerAdapter
 

为了便于描述，把FragmentStatePagerAdapter记为`FSPA`，FragmentPagerAdapter记为`FPA`;



	public class ViewPager extends ViewGroup 
ViewPager是继承于ViewGropu的，我们先来看下一个view的三个步骤onMeasure，onLayout，onDraw三个函数，在去看下设置adapter的内容，以及选中特定页面的逻辑

# onMeasure

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    
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

# OnLayout
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

 

## scrollToItem
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

# draw

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
我们在水平滑动到左边缘和右边缘的时候，我们可以看到一个半透明的东西，就像那个listview下拉的时候，顶部有一个东西在表示到。


# onDraw(Canvas canvas)

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
                    if (ii != null && ii.position == mCurItem) {
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

