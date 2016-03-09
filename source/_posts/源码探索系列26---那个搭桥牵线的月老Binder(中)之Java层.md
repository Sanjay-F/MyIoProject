title: 源码探索系列26---那个搭桥牵线的月老Binder(中)之Java层
date: 2016-03-09 13:30:46
tags: [android,源码,Binder]
categories: android

------------------------------------------
 
 上一篇文章，我们对Binder做了一个简单的介绍。现在想从Binder机制的Java层来说下。

![Java层的Binder框架](http://7xl9zd.com1.z0.glb.clouddn.com/binder_Binder%E7%9A%84%E5%B1%82%E7%BA%A7%E6%A1%86%E6%9E%B6.png)

安卓本身在`natvie层`搭建了整套机制的基础，然后在`Java层`也重新包装了下，方便我们使用。
因此本篇想尝试就只剖析Java层的架构为主，对于natvie层只是简单掠过，虽然在native层我们可以看到他加载虚拟Binder设备，打开Binder驱动，注册service的具体内容。不过我们还是一步一步来。有兴趣的伙伴可以自己去看下。

那么，就让我们开始探索之旅吧。

<!--more-->

# 起航

我们按照通讯模型，先从向ServiceManager添加服务作为探索的起点ServiceManager.addService()。

	 
	public static void addService(String name, IBinder service) {
        try {
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }


	private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(
					        BinderInternal.getContextObject());
        return sServiceManager;
    }

这个BinderInternal是仅供Binder框架使用的类，内部有一个static final 的GcWatcher类，负责垃圾回收工作。他的getContextObject()是一个native方法，他会创建一个BinderProxy对象，然后通过JNI把它和一个Native的BpProxy对象挂钩。替我们去和Binder驱动通讯（可以看一开始的图）。

接着我们看下asInterface的内容
	
	static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }        
        return new ServiceManagerProxy(obj);
    }
他把我们的 BinderProxy用SMP包成一个IServiceManager返回。这个SMP是一个内部类，实现IServiceManager 接口，主要的是打包请求数据，然后交给BinderProxy对象去处理。例如下面的片段代码一样

	class ServiceManagerProxy implements IServiceManager {
	    public ServiceManagerProxy(IBinder remote) {
	        mRemote = remote;
	    } 
	    
	    public void addService(String name, IBinder service, boolean allowIsolated)
	            throws RemoteException {
	        Parcel data = Parcel.obtain();
	        Parcel reply = Parcel.obtain();
	        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
	        data.writeInterfaceToken(IServiceManager.descriptor);
	        //打包请求数据，然后交给BinderProxy对象去处理,
			//后者再转个BpBinder，然后和Binder驱动通讯
	        data.writeString(name);
	        data.writeStrongBinder(service);
	        data.writeInt(allowIsolated ? 1 : 0);
	        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
	        reply.recycle();
	        data.recycle();
	    }
	     
		    ...
    }
这样我们就看完注册流程了。背后的native代码。我们下次再深入的看。

---

接下来，我们需要去看下我们的客户端的流程
我们再客户端调用返回的proxy代理对象的方法


	IDemoAidlInterface demoAidlInterface = IDemoAidlInterface.Stub.
                    asInterface(service);
                boolean relation=demoAidlInterface.beMyGirlFriend(666666, true);


接着在我们的内部类Proxy里面(这段是在系统自动为我们生成的文件的内容)

	 @Override
            public boolean beMyGirlFriend(int FaceScore, boolean isCurSingle) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(FaceScore);
                    _data.writeInt(((isCurSingle) ? (1) : (0)));
                    mRemote.transact(Stub.TRANSACTION_beMyGirlFriend, _data, _reply, 0);
                    _reply.readException();
                    _result = (0 != _reply.readInt());
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
 背后是由binder的transact函数去执行的。在开头的图片已经有指出，这个Binder的具体实现是BinderProxy类，他就在Binder.java里面。

	public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }

