title: 源码探索系列37---Android的属性动画
date: 2016-05-23 22:45:46
tags: [android,animator,property]
categories: android

------------------------------------------


The Android framework provides two animation systems: `property animation` and `view animation.` Both animation systems are viable options, but the property animation system, in general, is the preferred method to use, because it is `more flexible` and offers `more features.` In addition to these two systems, you can utilize `Drawable animation,` which allows you to load drawable resources and display them one frame after another.

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/37_anim_valueanimator.png)
<!--more-->

# 简介

安卓有三种动画
 
2. Tween Animation，即补间动画，它提供了
四种常见效果：位移（translate），缩放（scale），旋转（rotate）和 淡入淡出（alpha）效果；类图如下
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/37_anim_QQ%E6%88%AA%E5%9B%BE20160523130740.png)
这个框架应该是我们多年前学安卓时候最早接触的几个基本动画。但这个动画有点坑，我们播放的动画，最后的移动位置是“假的”，还需要手动代码设置下。
因为它的实现原理是每次绘制视图时，View所在的ViewGroup中的drawChild函数获取该View的Animation的Transformation值，然后调用canvas.concat(transformToApply.getMatrix())，通过矩阵运算完成动画帧，如果动画没有完成，继续调用invalidate()函数，启动下次绘制来驱动动画，动画过程中的帧之间间隙时间是绘制函数所消耗的时间，这可能会导致动画消耗比较多的CPU资源，最重要的是，动画改变的只是显示，并不能相应事件。这真的是“障眼法”。

3.  Proper Animation，属性动画。从安卓3.0开始引入，这个就相对强大了，叫属性动画是可以真的操作对象的属性。它能够自动驱动，用setFrameDelay(longframeDelay)设置动画帧之间的间隙时间，调整帧率，减少动画过程中频繁绘制界面，而在不影响动画效果的前提下减少CPU资源消耗。因此，Anroid推出的强大的属性动画框架，基本可以实现所有的动画效果。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/37_anim_QQ%E6%88%AA%E5%9B%BE20160523131345.png)

1. Frame Animation，即帧动画，它会按顺序展示一组图片（如gif、电影之类的效果）。这个动画，个人用得很少。
 
在知道这三大动画家后，现在打算主要先说下属性动画。因为他大部分时候是我们的首选。


# Demo
API:23


在开始深入前，我们先来看下简单的demo，然后再以这个为线索，进行深入。

## ValueAnimator 
 
	ValueAnimator valueAnim = ValueAnimator.ofFloat(1, 0);
    valueAnim.setDuration(1000);
    valueAnim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            float targetValue = (Float) animation.getAnimatedValue();                 
            aniView.setAlpha(targetValue);
        }
    });
    valueAnim.start();

上面的代码片段，我们设置了用一秒的时间，把我们的aniView的透明度从1变成0的过程。
ValueAnimator有上面类似的用法，设置我们想要的变化，然后在Animator计算好后，通过回调来告诉我们，我们通过获取目标值来更新我们的界面，从而完成动画。
 

## ObjectAnimator

	ObjectAnimator.ofFloat(aniView, "scaleX",1.0f, 2f).setDuration(2000).start();
                
上面的代码片段，我们设置用2秒的时间，把我们的`aniView`进行横向（view.scaleX）拉长。
动画的最终效果实现了“难过到变形”的样子。


**动画生成的原理**就是通过**差值器**计算出来的一定规律变化的**数值**作用到对象上来实现对象效果的变化，因此我们也可以使用ObjectAnimator来生成这些数，然后像ValueAnimator一样加多个监听回调，在监听回调处完成我们想要的效果。



# 前进

在看完demo后，我们从这里切入，开始看下背后到底做了些什么，让ObjectAnimator可以操纵我们的view更新属性，是通过反射机制去做嘛？
让我们从ofFlaot开始看起吧


## ofFloat()

	public static ObjectAnimator ofFloat(Object target, String propertyName, 
										float... values) {
	
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setFloatValues(values);
        return anim;
    }
