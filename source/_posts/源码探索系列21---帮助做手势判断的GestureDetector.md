title: 源码探索系列21---帮助做手势判断的GestureDetector
date: 2016-01-11 15:37
tags: [android,源码,GestureDetector]
categories: android

------------------------------------------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/rdn_5215744b835e8.jpg)

很久前开始写安卓程序时候，对双击事件的判断都是直接在touch事件里面做判断的，后来听说了有个`GestureDetector`，发现真是大法好啊。

今天我们就来看下这个类到底做了什么，可以识别出多种不同的手势。

# 前进
API:23

开始分析前，我们看下使用方法

	//在我们关注的地方由mGestureDetector来处理MotionEvent
	mDoubleBtn.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                mGestureDetector.onTouchEvent(event);
                return false;
            }
        });

    mGestureDetector = new GestureDetector(this, onGestureListener);
     mGestureDetector.setOnDoubleTapListener(new GestureDetector.OnDoubleTapListener() {
         @Override
         public boolean onSingleTapConfirmed(MotionEvent e) {

             return false;
         }

         @Override
         public boolean onDoubleTap(MotionEvent e) {
             Log.e("double", " double");
             return false;
         }

         @Override
         public boolean onDoubleTapEvent(MotionEvent e) {

             return false;
         }
     });
    
	    //监听回调
	OnGestureListener onGestureListener= new OnGestureListener() {
        @Override
        public boolean onDown(MotionEvent e) {
            return false;
        }

        @Override
        public void onShowPress(MotionEvent e) {
        }

        @Override
        public boolean onSingleTapUp(MotionEvent e) {
            return false;
        }

        @Override
        public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
            return false;
        }

        @Override
        public void onLongPress(MotionEvent e) {
        }

        @Override
        public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
            return false;
        }
    };

 上面简单的演示了一些基本的用法。（额外的提下，在前面我们的解析View的事件传递过程的时候，我们就有提到，注册事件监听的优先级是高于View本身的onTouch函数的） 

---

我们先来看下构造函数

	public GestureDetector(Context context, OnGestureListener listener) {
        this(context, listener, null);
    }
 
    public GestureDetector(Context context,OnGestureListener listener,Handler handler){
        if (handler != null) {
            mHandler = new GestureHandler(handler);
        } else {
            mHandler = new GestureHandler();
        }
        mListener = listener;
        if (listener instanceof OnDoubleTapListener) {
            setOnDoubleTapListener((OnDoubleTapListener) listener);
        }
        if (listener instanceof OnContextClickListener) {
            setContextClickListener((OnContextClickListener) listener);
        }
        init(context);
    }
    
哎呀，看到这构造函数他会对`listener`做类型的判断，前面我加了`onGestureListener`，虽然没用到，但因为不加他会在`init（）`里面扔异常出来。

	private void init(Context context) {
        if (mListener == null) {
            throw new NullPointerException("OnGestureListener must not be null");
        }      
        mIsLongpressEnabled = true;  
        ...
	}
看到这，我智商就不够了。`OnDoubleTapListener`，`OnGestureListener`和`OnContextClickListener`都是接口来的，如果写成下面这样，是会遇到`java.lang.ClassCastException`的啊...
另外那个`OnContextClickListener`是后面在加上去的，我看v18版的时候都没有这个接口。也不加多一个`@since` ，小气。


	mGestureDetector = new GestureDetector(this,
	 (GestureDetector.OnGestureListener) new GestureDetector.OnDoubleTapListener() {});


那个`GestureHandler`是用来接收消息，回调我们的监听函数的。

	 private class GestureHandler extends Handler {
        GestureHandler() {
            super();
        }
        GestureHandler(Handler handler) {
            super(handler.getLooper());
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case SHOW_PRESS:
                mListener.onShowPress(mCurrentDownEvent);
                break;                
            case LONG_PRESS:
                dispatchLongPress();
                break;                
            case TAP:
                 if (mDoubleTapListener != null) {
                    if (!mStillDown) {                        mDoubleTapListener.onSingleTapConfirmed(mCurrentDownEvent);
                    } else {
                        mDeferConfirmSingleTap = true;
                    }
                }
                break; 
             ...
            }
        }
    }

