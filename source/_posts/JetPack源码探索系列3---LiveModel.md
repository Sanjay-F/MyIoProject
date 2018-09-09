title: JetPack源码探索系列3---VidwModel

date: 2018-08-26 21:45:46

tags: [android,JetPack,VidwModel]

categories: android
 

------------------------------------------
 

# VidwModel 

前面在使用LiveData时候，我们已经看到了ViewModel，不过没有去深入的说，因为那篇文章是说LiveData的，这次我们说会ViewModle去。
它主要是以关联生命周期的方式来存储和管理UI相关的数据的类，即使configuration发生改变，数据仍然可以存在不会销毁
前面我们看到我们的LiveData是写在viewModle里面的变量啊。


jetPack在引入ViewModel时有以下几点：

1. 数据管理：     
 Activity或Fragment这类应用组件都有自己的生命周期，他们的生命周期都是被Framework所管理。Framework可能会根据用户的一些操作以及设备的状态对Activity或Fragment进行销毁和重建。作为开发者，这些行为我们是无法干预的。伴随着Activity或Fragment的销毁和重建，它们当中的数据也会随着一起销毁和重建。对于一些简单的数据，Activity可以使用onSaveInstanceState()方法，并从onCreate的bundle中重新获取，但这一方法仅仅适合一些简单的UI状态，对于列表型这种庞大的数据类型并不适合

2. 异步与生命周期问题：   
Activity或Fragment经常会做一些异步的耗时操作，随之就需要管理这些异步操作得到的数据，并在destroyed的时候清理它们，从而避免内存溢出这类问题的发生。但是这样的处理会随着项目扩大而变得十分复杂

