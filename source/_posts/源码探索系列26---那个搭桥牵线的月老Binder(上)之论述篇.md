title: 源码探索系列26---那个搭桥牵线的月老Binder(上)之论述篇
date: 2016-03-07 14:47:46
tags: [android,源码,Binder]
categories: android

------------------------------------------

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/binder_201109131125266418.jpg)
 
Binder，大名鼎鼎，有了他，我们才能和另外一个陌生人（进程）发生沟通啊.
那么这个搭桥牵绳的中间人---月老Binder，到底是怎么做到的呢？
我们先来简单的了解下，然后再深入的探索。
篇幅会比较长，所以分了上中下篇，上篇简述基本的原理，下篇尝试从java层和Native层源码的角度来理解。  
好了，那么我们就开始说下这个月老Binder的故事吧！

<!--more--> 

# 起航
API:23

## 什么是跨进程通讯（IPC）以及为何要

我们知道Android是基于Linux内核的。不同的进程间是存在一个隔离的，就像动物间存在生殖隔离一样，彼此那是真的"老死不相往来"，和老派的英国绅士一样，是不会主动发生往来的。做这种`进程隔离` 是有目的的，他可以避免进程A写入进程B的情况发生。 进程的隔离实现，使用了虚拟地址空间。进程A的虚拟地址和进程B的虚拟地址不同，这样就防止进程A将数据信息写入进程B。

	简单的说 这种进程隔离技术 保护 操作系统中 进程 互不干扰。

因此，不同进程间的数据不共享 且 不会察觉到还有其他进程的存在。
但实际开发中，我们还是需要和别的进程做点沟通的（例如那些全家桶app们），因此我们需要一种机制来做这事。


## 如何通讯

显然，我们的程序是运行在用户空间（User space）的，如果一个用户空间想与另外一个用户空间进行通信得怎么做呢？ 显然他们需要一个支持他们通讯的东西，而这东西在Linux下，需要操作系统内核支持。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/binder_Linux_os_.png)
> Linux下，用户空间访问内核空间的唯一方式就是系统调用；	
 Linux Kernel 掌管所有，像这种小事人家肯定可以做到。	
  
Linux 内核原生支持的通信机制有Socket，System V，管道等。不过这些都满足不了谷歌对**性能和安全**的需求，因此他们自己弄了这个Binder机制，由于需要运行在内核，所以写了个**Binder驱动模块**并用[可加载核心模块技术（Loadable Kernel Module，LKM）  ](https://zh.wikipedia.org/wiki/%E5%8F%AF%E8%BC%89%E5%85%A5%E6%A0%B8%E5%BF%83%E6%A8%A1%E7%B5%84)把这个不是Linux原生的给安上，因为在运行时它会被连接到内核，运行在内核空间。这样有了这个 Binder驱动内应，我们安卓就可以根据需要，定制接口，完成通讯。
（有驱动，就有对应的硬件咯？有的，不过是一个虚拟的硬件，就像我们用Ultraiso/Nero这些刻录软件一样会有一个虚拟的光驱。 这个虚拟的binder硬件在“/dev/binder”）

大致像下图:

![Binder关系图](http://7xl9zd.com1.z0.glb.clouddn.com/binder_binder_driver.png)
 
 这个Binder相对出传统的Socket方式，**传输效率高，开销小**。 Socket则是主要用在跨网络的进程间通信和本机上进程间的低速通信情景下的，不符合手机内程序通讯的场景。
 
 另外，传统的进程通信方式对于通信双方的身份并**没有做出严格的验证**，只有在上层协议上进行架设；比如Socket通信ip地址是客户端手动填入的，都可以进行伪造；而Binder机制从协议本身就支持对通信双方做**身份校检**，因而大大提升了安全性（这个在后面我们写java的Binder时候提到）。这个也是Android权限模型的基础。（更具体的[Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)）

 

## 通讯流程
 
Binder框架定义了四个角色：**Server**，**Client**，**ServiceManager**（以后简称SM）以及**Binder驱动**。这四个角色的关系和互联网类似：Server是服务器，Client是客户终端，SM是域名服务器（DNS），Binder驱动是路由器。 
 
所以我们可以有这么一个大致的流程：

	server通过Binder向SM注册自己，
	接着我们的client想与server通讯的话，通过binder找SM要联系信息，
	有这个信息，Client就可以建立起连接，开始通讯。   
  如果你看过`EventBus`的代码，那么上面的整个流程你应该不陌生。
  
为了讨论的方便，老套路的先用AIDL(Android Interface Definition Language)写个demo先。如果你很清楚，可以跳过这部分，看下一步的流程分析 。

下面的代码简单的演示client跨进程的调用RemoteServer的beMyGirlFriend()接口。

	interface IDemoAidlInterface {
    /**
     *根据传送的颜值和是否单身两个条件，看是否能做女朋友 ^_^
     *true为可以，false为条件不符合
     */     
	 boolean beMyGirlFriend(in int FaceScore,in boolean isCurSingle);             
	}

嗯，我们写好接口了。接着是我们的服务

	public class RemoteService extends Service {

	    public RemoteService() {
	    }
	    	
	    private Binder mBinder = new IDemoAidlInterface.Stub() {
	        @Override
	        public boolean beMyGirlFriend(int FaceScore, boolean isCurSingle) 
	        throws RemoteException {
	            return FaceScore > 95 && isCurSingle;
	        }
	    };
	
	    @Override
	    public IBinder onBind(Intent intent) {
	        return  mBinder;
	    }
	}
	
然后是我们的Client 客户端的使用

	public class MainActivity extends AppCompatActivity {

	    private String TAG = MainActivity.class.getSimpleName();
	    private ServiceConnection mConnection = new ServiceConnection() {
	        @Override
	        public void onServiceConnected(ComponentName name, IBinder service) {	
	            Log.e(TAG, "onServiceConnected: ");
	            IDemoAidlInterface demoAidlInterface = IDemoAidlInterface.Stub.
											            asInterface(service);
	            try {
	                boolean relation=demoAidlInterface.beMyGirlFriend(666666, true);
	                Log.e(TAG,"can?="+relation );
	            } catch (RemoteException e) {
	                e.printStackTrace();
	            }
	        }
	
	        @Override
	        public void onServiceDisconnected(ComponentName name) {
	            Log.e(TAG, "onServiceDisconnected: ");
	        }
	    };
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        bindService(new Intent(this, RemoteService.class),
				         mConnection, BIND_AUTO_CREATE);
	    }
	
	    @Override
	    protected void onDestroy() {
	        unbindService(mConnection);
	        super.onDestroy();
	    }
	}


### 流程分析---通信模型

好了，有上的代码背景，我们继续分析

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/binder_binder%E9%80%9A%E8%AE%AF%E6%B5%81%E7%A8%8B%20%281%29.png)
 