ObjectAnimator处理ofFloat外，还有ofInt和ofObjects的。
我们看到第三个参数是一个可变长的参数。当只有一个参数的时候，表示这个值为目标值。如果有两个表示为从第一个起始值到第二个结束值。如果有三个和以上，出去两端的中间数值，表示会经过的值。
例如

		ValueAnimator valueAnim = ValueAnimator.ofFloat(1f, 0.5f,0.3f,0.9f,0);
像这样写的时候，我们的回调从1逐渐变到0.3，然后逐渐变成0.9,再逐渐减小到0；

### ObjectAnimator()

	private ObjectAnimator(Object target, String propertyName){
        mTarget = target;
        setPropertyName(propertyName);
    }
 
    public void setPropertyName(String propertyName) {
        // mValues could be null if this is being constructed piecemeal. Just record the
        // propertyName to be used later when setValues() is called if so.
        if (mValues != null) {
	        //保存我们的属性在valuesHolder，挺重要的哈。
	        //后面修改就靠它了！
            PropertyValuesHolder valuesHolder = mValues[0];
            String oldName = valuesHolder.getPropertyName();
            valuesHolder.setPropertyName(propertyName);
            
            mValuesMap.remove(oldName);
            mValuesMap.put(propertyName, valuesHolder);
        }
        mPropertyName = propertyName;
        // New property/values/target should cause re-initialization prior to starting
        mInitialized = false;
    }

构造一个ObjectAnimator，同时保存我们的属性名，留着后面需要的时候使用。

我们的动画一次只能改一个属性，如果想要同时修改多个，那要用`AnimatorSet`，像下面这样

	ObjectAnimator anim1 = ObjectAnimator.ofFloat(aniView, "scaleX",1.0f, 2f);
    ObjectAnimator anim2 = ObjectAnimator.ofFloat(aniView, "scaleY",1.0f, 2f);
    
    AnimatorSet animSet = new AnimatorSet();
    animSet.setDuration(2000);
    animSet.setInterpolator(new AccelerateInterpolator());
    animSet.playTogether(anim1, anim2);
    animSet.start();

这样就可以同时惊喜X和Y两个方向拉伸的动画。

## setDruing()

	public ValueAnimator setDuration(long duration) {
	
        if (duration < 0) {
            throw new IllegalArgumentException("Animators cannot have negative duration: 
            " +duration);
        }
        mUnscaledDuration = duration;
        mDuration = (long)(duration * sDurationScale);
        return this;
    }
    
这里主要是保存下动画的持续时间，对于负数时间会抛错。另外这个sDurationScale默认为1，目前被藏起来不给用了。系统默认的是300.


## start()
然后到了我们比较关心的开始动画了。

	@Override
    public void start() {
    
        // See if any of the current active/pending animators need to be canceled
        AnimationHandler handler = sAnimationHandler.get();
        if (handler != null) {
            int numAnims = handler.mAnimations.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mAnimations.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mAnimations.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                        anim.cancel();
                    }
                }
            }
            numAnims = handler.mPendingAnimations.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mPendingAnimations.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mPendingAnimations.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                        anim.cancel();
                    }
                }
            }
            numAnims = handler.mDelayedAnims.size();
            for (int i = numAnims - 1; i >= 0; i--) {
                if (handler.mDelayedAnims.get(i) instanceof ObjectAnimator) {
                    ObjectAnimator anim = (ObjectAnimator) handler.mDelayedAnims.get(i);
                    if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                        anim.cancel();
                    }
                }
            }
        }
	    ...
        super.start();
    }

这个先取消一堆动画，包括在进行中的，delay的和pending的，然后再调用父类的start()。

他这里有两种结束动画，一种是用`cancel()`，另外一个是用`end()`。前者让动画立即停止，同时画面是停在当前的位置；而后者end()则直接跳到最终状态。

关于开头调用的那句AnimationHandler handler = sAnimationHandler.get();。这个sAnimationHandler是一个静态的ThreadLocal。

	protected static ThreadLocal<AnimationHandler> sAnimationHandler =
            new ThreadLocal<AnimationHandler>();
 
 这个类的使用在Handler的源码部分我们有遇到过，他功效就是获得我当前线程自己的副本的奇效。
 关于这个AnimationHandler，他不是一个真的handler哈，人家是实现了Runnable。 关于他会在后面会讲到，现在先不细说，然后我们继续看下父类的start()

