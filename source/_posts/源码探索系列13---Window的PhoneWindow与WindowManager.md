title: 源码探索系列13---Window的PhoneWindow与WindowManager
date: 2015-12-28 17:36
tags: [android,源码,window,windowManager,PhoneWindow]
categories: android
------------------------------------------

关于Window，PhoneWindow和WindowManager三者的关系是：

> Window是一个抽象类，他的具体实现是PhoneWindow
> 我们通过WindowManager来管理Window。

我们的所有的界面，例如Activity，Toast，Dialog等都是靠Window来呈现，因此他是View的管理者咯。

很久前用过他的一个功能就是拿来做消息提醒，当有用户发来特定消息时候，就跑出来一个小悬浮窗来提示用户。现在看得多的就是各种管家关于内存的提示！这真的很让人纠结的安卓机啊！特别是某米的手机，当年发现他不支持这个功能！搞到得单独为他开发一个别的方式....

好了，说这么多，我们该从哪里开始说起呢？

<!--more-->

# 起航

API：23

 我们先挑WindowManager 开始吧。

	public interface WindowManager extends ViewManager 

我们的WiddowManger是一个接口，不是实际干活的人，看下他继承的ViewManager	

	public interface ViewManager{ 
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
	}
定义了我们使用到的三个操作。增加，更新，移除。
那么背后到底谁在干活呢，是`WindowManagerImpl`
不过很有意思的是，他也是一个壳，实际干活的都交给了WindowManagerGlobal去干了。

	public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    
		@Override
	    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
	        applyDefaultToken(params);
	        mGlobal.addView(view, params, mDisplay, mParentWindow);
	    }
		@Override
		public void removeView(View view) {
		     mGlobal.removeView(view, false);
		}

	    @Override
	    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
	        applyDefaultToken(params);
	        mGlobal.updateViewLayout(view, params);
	    }
	    ...
	}
## addView

我们继续跟下，看在addView背后做了什么

	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        
         ... 

		final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        ViewRootImpl root;
        View panelParentView = null;
        synchronized (mLock) {
        
            ...

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }
        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            ...
        }
    }
    
	private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
我们看到他主要是创建了新的ViewRootImpl，然后将View，ViewRootImpl和params保存起来，最后将View添加进去ViewRootImpl里面去。这里有种感觉，removeView做的就可能和上面的相反，从数组移除，然后调用root.removeView()的类似函数去移除。这是猜测，我们先留着，看完我们的主线

我们来看下这个setView里面做了什么

	public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view; 

				... 
                requestLayout();
                
				... 
               
	            try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
					...
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
            }
        }
    }
通过requestLayout，然后去更新界面。

	@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
    
	void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
  那个mTraversalRunnable最后回去调用   performTraversals()，执行对界面的更新。
	  
	  final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }  

	void doTraversal() {
        if (mTraversalScheduled) { 
           ...
            performTraversals(); 
            ..
        }
    }
关于这个函数，我们前面在讲View的绘制的时候有说到，他最终去调用我们View的measure，layout，draw等方法，在这里就不深入说了。    
	 继续回主线
	 
	  res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
	                   getHostVisibility(), mDisplay.getDisplayId(),
	                   mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
	                   mAttachInfo.mOutsets, mInputChannel);
这mWindowSession是一个IWindowSession类，就一个Bidner，具体干活的是Session类。

	@Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
            
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
                
    }
这里看到他最终是用WindowManagerService去添加Window的。这个WMS深似海和AMS一样的坑啊.每个函数都是些几百号才能结束的.我们下次有空再深入看下，还有PW很多内容没说呢，已经写了很长了。。

## removeView
和原来的套路一样，我们到WMG去看下，验证下和一开始的想法是不是一样的

	@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
    
	public void removeView(View view, boolean immediate) {
		...
        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            } 
        }
    }
    
