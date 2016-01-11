
title:  源码探索系列22---顺滑滚动的Scroller
date: 2016-01-12 01:05:46
tags: [android,源码,Scroller]
categories: android

------------------------------------------

我们自定义View的时候难免会需要到滚动的状态，而Scroller可以帮助我们做到顺滑的滚动。
使用的基本套路是下面这样的，然后我们需要滚的的时候调用下smoothScrollTo函数就可以了。

	Scroller mScroller = new Scroller(context);

    private void smoothScrollTo(int destX, int destY) {
        int scrollX = getScrollX();
        int delta = destX - scrollX;
        mScroller.startScroll(scrollX, 0, delta, 0, 1000);
    }

    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
        } 
    }
我们知道这个Scroller本身是不能移动我们View的，最终还是要靠本身的scrollTo函数。通过我们希望的滑动距离和时间，然后计算出每次滚动的具体，一直就那样一点点的滚动，从而实现弹性滑动的效果。

现在我们就去看下他背后是做了什么。

<!--more-->


# 起航
API:23

我们先看下构造函数的

	public Scroller(Context context) {
        this(context, null);
    }
    
     public Scroller(Context context, Interpolator interpolator) {
        this(context, interpolator,
                context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB);
    }
     
    public Scroller(Context context, Interpolator interpolator, boolean flywheel) {
        mFinished = true;
        if (interpolator == null) {
            mInterpolator = new ViscousFluidInterpolator();
        } else {
            mInterpolator = interpolator;
        }
        mPpi = context.getResources().getDisplayMetrics().density * 160.0f;
        mDeceleration = computeDeceleration(ViewConfiguration.getScrollFriction());
        mFlywheel = flywheel;

        mPhysicalCoeff = computeDeceleration(0.84f); // look and feel tuning
    }
我们直接调用`new Scroller（context）`的时候，他会用默认插值器`ViscousFluidInterpolator()`，设置滚动是否结束的标记为真，同时计算下Deceleration的值。对于0.84是什么鬼，我只想说，对于魔术字，我也不懂。

---
好了，我们继续去看下那个`startScroll()`里面的内容

	public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
嗯，基本都是配置一堆的值。没什么特别的，我们得结合别的来说。
接下来我们看下他配套的函数

	public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE: //当前我们要关心的mode
            
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                final float t = (float) timePassed / mDuration;
                final int index = (int) (NB_SAMPLES * t);
                float distanceCoef = 1.f;
                float velocityCoef = 0.f;
                if (index < NB_SAMPLES) {
                    final float t_inf = (float) index / NB_SAMPLES;
                    final float t_sup = (float) (index + 1) / NB_SAMPLES;
                    final float d_inf = SPLINE_POSITION[index];
                    final float d_sup = SPLINE_POSITION[index + 1];
                    velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                    distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                }

                mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
                
                mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                // Pin to mMinX <= mCurrX <= mMaxX
                mCurrX = Math.min(mCurrX, mMaxX);
                mCurrX = Math.max(mCurrX, mMinX);
                
                mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                // Pin to mMinY <= mCurrY <= mMaxY
                mCurrY = Math.min(mCurrY, mMaxY);
                mCurrY = Math.max(mCurrY, mMinY);

                if (mCurrX == mFinalX && mCurrY == mFinalY) {
                    mFinished = true;
                }

                break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
很简单的，`computeScrollOffset()`判断标记是否结束和时间是否过了，没有就分模式处理。
我们的当前的模式是`Scroll_mode`，那个插值器根据我们的设置的值来计算出一个插值，再和我们的增量相乘的到这次要滚动的距离，最后和我们的其实距离相加，最是我们这次要滚到的点啦。

	final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
	                mCurrX = mStartX + Math.round(x * mDeltaX);
	                mCurrY = mStartY + Math.round(x * mDeltaY);
这样我们的`computeScroll()` 函数里面，就可以获得者连个值，来滑动界面的。知道

	            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());



---

接着还有的另外一个模式就是Fling

	public void fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY) {
        // Continue a scroll or fling in progress
        if (mFlywheel && !mFinished) {
            float oldVel = getCurrVelocity();

            float dx = (float) (mFinalX - mStartX);
            float dy = (float) (mFinalY - mStartY);
            float hyp = (float) Math.hypot(dx, dy);

            float ndx = dx / hyp;
            float ndy = dy / hyp;

            float oldVelocityX = ndx * oldVel;
            float oldVelocityY = ndy * oldVel;
            if (Math.signum(velocityX) == Math.signum(oldVelocityX) &&
                    Math.signum(velocityY) == Math.signum(oldVelocityY)) {
                velocityX += oldVelocityX;
                velocityY += oldVelocityY;
            }
        }

        mMode = FLING_MODE;
        mFinished = false;

        float velocity = (float) Math.hypot(velocityX, velocityY);
     
        mVelocity = velocity;
        mDuration = getSplineFlingDuration(velocity);
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;

        float coeffX = velocity == 0 ? 1.0f : velocityX / velocity;
        float coeffY = velocity == 0 ? 1.0f : velocityY / velocity;

        double totalDistance = getSplineFlingDistance(velocity);
        mDistance = (int) (totalDistance * Math.signum(velocity));
        
        mMinX = minX;
        mMaxX = maxX;
        mMinY = minY;
        mMaxY = maxY;

        mFinalX = startX + (int) Math.round(totalDistance * coeffX);
        // Pin to mMinX <= mFinalX <= mMaxX
        mFinalX = Math.min(mFinalX, mMaxX);
        mFinalX = Math.max(mFinalX, mMinX);
        
        mFinalY = startY + (int) Math.round(totalDistance * coeffY);
        // Pin to mMinY <= mFinalY <= mMaxY
        mFinalY = Math.min(mFinalY, mMaxY);
        mFinalY = Math.max(mFinalY, mMinY);
    }


当我们想中断动画的时候，可以调用`abortAnimation()`，来直接停止滑动，直接滚动到结点。

	public void abortAnimation() {
	        mCurrX = mFinalX;
	        mCurrY = mFinalY;
	        mFinished = true;
	    }
如果想在拉长下时间，可以调用extendDuration()函数，来延长下时间。

	public void extendDuration(int extend) {
        int passed = timePassed();
        mDuration = passed + extend;
        mDurationReciprocal = 1.0f / mDuration;
        mFinished = false;
    }
    
    public int timePassed() {
        return (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    }


# 后记

写得好没精神..不知不觉一点多了。。。呵呵

明天再做补充。有点困了，还没洗澡。明天居然要早到。