title: JetPack源码探索系列1---Lifecycle

date: 2018-08-23 22:45:46

tags: [android,JetPack,Lifecycle]

categories: android
 

------------------------------------------

  
  

安卓提供了新的JetPack工具包，把原本我们早期开发安卓所遇到的一些基础问题，重新做了封装，便于我们使用。

  

这些组件可帮助我们摆脱编写样板代码的工作并简化复杂任务，以便您可以专注于您关心的代码。


![](https://i.imgur.com/aTR60lI.png)

这次的源码探索系列，就针对这块来做新的探索，虽然Flutter已经逐渐的成熟，大有逐渐替代之势，不过还是继续看下这块的源码，对其思路做了解，以方便以后的框架设计，提供新的思路来做。

系列文章基本按照这个列表的内容来做，对于基础包就不做过度描述，Appcompat最近运用于做一些转场动画，不过这个只针对5.0的贡献元素等的一次封装，没太大意义，就不管了，KTX是对kotlin的扩展使用，也不深入去说，Multidex这个都知道，只是挪过来放一起，就不去看了。

所以这次的探索系列文章，主要针对**架构**这一列.先做记录，然后对UI，行为这块做些记录。像TV,Wear OS 这些选择看了即可，需要时候再看。
同时作为踩过dataBind的问题的人，现在对他是完全的嫌弃的，他带来太多的问题了。

现在就开始从Lifecycles开始写起.点击more查看更多哦！

<!--more-->

# LifeCycles
早期开发安卓经常遇到一些生命周期的问题，例如要处理些在Fragment发起网络请求，接着Fragment被销毁，突然这时候请求回来了，这就往往容易出一堆的问题，为此我们可以使用LIfeCycle来做些处理

**Lifecycles**：生命周期感知组件，处理生命周期的相关的操作。Lifecycle 是一个类，它持有关于组件（如 Activity 或 Fragment）生命周期状态的信息，并且允许其他对象观察此状态。

使用起来像下面的样子：

	public interface IPresenter extends LifecycleObserver {
	
	    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
	    void onCreate(@NotNull LifecycleOwner owner);
	
	    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
	    void onDestroy(@NotNull LifecycleOwner owner);
	
	    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
	    void onLifecycleChanged(@NotNull LifecycleOwner owner, @NotNull Lifecycle.Event event);
	}

	public class BasePresenter implements IPresenter {
	
	    private static final String TAG = "BasePresenter";    
	
	    @Override
	    public void onLifecycleChanged(@NotNull LifecycleOwner owner, @NotNull Lifecycle.Event event) {
	
	    }
	
	    @Override
	    public void onCreate(@NotNull LifecycleOwner owner) {
	        Log.d(TAG, "BasePresenter.onCreate" + this.getClass().toString());
	    }
	
	    @Override
	    public void onDestroy(@NotNull LifecycleOwner owner) {
	        Log.d(TAG, "BasePresenter.onDestroy" + this.getClass().toString());
	    }
	}


	public class MainActivity extends AppCompatActivity {
	    private IPresenter mPresenter;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState); 
	        setContentView(R.layout.activity_main);

	        mPresenter = new MainPresenter(this);
	        getLifecycle().addObserver(mPresenter);
			//添加LifecycleObserver，之后会自动调用presenter里面对应的生命周期方法
	    } 
	}

这个就像它自己说的，把一些模板代码给帮我们封装好了，但稍微成熟的app像这样的事情其实都做了，只是对于新开的项目可以尝试这些东西。

基于上面的内容，我们可以先思考下，他到底是怎么做到生命周期的回调的？
我们看他是在方法前面加多注解的，依据我们看这么多年注解的框架来理解，有可能是APT那样的生成的新的代码，来帮我们做了这些事情，
要不是被系统识别到，自动调用。带着这个想法，我们开始看下源码，到底是怎么做到的？最后再贴UML图


