title: Flutter入门系列10---关于状态管理BLoC

date: 2018-04-10 23:14:46

tags: [android,Flutter,BLoC]

categories: flutter

------------------------------------------


做过web的同学应该速度React+Redux这个东西，到了Flutter，我们也需要有个东西来做不同页面的跳转逻辑，数据的分发管理等问题。

因为一个程序大了，界面很多的，逻辑很复杂，这时候需要有个东西来做管理啊

目前我们工程有尝试用到了[BLoC](https://hk.saowen.com/a/fbb6e484de022173fe85248875286060ce40d069c97420bc0be49d838e19e372)这个东西，在这里做简单介绍下


BLoC的全称是 业务逻辑组件（Business Logic Component）。
如果你对响应式编程熟悉的话，应该很好懂这个到底是什么

就是用reactive programming方式构建应用，一个由流构成的完全异步的世界。从而达到界面与业务分离的逻辑.

或者简单粗暴的理解为，这个就是类似安卓的RX事件总线。
 

<!--more-->

# 起航

  


便于说明，我们先写个简单的demo，介绍如何基于由Stream提供的数据构建Widget

	import 'dart:async';
	import 'package:flutter/material.dart';
	
	class CounterPage extends StatefulWidget {
	  @override
	  _CounterPageState createState() => _CounterPageState();
	}
	
	class _CounterPageState extends State<CounterPage> {
	  int _counter = 0;
	  final StreamController<int> _streamController = StreamController<int>();
	
	  @override
	  void dispose(){
	    _streamController.close();
	    super.dispose();
	  }
	
	  @override
	  Widget build(BuildContext context) {
	    return Scaffold(
	      appBar: AppBar(title: Text('Stream version of the Counter App')),
	      body: Center(
	        child: StreamBuilder<int>(
			  //当有新的数据，会刷新页面
	          stream: _streamController.stream,
	          initialData: _counter,
	          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
	            return Text('You hit me: ${snapshot.data} times');
	          }
	        ),
	      ),
	      floatingActionButton: FloatingActionButton(
	        child: const Icon(Icons.add),
	        onPressed: (){
				//点击时候，我们往Stream扔数据
	          _streamController.sink.add(++_counter);
	        },
	      ),
	    );
	  }
	}

上面利用stream的方式，更改了我们之前需要用setState去刷新的逻辑。
即 StreamBuilder监听Stream，每当某些数据输出Stream时，它会自动重建，调用其builder回调。


	StreamBuilder<T>(
	    key: ...optional, the unique ID of this Widget...
	    stream: ...the stream to listen to...
	    initialData: ...any initial data, in case the stream would initially be empty...
	    builder: (BuildContext context, AsyncSnapshot<T> snapshot){
	        if (snapshot.hasData){
	            return ...the Widget to be built based on snapshot.data
	        }
	        return ...the Widget to be built if no data is available
	    },
	)

通过以上的方式，不同的组件widget也可以去做只关心数据的展示和处理逻辑。

看到这里，你是否也有当年一开始用EventBus那样的恐惧，一开始开心，后面事件满天飞，开始害怕了？

# BLoc

接下来正式介绍BLoC