---

接着我们去`GestureDetector` 的`onTouchEvent(event)`看下里面写了什么吧。
 
 整个也就一百多行，现在都感觉挺短的，先把整体的贴在这里，有个整体的印象，再分别讨论具体case的内容 

	public boolean onTouchEvent(MotionEvent ev) {
	
	    ...
        final int action = ev.getAction();

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        final boolean pointerUp =
                (action & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_POINTER_UP;
        final int skipIndex = pointerUp ? ev.getActionIndex() : -1;

        // Determine focal point
        float sumX = 0, sumY = 0;
        final int count = ev.getPointerCount();
        for (int i = 0; i < count; i++) {
            if (skipIndex == i) continue;
            sumX += ev.getX(i);
            sumY += ev.getY(i);
        }
        final int div = pointerUp ? count - 1 : count;
        final float focusX = sumX / div;
        final float focusY = sumY / div;

        boolean handled = false;

        switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_POINTER_DOWN:
			 ...
        case MotionEvent.ACTION_POINTER_UP:
             ...
        case MotionEvent.ACTION_DOWN:
	        ...
         case MotionEvent.ACTION_UP:
             ... 

        case MotionEvent.ACTION_CANCEL:
			 ...  
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 0);
        }
        return handled;
    }

## 各个Action

现在我们分各个`Action`来看下具体的内容

### ACTION_POINTER_DOWN

	case MotionEvent.ACTION_POINTER_DOWN:
            mDownFocusX = mLastFocusX = focusX;
            mDownFocusY = mLastFocusY = focusY;
            // Cancel long press and taps
            cancelTaps();
            break;
            
	private void cancelTaps() {
        mHandler.removeMessages(SHOW_PRESS);
        mHandler.removeMessages(LONG_PRESS);
        mHandler.removeMessages(TAP);
        mIsDoubleTapping = false;
        mAlwaysInTapRegion = false;
        mAlwaysInBiggerTapRegion = false;
        mDeferConfirmSingleTap = false;
        mInLongPress = false;
        mInContextClick = false;
        mIgnoreNextUpEvent = false;
    }
对于ACTION_POINTER_DOWN的情况，他记录当前的点，然后取消各种事件

### ACTION_DOWN

	case MotionEvent.ACTION_DOWN:
	
     if (mDoubleTapListener != null) {      
        //不像前面那样单独开个函数取消啊
        boolean hadTapMessage = mHandler.hasMessages(TAP);
        if (hadTapMessage) mHandler.removeMessages(TAP);
                        
        if ((mCurrentDownEvent != null) && (mPreviousUpEvent != null) &&hadTapMessage
	         &&isConsideredDoubleTap(mCurrentDownEvent,mPreviousUpEvent, ev)) {
	         
            // This is a second tap
            mIsDoubleTapping = true;
            // Give a callback with the first tap of the double-tap
            handled |= mDoubleTapListener.onDoubleTap(mCurrentDownEvent);
            // Give a callback with down event of the double-tap
            handled |= mDoubleTapListener.onDoubleTapEvent(ev);
        } else {
            // This is a first tap
            mHandler.sendEmptyMessageDelayed(TAP, DOUBLE_TAP_TIMEOUT);
        }
      }

      mDownFocusX = mLastFocusX = focusX;
      mDownFocusY = mLastFocusY = focusY;
      if (mCurrentDownEvent != null) {
          mCurrentDownEvent.recycle();
      }
      mCurrentDownEvent = MotionEvent.obtain(ev);
      mAlwaysInTapRegion = true;
      mAlwaysInBiggerTapRegion = true;
      mStillDown = true;
      mInLongPress = false;
      mDeferConfirmSingleTap = false;

      if (mIsLongpressEnabled) {
          mHandler.removeMessages(LONG_PRESS);
          mHandler.sendEmptyMessageAtTime(LONG_PRESS, mCurrentDownEvent.getDownTime()
                  + TAP_TIMEOUT + LONGPRESS_TIMEOUT);
      }
      
      mHandler.sendEmptyMessageAtTime(SHOW_PRESS, mCurrentDownEvent.getDownTime() + TAP_TIMEOUT);
      handled |= mListener.onDown(ev);
      
      break;