在详细看之前，我们先看下官方接口的文档说明：

	Defines an object that has an Android Lifecycle. Fragment and FragmentActivity classes 
	implement LifecycleOwner interface which has the getLifecycle method to access the Lifecycle.
	You can also implement LifecycleOwner in your own classes.

	Lifecycle.Event.ON_CREATE, Lifecycle.Event.ON_START, Lifecycle.Event.ON_RESUME events
	in this class are dispatched after the LifecycleOwner's related method returns.

 	`==============注意上下两排的调用顺序区别====================`

	Lifecycle.Event.ON_PAUSE, Lifecycle.Event.ON_STOP, Lifecycle.Event.ON_DESTROY events in this 
	class are dispatched before the LifecycleOwner's related method is called.  
	很好理解，初始化一些环节要想准备好我们才可以调用，
	销毁时候，我们是需要做提前的销毁操作，不然很多变量等信息就都没了啊。
 
	For instance, Lifecycle.Event.ON_START will be dispatched after onStart returns, 
	Lifecycle.Event.ON_STOP will be dispatched before onStop is called.
	This gives you certain guarantees on which state the owner is in.
	If you use Java 8 Language, then observe events with DefaultLifecycleObserver. 
	To include it you should add "android.arch.lifecycle:common-java8:<version>"
	to your build.gradle file.

	   class TestObserver implements DefaultLifecycleObserver {
	        @Override
	       public void onCreate(LifecycleOwner owner) {
	           // your code
	       }
	   }
	===注意下这里，对于java7的区别，现在为了上RX和lambada之类的，我们项目是已经升级Java到8了===
   
	If you use Java 7 Language, Lifecycle events are observed using annotations.
	Once Java 8 Language becomes mainstream on Android, annotations will be deprecated, 
	so between DefaultLifecycleObserver and annotations,
	you must always prefer DefaultLifecycleObserver.

	   class TestObserver implements LifecycleObserver {
	      @OnLifecycleEvent(ON_STOP)
	     void onStopped() {}
	   }
	   
	Observer methods can receive zero or one argument. 
	If used, the first argument must be of type LifecycleOwner.
	Methods annotated with Lifecycle.Event.ON_ANY can receive the second argument, 
	which must be of type Lifecycle.Event.

	   class TestObserver implements LifecycleObserver {
	      @OnLifecycleEvent(ON_CREATE)
	     void onCreated(LifecycleOwner source) {}
	      @OnLifecycleEvent(ON_ANY)
	     void onAny(LifecycleOwner source, Event event) {}
	   }	
			

个人就不翻译了，很简单，都看的懂,说到这里得提下，大部分框架在类文件的开头没写很多的注释，
说明整体的思虑设计之类信息，其实挺不友好的，久了自己也会忘记代码细节内容的
，后面接盘的人也难以快速的理解原本的思路。


接下我们看具体实现


	@Override
    public void addObserver(@NonNull LifecycleObserver observer) {
		
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;		
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }

        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }

可以看到，根据observer和initialState构造ObserverWithState对象statefulObserver，
然后将该对象存入mObserverMap，可以简单的把它理解成用来保存观察者的Map。然后等到回到时候去查map来调用对应方法？

现在我们看下怎么等到通知呢？其实在FragmentManager是可以收到对应的生命周期回到的，Activity也是类似的，就不贴了
![](https://i.imgur.com/XsXGMMy.png)

调用你这个performXXx函数然后呢？

![](https://i.imgur.com/3b6jhWl.png)
我们看到会去调用`handleLifecycleEvent（）`
他的入口是这个

	 public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
	        State next = getStateAfter(event);
	        moveToState(next);
	    }

	  static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
读取到对应的状态，然后到下个状态，像个状态机一样的

	private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;	//赋值刷新当前的状态为新状态!

        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }	

关键在Sync方法，我们去看下

	// happens only on the top of stack (never in reentrance),
    // so it doesn't have to take in account parents
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }

这里需要说下，这些状态是一堆枚举的变量，谷歌自己说要少用枚举，多用注解的使用方式。哎。弄了点小技巧，用枚举来做状态的计算，确定是`backwardPass`还是`forwardPass`往前

	@SuppressWarnings("WeakerAccess")
    public enum Event {
        /**
         * Constant for onCreate event of the {@link LifecycleOwner}.
         */
        ON_CREATE,
        /**
         * Constant for onStart event of the {@link LifecycleOwner}.
         */
        ON_START,
        /**
         * Constant for onResume event of the {@link LifecycleOwner}.
         */
        ON_RESUME,
        /**
         * Constant for onPause event of the {@link LifecycleOwner}.
         */
        ON_PAUSE,
        /**
         * Constant for onStop event of the {@link LifecycleOwner}.
         */
        ON_STOP,
        /**
         * Constant for onDestroy event of the {@link LifecycleOwner}.
         */
        ON_DESTROY,
        /**
         * An {@link Event Event} constant that can be used to match all events.
         */
        ON_ANY
    }


我们拿往前的代码看下：

	 private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

 这里确实去走了一遍遍历map的结构！最后是去分发消息，这个也就很好的理解了。类似于EventBus的结构，注册保存到map，等消息来，遍历回调的过程。
	
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
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }

看我整体我们也就明白了；

贴一张别人的图，就不自己画了。

![](https://i.imgur.com/940ECip.png)