![](https://i.imgur.com/xt3xilA.png)


BLoC模式由来自Google的Paolo Soares和Cong Hui设计，并在2018年DartConf期间（2018年1月23日至24日）首次展示。 在YouTube上观看此视频。

 

简而言之，业务逻辑（Business Logic ）需要：

- 转移到一个或几个BLoC，
- 尽可能从表现层中删除。 换句话说，UI组件应该只关心UI事物而不关心业务，
- 依赖Streams独家使用输入（Sink）和输出（流），
- 保持平台独立，
- 保持环境独立。

事实上，BLoC模式最初被设想为允许独立于平台重用相同的代码：Web应用程序，移动应用程序，后端。



我们看回上面的图，就是像事件总线一样来做解耦工作

1. Widgets通过Sinks向BLoC发送事件，
2. BLoC通过Stream通知Widgets，
3. 由BLoC实现的业务逻辑不是他们关注的问题。

从上面来看，我们可以直接看到使用BLoC的一个巨大的好处。
业务逻辑与UI的分离：

- 我们可以随时更改业务逻辑，对应用程序的影响最小，
- 我们可能会更改UI而不会对业务逻辑产生任何影响，
- 现在，测试业务逻辑变得更加容易。



现在我们来现在，基于BLoC来改造前面的点击按钮事件

	void main() => runApp(new MyApp());
	
	class MyApp extends StatelessWidget {
	  @override
	  Widget build(BuildContext context) {
	    return new MaterialApp(
	        title: 'Streams Demo',
	        theme: new ThemeData(
	          primarySwatch: Colors.blue,
	        ),
	        home: BlocProvider<IncrementBloc>(
	          bloc: IncrementBloc(),
	          child: CounterPage(),
	        ),
	    );
	  }
	}
	
	class CounterPage extends StatelessWidget {
	  @override
	  Widget build(BuildContext context) {
		//职责分析
	    final IncrementBloc bloc = BlocProvider.of<IncrementBloc>(context);
	
	    return Scaffold(
	      appBar: AppBar(title: Text('Stream version of the Counter App')),
	      body: Center(
	        child: StreamBuilder<int>(
	          stream: bloc.outCounter,
	          initialData: 0,
	          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
	            return Text('You hit me: ${snapshot.data} times');
	          }
	        ),
	      ),
	      floatingActionButton: FloatingActionButton(
	        child: const Icon(Icons.add),
	        onPressed: (){
	          bloc.incrementCounter.add(null);
	        },
	      ),
	    );
	  }
	}
	
	class IncrementBloc implements BlocBase {
	  int _counter;
	
	  //
	  // Stream to handle the counter
	  //
	  StreamController<int> _counterController = StreamController<int>();
	  StreamSink<int> get _inAdd => _counterController.sink;
	  Stream<int> get outCounter => _counterController.stream;
	
	  //
	  // Stream to handle the action on the counter
	  //
	  StreamController _actionController = StreamController();
	  StreamSink get incrementCounter => _actionController.sink;
	
	  //
	  // Constructor
	  //
	  IncrementBloc(){
	    _counter = 0;
	    _actionController.stream
	                     .listen(_handleLogic);
	  }
	
	  void dispose(){
	    _actionController.close();
	    _counterController.close();
	  }
	
	  void _handleLogic(data){
	    _counter = _counter + 1;
	    _inAdd.add(_counter);
	  }
	}

看我这部分demo你可能第一反应是，至于吗，怎么导致代码变庞大了。我也是这么一个第一感受的，不过就像ＲｘＪａｖａ一样，其本身不是来做缩短代码的，而是框架性的去做解耦功能的。

做技术选型时候，我们会去做很累是否有必要引入这么强大的功能库，类似ｗｅｂ去加入`React+Redux`功能一样，对于小型简单几个页面的，这样反而适得其反，但对于中大型的app来说，是有必要的。

如果我说，上面代码还不是全部。。的话


但还是得说有必要的，我们来分析看下代码：

回去看CounterPage里面的内容

1. 责任分离 。
 你会发现Build其中绝对没有任何业务逻辑。此页面现在仅负责：
	- 显示计数器，现在只在必要时刷新（即使页面不必知道）
	- 提供按钮，当按钮按下时，将会在counter面板上请求一个动作
	- 此外，整个业务逻辑集中在一个单独的类“IncrementBloc”中。
现在如果你需要更改业务逻辑，您只需更新方法**_handleLogic**。 也许新的业务逻辑会要求做非常复杂的事情但CounterPage永远不会知道它，这非常好！

2. 自由组织布局:   
由于使用了Streams，你现在可以独立于业务逻辑组织布局。
可以从应用程序中的任何位置启动任何操作：只需调用.incrementCounter sink即可。
您可以在任何页面的任何位置显示counter，只需听取.outCounter stream。 

3. 减少“build”的数量  
不使用setState()而是使用StreamBuilder大大减少了“build”的数量。
从性能角度来看，这是一个巨大的进步。

##  BLoc 的传递

当然也不能说它完美无瑕，现在就有一个基础问题，跨不同组件等，需要拿到这个controller，好塞数据，监听。
有几种方法可以访问它：

1. 通过全局单例   
这种方式可以实现，但不是真的推荐。 此外，由于Dart中没有类析构函数，因此你永远无法正确释放资源。
2. 作为局部变量   
你可以实例化BLoC的局部实例。 在某些情况下，此解决方案完全符合某些需求。 在这种情况下，你应该始终考虑在StatefulWidget中初始化，以便您 
可以利用dispose()方法来释放相关资源。
3. 由父级提供  
使其可访问的最常见方式是通过父级Widget访问，通过StatefulWidget实现。