我们看到开头在处理双击事件的方法是通过给我们的Handler发一个空延迟消息，然后那个时间是300ms的间隔。


	private static final int DOUBLE_TAP_TIMEOUT = ViewConfiguration.
													getDoubleTapTimeout();

    private static final int DOUBLE_TAP_TIMEOUT = 300;


### ACTION_MOVE

	case MotionEvent.ACTION_MOVE:
	
	     if (mInLongPress || mInContextClick) {
	         break;
	     }
	     final float scrollX = mLastFocusX - focusX;
	     final float scrollY = mLastFocusY - focusY;
	     if (mIsDoubleTapping) {
	         // Give the move events of the double-tap
	         handled |= mDoubleTapListener.onDoubleTapEvent(ev);
	     } else if (mAlwaysInTapRegion) {
	         final int deltaX = (int) (focusX - mDownFocusX);
	         final int deltaY = (int) (focusY - mDownFocusY);
	         int distance = (deltaX * deltaX) + (deltaY * deltaY);
	         if (distance > mTouchSlopSquare) {
	             handled = mListener.onScroll(mCurrentDownEvent, ev, scrollX, scrollY);
	             mLastFocusX = focusX;
	             mLastFocusY = focusY;
	             mAlwaysInTapRegion = false;
	             mHandler.removeMessages(TAP);
	             mHandler.removeMessages(SHOW_PRESS);
	             mHandler.removeMessages(LONG_PRESS);
	         }
	         if (distance > mDoubleTapTouchSlopSquare) {
	             mAlwaysInBiggerTapRegion = false;
	         }
	     } else if ((Math.abs(scrollX) >= 1) || (Math.abs(scrollY) >= 1)) {
	         handled = mListener.onScroll(mCurrentDownEvent, ev, scrollX, scrollY);
	         mLastFocusX = focusX;
	         mLastFocusY = focusY;
	     }
	     break;
对于我们的move的情况，我们看到的是但移动的距离distance如果超过mTouchSlopSquare，就认为是一次挪动，移除原先的TAP,PRESS消息。也不是单独弄个函数来搞。这人写的时候很有权衡啊。

我们看下他对这个值得设置，回到我们的`init`函数

	 private void init(Context context) {
	 
        ... 
        int touchSlop, doubleTapSlop, doubleTapTouchSlop;
        if (context == null) {
            //noinspection deprecation
            touchSlop = ViewConfiguration.getTouchSlop();
            doubleTapTouchSlop = touchSlop; // Hack rather than adding a hiden method for this
            doubleTapSlop = ViewConfiguration.getDoubleTapSlop();
            //noinspection deprecation
            mMinimumFlingVelocity = ViewConfiguration.getMinimumFlingVelocity();
            mMaximumFlingVelocity = ViewConfiguration.getMaximumFlingVelocity();
        } else {
            final ViewConfiguration configuration = ViewConfiguration.get(context);
            touchSlop = configuration.getScaledTouchSlop();
            doubleTapTouchSlop = configuration.getScaledDoubleTapTouchSlop();
            doubleTapSlop = configuration.getScaledDoubleTapSlop();
            mMinimumFlingVelocity = configuration.getScaledMinimumFlingVelocity();
            mMaximumFlingVelocity = configuration.getScaledMaximumFlingVelocity();
        }
        mTouchSlopSquare = touchSlop * touchSlop;
        mDoubleTapTouchSlopSquare = doubleTapTouchSlop * doubleTapTouchSlop;
        mDoubleTapSlopSquare = doubleTapSlop * doubleTapSlop;
    }
 前面没有对`init`进行说明，就是觉得好时机没到，现在可以了。 ^_^
 这个我们来看下，对于没有加context的，则用默认的大小配置。