1. 我们的Server同SM注册自己；
主要是告诉SM，在我这个进程，有能执行一系列方法（beMyGirlFriend）的对象（IDemoAidlInterface）的信息。而SM自己内部会维护一张表格，保存这些信息，供以后查询用。
2. 我们的Client同SM查询
   我们需要找那个刚注册的进程对象（RemoteService）；但这里有一个细节，我们的请求在经过Binder驱动时候，他会做一些处理。它并没给Client进程返回一个真正的对象，而是返回一个代理对象（ProxyObjec），虽然这个代理对象具有真实对象的所有方法，但它是不干活的，他只是保存请求数据，然后再交给实际干活的，由他去执行。(这里返回的IDemoAidlInterface就是这样啦)
	   
	      IDemoAidlInterface demoAidlInterface = IDemoAidlInterface.Stub.
												            asInterface(service);											            
虽然我们知道是假的，是一个代理的ProxyObject，但Client不知道，反正它关心的还是这次调用能不能做到自己要的就好了。

3. 调用方法
 在Client调用了代理对象的方法后，驱动就会收到这个请求消息，他自己查表，知道是哪个Server，接着把请求的参数打包给Server，server进程在执行好对应的方法后，再把结果返回给驱动。驱动再返回给Client；这样就完成了一次调用。

通过上面的分析，我们知道Binder对象**实际 并不是 能进行跨进程传递的对象**。虽然在我们的Client有他的存在，而且真的能调用方法，返回个结果的。但只是看起来像是这样子。实际上我们的Client只是对代理的操纵罢了。
看到这里，我相信你对这个中间人月老Binder有了一定的了解了。

写到这类，让我想起一开始 在写源码探索系列的**四大金刚之BroadCast**时候，我写到的类似内容。不过那篇文章被CSDN吃了。到现在没心再写。下次有空再补吧.... 