3. 分忧：  
Activity或Fragment本身需要处理很多用户的输入事件并和操作系统打交道，当它们还要花时间管理那些数据资源时，它们所在的类就会变得异常庞大，造就出所谓的god activities和god fragments，这样很尴尬

  
正因为这样，所以类似MVP结构的重要一环一样，这个也会有很长的生命周期，便于我们对数据做统一管理
![](https://i.imgur.com/V9VHrOn.png)

<!--more-->

# 起航 

关于使用在是上一篇说了，这里不重复，直接看源码把。之前已经说了，ViewModel即使发生configuration改变（比如旋转屏幕），数据仍然可以存在不会销毁，到底怎么做到的呢？

先看文档：

	/**
	 * ViewModel is a class that is responsible for preparing and managing the data for
	 * an {@link android.app.Activity Activity} or a {@link android.support.v4.app.Fragment Fragment}.
	 * It also handles the communication of the Activity / Fragment with the rest of the application
	 * (e.g. calling the business logic classes).
	 * <p>有点类似于MVP的p，做一些持久化的数据管理之类事情。
	 * 
	 * A ViewModel is always created in association with a scope (an fragment or an activity) and will
	 * be retained as long as the scope is alive. E.g. if it is an Activity, until it is
	 * finished.
	 * <p>
	 * In other words, this means that a ViewModel will not be destroyed if its owner is destroyed for a
	 * configuration change (e.g. rotation). The new instance of the owner will just re-connected to the
	 * existing ViewModel.
	 * <p>
	 * #设置这个的目的：
	 * The purpose of the ViewModel is to acquire and keep the information that is necessary for an
	 * Activity or a Fragment. The Activity or the Fragment should be able to observe changes in the
	 * ViewModel. ViewModels usually expose this information via {@link LiveData} or Android Data
	 * Binding. You can also use any observability construct from you favorite framework.
	 * <p>
	 * ViewModel's only responsibility is to manage the data for the UI. It <b>should never</b> access
	 * your view hierarchy or hold a reference back to the Activity or the Fragment.
	 * <p>
	 * Typical usage from an Activity standpoint would be:
	 * <pre>
	 * public class UserActivity extends Activity {
	 *
	 *     {@literal @}Override
	 *     protected void onCreate(Bundle savedInstanceState) {
	 *         super.onCreate(savedInstanceState);
	 *         setContentView(R.layout.user_activity_layout);
	 *         final UserModel viewModel = ViewModelProviders.of(this).get(UserModel.class);
	 *         viewModel.userLiveData.observer(this, new Observer<User>() {
	 *            {@literal @}Override
	 *             public void onChanged(@Nullable User data) {
	 *                 // update ui.
	 *             }
	 *         });
	 *         findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
	 *             {@literal @}Override
	 *             public void onClick(View v) {
	 *                  viewModel.doAction();
	 *             }
	 *         });
	 *     }
	 * }
	 * </pre>
	 *
	 * ViewModel would be:
	 * <pre>
	 * public class UserModel extends ViewModel {
	 *     public final LiveData&lt;User&gt; userLiveData = new LiveData<>();
	 *
	 *     public UserModel() {
	 *         // trigger user load.
	 *     }
	 *
	 *     void doAction() {
	 *         // depending on the action, do necessary business logic calls and update the
	 *         // userLiveData.
	 *     }
	 * }
	 * </pre>
	 *
	 * <p>
	 * ViewModels can also be used as a communication layer between different Fragments of an Activity.
	 * Each Fragment can acquire the ViewModel using the same key via their Activity. This allows
	 * communication between Fragments in a de-coupled fashion such that they never need to talk to
	 * the other Fragment directly.
	 * <pre>
	 * public class MyFragment extends Fragment {
	 *     public void onStart() {
	 *         UserModel userModel = ViewModelProviders.of(getActivity()).get(UserModel.class);
	 *     }
	 * }
	 * </pre>
	 * </>
	 */
	public abstract class ViewModel {
		    /**
		     * This method will be called when this ViewModel is no longer used and will be destroyed.
		     * <p>
		     * It is useful when ViewModel observes some data and you need to clear this subscription to
		     * prevent a leak of this ViewModel.
		     */
	    @SuppressWarnings("WeakerAccess")
	    protected void onCleared() {
	    }
	}

说真的，这个类真的是什么都没有，但是文档很长，描述了这个到底做了什么事情，也便于我们后续去看源码的时候，可以更好的理解对应的逻辑

![](https://i.imgur.com/WM1dXf4.png)
看完我们明白，这个ViewModel很像是MVP总的P，负责和view做交互逻辑的，对吗？

# 源码


我们从上一个评论案例入口来说：

	mViewModel = ViewModelProviders.of(this).get(CommentsViewModel.class);

## of()
这个入口从这里进去后，我们知道了该怎么看源码了。

	/**
	 * Utilities methods for {@link ViewModelStore} class.
	 */
	public class ViewModelProviders {

		/*
	     * Creates a {@link ViewModelProvider}, which retains ViewModels while a scope of given
	     * {@code fragment} is alive. More detailed explanation is in {@link ViewModel}.
	     * <p>
	     * It uses {@link ViewModelProvider.AndroidViewModelFactory} to instantiate new ViewModels.
	     *
	     * @param fragment a fragment, in whose scope ViewModels should be retained
	     * @return a ViewModelProvider instance
	     */
		@MainThread
	    public static ViewModelProvider of(@NonNull Fragment fragment) {
	        return of(fragment, null);
	    }
	
		 @MainThread
	    public static ViewModelProvider of(@NonNull Fragment fragment, @Nullable Factory factory) {
	        Application application = checkApplication(checkActivity(fragment));
	        if (factory == null) {
	            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
	        }
	        return new ViewModelProvider(ViewModelStores.of(fragment), factory);
	    }
		

		/**
	     * Creates {@code ViewModelProvider}, which will create {@code ViewModels} via the given
	     * {@code Factory} and retain them in the given {@code store}.
	     *
	     * @param store   {@code ViewModelStore} where ViewModels will be stored.
	     * @param factory factory a {@code Factory} which will be used to instantiate
	     *                new {@code ViewModels}
	     */
		public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
	        mFactory = factory;
	        this.mViewModelStore = store;
	    }
	}

我们看这个是new了一个新的ViewModelProviders出来的，构造了两个fragemnt,factory参数，
我们看下构造参数时候的ViewModelStores.of静态方法:

	@MainThread
    public static ViewModelStore of(@NonNull Fragment fragment) {
        if (fragment instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) fragment).getViewModelStore();
        }
        return holderFragmentFor(fragment).getViewModelStore();
    }

	 @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static HolderFragment holderFragmentFor(Fragment fragment) {
        return sHolderFragmentManager.holderFragmentFor(fragment);
    }