## super.start()

	@Override
    public void start() {
        start(false);
    } 

	private void start(boolean playBackwards) {
		 ...
        mPlayingBackwards = playBackwards;
        mCurrentIteration = 0;
        mPlayingState = STOPPED;
        mStarted = true;
        mStartedDelay = false;
        mPaused = false;
        
    	//这个函数人如其名，看是否有handler了，没有就创建一个。       
        AnimationHandler animationHandler = getOrCreateAnimationHandler();        
        animationHandler.mPendingAnimations.add(this);
        
        if (mStartDelay == 0) {
            // This sets the initial value of the animation, 
            // prior to actually starting it running
            我们传0过去，表示开始时间为当前，没有延迟
            setCurrentPlayTime(0);
            mPlayingState = STOPPED;
            mRunning = true;
            //通知监听，调他们的onAnimationStart(this)
            notifyStartListeners();
        }
        animationHandler.start();
    }
因为我们的mStartDelay默认为0，也没人设置了延迟多久，所以会走到if里面去。

###  setCurrentPlayTime()

	 public void setCurrentPlayTime(long playTime) {
        float fraction = mUnscaledDuration > 0 ? (float) playTime / mUnscaledDuration : 1;
        setCurrentFraction(fraction);
    }
### setCurrentFraction()

	public void setCurrentFraction(float fraction) {
        initAnimation();
        if (fraction < 0) {
            fraction = 0;
        }
        int iteration = (int) fraction;
        if (fraction == 1) {
            iteration -= 1;
        } else if (fraction > 1) {
            if (iteration < (mRepeatCount + 1) || mRepeatCount == INFINITE) {
                if (mRepeatMode == REVERSE) {
                    mPlayingBackwards = (iteration % 2) != 0;
                }
                fraction = fraction % 1f;
            } else {
                fraction = 1;
                iteration -= 1;
            }
        } else {
            mPlayingBackwards = mReversing;
        }
        mCurrentIteration = iteration;
        long seekTime = (long) (mDuration * fraction);
        long currentTime = AnimationUtils.currentAnimationTimeMillis();
        mStartTime = currentTime - seekTime;
        mStartTimeCommitted = true; // do not allow start time to be compensated for jank
        if (mPlayingState != RUNNING) {
            mSeekFraction = fraction;
            mPlayingState = SEEKED;
        }
        if (mPlayingBackwards) {
            fraction = 1f - fraction;
        }
        animateValue(fraction);
    }
    
#### initAnimation()---**
有点重要的一个函数，至于原因会在后面提到，先卖个关子

	@CallSuper
    void initAnimation() {
        if (!mInitialized) {
            int numValues = mValues.length;
            for (int i = 0; i < numValues; ++i) {
                mValues[i].init();
            }
            mInitialized = true;
        }
    }
    
## animationHandler.start()
 
	 public void start() {
            scheduleAnimation();
        }

纯粹一个壳，调用scheduleAnimation()

## scheduleAnimation()

	private void scheduleAnimation() {
            if (!mAnimationScheduled) {
                mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, this, null);
                mAnimationScheduled = true;
            }
        }

## mChoreographer.postCallback()
	 
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
    
    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }
	
	 private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
	      ...

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }


	private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }

我们看最后层层转包，最后通过真的Handler，让我们AnimationHandler跑在了UI线程，从而让我们动画可以显示出来。然后回调我们的AnimationHandler ，看下他的run方法是什么

## AnimationHandler.run()

		@Override
        public void run() {
            mAnimationScheduled = false;
            doAnimationFrame(mChoreographer.getFrameTime());
        }