我们是有传入`context`的，他可以根据屏幕大小做适配，得出一个更佳的值

	public static ViewConfiguration get(Context context) {
        final DisplayMetrics metrics = context.getResources().getDisplayMetrics();
        final int density = (int) (100.0f * metrics.density);

        ViewConfiguration configuration = sConfigurations.get(density);
        if (configuration == null) {
            configuration = new ViewConfiguration(context);
            sConfigurations.put(density, configuration);
        }

        return configuration;
    } 
做适配可以参考这模式，来一个配置类。


### ACTION_UP

	case MotionEvent.ACTION_UP:
	
      mStillDown = false;
      MotionEvent currentUpEvent = MotionEvent.obtain(ev);
      if (mIsDoubleTapping) {
          // Finally, give the up event of the double-tap
          handled |= mDoubleTapListener.onDoubleTapEvent(ev);
      } else if (mInLongPress) {
          mHandler.removeMessages(TAP);
          mInLongPress = false;
      } else if (mAlwaysInTapRegion && !mIgnoreNextUpEvent) {
          handled = mListener.onSingleTapUp(ev);
          if (mDeferConfirmSingleTap && mDoubleTapListener != null) {
              mDoubleTapListener.onSingleTapConfirmed(ev);
          }
      } else if (!mIgnoreNextUpEvent) {

          // A fling must travel the minimum tap distance
          final VelocityTracker velocityTracker = mVelocityTracker;
          final int pointerId = ev.getPointerId(0);
          velocityTracker.computeCurrentVelocity(1000, mMaximumFlingVelocity);
          final float velocityY = velocityTracker.getYVelocity(pointerId);
          final float velocityX = velocityTracker.getXVelocity(pointerId);

          if ((Math.abs(velocityY) > mMinimumFlingVelocity)
                  || (Math.abs(velocityX) > mMinimumFlingVelocity)){
              handled = mListener.onFling(mCurrentDownEvent, ev, velocityX, velocityY);
          }
      }
      if (mPreviousUpEvent != null) {
          mPreviousUpEvent.recycle();
      }
      // Hold the event we obtained above - listeners may have changed the original.
      mPreviousUpEvent = currentUpEvent;
      if (mVelocityTracker != null) {
          // This may have been cleared when we called out to the
          // application above.
          mVelocityTracker.recycle();
          mVelocityTracker = null;
      }
      mIsDoubleTapping = false;
      mDeferConfirmSingleTap = false;
      mIgnoreNextUpEvent = false;
      mHandler.removeMessages(SHOW_PRESS);
      mHandler.removeMessages(LONG_PRESS);
      break;


### ACTION_CANCEL

	case MotionEvent.ACTION_CANCEL:
            cancel();
            break;
            
	private void cancel() {
        mHandler.removeMessages(SHOW_PRESS);
        mHandler.removeMessages(LONG_PRESS);
        mHandler.removeMessages(TAP);
        mVelocityTracker.recycle();
        mVelocityTracker = null;
        mIsDoubleTapping = false;
        mStillDown = false;
        mAlwaysInTapRegion = false;
        mAlwaysInBiggerTapRegion = false;
        mDeferConfirmSingleTap = false;
        mInLongPress = false;
        mInContextClick = false;
        mIgnoreNextUpEvent = false;
    }
            
# 后记

我们看到这个判断很大程度是利用Handler来处理的，并不是用一些变量来寄存，然后判断。

 写到这类，突然有种像那些人说的，需要“取材"，每次要写文章，就憋着想写点什么好...
 现在是靠看以前写的项目里面看到什么没写过的就抓主来写下，不知道这个可以支撑多久...
 
估计也只能再硬憋几篇文章，就写不下去，得开结尾篇了，然后写点第三方库的内容，例如Volley，EventBus，ButterKnifer，Universial-Image-Loader，FastJson等。这些库也就差FastJson没去看过了，在csdn都给他们写过分析稿。就是能坑爹的CSDN吃了我几篇文章还没吐出来，不见了。哎，搭个高可靠的服务器还真的不容易。
