title: 源码探索系列43---关于FragmentStatePagerAdapter和FragmentPagerAdapter
date: 2016-09-09 13:11
tags: [android,ViewPager,FragmentStatePagerAdapter,FragmentPagerAdapter]
categories: android
--- 



 

对于这两个adapter的区别，我们在对代码做一次遍历后给出

为了便于沟通，我们把前者缩写成`FPA`,后者缩写成`FSPA`

现在我们先来看下简单的使用情况

	public abstract class FragmentPagerAdapter extends PagerAdapter 
	public abstract class FragmentStatePagerAdapter extends PagerAdapter 

通过这两个类的声明，我们知道他们都是继承与PagerAdapter的抽象类，需要实现两个接口
 

 
		public abstract Fragment getItem(int position);
		//返回对应的position要展示的Fragment
		
		public abstract int getCount();
		//返回要展示的总fragment数目
	
我想应该你有写过类似的代码，直接自己继承与PagerAdapter，然后重写多下面两个函数来创建/销毁界面

	public Object instantiateItem(ViewGroup container, int position)
	public void destroyItem(ViewGroup container, int position, Object object)

<!--more-->


## 起航
SDK23-support-V4

在开始介绍前，我们弄了下面一个简单的类，用来做说明用
	 

	public class MyFragmentPagerAdapter extends FragmentPagerAdapter {
	
		    private List<Fragment> mFragments = new ArrayList<>();
		
		    public MyFragmentPagerAdapter(FragmentManager fm) {
		        super(fm);
		    }
		
		    public void setmFragments(List<Fragment> mFragments) {
		        this.mFragments = mFragments;
		    }
		
		    @Override
		    public int getCount() {
		        return mFragments.size();
		    }
		
		    @Override
		    public Fragment getItem(int position) {
		        return mFragments.get(position);
		    }
	}
	

使用起来也很简单，就像下面这样：

    MyFragmentPagerAdapter pagerAdapter=new MyFragmentPagerAdapter(
											getFragmentManager());

    List<Fragment> fragments = new ArrayList<>();         
    fragments.add(yourFragment); 
    pagerAdapter.setmFragments(fragments);
    
    yourViewPage.setAdapter(pagerAdapter)



#  FragmentPagerAdapter

