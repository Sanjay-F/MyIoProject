title: JetPack源码探索系列2---LiveData

date: 2018-08-24 20:21:00

tags: [android,JetPack,LiveData]

categories: android
 

------------------------------------------

  
  
 

# LiveData
LiveData是一个可以感知Activity、Fragment生命周期的数据容器。
当LiveData所持有的数据发生改变时，它会通知相应的观察者进行数据更新。
由于Activity、Fragment都已实现LifecycleOwner接口，

所以持有LifecycleOwner引用的LiveData在LifecycleOwner的生命周期处于started或resumed时可以作出相应更新，
而在LifecycleOwner处于被销毁时停止更新。

目前在项目使用了这个是改造了 **网络请求，数据回包**的问题。

发送网络请求，然后用观察者来处理对应数据，省去自己在对回包时候的生命周期判断的恶心判断逻辑。


从官方文档来看，LiveData的使用有以下几大好处

1. 保证UI状态和数据的统一：  
   LiveData采用了观察者设计模式。当生命周期状态改变时，LiveData会通知Observer对象。每次应用程序数据更改时，都会通知观察者对象，从而更新UI。
2. 减少内存泄漏：    
   LiveData能够感知到组件的生命周期，观察者绑定到生命周期对象，并在其相关生命周期被破坏后自行清理。
3. 当Activity停止时不会引起崩溃：   
   这是因为组件处于非激活状态时，不会收到LiveData中数据变化的通知
4. 不需要额外的手动处理来响应生命周期的变化：  
   这一点同样是因为LiveData能够感知组件的生命周期，所以就完全不需要在代码中告诉LiveData组件的生命周期状态。
5. 组件和数据相关的内容能实时更新：   
   组件在前台的时候能够实时收到数据改变的通知，这是可以理解的。当组件从后台到前台来时，LiveData能够将最新的数据通知组件，这两点就保证了组件中和数据相关的内容能够实时更新。
6. 资源共享：   
   通过继承LiveData类，然后将该类定义成单例模式，在该类封装监听一些系统属性变化，然后通知LiveData的观察者。


**我们经常有这样的场景：**

feed流-详情页这样的例子是最实在的，经常会跳去某个item的详情页，然后在里面做些处理，回到feed流页面，要保持数据的一致

 
 
<!--more-->

# 起航 

先说下使用情况：
以拉取评论数据为例子，我们在fragment里面写这样的逻辑，从而触发网络请求：


	ViewModelProviders.of(activity)
                .get(CommentsViewModel.class)
                .getCommentsLiveData()
                .observe(owner, yourObserver);
	//添加监听的yourObserver

	Observer<CommentsData> yourObserver = new Observer<CommentsData>() {
        @Override
        public void onChanged(@Nullable CommentsData commentsData) {
				//数据刷新，做一些刷新页面数据的操作
		}
	}


	ViewModelProviders.of(activity)
                .get(CommentsViewModel.class)
                .getCommentsLiveData()
                .getComments(pageIndex, feedId, feedType);
	//发起网络请求

然后需要说下liveData一般是包在XxxModel里面的，类似下面这样。这个至于为什么，后面再说 

	public class CommentsViewModel extends ViewModel {
	    // 创建一个DetailPagPageLiveData
	    private CommentsLiveData mCommentsLiveData = new CommentsLiveData();
	     
	
	    public CommentsLiveData getCommentsLiveData() {
	        return mCommentsLiveData;
	    }
	}

然后看我们的liveData怎么写

	public class CommentsLiveData extends LiveData<CommentsData> {
	    
	    // 获取评论数据
	    public void getComments(int pageIndex,
	                            String feedId){
	
	
	        CommentsData commentsData = getDataFromServer(pageIndex,feedId);
	        setValue(commentsData);
	    }
	}

这个我们继承LiveData，然后写对应的拉取评论数据的接口,拿到服务器的数据后就保存setValue(),之后会自动调用对应的监听对象。

好了，大致使用流程就是这样，现在我们开始来翻源码看到底做了什么事情吧。

# 查看源码前看文档

	/**
	 * LiveData is a data holder class that can be observed within a given lifecycle.
	 * This means that an {@link Observer} can be added in a pair with a {@link LifecycleOwner}, and
	 * this observer will be notified about modifications of the wrapped data only if the paired
	 * LifecycleOwner is in active state.   #看下官方这个注释，active state时候.
	 *  这个好理解，如果界面是出于onPause或者onStop的状态，给你数据，你也不应该去做刷新操作等，完全有可能那个界面已经销毁了啊。。
	 * 
	 * LifecycleOwner is considered as active, if its state is
	 * {@link Lifecycle.State#STARTED} or {@link Lifecycle.State#RESUMED}. An observer added via
	 * {@link #observeForever(Observer)} is considered as always active and thus will be always notified
	 * about modifications. For those observers, you should manually call
	 * {@link #removeObserver(Observer)}.
	 *   提供了针对永远都要监听的接口方法observeForever()，对应我们需要手动移除
	 *
	 * <p> An observer added with a Lifecycle will be automatically removed if the corresponding
	 * Lifecycle moves to {@link Lifecycle.State#DESTROYED} state. This is especially useful for
	 * activities and fragments where they can safely observe LiveData and not worry about leaks:
	 * they will be instantly unsubscribed when they are destroyed.
	 *	自动帮我们移除observer，这个挺好的特效
	 * <p>
	 * 
	 * In addition, LiveData has {@link LiveData#onActive()} and {@link LiveData#onInactive()} methods
	 * to get notified when number of active {@link Observer}s change between 0 and 1.
	 * This allows LiveData to release any heavy resources when it does not have any Observers that
	 * are actively observing.
	 *  提供了两个方法，便于我们自己做资源释放逻辑 
	 * <p>
	 * This class is designed to hold individual data fields of {@link ViewModel},
	 * but can also be used for sharing data between different modules in your application
	 * in a decoupled fashion.
	 *	这个也支持让我们在解耦的方式来去做数据共享，这个是解决了一个很多的数据在不同界面传递导致的出错问题
	 * @param <T> The type of data held by this instance
	 * @see ViewModel
	 */ 
	public abstract class LiveData<T> {}