## scheduleAnimation()

	private void doAnimationFrame(long frameTime) {
 
			//我们前面才加了一个，所以必然不会为0
            while (mPendingAnimations.size() > 0) {
                ArrayList<ValueAnimator> pendingCopy =
                        (ArrayList<ValueAnimator>) mPendingAnimations.clone();
                mPendingAnimations.clear();
                int count = pendingCopy.size();
                for (int i = 0; i < count; ++i) {
                    ValueAnimator anim = pendingCopy.get(i);
                    // If the animation has a startDelay, place it on the delayed list
                    if (anim.mStartDelay == 0) {
	                    //前面我们知道这个就是0,所以开启动画。
	                    //实际就是把这个动画加到animations这个执行队列去
                        anim.startAnimation(this);
                    } else {
                        mDelayedAnims.add(anim);
                    }
                }
            }

            // Next, process animations currently sitting on the delayed queue, adding
            // them to the active animations if they are ready
            int numDelayedAnims = mDelayedAnims.size();
            for (int i = 0; i < numDelayedAnims; ++i) {
                ValueAnimator anim = mDelayedAnims.get(i);
                if (anim.delayedAnimationFrame(frameTime)) {
                    mReadyAnims.add(anim);
                }
            }
            int numReadyAnims = mReadyAnims.size();
            if (numReadyAnims > 0) {
                for (int i = 0; i < numReadyAnims; ++i) {
                    ValueAnimator anim = mReadyAnims.get(i);
                    anim.startAnimation(this);
                    anim.mRunning = true;
                    mDelayedAnims.remove(anim);
                }
                mReadyAnims.clear();
            }

			//接下来才是我们比较关心的几行
            // Now process all active animations. The return value from animationFrame()
            // tells the handler whether it should now be ended
            int numAnims = mAnimations.size();
            for (int i = 0; i < numAnims; ++i) {
                mTmpAnimations.add(mAnimations.get(i));
            }
            for (int i = 0; i < numAnims; ++i) {
                ValueAnimator anim = mTmpAnimations.get(i);
                if (mAnimations.contains(anim) && anim.doAnimationFrame(frameTime)) {
                    mEndingAnims.add(anim);
                }
            }
            mTmpAnimations.clear();
            if (mEndingAnims.size() > 0) {
                for (int i = 0; i < mEndingAnims.size(); ++i) {
                    mEndingAnims.get(i).endAnimation(this);
                }
                mEndingAnims.clear();
            }

            // If there are still active or delayed animations, schedule a future call to
            // onAnimate to process the next frame of the animations.
            if (!mAnimations.isEmpty() || !mDelayedAnims.isEmpty()) {
                scheduleAnimation();
            }
        }		


### startAnimation()

	private void startAnimation(AnimationHandler handler) {
        if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
            Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, getNameForTrace(),
                    System.identityHashCode(this));
        }
        initAnimation();
        //饶了一个圈，把动画从pending队伍弄到执行队列去。
        handler.mAnimations.add(this);
        if (mStartDelay > 0 && mListeners != null) {
            // Listeners were already notified in start() if startDelay is 0; this is
            // just for delayed animations
            notifyStartListeners();
        }
    }


##  ValueAnimator.doAnimationFrame()

	final boolean doAnimationFrame(long frameTime) {
        if (mPlayingState == STOPPED) {
            mPlayingState = RUNNING;
            if (mSeekTime < 0) {
                mStartTime = frameTime;
            } else {
                mStartTime = frameTime - mSeekTime;
                // Now that we're playing, reset the seek time
                mSeekTime = -1;
            }
        }
        if (mPaused) {
            if (mPauseTime < 0) {
                mPauseTime = frameTime;
            }
            return false;
        } else if (mResumed) {
            mResumed = false;
            if (mPauseTime > 0) {
                // Offset by the duration that the animation was paused
                mStartTime += (frameTime - mPauseTime);
            }
        }
        // The frame time might be before the start time during the first frame of
        // an animation.  The "current time" must always be on or after the start
        // time to avoid animating frames at negative time intervals.  In practice, this
        // is very rare and only happens when seeking backwards.
        final long currentTime = Math.max(frameTime, mStartTime);
        return animationFrame(currentTime);
    }
    