根据索引去移除View，没什么好说的，继续
	
	private void removeViewLocked(int index, boolean immediate) {
	
	       ViewRootImpl root = mRoots.get(index);
	       View view = root.getView();
	
	       if (view != null) {
	           InputMethodManager imm = InputMethodManager.getInstance();
	           if (imm != null) {
	               imm.windowDismissed(mViews.get(index).getWindowToken());
	           }
	       }
	       
	       boolean deferred = root.die(immediate);
	       if (view != null) {
	           view.assignParent(null);
	           if (deferred) {
	               mDyingViews.add(view);
	           }
	       }
	   }
关于这个InputMethodManager，在前面的添加Window的时候就一直出现，这里没把它去掉，是因为想说件事，在整个建立的过程中，是涉及到了Input子系统事件分发流程，建立一个通讯的过程，这个也是一匹布那么长，我们忽略跳过他。。我们继续看下那个ViewRootImpl的`die()`方法。

	boolean die(boolean immediate) {
        // Make sure we do execute immediately 
        //if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        }  
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
我们的immediate是一个false，所以执行的只是下面的用handler去发送一个消息就结束了	，然后返回。
这里我们回去看下那个windowMangerImpl的函数，他有两个remove方法

	@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

    @Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
结合上面的函数，我们的出结论，实际移除有同步和异步的方式，同步的方法可能带来一些问题。
异步方式是发送消息返回true，而`removeViewLocked`还把View加入`mDyingViews.add(view)`死亡数组去。呵呵，我们继续看下去

	case MSG_DIE:
	     doDie();
	     break;
这个消息执行的也是doDie方法啊。。

	void doDie() {
		...
        dispatchDetachedFromWindow();
		WindowManagerGlobal.getInstance().doRemoveView(this);
        ... 
    }

看下dispatchDetachedFromWindow去做了什么
	
	void dispatchDetachedFromWindow() {
	
        if (mView != null && mView.mAttachInfo != null) {
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(false);
            mView.dispatchDetachedFromWindow();
        }
        mAccessibilityInteractionConnectionManager.ensureNoConnection();
        mAccessibilityManager.removeAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mAccessibilityManager.removeHighTextContrastStateChangeListener(
                mHighContrastTextManager);
        removeSendWindowContentChangedCallback();

        destroyHardwareRenderer();

        setAccessibilityFocus(null, null);

        mView.assignParent(null);
        mView = null;
        mAttachInfo.mRootView = null;

        mSurface.release();
		...
        try {
            mWindowSession.remove(mWindow);
        } catch (RemoteException e) {
        }
        mDisplayManager.unregisterDisplayListener(mDisplayListener);

        unscheduleTraversals();
    }
整个过程先调用View的`dispatchDetachedFromWindow()`，然后去一堆变量，接着是Session去remove()。

	public void remove(IWindow window) {
        mService.removeWindow(this, window);
    }
好了，我们看到他去调用了AMS的removeWindow方法，和一开添加的时候的设想一直。    

最后在我们的`doDie（）`里面是用`WindowManagerGlobal.getInstance().doRemoveView(this);`移除

	void doRemoveView(ViewRootImpl root) {
        synchronized (mLock) {
            final int index = mRoots.indexOf(root);
            if (index >= 0) {
                mRoots.remove(index);
                mParams.remove(index);
                final View view = mViews.remove(index);
                mDyingViews.remove(view);
            }
        }
        if (HardwareRenderer.sTrimForeground && HardwareRenderer.isAvailable()) {
            doTrimForeground();
        }
    }
这部分工作和我们一开始的设想的内容一致啦。

好了，大致的过程我们基本就看完了，最后的update部分类似，就不细写了。

# 前进

看完了Window的增加，我们来看下我们的Window吧。
关于Window和Activity的建立过程，查了下资料，背后还是挺复杂的，我们就从一个我们属性的方法入手

	setContentView(R.layout.activity_main);

	public void setContentView(int layoutResID) {
	        getWindow().setContentView(layoutResID);
	        initWindowDecorActionBar();
	}
这里的getWindow函数就是我们的Window，他的具体实现是PhoneWindow，我们跳过去看下
	
	@Override
    public void setContentView(int layoutResID) { 
    
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
	     ...
    }
先初始化decorView，然后有句熟悉的`mLayoutInflater.inflate(layoutResID, mContentParent)`
传递了mContentParent去inflate，我们需要去看`installDecor()`里面做了什么。
 
	private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor(); 
            mDecor.setIsRootNamespace(true); 
            ...
        }
 
        if (mContentParent == null) {
	        mContentParent = generateLayout(mDecor);
	        ...
	    }
    }
    
