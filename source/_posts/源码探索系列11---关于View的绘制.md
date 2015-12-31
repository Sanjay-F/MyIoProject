title: 源码探索系列11---关于View的绘制
date: 2015-12-27 02:36
tags: [android,源码,View,绘制]
categories: android
------------------------------------------

我们开发过程，基本需要自定义View，画一些自己的小插件出来
这需要我们掌握整个View的绘画过程和一些别的小技巧。
这里总结下整个View的源码中涉及到的一些绘制过程的核心部分，
之后再来看下整体的内容，毕竟整个源码有近2W1行，不是随便一时半会能搞定的，还是得下不少功夫。

<!--more-->

# 起航 ------ 绘制流程

API：23

一般View的“生命周期”即绘画的流程像下面这样。
```flow
st=>start: View的绘画流程
 
op=>operation: measure()

op2=>operation: layout()

op3=>operation: draw()

e=>end: 结束

st->op->op2->op3->e

```
 这个是一般的流程都这样，
 
1. 我们的`measure`负责去测量View的`Width`和`Height`,
2.  然后我们的`layout`负责去确定其在父容器的位置，
3.  最后由`draw`来负责在屏幕上画内容。

但实际还有一些别的步骤流程，如这些函数由上一层来调用， 就像我们的Activity的`onCreate`等！
不过现在先不提及。我们继续看各个阶段具体到底是做什么先。

## measure

	public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
       ...
       
       onMeasure(widthMeasureSpec, heightMeasureSpec);
       
       ...
    }

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
	public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
      
我们的measure函数是个final类型的，里面主要是调用了onMeasure函数，由他做具体测量。
这里需要补充一部分内容，关于`MeasureSpec.EXACTLY`，`MeasureSpec.AT_MOST`和`MeasureSpec.UNSPECIFIED`

- **EXACTLY：**
 这个词的意思是父容器已经检测出View的精确大小（eg:width=200dp/match_parent）,这时我们的View的最终大小值就是specSize的值。
-  **AT_MOST：** 
 这个词的意思是父容器指定了一个大小（eg:width=wrap_content）,这时我们的View的大小是要小于等于specSize的值，最终大小到底是多大，要看View的具体实现。
-  **UNSPECIFIED：**
这个词的意思是父容器不对View有任何大小的限制，需要多大就设置多大，但这一般是系统内部用来表示一种测量的状态。当然还有别的用处，例如我们的`ScrollView`，他就可以用这个来告诉子View，大小无限，任意画。
 
上面的解释看起来这个View的MeasureSpec类型由我们的LayoutParams来设置，但实际这个MeasureSpec是由View和父容器一起决定的。这个好理解，例如我们的LinearLayout里面有个View，前者设置最高为200dp，后者为300dp，最终这个子View大小不由自己设置的300dp决定。具体的测量过程，下次再开贴说，就不插在这里了。我们继续主线

这样我们回看上面应该就好理解getDefaultSize（）里面的到底是什么意思了。
在  `MeasureSpec.UNSPECIFIED:`的状况下，大小是`result = size;`，由传过来的参数觉得，我们看下具体做了什么
            
    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
    }
 
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
我们拿`getSuggestedMinimumHeight()`来看下
里面含义就是：

- 如果我没背景，那么就是mMinHeight大小，这个值对应于我们写的`android:minHeight="20dp"`属性，他的默认值是0。

		case R.styleable.View_minWidth:
	                    mMinWidth = a.getDimensionPixelSize(attr, 0);
	                    break;

-	如果我有背景，那就选背景的最小高度和mMinHeight中最大的。
  这个背景的`getMinimumHeight()`内容是
	  
		  /**
	     * Returns the minimum height suggested by this Drawable. If a View uses this
	     * Drawable as a background, it is suggested that the View use at least this
	     * value for its height. (There will be some scenarios where this will not be
	     * possible.) This value should INCLUDE any padding.
	     *
	     * @return The minimum height suggested by this Drawable. If this Drawable
	     *         doesn't have a suggested minimum height, 0 is returned.
	     */
		  public int getMinimumHeight() {
	        final int intrinsicHeight = getIntrinsicHeight();
	        return intrinsicHeight > 0 ? intrinsicHeight : 0;
	    }	    
