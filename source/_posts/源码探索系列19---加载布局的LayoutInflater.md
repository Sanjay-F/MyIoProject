title:  源码探索系列19---加载布局的LayoutInflater
date: 2016-01-10 00:31:46
tags: [android,源码]
categories: android

------------------------------------------


对于这个layoutInflater我们肯定不陌生，每次写各种Adatper都遇到他，基本就像下面这样的套路。

	@Override
    public View getView(int i, View view, ViewGroup viewGroup) {
        SimpleViewHolder viewHolder;
        if (view == null) {
            view = LayoutInflater.from(mContext).inflate(R.layout.listitem_demo, null);
            viewHolder = new SimpleViewHolder();
            viewHolder.bindView(view);
            view.setTag(viewHolder);
        } else {
            viewHolder = (SimpleViewHolder) view.getTag();
        }

那么问题来了，这个家伙是怎么加载到他的呢？
我们今天去看下。

<!--more-->

# 起航
API:23

不过这个LayoutInflater是一个抽象类啊，所以第一个问题来了，我们得去找到背后实际干活的人！

	public abstract class LayoutInflater 

既然这样，我们就顺藤摸瓜当前去看下

	public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }

好啦，他是去调用了`context.getSystemService()`函数，这个context我们已经很熟悉了，他的具体实现是`ContextImpl`，当然他的getService我们也挺熟悉他的内容了。不过还是继续看下吧

	@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
我们到`SystemServiceRegistry`里面看下

	public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
    
这个是用保存在`HashMap`结构的`SYSTEM_SERVICE_FETCHERS`里面的数据。他的初始化是在开头的静态代码里面。

	final class SystemServiceRegistry {
	
	    private final static String TAG = "SystemServiceRegistry";

	     private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS 
			     = new HashMap<String, ServiceFetcher<?>>();
 	
	    static { 
		 ...
		 registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
            
    	 ...
	    }

		private static <T> void registerService(String serviceName, Class<T> serviceClass,
	            ServiceFetcher<T> serviceFetcher) {
	        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
	        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
	    }

不过在这个静态代码块里面还有很多别的service的注册，下次需要知道那个服务的具体实现就直接看这里吧。好了，我们看到背后实际干活的是`PhoneLayoutInflater`这个类，不够暂时不打算继续深入的看这个类，因为他内容不多，我们继续看原来`LayoutInflater.from(mContext).inflate()`后半句的`inflate`函数的内容，后面再补充介绍这个`PhoneLayoutInflater`

	public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
    
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
         final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
我们看到，他会去拿`Resource`，然后靠他去根据这个LayoutRes的Id去获取布局，最后再调用`inflate(parser, root, attachToRoot)`去生成我们的View。
这里额外的提下关于`@LayoutRes` 这类注解，以前[写过篇文章简单介绍过他](http://blog.csdn.net/sanjay_f/article/details/50325145)，他在我们实际项目开发中还是有很多作用的！起到对传入参数的检测。关于这个Resource，我们在前一篇文章已经做过简单介绍了，现在我们也看下他的getLayhout函数去看下吧。

	public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
        return loadXmlResourceParser(id, "layout");
    }
    
	XmlResourceParser loadXmlResourceParser(int id, String type)
            throws NotFoundException {
        synchronized (mAccessLock) {
            TypedValue value = mTmpValue;
            if (value == null) {
                mTmpValue = value = new TypedValue();
            }
            getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                return loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException(
                    "Resource ID #0x" + Integer.toHexString(id) + " type #0x"
                    + Integer.toHexString(value.type) + " is not valid");
        }
    }
很有意思，这里显示的只有一个返回路径，所以基本可以确实下面的函数会被执行。

	public void getValue(@AnyRes int id, TypedValue outValue, boolean resolveRefs)
            throws NotFoundException {
        boolean found = mAssets.getResourceValue(id, 0, outValue, resolveRefs);
        if (found) {
            return;
        }
        throw new NotFoundException("Resource ID #0x"
                                    + Integer.toHexString(id));
    }
继续前进
	
	/*package*/ final boolean getResourceValue(int ident,
                                               int density,
                                               TypedValue outValue,
                                               boolean resolveRefs)
    {
        int block = loadResourceValue(ident, (short) density, outValue, resolveRefs);
        if (block >= 0) {
            if (outValue.type != TypedValue.TYPE_STRING) {
                return true;
            }
            outValue.string = mStringBlocks[block].get(outValue.data);
            return true;
        }
        return false;
    }
在我们的AssetManager里面，会根据传过来的参数去加载对应的布局，不过这个方法是native，就不深入看了

	private native final int loadResourceValue(int ident, short density, 
						TypedValue outValue,boolean resolve);

继续回到上面那里，我们看下执行后的下一句的内容，即`loadXmlResourceParser`函数里面的

	getValue(id, value, true);
	
    if (value.type == TypedValue.TYPE_STRING) {
        return loadXmlResourceParser(value.string.toString(), id,
                value.assetCookie, type);
    }	

	/*package*/ XmlResourceParser loadXmlResourceParser(String file, int id,
            int assetCookie, String type) throws NotFoundException {
            
        if (id != 0) {
            try {
                // These may be compiled...
                synchronized (mCachedXmlBlockIds) { 
                    final int num = mCachedXmlBlockIds.length;
                    for (int i=0; i<num; i++) {
                        if (mCachedXmlBlockIds[i] == id) {
                             return mCachedXmlBlocks[i].newParser();
                        }
                    }
 
                    // Not in the cache, create a new block and put it at
                    // the next slot in the cache.
                    XmlBlock block = mAssets.openXmlBlockAsset(
                            assetCookie, file);
                    if (block != null) {
                        int pos = mLastCachedXmlBlockIndex+1;
                        if (pos >= num) pos = 0;
                        mLastCachedXmlBlockIndex = pos;
                        XmlBlock oldBlock = mCachedXmlBlocks[pos];
                        if (oldBlock != null) {
                            oldBlock.close();
                        }
                        mCachedXmlBlockIds[pos] = id;
                        mCachedXmlBlocks[pos] = block;
                        //System.out.println("**** CACHING NEW XML BLOCK!  id="
                        //                   + id + ", index=" + pos);
                        return block.newParser();
                    }
                }
            } catch (Exception e) {
            ...
             }
        } 
	}
我们看到他会先查下缓存，没有再去新建，不过有意思的是，他是对XmlResourceParser做缓存，返回的是调用newParser()函数的新new的一个。

这样我们继续回主线，到我们的`LayoutInflater`类的`inflate`函数里面去，
	
	public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources(); 
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
看完了加回XmlResourceParser ，我们看怎么生成View的部分

	public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
             
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {                 
	           ...

                final String name = parser.getName();
                                
	             // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;
                
				 ...

                // Inflate all children under temp against its context.
                //解析Temp视图下的子View
                rInflateChildren(parser, temp, attrs, true);
 
				 ...
				 
                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
             }

            } catch (Exception e) {
                ...
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }
			...

            return result;
        }
    }