### 代码分析
好了，有了上面的铺垫，我们可以先来看下我们系统给我们自动生成的Binder的内容了。当然你熟悉后完全可以手动。
(文件在你的项目 build/generated/source/aidl内)
 

	public interface IDemoAidlInterface extends android.os.IInterface {
	    /**
	     * Local-side IPC implementation stub class.
	     */
	    public static abstract class Stub extends android.os.Binder implements 
	    com.example.sanjay.binderdemo.IDemoAidlInterface {
	    
	        private static final java.lang.String DESCRIPTOR =
		         "com.example.sanjay.binderdemo.IDemoAidlInterface";
	
	        /**
	         * Construct the stub at attach it to the interface.
	         */
	        public Stub() {
	            this.attachInterface(this, DESCRIPTOR);
	        }
	
	
	        /**
	         * Cast an IBinder object into an IDemoAidlInterface interface,
	         * generating a proxy if needed.
	         */
	        public static com.example.sanjay.binderdemo.IDemoAidlInterface asInterface(android.os.IBinder obj) {
	            if ((obj == null)) {
	                return null;
	            }
	            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	            if (((iin != null) && (iin instanceof com.example.sanjay.binderdemo.IDemoAidlInterface))) {
	                return ((com.example.sanjay.binderdemo.IDemoAidlInterface) iin);
	            }
	            return new com.example.sanjay.binderdemo.IDemoAidlInterface.Stub.Proxy(obj);
	        }
     	   //IDemoAidlInterface.Stub.asInterface(service);
	      //参数的这个`IBinder`类Obj就是驱动给我们的。
	      //对于Binder的访问，如果是在同一个进程，即实际并不需要跨进程，那么直接返回原始的Binder实体；
	      // 如果在不同进程，那么就给他一个代理对象 ，让这个代理实现对远程对象的访问。 
	      
	      
	        @Override
	        public android.os.IBinder asBinder() {
	            return this;
	        }
 	
               //如果是远程调用，处在不同的进程中。则用Proxy里面的方法
	        private static class Proxy implements com.example.sanjay.binderdemo.
	        IDemoAidlInterface {
	            private android.os.IBinder mRemote;
	
	            Proxy(android.os.IBinder remote) {
	                mRemote = remote;
	            }
	
	            @Override
	            public android.os.IBinder asBinder() {
	                return mRemote;
	            }
	
	            public java.lang.String getInterfaceDescriptor() {
	                return DESCRIPTOR;
	            }
	
	            /**
	             * 根据传送的颜值和是否单身两个条件，看是否能做女朋友 ^_^
	             * true为可以，false为条件不符合
	             */
	            @Override
	            public boolean beMyGirlFriend(int FaceScore, boolean isCurSingle)
	             throws android.os.RemoteException {
	                android.os.Parcel _data = android.os.Parcel.obtain();
	                android.os.Parcel _reply = android.os.Parcel.obtain();
	                boolean _result;
	                try {
	                    _data.writeInterfaceToken(DESCRIPTOR);
	                    _data.writeInt(FaceScore);
	                    _data.writeInt(((isCurSingle) ? (1) : (0))); 
	                    mRemote.transact(Stub.TRANSACTION_beMyGirlFriend, _data, _reply, 0);
	                    //这个mRemote是在前面调用asInterface时候初始化的，所以它就是BinderProxy；
	                    //而这个transact是个native方法。他最终会去和我们的Driver通讯。
	                    //驱动去唤醒Server进程，调用Server进程本地对象的`onTransact`函数。
	                    //而这个过程，调用这个方法的Client**线程会被挂起**，等待返回，
	                    //因此如果**你是在UI线程调用的话，会有可能导致ANR！！！**，
	                    //所以最好是开个线程，然后再用Handler处理下。
	                    //而且这方法本身有可能很耗时，处理时间长。	                    

	                    _reply.readException();
	                    _result = (0 != _reply.readInt());
	                } finally {
	                    _reply.recycle();
	                    _data.recycle();
	                }
	                return _result;
	            }
	        }
	
	        static final int TRANSACTION_beMyGirlFriend = 
							        (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
	    }
	
 
	        @Override
	        public boolean onTransact(int code, Parcel data, Parcel reply,int flags) 
	        throws android.os.RemoteException {
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
	               // 根据我们在proxy的Transact的Stub.TRANSACTION_beMyGirlFriend的code
               		// 运行到这里，然后调用本地对象的方法，处理结果返回给驱动
               		// 驱动唤醒挂起的Client进程里面的线程并将结果返回。
               		//到此，一次跨进程调用就完美结束啦。
	                    return true;
	                }
	            }
	            return super.onTransact(code, data, reply, flags);
	        }

	    /**
	     * 根据传送的颜值和是否单身两个条件，看是否能做女朋友 ^_^
	     * true为可以，false为条件不符合
	     */
	    public boolean beMyGirlFriend(int FaceScore, boolean isCurSingle) 
	    throws android.os.RemoteException;
	}


## 什么是Binder

等等，为何这里突然说起，设么是binder起来了。
看了上面那么多个叫Binder的名字出现，不知道你有没在知与不知中，悄悄搞混了些什么，所以现在来清理下这个概念。
 