现在我们先从FragmentPagerAdapter开始说起，整个类很短，只有一百多行，没有AMS,PMS代码里面一个函数长，呵呵。哪些几万行的一个类，直接贴整个类上来。



	public abstract class FragmentPagerAdapter extends PagerAdapter {
 	
	    private final FragmentManager mFragmentManager;
	    private FragmentTransaction mCurTransaction = null;
	    private Fragment mCurrentPrimaryItem = null;
	
	    public FragmentPagerAdapter(FragmentManager fm) {
	        mFragmentManager = fm;
	    }
	 
	    public abstract Fragment getItem(int position);

	
	    @Override
	    public void startUpdate(ViewGroup container) {}	    
	    
这个函数我们在看ViewPager的时候就有看到过，他和FinishUpdate是配套的，用来表示一些更新操作的开始

下面我们看下关于instantiateItem(）的内容
	
	    @Override
	    public Object instantiateItem(ViewGroup container, int position) {
	    
	        if (mCurTransaction == null) {
	            mCurTransaction = mFragmentManager.beginTransaction();
	        }
	
	        final long itemId = getItemId(position);
			// public long getItemId(int position) {return position;}
	
	        // Do we already have this fragment?
	        String name = makeFragmentName(container.getId(), itemId);
			//makeFragmentName(int viewId,long id){
			//       return "android:switcher:" +viewId+":"+id;  }
						
	        Fragment fragment = mFragmentManager.findFragmentByTag(name);
	        
	        if (fragment != null) { 
	            mCurTransaction.attach(fragment);
	        } else {
	            fragment = getItem(position);
	            mCurTransaction.add(container.getId(), fragment,
	                    makeFragmentName(container.getId(), itemId));
	        }
	        
	        
	        //这个mCurrentPrimaryItem是用来标记当前用户看到的界面	        
	        if (fragment != mCurrentPrimaryItem) {
	            fragment.setMenuVisibility(false);
	            fragment.setUserVisibleHint(false);
	        }	
	        return fragment;
	    }
	
对于这个函数，感觉是不是和自己写listview的adapter很类似的套路？看有没可以复用的，没有就新建一个getItem（Postion），接着设置一些内容，最后就是返回。


	    @Override
	    public void destroyItem(ViewGroup container, int position, Object object) {
	        if (mCurTransaction == null) {
	            mCurTransaction = mFragmentManager.beginTransaction();
	        }        
	        mCurTransaction.detach((Fragment)object);
	    }
	    
	
关于这个函数，我们前面在看ViewPager	的时候已经有见过了，就是用来设置当前可以看到的到底是哪个，我们在instantiateItem(）函数里面也看到用mCurrentPrimaryItem来做判断的地方

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

通过设置primaryItem，在这里面更改了这两个Fragment的对用户的可见性`setUserVisibleHint(）`。

	
	    @Override
	    public void finishUpdate(ViewGroup container) {
	        if (mCurTransaction != null) {
	            mCurTransaction.commitAllowingStateLoss();
	            mCurTransaction = null;
	            mFragmentManager.executePendingTransactions();
	        }
	    }
最后关于这个finishUpdate函数，我们也在ViewPager看过，这时候就是去commit操作。
而且它调用的是`commitAllowingStateLoss()`而不是`commit()`
  
整个函数的最后就是这两个函数啦！什么鬼也没有  
	
	    @Override
	    public Parcelable saveState() {
	        return null;
	    }
	
	    @Override
	    public void restoreState(Parcelable state, ClassLoader loader) {
	    } 
	}
 


# FragmentStatePagerAdapter


接下来我们来看下`FragmentStatePagerAdapter`，他与上面的`FragmentPagerAdapter`在名字是就差别在`State`这个单词上，那么这个类是怎么体现State这个单词的呢？


	public abstract class FragmentStatePagerAdapter extends PagerAdapter {
		 
	    private final FragmentManager mFragmentManager;
	    private FragmentTransaction mCurTransaction = null; 
	    private Fragment mCurrentPrimaryItem = null;
	    
		//在参数上，我们发现它多了这么两个变量
	    private ArrayList<Fragment.SavedState> mSavedState = new 
										    ArrayList<Fragment.SavedState>();										    
	    private ArrayList<Fragment> mFragments = new ArrayList<Fragment>();
	
	    ...
	
我们直入主题，看下`instantiateItem(）`函数的内容，做个对比
	
	    @Override
	    public Object instantiateItem(ViewGroup container, int position) {
	    
	        //我们看到，如果有缓存了这个Fragment，那就直接返回 
	        //不再通过调用getItem去获取，即本地缓存了一份备份	       
	        if (mFragments.size() > position) {
	            Fragment f = mFragments.get(position);
	            if (f != null) {
	                return f;
	            }
	        }
	
	        if (mCurTransaction == null) {
	            mCurTransaction = mFragmentManager.beginTransaction();
	        }
	
	        Fragment fragment = getItem(position);	         
	        if (mSavedState.size() > position) {
	            Fragment.SavedState fss = mSavedState.get(position);
	            if (fss != null) {
	                fragment.setInitialSavedState(fss);
	            }
	        }
	        //对于保存过SavedState的，就重新恢复他原来的状态
	        //那么这信息是在什么时候存在mSavedState的呢？
	        //在destroyItem（）函数里面
	        
	        while (mFragments.size() <= position) {
	            mFragments.add(null);//为何这里add一个null？看下去
	        }
	        	        
	        fragment.setMenuVisibility(false);
	        fragment.setUserVisibleHint(false);
	        mFragments.set(position, fragment); //因为在这里调用了set
	        	        
	        mCurTransaction.add(container.getId(), fragment);
	
	        return fragment;
	    }
	
在销毁是，我们看这个也和FPA类似
	
	    @Override
	    public void destroyItem(ViewGroup container, int position, Object object) {
	        Fragment fragment = (Fragment) object;
	
	        if (mCurTransaction == null) {
	            mCurTransaction = mFragmentManager.beginTransaction();
	        } 
	        
	        while (mSavedState.size() <= position) {
	            mSavedState.add(null);
	        }
	        mSavedState.set(position, fragment.isAdded()
	                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
	        //我们的fragment的状态信息是在这里产生的
	                
	        mFragments.set(position, null);
	
	        mCurTransaction.remove(fragment);
	        //这里调用的是remove，而FPA是detach而已
	    }
  
		...

通过这个destroyItem和instantiateItem函数，我们对为何有加多state有一个大概缘由，不过相比较而言，FSPA还有下面两个函数，下面两个函数是在FPA我们看到没实现的，我们看具体到底做了什么
	
	    @Override
	    public Parcelable saveState() {
	        Bundle state = null;
	        if (mSavedState.size() > 0) {
	            state = new Bundle();
	            Fragment.SavedState[] fss = new Fragment.SavedState[mSavedState.size()];
	            mSavedState.toArray(fss);
	            state.putParcelableArray("states", fss);
	        }
	        for (int i=0; i<mFragments.size(); i++) {
	            Fragment f = mFragments.get(i);
	            if (f != null && f.isAdded()) {
	                if (state == null) {
	                    state = new Bundle();
	                }
	                String key = "f" + i;
	                mFragmentManager.putFragment(state, key, f);
	            }
	        }
	        return state;
	    }

可以在这函数看到他会把framgnet保存到Manager里面，那么问题来了，这个函数putFragment()做了什么 
根据官方的注释：

> Put a reference to a fragment in a Bundle. This Bundle can be persisted as saved state, 
> and when later restoring getFragment(Bundle, String) will return the current instance of the same fragment.

他会被保存下来，后期我们通过bundle和key的配合方式去找回我们的特定fragment。
所以我们来看下我们的`restoreState`

	    @Override
	    public void restoreState(Parcelable state, ClassLoader loader) {
	        if (state != null) {
	            Bundle bundle = (Bundle)state;
	            bundle.setClassLoader(loader);
	            Parcelable[] fss = bundle.getParcelableArray("states");
	            mSavedState.clear();
	            mFragments.clear();
	            if (fss != null) {
	                for (int i=0; i<fss.length; i++) {
	                    mSavedState.add((Fragment.SavedState)fss[i]);
	                }
	            }
	            Iterable<String> keys = bundle.keySet();
	            for (String key: keys) {
	                if (key.startsWith("f")) {
	                    int index = Integer.parseInt(key.substring(1));
	                    Fragment f = mFragmentManager.getFragment(bundle, key);
						//按照约定的，传bundle和key回去来获得对应的fragment
						
	                    if (f != null) {
	                        while (mFragments.size() <= index) {
	                            mFragments.add(null);
	                        }
	                        f.setMenuVisibility(false);
	                        mFragments.set(index, f);
	                    } else {
	                        Log.w(TAG, "Bad fragment at key " + key);
	                    }
	                }
	            }
	        }
	    }
	}
看我这里，有一点疑惑的地方，就是这两个函数是什么时候被调用的呢？

这个是在ViewPager里面调用

	@Override
    public Parcelable onSaveInstanceState() {
        Parcelable superState = super.onSaveInstanceState();
        SavedState ss = new SavedState(superState);
        ss.position = mCurItem;
        if (mAdapter != null) {
            ss.adapterState = mAdapter.saveState();
        }
        return ss;
    }
	
	@Override
    public void onRestoreInstanceState(Parcelable state) {
        if (!(state instanceof SavedState)) {
            super.onRestoreInstanceState(state);
            return;
        }

        SavedState ss = (SavedState)state;
        super.onRestoreInstanceState(ss.getSuperState());

        if (mAdapter != null) {
            mAdapter.restoreState(ss.adapterState, ss.loader);
            setCurrentItemInternal(ss.position, false, true);
        } else {
            mRestoredCurItem = ss.position;
            mRestoredAdapterState = ss.adapterState;
            mRestoredClassLoader = ss.loader;
        }
    }
    
然后不要问我这两个函数是什么时候被调用的了。这两个函数是View的基本方法。和我们的Fragment，Activity都包含的类似的两个方法一样，在系统销毁的时候调用你。

# FSPA& FPA的区别

## 缓存
通过上面的代码，我们知道这两者的一大区别就是前者会缓存这个state，而后者不会这么做。
本地有多一份fragment列表的备份。

这点我们可以在 destroyItem()函数看到区别

FSPA在destroyItem的时候用的是remove，销毁该fragment， 而FPA调用的是detach()，他所做的只是将view从UI中移除，此时fragment的状态依然由FragmentManager维护。

所以我们看FSPA有如下的说明,他相对于FPA果要节约内存，就可以用这个

> This version of the pager is more useful when there are a large number of pages, 
> working more like a list view.  
>  When pages are not visible to  the user, their entire fragment may be destroyed, 
>  only keeping  the saved   state of that fragment.  
>  This allows the pager to hold on  to much less memory associated with each visited page 
>  as compared  to  FragmentPagerAdapter   at the cost of potentially more overhead 
>  when  switching between pages.



# REF

https://developer.android.com/reference/android/app/Fragment.html