##  animationFrame()

	boolean animationFrame(long currentTime) {
        boolean done = false;
        switch (mPlayingState) {
        case RUNNING:
        case SEEKED:
        
            //用currentTime和mStartTime的差值计算动画执行的进度
            float fraction=mDuration>0?(currentTime-mStartTime)/mDuration: 1f;
            
            if (fraction >= 1f) {
		        //判断是否结束
                if (mCurrentIteration < mRepeatCount || mRepeatCount == INFINITE) {
                    // Time to repeat
                    if (mListeners != null) {
                        int numListeners = mListeners.size();
                        for (int i = 0; i < numListeners; ++i) {
                            mListeners.get(i).onAnimationRepeat(this);
                        }
                    }
                    if (mRepeatMode == REVERSE) {
                        mPlayingBackwards = !mPlayingBackwards;
                    }
                    mCurrentIteration += (int)fraction;
                    fraction = fraction % 1f;
                    mStartTime += mDuration;
                } else {
                    done = true;
                    fraction = Math.min(fraction, 1.0f);
                }
            }
            if (mPlayingBackwards) {
                fraction = 1f - fraction;
            }
             //这个实际负责计算animationValue和onAnimationUpdate回调 
            animateValue(fraction);
            break;
        }

        return done;
    }

## animateValue()---回调更新界面

	void animateValue(float fraction) {
		//通过插值器重新计算一个fraction，
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
           //真正计算animationValue的方法  
            mValues[i].calculateValue(fraction);
        }
       
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
	            //我们期待的回调更新
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }

跑了一大圈，终于看到了回调部分，这样我们就可以来更新我们的程序了。

但我们目前还没看到ObjectAnimator是怎么更新我们的targetView的。
这部分是在哪里呢？我们深挖下


## PropertyValuesHolder.calculateValue() 

	void calculateValue(float fraction) {
	        Object value = mKeyframes.getValue(fraction);
	        mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
	    }
这计算别后是找KeyFrameSet去干活的。