他会去调用native的`transactNative(code, data, reply, flags);`背后实现我们就不深究，直接到结果：

下面代码在在Binder.java类文件里面，是被调用的类的Binder实现。

	// Entry point from android_util_Binder.cpp's onTransact 
    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        // theoretically, we should call transact, which will call onTransact,
        // but all that does is rewind it, and we just got these from an IPC,
        // so we'll just call it directly.
        boolean res;
        // Log any exceptions as warnings, don't silently suppress them.
        // If the call was FLAG_ONEWAY then these exceptions disappear into the ether.
        res = onTransact(code, data, reply, flags); // <-重要的一句
        
        ....
 
        checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
        reply.recycle();
        data.recycle();

        // Just in case -- we are done with the IPC, so there should be no more strict
        // mode violations that have gathered for this thread.  Either they have been
        // parceled and are now in transport off to the caller, or we are returning back
        // to the main transaction loop to wait for another incoming transaction.  Either
        // way, strict mode begone!
        StrictMode.clearGatheredViolations();

        return res;
    }

上面的onTransact函数，会跳到我们自己实现的接口的内容去，具体在我们的电脑会自动帮我们生成的类文件里面。这样就回到了我们**Server**的onTransact()里面去调用我们的方法。 

	@Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_beMyGirlFriend: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    boolean _arg1;
                    _arg1 = (0 != data.readInt());
                    boolean _result = this.beMyGirlFriend(_arg0, _arg1);
                    reply.writeNoException();
                    reply.writeInt(((_result) ? (1) : (0)));
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

最后通过保存结果到reply中去。这里的replay“等同于”我们Client里面调用的那个reply。这个Parcel类就是前篇文章提到的保存调用信息的工具。

关于bindeService()的调用，我们在介绍[四大金刚之Service](http://sanjay-f.github.io/2015/12/18/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%977---%E5%9B%9B%E5%A4%A7%E9%87%91%E5%88%9A%E4%B9%8BService/)的时候有提到过。就不再细看了。我们直接看结果吧。


# 框架的学习

其实，说实在的，真的意义可能不是很大，完全只是知道一个流程，一时半会也无法做些什么实践。
也不能基于此做些修改，突破性能之类的。可能下次你要比较性能时候，你清楚如果背后是Binder，那么性能可能稍微弱些，毕竟人家跨进程通讯，一般比不上本地通讯，就像出国游和去隔壁城市一样。或者说类似本地广播和BroadCast。

但还有一个堂而皇之的理由，就是本节的标题，框架的学习...一个案例就是用在搭建网络框架部分。
大致意思如下图

 ![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/binder_Binder_%E6%A1%86%E6%9E%B6%E7%9A%84%E8%A1%8D%E7%94%9F%20%283%29.png)

我们定义了一个代理的请求`ProxyRequest`，他是我们调用的对象，由这个对象来保存我们所有的请求信息。之后这个对象跑到我们的`RequestManager`去，它根据这些请求信息，来装换成实际的请求类（如Volley的 JsonRequest ），再发给我们实际处理请求的类(如HttpClient，Volley，OkHttp)去处理。

通过这样的在实际的网络框架上加多一层`包装层`，我们可以让我们的调用者不知道背后的`干活人`，达到某种程度的解耦，以后当我们需要换一个网络请求框架的时候，只需要修改下在RequestManager层的转换函数，而且我们对所有的请求也可以做统一的管理，这是很重要的！控制性能，对请求结果的统一处理等。



# 后记

大致的写了java层的Binder框架，感觉写得很不够到位，在深入底层和得其意上没做到好的平衡，不过也还好，还可以持续改，先起个头，后面也能继续优化，这就是开发的一些好处吧，可以持续的改进，而且没太高的成本。不像一些科学实验，想再重现，改进的成本太高了。例如Musk的飞天火箭，每次的实验成功都是天文数字。
我给自己个台阶下了   ( — . —  ) "