现在我们手把手的来看下demo，通过**BlocProvider**的方式来做。这个也是开头demo的案例使用的方式

	// Generic Interface for all BLoCs
	abstract class BlocBase {
	  void dispose();
	}
	
	// Generic BLoC provider
	class BlocProvider<T extends BlocBase> extends StatefulWidget {
	  BlocProvider({
	    Key key,
	    @required this.child,
	    @required this.bloc,
	  }): super(key: key);
	
	  final T bloc;
	  final Widget child;
	
	  @override
	  _BlocProviderState<T> createState() => _BlocProviderState<T>();
	
	  static T of<T extends BlocBase>(BuildContext context){
	    final type = _typeOf<BlocProvider<T>>();
	    BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
	    return provider.bloc;
	  }
	
	  static Type _typeOf<T>() => T;
	}
	
	class _BlocProviderState<T> extends State<BlocProvider<BlocBase>>{
	  @override
	  void dispose(){
	    widget.bloc.dispose();
	    super.dispose();
	  }
	
	  @override
	  Widget build(BuildContext context){
	    return widget.child;
	  }
	}

以上的代码写好后，我们通过这样的方式来调用

	 home: BlocProvider<IncrementBloc>(
	          bloc: IncrementBloc(),
	          child: CounterPage(),
	        ),
  
通过这些代码，我们只需实例化一个新的BlocProvider，它将处理一个IncrementBloc，并将CounterPage作为子项呈现。

这样从BlocProvider开始的子树的任何Widget都将能够通过以代码访问IncrementBloc：

	IncrementBloc bloc = BlocProvider.of<IncrementBloc>(context);


## 可以使用多个BLoC吗？
当然，这是非常可取的。建议如下：

- （如果有任何业务逻辑）每个页面的顶部有一个BLoC，我认为这个就像每个具体业务逻辑有一个Context一样
- 为什么不是ApplicationBloc来处理应用程序状态？
- 每个“足够复杂的组件”都有相应的BLoC。


  以下示例代码在整个应用程序的顶部显示ApplicationBloc，然后在CounterPage顶部显示IncrementBloc。
该示例还显示了如何检索两个bloc。

	void main() => runApp(
	  BlocProvider<ApplicationBloc>(
	    bloc: ApplicationBloc(), //注意这里加入了一个新的app级别
	    child: MyApp(),
	  )
	);
	
	class MyApp extends StatelessWidget {
	  @override
	  Widget build(BuildContext context){
	    return MaterialApp(
	      title: 'Streams Demo',
	      home: BlocProvider<IncrementBloc>( // widget级别
	        bloc: IncrementBloc(),
	        child: CounterPage(),
	      ),
	    );
	  }
	}
	
	class CounterPage extends StatelessWidget {
	  @override
	  Widget build(BuildContext context){
		//检索两个bloc
	    final IncrementBloc counterBloc = BlocProvider.of<IncrementBloc>(context);
	    final ApplicationBloc appBloc = BlocProvider.of<ApplicationBloc>(context);
	    
	    ...
	  }
	}

# 实战使用建议

## 为什么不使用InheritedWidget？

在与BLoC相关的大多数文章中，你会看到通过InheritedWidget实现Provider。
当然，没有什么能阻止这种类型的实现。 然而，

- dispose     
一个InheritedWidget没有提供任何dispose方法，请记住，在不再需要资源时总是释放资源是一种很好的做法。

- 当然，没有什么能阻止你将InheritedWidget包装在另一个StatefulWidget中，但是，使用InheritedWidget增加了什么呢？
- 最后，如果不受控制，使用InheritedWidget经常会导致副作用（请参阅下面的InheritedWidget上的Reminder）。

这三点解释了我为什么选择通过StatefulWidget实现BlocProvider，这样做可以让我在Widget dispose时释放相关资源。
## Flutter无法实例化泛型类型

不幸的是，Flutter无法实例化泛型类型，我们必须将BLoC的实例传递给BlocProvider。 为了在每个BLoC中强制执行dispose()方法，所有BLoC都必
须实现BlocBase接口。


## 这个不是为Flutter而生的库，技术选型要谨慎

> At first, the BLoC pattern was conceived to share the very same code across platforms (AngularDart, …) and, in this perspective, that statement makes full sense.
> 
> However, if you only intend to develop a Flutter application, this is, based on my humble experience, a little bit overkill.
> 
> If we stick to the statement, no getter or setter are possible, only sinks and streams. The drawback is “all this is asynchronous”.

作者认为，如果你只是拿这个来纯粹做个flutter，有点杀猪用牛刀的意味，大材小用了。。