去拿到Fragment对应的holder出来

	/**
	 * @hide
	 */
	@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
	public class HolderFragment extends Fragment implements ViewModelStoreOwner {}
这个holderFragment是系统的一个隐藏类，具体是做什么呢？先不管，我们看下后面的getViewModelStore内容

	private ViewModelStore mViewModelStore = new ViewModelStore();
	@Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }

我们看下重要的方法holderFragmentFor里面的内容：

	HolderFragment holderFragmentFor(Fragment parentFragment) {
            FragmentManager fm = parentFragment.getChildFragmentManager();
            HolderFragment holder = findHolderFragment(fm);
            if (holder != null) {
                return holder;
            }
            holder = mNotCommittedFragmentHolders.get(parentFragment);
            if (holder != null) {
                return holder;
            }

            parentFragment.getFragmentManager()
                    .registerFragmentLifecycleCallbacks(mParentDestroyedCallback, false);
            holder = createHolderFragment(fm);
            mNotCommittedFragmentHolders.put(parentFragment, holder);
            return holder;
        }

		private static HolderFragment createHolderFragment(FragmentManager fragmentManager) {
            HolderFragment holder = new HolderFragment();
            fragmentManager.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();
            return holder;
        }

holderFragmentFor()负责创建Fragment并与其所在的Activity的Lifecycle相关联

在看进去这个结构：

	public class ViewModelStore {

	    private final HashMap<String, ViewModel> mMap = new HashMap<>();
	
	    final void put(String key, ViewModel viewModel) {
	        ViewModel oldViewModel = mMap.put(key, viewModel);
	        if (oldViewModel != null) {
	            oldViewModel.onCleared();
	        }
	    }
	
	    final ViewModel get(String key) {
	        return mMap.get(key);
	    }
	
	    /**
	     *  Clears internal storage and notifies ViewModels that they are no longer used.
	     */
	    public final void clear() {
	        for (ViewModel vm : mMap.values()) {
	            vm.onCleared();
	        }
	        mMap.clear();
	    }
	}


看都这里，我们可以大胆的猜测，是否因为在某个更持久的地方，放了这个viewModel，所以才保证了一直可以拿到这个ViewModel数据？
通过把这个数据放在Fragment对象里面，这样只要不是销毁，无论经历什么生命周期，都可以去通过这个地方拿到的意思，从而达到效果?


在HolderFragment创建的同时也完成了ViewModelStore的创建，而ViewModelStore里面保存的都是ViewModel，所以ViewModel也就被保存下来了.而当HolderFragment销毁时，其所保存的ViewModel对象就会被清理掉

 

## get()
这样前面部分也就看完了，我们来看下后面的get内容：

	mViewModel = ViewModelProviders.of(this).get(CommentsViewModel.class);


	 /**
     * Returns an existing ViewModel or creates a new one in the scope (usually, a fragment or
     * an activity), associated with this {@code ViewModelProvider}.
     * <p>
     * The created ViewModel is associated with the given scope and will be retained
     * as long as the scope is alive (e.g. if it is an activity, until it is
     * finished or process is killed).
     *
     * @param modelClass The class of the ViewModel to create an instance of it if it is not
     *                   present.
     * @param <T>        The type parameter for the ViewModel.
     * @return A ViewModel that is an instance of the given type {@code T}.
     */
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
	private static final String DEFAULT_KEY =
            "android.arch.lifecycle.ViewModelProvider.DefaultKey";


	 @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }

        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }
系统会优先从已有的ViewModelStore中去查找这个Key对应的ViewModel，如果找到则直接返回，否则则通过mFactory.create(modelClass)创建并保存到ViewModelStore中再返回。我们来看下create()

看到这里我们也就终于知道了，确实就是保存在store里面去了。


我们看下cereate里面的

	 	@Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
                //noinspection TryWithIdenticalCatches
                try {
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (IllegalAccessException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InstantiationException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InvocationTargetException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                }
            }
            return super.create(modelClass);
        }
没有就反射调用给你new一个。哈哈

好了这样我们也就明白整体是怎样的拉！