## keyFrameSet.getValue

	public Object getValue(float fraction) {

        // Special-case optimization for the common case of only two keyframes
        //这个mNumKeyframes就是在创建animator时传的VAL可变长参数对应的具体长度
        if (mNumKeyframes == 2) {
            if (mInterpolator != null) {
                fraction = mInterpolator.getInterpolation(fraction);
            }
            return mEvaluator.evaluate(fraction, mFirstKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        
        if (fraction <= 0f) {
            final Keyframe nextKeyframe = mKeyframes.get(1);
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = mFirstKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /            
					                (nextKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, mFirstKeyframe.getValue(),
                    nextKeyframe.getValue());
        } else if (fraction >= 1f) {
            final Keyframe prevKeyframe = mKeyframes.get(mNumKeyframes - 2);
            final TimeInterpolator interpolator = mLastKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = prevKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (mLastKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        Keyframe prevKeyframe = mFirstKeyframe;
        for (int i = 1; i < mNumKeyframes; ++i) {
            Keyframe nextKeyframe = mKeyframes.get(i);
            if (fraction < nextKeyframe.getFraction()) {
                final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
                if (interpolator != null) {
                    fraction = interpolator.getInterpolation(fraction);
                }
                final float prevFraction = prevKeyframe.getFraction();
                float intervalFraction = (fraction - prevFraction) /
                    (nextKeyframe.getFraction() - prevFraction);
                return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                        nextKeyframe.getValue());
            }
            prevKeyframe = nextKeyframe;
        }
        // shouldn't reach here
        return mLastKeyframe.getValue();
    }

关于这个Evaluator，它是一个接口，而且只有一个方法evaluate()。
背后有不少的实现类如IntEvaluator，FloatEvaluator，RectEvaluator等。
我们挑一个floatEvaluator看
 
	public Float evaluate(float fraction, Number startValue, Number endValue) {
	        float startFloat = startValue.floatValue();
	        return startFloat + fraction * (endValue.floatValue() - startFloat);
	    }    
 
 只是简单的做一个计算，但问题来了，看到这里我们也没看到关于我们的ObjectAnimator在哪里做了什么更新view的操作啊。难道是我们漏了什么？
 
   是的，漏了，前面我们都是跑在他的父类ValueAnimator看的。没看它本身是否有重写的地方，我们需要回去ObjectAnimator是怎样的。

在一开始的父类ValueAnimator的setCurrentFraction()函数里面。有一小段代码

	public void setCurrentFraction(float fraction) {
	        initAnimation();
 
			 ...
	 }
	 
在开头的地方他调用了initAnimation函数，而重点内容就在它身上了
	 
# ObjectAnimator.initAnimation()

	@Override
    void initAnimation() {
        if (!mInitialized) {
            // mValueType may change due to setter/getter setup; 
            //do this before calling super.init(),
            // which uses mValueType to set up the default type evaluator.
            final Object target = getTarget();
            if (target != null) {
                final int numValues = mValues.length;
                for (int i = 0; i < numValues; ++i) {
                    mValues[i].setupSetterAndGetter(target);
                }
            }
            super.initAnimation();
        }
    }
我们看到它调用了一个看上去很可疑的函数，就是PropertyValueHolder的
setupSetterAndGetter()函数。我们具体去看下


## PropertyValueHolder.setupSetterAndGetter()


	void setupSetterAndGetter(Object target) {
        mKeyframes.invalidateCache();
        //开头检测是否有我们传进来的这个属性
        //没有的话就打印一个警告日志，不应该抛个错？
        if (mProperty != null) {
            // check to make sure that mProperty is on the class of target
            try {
                Object testValue = null;
                List<Keyframe> keyframes = mKeyframes.getKeyframes();
                int keyframeCount = keyframes == null ? 0 : keyframes.size();
                for (int i = 0; i < keyframeCount; i++) {
                    Keyframe kf = keyframes.get(i);
                    if (!kf.hasValue() || kf.valueWasSetOnStart()) {
                        if (testValue == null) {
                            testValue = convertBack(mProperty.get(target));
                        }
                        kf.setValue(testValue);
                        kf.setValueWasSetOnStart(true);
                    }
                }
                return;
            } catch (ClassCastException e) {
                Log.w("PropertyValuesHolder","No such property (" + mProperty.getName() +
                        ") on target object " + target + ". Trying reflection instead");
                mProperty = null;
            }
        }
        
        // We can't just say 'else' here 
        //because the catch statement sets mProperty to null.
        //这句注释让我感觉曾经它们踩过坑，直接写了一个else在这里。
        if (mProperty == null) {
            Class targetClass = target.getClass();
            if (mSetter == null) {
                setupSetter(targetClass);
            }
            List<Keyframe> keyframes = mKeyframes.getKeyframes();
            int keyframeCount = keyframes == null ? 0 : keyframes.size();
            for (int i = 0; i < keyframeCount; i++) {
                Keyframe kf = keyframes.get(i);
                if (!kf.hasValue() || kf.valueWasSetOnStart()) {
                    if (mGetter == null) {
                        setupGetter(targetClass);
                        if (mGetter == null) {
                            // Already logged the error - just return to avoid NPE
                            return;
                        }
                     }
 
	                Object value = convertBack(mGetter.invoke(target));
					//通过get方法设置startValue  
	                kf.setValue(value);
	                kf.setValueWasSetOnStart(true);
				    ...
                }
            }
        }
    }

###  setupSetter()

	void setupSetter(Class targetClass) {
        Class<?> propertyType = mConverter == null ? mValueType : mConverter.getTargetType();
        mSetter = setupSetterOrGetter(targetClass, sSetterPropertyMap, "set", propertyType);
    }

####  setupSetterOrGetter()

	private Method setupSetterOrGetter(Class targetClass,
            HashMap<Class, HashMap<String, Method>> propertyMapMap,
            String prefix, Class valueType) {
        Method setterOrGetter = null;
        synchronized(propertyMapMap) {
            // Have to lock property map prior to reading it, to guard against
            // another thread putting something in there after we've checked it
            // but before we've added an entry to it
            HashMap<String, Method> propertyMap = propertyMapMap.get(targetClass);
            boolean wasInMap = false;

            if (propertyMap != null) {
                wasInMap = propertyMap.containsKey(mPropertyName);
                if (wasInMap) {
                    setterOrGetter = propertyMap.get(mPropertyName);
                }
            }
            if (!wasInMap) {
 
                setterOrGetter = getPropertyFunction(targetClass, prefix, valueType);
                if (propertyMap == null) {
                    propertyMap = new HashMap<String, Method>();
                    propertyMapMap.put(targetClass, propertyMap);
                }
                propertyMap.put(mPropertyName, setterOrGetter);
            }
        }
        return setterOrGetter;
    }
这函数主要做两部分，一个查缓存，没有命中就反射,然后保存起来。

####  getPropertyFunction()

	private Method getPropertyFunction(Class targetClass, String prefix, Class valueType){
        // TODO: faster implementation...
        Method returnVal = null;
        String methodName = getMethodName(prefix, mPropertyName);
        Class args[] = null;
        if (valueType == null) {
            try {
                returnVal = targetClass.getMethod(methodName, args);
            } catch (NoSuchMethodException e) {
                // Swallow the error, log it later
            }
        } else {
            args = new Class[1];
            Class typeVariants[];
            if (valueType.equals(Float.class)) {
                typeVariants = FLOAT_VARIANTS;
            } else if (valueType.equals(Integer.class)) {
                typeVariants = INTEGER_VARIANTS;
            } else if (valueType.equals(Double.class)) {
                typeVariants = DOUBLE_VARIANTS;
            } else {
                typeVariants = new Class[1];
                typeVariants[0] = valueType;
            }
            for (Class typeVariant : typeVariants) {
                args[0] = typeVariant;
                try {
                    returnVal = targetClass.getMethod(methodName, args);
                    if (mConverter == null) {
                        // change the value type to suit
                        mValueType = typeVariant;
                    }
                    return returnVal;
                } catch (NoSuchMethodException e) {
                    // Swallow the error and keep trying other variants
                }
            }
            // If we got here, then no appropriate function was found
        }

        if (returnVal == null) {
            Log.w("PropertyValuesHolder", "Method " +
                    getMethodName(prefix, mPropertyName) + "() with type " + valueType +
                    " not found on target class " + targetClass);
        }

        return returnVal;
    }


看到这个函数，我们总算是有个底了，它最后就是通过反射来处理的。
缓存了字段，后面估计要更新就是查这个缓存然后调用下来更新！


# ObjectAnimator.animateValue()

	@CallSuper
    @Override
	void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }

我们的ObjectAniamtor还重写了父类这个函数，倒是这开头的丢失对target的应用，然后暂停感觉到奇妙，难道是被GC了，还是怎么就没了？估计背后曾经也有一个坑的故事！

在调用父类的animateValue后，就去用setAnimatedValue()来更新界面的值！

## PropertyValueHolder.setAnimatedValue()

	void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }

这个mSetter我们前面看到过了，是通过setupSetterOrGetter来获取初始化的一个Methond！

# 小结

1. 对比ValueAnimator和ObjectAnimator
`ValueAnimator`提供基本的计算属性值的方法，改变属性的话，需要我们自己在回调里面处理，因此相对灵活，不需要我们的View提供指定的设置(get/set)接口。
而`ObjectAnimator`继承自`ValueAnimator`，需要指定一个对象及该对象的一个属性，当属性值计算完成时自动设置为该对象的相应属性，即完成了`Property Animation`的全部两步操作。这导致的一个条件就是，对象**必须**有要修改的`属性`对应的`setYourPropertyName()`和`getYourPropertyName()`才行。
有看到对于有些View不提供自己各种get/set的方法，或者有提供，不过是间接的。这时候会加多一层壳，加多个Wrapper。然后来调用。
2. 兼容
属性动画是V11才引入额，以前做动画为了兼容用过nineoldandroids这个库，非常的有名啊。后来项目对动画的要求没那么多了，就很少使用了。今天顺便看下背后代码的实现，当作温习一次吧。


# REF:

1. [鸿洋的介绍如何使用属性动画](http://blog.csdn.net/lmj623565791/article/details/38067475/)
2. [Property Animation谷歌官方介绍](https://developer.android.com/guide/topics/graphics/prop-animation.html)  