自带的解释已经很具体了，返回Drawable的最小高度，没有的话就返回0；可能有些奇怪，说得好像我们的Drawable可以没高是的。确实有些没有，例如我们在自定义一些我们的**圆角**的Button在不同点击效果时候用到的`<shape>`标签写的背景，他就没有。

继续回主线

	protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        ...
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

	private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
    
最后就设置了测量的大小，是的测量的大小，不是最终的大小，最终的大小还是需要根据实际做调整的。
这样我们的measure，测量过程就基本结束了。

### **一些题外话：**

这里补充一个早年无知时候遇到的坑，那时候项目要求弄一个像下面这样的一个带有气泡框的进度条， 

![这里写图片描述](http://img.blog.csdn.net/20151227115323242)

那时候就直接类似于下面这样，继承View，然后重写onDraw函数，在里面绘制好整个样子。

	public class BubbleProgressBar extends View {

			public void onDraw(@NonNull Canvas canvas) {
			      //画进度和泡泡框
			}
	}	
但画好后，遇到个问题，这个View居然自动填充满整个界面，我设置的是`wrap_content`，感觉应该是系统帮我搞好，弄成很小的一个啊，怎么就那么大呢？  后来查了资料发现，如果我们是直接继承于View，那需要重写下那个measure函数，要不然他就会自动填满，为啥呢？回看那个`getDefaultSize`函数

	case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
           result = specSize;
         break;
我们的`wrap_content`就是那个`AT_MOST`和`EXACTLY`是同条路，实际就等于写了`Match_parent`。
所以我们得根据情况来做判断，来给点指定大小
	
	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
    
        if(heightMode==MeasureSpec.AT_MOST widthMode == MeasureSpec.AT_MOST ){
           setMeasuredDimension(mOurDefalutHeight,mOurDefalutWidth);
		} 		
		 ...
	}	
现在想想，大概当年设计这个View类的人遇到了这个默认初始化大小应该是多大才合适的问题，所以干脆直接来个填充全局的方式。

# 前进 ------ Layout过程

看完了测量出界面的大小，我们需要开始下一步layout的过程了。
layout主要是用来确定View的位置的，具体如下
	
	public void layout(int l, int t, int r, int b) {
       
        ...

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
 
	     ...
    }

整个流程大致是先用setFrame（）函数去设置我们的View的位置，然后调用onLayout来让服从其确定之元素的位置，由于这个onLayout做的是具体的布局工作，需要具体的继承的人去做，例如我们的LinearLayout有水平和垂直之分，所以在View中里面什么也没有。最后是调用监听函数，通知他们onLayoutChange（）了。

	 protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
 
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

			 ...
        }
        return changed;
    }

	/**
     * Called from layout when this view should  
     * assign a size and position to each of its children.
     *
     * Derived classes with children should override this method 
     * and call layout on each of their children. 
     */
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }

好了，基本的layout过程就这么结束了，我们的View的布局也就确定了。
接下来就看下draw过程了。


# 前进前进------画界面的Draw

这个画的过程，主要就是把View绘制到屏幕上去，根据写的注释，我们看到View的绘制过程有这里六个步骤。其中两个可以忽略的。

	 /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background  
         *      2. If necessary, save the canvas' layers to prepare for fading 
         *      3. Draw view's content
         *      4. Draw children  
         *      5. If necessary, draw the fading edges and restore layers  
         *      6. Draw decorations (scrollbars for instance) 
         */
继续的步骤如下：


	public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

 
        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);

            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */
         
			... 画特效部分
     }