我个人还是建议，在详细看代码前，先看下官方的文档介绍，这个是便于我们快速的去理解整体的结构的点，帮助我们去从整个大的背景去理解，看整个，不要只见树木不见森林。大局观是需要的。


## 开始看添加观察者

我们先从添加观察者开始看源码，

		/**
	     * Adds the given observer to the observers list within the lifespan of the given
	     * owner. The events are dispatched on the main thread. If LiveData already has data
	     * set, it will be delivered to the observer.
	     * <p>
	     * The observer will only receive events if the owner is in {@link Lifecycle.State#STARTED}
	     * or {@link Lifecycle.State#RESUMED} state (active).
	     * <p>
	     * If the owner moves to the {@link Lifecycle.State#DESTROYED} state, the observer will
	     * automatically be removed.
	     * <p>
	     * When data changes while the {@code owner} is not active, it will not receive any updates.
	     * If it becomes active again, it will receive the last available data automatically.
	     * <p>
	     * LiveData keeps a strong reference to the observer and the owner as long as the
	     * given LifecycleOwner is not destroyed. When it is destroyed, LiveData removes references to
	     * the observer &amp; the owner.
	     * <p>
	     * If the given owner is already in {@link Lifecycle.State#DESTROYED} state, LiveData
	     * ignores the call.
	     * <p>
	     * If the given owner, observer tuple is already in the list, the call is ignored.
	     * If the observer is already in the list with another owner, LiveData throws an
	     * {@link IllegalArgumentException}.
	     *
	     * @param owner    The LifecycleOwner which controls the observer
	     * @param observer The observer that will receive the events
    	 */
		
		private SafeIterableMap<Observer<T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

	    @MainThread
	    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
	        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
	            // ignore
	            return;
	        }
	        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
	        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
	        if (existing != null && !existing.isAttachedTo(owner)) {
	            throw new IllegalArgumentException("Cannot add the same observer"
	                    + " with different lifecycles");
	        }
	        if (existing != null) {
	            return;
	        }
	        owner.getLifecycle().addObserver(wrapper);
			//添加生命周期，但到了onDestory自动移除observer
	    }

为何要贴这么长的代码，因为作为框架，每个方法都加了很长的注释，这个有助于后人去理解你的设计，经常看一些第三方库，没有任何注解，只能凭感觉，实在是可怕。
希望大家以后写开源库可以多加注释，让打击更好的去理解。

我们看那个LifecycleBoundObserver,里面有一个onStateChanged函数，可以看到这个会有移除obser的逻辑，这也就是为何说会有自动清除观察者的地方

	 class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
        @NonNull final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }

他的调用，在之前的文字有提到,这个是因为在addObserver()的

	static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);// 这里被调用
            mState = newState;
        }
    }

我们去看函数，有**owner.getLifecycle().addObserver(wrapper);**这一段，就是通过他最后去添加这个state的。



这个观察者是放到一个SafeIterableMap.这个是安卓自己的不是java的sdk里面的。搞笑的是这个类的注释

	/**
	 * LinkedList, which pretends to be a map and supports modifications during iterations.
	 * It is NOT thread safe.
	 *
	 * @param <K> Key type
	 * @param <V> Value type
	 * @hide
	 */
	@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
	public class SafeIterableMap<K, V> implements Iterable<Map.Entry<K, V>> {
	}
这个是来模拟map的，名字写着是safe，但实际上并不是线程安全的。。。我想因为这个主要通过@MainThread的原因，保证了安全，所以就没必要做线程安全了。

继续回主线，那个observer被添加到这个模拟map结构类里面去。

后面我们看下刷新逻辑

# setValue（）

	 protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    } 

    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }

提供两个方法给不同线程的使用，我们是通过setValue来通知obser的

	private void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }

我们传的是null过来，所以走下面else分支

	 private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //noinspection unchecked
        observer.mObserver.onChanged((T) mData);
    }

这个b4的单词也是骚啊，不写before，写成B4。如setValue说的，会先做active的判断，同时检测是否shouldBeActive（）；
这个函数内容如下，就是检测是否为onStart之后的状态，是的话就做通知，同时还有个mVersion标记，避免异步修改导致的数据版本落后，使旧的数据刷新覆盖新的

	@Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

正常的话就是回调我们的onChagne函数。这样整体流程也就清晰了。
 

时序图：
![](https://i.imgur.com/K8kyGcE.png)

使用流程图
![](https://i.imgur.com/G3Cd0n3.png)