我们看到他主要是通过`createViewFromTag(）`函数来生成具体的view的，而且我们的root是null，所以就返回的就是他了，不同加到root去。

	private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
	        return createViewFromTag(parent, name, context, attrs, false);
	    }
	
	View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,boolean ignoreThemeAttr) {
	
		...
		
		//这算彩蛋嘛？ 这个TAG_1995值是 "blink";
        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
	        ...
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {//显然这句写得不是很好
	                    //是否包含内置View控件的解析用是否含“.”来做判断也太魔术了。
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        }  
        ...
    }
    
我们看到他对自定义控件View的解析和内置的View解析是调用不同的方法，难道这两者解析过程是不一样的？ 我们看下这个onCreateView做了什么。

	protected View onCreateView(View parent, String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return onCreateView(name, attrs);
    }
    
    //加多了一个安卓的前缀。。
	protected View onCreateView(String name, AttributeSet attrs)
            throws ClassNotFoundException {
        return createView(name, "android.view.", attrs);
    }
    
    public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        Class<? extends View> clazz = null;

        try { 

			//试下先从缓存获取构造函数。
			//那后面的逻辑估计就是生成一个，然后保存到缓存去咯？
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                //用ClassLoader加载该类
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);
                
                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                //缓存 构造函数
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
               ...
            }

            Object[] args = mConstructorArgs;
            args[1] = attrs;
			//反射构造View
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            return view;

        } 
        ...
    }
    
 好了，看到这里，我们就知道了他的View的创建过程了，另外这个判断最终也没做到什么！最后调用的函数都是同一个！只是谷歌方便我们写界面的时候，方便我们使用内置的View，自动补个`android.view`而且！基本我们想要的View就得到啦！

我们回主线，继续看下一句`rInflateChildren(parser, temp, attrs, true);`的内容。

	final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
	
	void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
    
我们看到的是通过DFS算法来构造视图树，每解析一个View就递归调用`rInflate`。然后再回溯将每个View元素添加到他们的`parent`中。

**注意：**
这里需要提出俩的是，不建议在布局文件中做过多View嵌套的一个原因，因为构造过程是递归的。


# 后记

今天去中山搞团建，挺好玩的，团队拿了第一，作为队长的我，成功运用`群华`的想法还是很功不可没的。
嗯，装逼下！哈哈

今天把图片补在后面，哈哈。

 ![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/01300000361240123790910502016.jpg)