提供了两个缺点的地方：


1. 你需要从BLoC中检索一些数据，以便使用这些数据作为应该立即显示这些参数的页面的输入（例如，想一个参数页面），如果我们不得不依赖Streams，这会使构建异步页面（很复杂）。通过Streams使其工作的示例代码可能如下所示......丑陋不是它。


		class FiltersPage extends StatefulWidget {
		  @override
		  FiltersPageState createState() => FiltersPageState();
		}
		
		class FiltersPageState extends State<FiltersPage> {
		  MovieCatalogBloc _movieBloc;
		  double _minReleaseDate;
		  double _maxReleaseDate;
		  MovieGenre _movieGenre;
		  bool _isInit = false;
		
		  @override
		  void didChangeDependencies() {
		    super.didChangeDependencies();
		
		    // As the context of not yet available at initState() level,
		    // if not yet initialized, we get the list of the 
		    // filter parameters
		    if (_isInit == false){
		      _movieBloc = BlocProvider.of<MovieCatalogBloc>(context);
		      _getFilterParameters();
		    }
		  }
		
		  @override
		  Widget build(BuildContext context) {
		    return _isInit == false
		      ? Container()
		      : Scaffold(
		    ...
		    );
		  }
		
		  ///
		  /// Very tricky.
		  /// 
		  /// As we want to be 100% BLoC compliant, we need to retrieve
		  /// everything from the BLoCs, using Streams...
		  /// 
		  /// This is ugly but to be considered as a study case.
		  ///
		  void _getFilterParameters() {
		    StreamSubscription subscriptionFilters;
		
		    subscriptionFilters = _movieBloc.outFilters.listen((MovieFilters filters) {
		        _minReleaseDate = filters.minReleaseDate.toDouble();
		        _maxReleaseDate = filters.maxReleaseDate.toDouble();
		
		        // Simply to make sure the subscriptions are released
		        subscriptionFilters.cancel();
		        
		        // Now that we have all parameters, we may build the actual page
		        if (mounted){
		          setState((){
		            _isInit = true;
		          });
		        }
		      });
		    });
		  }
		}

2. 在BLoC级别，您还需要转换某些数据的“假”注入，以触发提供您希望通过流接收的数据。使这项工作的示例代码可以是：


		
		class ApplicationBloc implements BlocBase {
		  ///
		  /// Synchronous Stream to handle the provision of the movie genres
		  ///
		  StreamController<List<MovieGenre>> _syncController = StreamController<List<MovieGenre>>.broadcast();
		  Stream<List<MovieGenre>> get outMovieGenres => _syncController.stream;
		
		  ///
		  /// Stream to handle a fake command to trigger the provision of the list of MovieGenres via a Stream
		  ///
		  StreamController<List<MovieGenre>> _cmdController = StreamController<List<MovieGenre>>.broadcast();
		  StreamSink get getMovieGenres => _cmdController.sink;
		
		  ApplicationBloc() {
		    //
		    // If we receive any data via this sink, we simply provide the list of MovieGenre to the output stream
		    //
		    _cmdController.stream.listen((_){
		      _syncController.sink.add(UnmodifiableListView<MovieGenre>(_genresList.genres));
		    });
		  }
		
		  void dispose(){
		    _syncController.close();
		    _cmdController.close();
		  }
		
		  MovieGenresList _genresList;
		}
		
		// Example of external call
		BlocProvider.of<ApplicationBloc>(context).getMovieGenres.add(null);


就原文作者而言，如果我没有任何与**代码移植/共享相关**的限制，他发现这太笨重了，他宁愿在需要时使用常规的getter / setter并使用Streams / Sinks来保持分离责任并在需要的地方广播信息，这比搞BLoC棒多了。

 


# 最后

项目的[代码地址](https://github.com/boeledi/Streams-Block-Reactive-Programming-in-Flutter)在这里。
这篇文章是基于[这篇](https://www.didierboelens.com/2018/08/reactive-programming---streams---bloc/?spm=a2c4e.11153940.blogcont640862.7.35fa2130JgD9aY)基础上改出来的。
#ref 

- https://hk.saowen.com/a/fbb6e484de022173fe85248875286060ce40d069c97420bc0be49d838e19e372

- https://cloud.tencent.com/developer/article/1345645

- https://www.didierboelens.com/2018/08/reactive-programming---streams---bloc/?spm=a2c4e.11153940.blogcont640862.7.35fa2130JgD9aY

  