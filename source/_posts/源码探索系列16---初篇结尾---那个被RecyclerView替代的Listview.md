title:  源码探索系列16---初篇结尾---那个被RecyclerView替代的Listview 
date: 2015-12-30 00:27:46
tags: [android,源码,ListView]
categories: android
------------------------------------------


作为这源码探索系列的初篇的终结，选择Listview来做最后一个探索的对象好像也挺好的。
所以我们就来简单的看下我们曾经最熟悉的Listview是怎么去绘制我们的各个View，如何复用的View。

# 起航

API：23

我们的`Listview`是继承于`AbsListView` 的

	public class ListView extends AbsListView 
而这个AbsListView里面帮我们做了大量的工作，包括与我们的adapter相关的工作，对View的复用等。
<!--more-->
我们就从我们的最熟悉的一句`setAdapter（）`开始吧

	@Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }

        resetList();
        mRecycler.clear();

        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
            mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }

        mOldSelectedPosition = INVALID_POSITION;
        mOldSelectedRowId = INVALID_ROW_ID;

        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);

        if (mAdapter != null) {
            mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
            mOldItemCount = mItemCount;
            mItemCount = mAdapter.getCount();
            checkFocus();

            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);

            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

            int position;
            if (mStackFromBottom) {
                position = lookForSelectablePosition(mItemCount - 1, false);
            } else {
                position = lookForSelectablePosition(0, true);
            }
            setSelectedPositionInt(position);
            setNextSelectedPositionInt(position);

            if (mItemCount == 0) {
                // Nothing selected
                checkSelectionChanged();
            }
        } else {
            mAreAllItemsSelectable = true;
            checkFocus();
            // Nothing selected
            checkSelectionChanged();
        }

        requestLayout();
    }
看惯了几百行一个的函数，现在整个这么长贴起来都不觉得长了。
现在来解释下，开头的这里两句，如原来有adapter的话，就注销掉数据观察者`mDataSetObserver`
	
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }
还记得我们每次修改数据后，都去调用下面这句吗？

	notifyDataSetChanged();
因为底层在数据变化上使用的是`观察者模型`来进行解耦操作。

	public void notifyDataSetChanged() {
	        mDataSetObservable.notifyChanged();
	    }
    
    public class DataSetObservable extends Observable<DataSetObserver> {
 
	    public void notifyChanged() {
	        synchronized(mObservers) { 
	            for (int i = mObservers.size() - 1; i >= 0; i--) {
	                mObservers.get(i).onChanged();
	            }
	        }
	    }
	 
	    public void notifyInvalidated() {
	        synchronized (mObservers) {
	            for (int i = mObservers.size() - 1; i >= 0; i--) {
	                mObservers.get(i).onInvalidated();
	            }
	        }
	    }
	}
在我们的AbsListview定义了这个DataSetObserver

	class AdapterDataSetObserver extends AdapterView<ListAdapter>.AdapterDataSetObserver {
        @Override
        public void onChanged() {
            super.onChanged();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }

        @Override
        public void onInvalidated() {
            super.onInvalidated();
            if (mFastScroll != null) {
                mFastScroll.onSectionsChanged();
            }
        }
    }
最后在onChange（）函数会去调用`requestLayout()` ，从而重新绘制界面。

我们继续看下去
	
		mRecycler.clear();

        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
            mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }
这里提下这个`mRecycler`，他和我们的界面重用有关，即他会保存我们的每个Item的View。
然后就是设置mAdapter这事了，如果我们原来就有设置头和脚的话，就会对我们传递过来的adapter做下修改，看来还算挺好的。
接着就是让父类设置下新的adapter，然后就是一堆的设置了，包括对新的额adapter设置数据观察员`mDataSetObserver`， 告诉`mRecycler`新的适配器有多少种View类型，好让他做准备，重新设置选中的item的位置。

最后就是请求刷新界面的啦

	requestLayout();
	
我们去看下怎么个布局，关于View的刷新过程，可以去看下**源码探索系列11---关于View的绘制**，具体不说，我们直接去看下`layout`函数，下面是`AbsListView`的`Onlayout`

	@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        mInLayout = true;
        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        layoutChildren();
        mInLayout = false;
        
		...
    }