我们再细看下各个步骤
	
	private void drawBackground(Canvas canvas) {
        final Drawable background = mBackground;
	    ...

        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        if ((scrollX | scrollY) == 0) {
            background.draw(canvas);
        } else {
            canvas.translate(scrollX, scrollY);
            background.draw(canvas);
            canvas.translate(-scrollX, -scrollY);
        }
    }
然后这个onDraw和我们onLayout一样，需要自己写，里面空空如也

	protected void onDraw(Canvas canvas) {
	    }
然后那个dispatchDraw(）也是，这个需要我们自己做，但这个更多的是针对于ViewGroup类的包含子View的。这样Draw事件就传递给下面，遍历所有的子View元素的Draw方法，绘制完所有。
	
	/**
     * Called by draw to draw the child views. This may be overridden
     * by derived classes to gain control just before its children are drawn
     * (but after its own view has been drawn).
     * @param canvas the canvas on which to draw the view
     */
    protected void dispatchDraw(Canvas canvas) {

    }
    
这样我们的Draw过程也就介绍了。

---
# 一些补充

看完一个完整的View的绘制过程，这里补充一些关于ViewGroup的内容
ViewGroup绘制过程中还需要让他的各个子View去绘制。

## measureChildren(）

	 protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
这里看到，他对于那些除了设置为Gone不可见的，都进行了绘制。
不过有一个点引起我的兴趣，这个`size`的大小不是取数组`children`的大小，而是`mChildrenCount`这个值。难道这背后有一个什么故事？查了下没什么结果。。。

	protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
绘制的过程也是直接调用他们的measure函数去执行。在获取到子View的MeasureSpec时，具体是：

	public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
    
这里面做的事情，主要的就是根据父容器的MeasureSpec同时结合View本身的LayoutParams来共同决定子View的MeasureSpec，所以子元素能用的大小就是父容器的尺寸减去padding
	
        int specSize = MeasureSpec.getSize(spec);
        int size = Math.max(0, specSize - padding);
前面在说View的时候也有提到过这个，具体的View的大小是需要和父容器协商的。

根据上面的内容的决定子View的大小的过程，我们可以总结出一个规律，就是如果我们设置了具体的大小（dp/px)那就是ChildSize，要不然是ParentSize除了UNPSECIFIED，


 
| childParams \   parentParams| EXACTLY | AT_MOST | UNSPECIFIED |
|---| :------------- |:-------------| :-----|
|**dp/px**| EXACTLY - childSize     | EXACTLY - childSize | EXACTLY - childSize |
|**match_parent**| EXACTLY - parentSize| EXACTLY - parentSize | UNSPECIFIED - 0 |
|**wrap_content**| EXACTLY - parentSize | EXACTLY - parentSize |UNSPECIFIED - 0 |


# 后记

一个View的绘制过程就这样结束了，也没太大负责的内容，但一个View里面的内容还是很多可以说的，
例如：

- 他内部的`post`机制，他可以让我们减少对Handler的使用。
- Touch事件的传递
- View的滑动

这些内容我们后面继续慢慢的补充吧。 

另外这个View的调用者是`ViewRoot`，他的具体实现是`ViewRootImpl`，在他的performTraversal函数里面，执行了我们的view的整个绘制周期的调用

```flow
st=>start: performTraversals（）
e=>end: 结束

op1=>operation: View.measure
op2=>operation: View.layout
op3=>operation: View.draw

cond1=>condition: 不用重新Measure?
cond2=>condition: 不用重新Layout? 
cond3=>condition: 不用重新Draw?

st->cond1->cond2 
cond1(yes)->cond2
cond1(no)->op1
cond1->cond2->cond3
cond2(yes)->cond3
cond2(no)->op2
cond3(yes)->e
cond3(no)->op3

```
更具体的调用流程如下： 

```sequence

performMeasure->measure: 

measure->onMeasure: 
onMeasure--> View.measure:
```

 另外我们的layout和draw的套路类似，就不细写.