我们看到的是，这个新生成了一个DecorView，最后通过这个mDecor去生成一个mContentParent。

	protected ViewGroup generateLayout(DecorView decor) {
      
        ...  		 

        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
        
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
 		 ...
      
        return contentParent;
    }
    
这里我们看这里会去inflate我们的View，然后将它赋值给我们的decor，需要补充说下这个inflate的内容就是我们的整个手机界面，前面会根据不同的选择判断来确定是要那个layoutResource的。
所以我们每次例如要求请求全屏等，就需要在这个setContentView前面先调用就是这个原因！

最后这个`contentParent`似乎看起来和我们的DecorView没什么关系，我们继续看下。
   
	public View findViewById(@IdRes int id) {
        return getDecorView().findViewById(id);
    }
    
 这个`getDecorView（）`返回的就是我们的`DecorView`类的mDecor，我们已经说过他就是PhoneWindow的一个内部类。通过给他一个ID来获取我们的ContentParent，直觉就是说这个是在DecorView里面的一个childView咯？当然是的，因为前面我们通过inflate得到的view加到我们的decor的时候，那个布局文件里面就有了这个ID了。
	 
	@Override
    public final View getDecorView() {
        if (mDecor == null) {
            installDecor();
        }
        return mDecor;
    }
到这里我们重新建立了我们的`decor`和我们的`mContentParent`的关系，即后者是前者的一个子View。
这样我们看完了`installDecor()`继续回到主线，他的下一句

	mLayoutInflater.inflate(layoutResID, mContentParent);
我们看到他是根据传过来的我们写的布局ID文件，然后加到我们的mContentParent里面去的。

 在继续看下去之前，我们补充一张图片，这里可以解释下，就是我们的DecorView里面包着我们的界面。
 我们的ActionBar和我们的自定义的布局文件。
 
![这里写图片描述](http://img.blog.csdn.net/20151227164259739)

有了这个概念，我们继续下面关于inflate部分的介绍:

	public View inflate(int resource, ViewGroup root) {
        return inflate(resource, root, root != null);
    }

	public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
       
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }

	public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
      
         final AttributeSet attrs = Xml.asAttributeSet(parser);
          Context lastContext = (Context)mConstructorArgs[0];
          mConstructorArgs[0] = mContext;
          View result = root;

          final String name = parser.getName();
          
         // Temp is the root view that was found in the xml
         final View temp = createViewFromTag(root, name, attrs, false);

         ViewGroup.LayoutParams params = null; 
         
         ...
         // Inflate all children under temp
         rInflate(parser, temp, attrs, true, true);
          ...

         if (root != null && attachToRoot) {
             root.addView(temp, params);
         }

	     ...
        return result;
   
    }
    
整个过程就成功的根据我们给的id去找到view，然后加到我们的root里面去。

这样我们的整个界面的过程是完成了。
另外对于这个PhoneWindow的内容还有很多没说，其余的设计到了界面的一些设置等等的内容。

	public void setStatusBarColor(int color)
	public void setNavigationBarColor(int color)
	public void setTitle(CharSequence title)
	public void setLogo(int resId) 
	public void setLogo(int resId) 

 



---

# 后记

1. **桥接模式**
这次看了下还是有点收获的，例如我们在`WindowManger`与windowGloabl里面看到了`桥接模式`的使用

2.  设置屏幕属性需要在`setContentView`前

	    requestWindowFeature(Window.FEATURE_NO_TITLE); //设置无标题
		getWindow().setFlags(WindowManager.LayoutParams.FILL_PARENT, 
						    WindowManager.LayoutParams.FILL_PARENT);  //设置全屏  
		setContentView(R.layout.activity_main );
以前可以这么写全屏，但需要在		setContentView(）函数前，原因我的Window的创建是在他里面。