这里我们看到，他会去重新绘制界面，同时还去调用了 `layoutChildren();`，这个方法是空的，之类需要重写他，我们去看下Listview里面写了什么，这个方法不短，也有个三百多行的。我们截取关键的点

	protected void layoutChildren() {
		...
        try {
		    ...   
            switch (mLayoutMode) {
                ...
             case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
             ...
             }
		  ...
         } finally {
           ...
        }
    }
这里有一个有意思的点，就是布局的时候，有两种方式一个是从下到上，另一种是从上到下的布局。平时都没去注意这个布局是从那开始的。我们看下`fillUp`，从下到上的布局过程的

	private View fillUp(int pos, int nextBottom) {
        View selectedView = null;
        int end = 0; 
        ...

        while (nextBottom > end && pos >= 0) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            View child = makeAndAddView(pos, nextBottom, false, mListPadding.left, selected);
            nextBottom = child.getTop() - mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos--;
        }
        mFirstPosition = pos + 1;
        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
这里重要的一句就是`makeAndAddView(）`函数，他返回的View就是我们的ListView中对应的每个Item。

	private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        View child;
        if (!mDataChanged) { 
            child = mRecycler.getActiveView(position);
            if (child != null) {
                 setupChild(child, position, y, flow, childrenLeft, selected, true);
                return child;
            }
        }
         
        child = obtainView(position, mIsScrap);
 
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);
        return child;
    }
这里我们又看到了`mRecycler`这家伙，前面已经有提到过他和我们的复用有关，这里的`getActiviView()`函数给了更直接的证据，很显然他保存着我们的View。如果这个Child不是空的话，就去`setUpChild()`,让我们去给我们绑定数据到View上，但这的前提当然就是我们的数据没变啊，View还可以用的情况下。

如果变了的话，他就调用`obtainView()`去New一个出来，再让我们去初始化我们的View。
我们先去看下`obtainView()`，在看那个`setupChilde()`

	View obtainView(int position, boolean[] isScrap) {
	     ...
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else {
                isScrap[0] = true;
                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }

		 ...
        return child;
    }
看到这里，我们看到了熟悉的一个地方`mAdapter.getView(position, scrapView, this);`	
这句话就是我们每次写adapter时候遇到的地方

	@Override
    public View getView(int position, View view, ViewGroup viewGroup) {
        ViewHolder holder;
        if (view == null) {
            view = mInflater.inflate(R.layout.list_item_folder, viewGroup, false);
            holder = new ViewHolder(view);
        } else {
            holder = (ViewHolder) view.getTag();
        }
        ...
	}
对于我们的Adapter返回的View，会被保存起来`mRecycler.addScrapView(scrapView, position);`
这样下次有人问题这个ListView是怎么做到复用的呢？
一定要说用一个`mRecycler`来保存的，这句话说出来就像看过源码的人。
通过这个mRecycler来保存我们每次infalte后的View，避免了频繁的去生成，从而提高效率的，当然我们都知道，这要求我们滚了一屏才可能，因为过了一屏，新的item的View才是复用原来的。

继续回主线，我们去看下setupChild的内容（）

	private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean recycled) {
            
        ...
        p.viewType = mAdapter.getItemViewType(position);

        if ((recycled && !p.forceAdd) || (p.recycledHeaderFooter
                && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
            attachViewToParent(child, flowDown ? -1 : 0, p);
        } else {
            p.forceAdd = false;
            if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                p.recycledHeaderFooter = true;
            }
            addViewInLayout(child, flowDown ? -1 : 0, p, true);
        }
	    ... 
    }
对于我们的child，分了两种，一种是相对特殊的头和脚，一种是正常的item。后者是直接加上去，没什么说，我们看下前者

	protected void attachViewToParent(View child, int index, LayoutParams params) {
        child.mLayoutParams = params;

        if (index < 0) {
            index = mChildrenCount;
        }

        addInArray(child, index);

        ...
    }
这里主要是把我们的头和脚加入到数组里面去，方便在使用`getChildAt(int)`的时候可以获得这个View。 

到这里基本对Listview的重要的部分就结束啦,他设计到的AdapterView下次再做详细的解析吧

# 后记

整个的关于源码探索系列的初篇就这样结尾啦！

后面有时间和想法，再写下中篇和大结局篇嘛？

开始写一点设计模式的内容，我们今天的Listview就有用到观察者和适配器模型两种！