*  通常我们对话中提到的Binder指的是一种**通信机制**；我们说AIDL使用Binder进行通信，指的就是Binder这种IPC机制本身。
* 对于**Server**进程来说，Binder指的是**Binder本地对象**
* 对于**Client**来说，Binder指的是**Binder代理对象**，它只是Binder本地对象的一个远程代理；对这个Binder代理对象的操作，会通过驱动最终转发到Binder本地对象上去完成；对于一个拥有Binder对象的使用者而言，它无须关心这是一个Binder代理对象还是Binder本地对象；对于代理对象的操作和对本地对象的操作对它来说没有区别，要的是结果。
* 对于传输过程而言，Binder是可以进行跨进程传递的对象；
Binder驱动会对具有跨进程传递能力的对象做特殊处理：自动完成代理对象和本地对象的转换。
因为在驱动中，在Binder对象进行跨进程传递的时候，Binder驱动会自动完成这Server和Client的两种Binder类型的转换；因此 Binder本地对象的代表是一个叫做`binder_node`的数据结构，Binder代理对象是用`binder_ref`代表的；有的地方把Binder本地对象直接称作Binder实体，把Binder代理对象直接称作Binder引用（句柄），其实指的是Binder对象在驱动里面的表现形式；明白意思即可。


> 面向对象思想的引入将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体（本地对象）位于一个进程中，而它的引用（代理对象）却遍布于系统的各个进程之中。最诱人的是，这个引用和java里引用一样既可以是强类型，也可以是弱类型，而且可以从一个进程传给其它进程，让大家都能访问同一Server，就象将一个对象或引用赋值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。形形色色的Binder对象以及星罗棋布的引用仿佛粘接各个应用程序的胶水，这也是Binder在英文里的原意。也是为何我起它叫搭桥牵线的月老的意思。
 

## Service Manager

 在前面的讨论中，我们对SM的描述都是一笔带过的，

现在在结尾来说下他，其实在我们建立通讯，让我们的Server向SM注册前，SM本身也是要建立的 。
首先有**一个进程**向驱动提出**申请为SM**；驱动**同意**之后，SM进程负责管理Service（注意这里是Service而不是Server，因为如果通信过程反过来的话，那么原来的客户端Client也会成为服务端Server）。

那么Service Manager是如何成为一个守护进程的？ 即Service Manager是如何告知Binder驱动程序它是Binder机制的上下文管理者呢。

欢迎查看老罗的这篇文章
[浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566) 
(¬_¬)
 

 
### 深入理解Java层的Binder
IBinder/IInterface/Binder/BinderProxy/Stub

我们使用AIDL接口的时候，经常会接触到这些类，那么这每个类代表的是什么呢？

* **IBinder是一个接口，它代表了一种跨进程传输的能力**；
只要实现了这个接口，就能将这个对象进行跨进程传递（但我们知道实际并不能）；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。
 
* IBinder负责数据传输，那么client与server端的调用契约（这里不用接口避免混淆）呢？
这里的IInterface代表的就是远程server对象具有什么能力。具体来说，就是aidl里面的接口。
 
* **Java层的Binder类，代表的其实就是Binder本地对象**。
BinderProxy类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。

* 在使用AIDL的时候，编译工具会给我们生成一个Stub的**静态内部类**；
这个类继承了Binder, 说明它是一个Binder本地对象，它实现了IInterface接口，表明它具有远程Server承诺给Client的能力；Stub是一个抽象类，具体的IInterface的相关实现需要我们手动完成，这里使用了**策略模式**。


# 后记

关于binder的文章已经很多，很全了，完全没必要多我这一篇。
还写下来，只是自己尝试总结下，加深理解。通篇写下来，发现还是有不少自己不清楚的地方，需要自己加强的。有种武侠小说中提到的一句话：**“  油尽灯枯还强行谷催 ”**。硬着臉皮寫這麼高難度的文章。
这个[google canvas](https://docs.google.com/drawings)还真不错。用起来比PPT作图好用！
而且是页面版，不用安装，就是要翻墙。在这里推荐下

 

# 参考资料

1. 《深入理解Android》邓凡平
1. [Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363) ，老罗哥的一篇文章
2.  [Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)，这哥们不知何许人也，没写几篇文章，不过一篇永流传啊。
3.   [Wiki的进程隔离介绍](https://zh.wikipedia.org/wiki/%E8%BF%9B%E7%A8%8B%E9%9A%94%E7%A6%BB)  
4. [Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)
5. [浅谈Service Manager成为Android进程间通信（IPC）机制Binder守护进程之路](http://blog.csdn.net/luoshengyang/article/details/6621566)  

