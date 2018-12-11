title: Flutter入门系列11---关于状态state本身

date: 2018-04-28 23:14:46

tags: [android,Flutter,BLoC]

categories: flutter

------------------------------------------


做过web的同学应该也知道，我们React的组件是有生命周期的，

![](https://i.imgur.com/715G2lI.png)

这个Flutter的widget也是有生命周期的
 
<!--more-->

# 起航


widget是immutable的，发生变化的时候需要重建，所以谈不上状态。StatefulWidget 中的状态保持其实是通过State类来实现的。State拥有一套自己的生命周期，下面做一个简单的介绍。

 ![](https://i.imgur.com/UOGGyTC.png)

整体分三个部分，类似于Android的onCreate-onResume-onDestory阶段

- 初始化（插入渲染树）
- 状态改变（在渲染树中存在）
- 销毁（从渲染树种移除）

细分的来看的话：

- initState 
当插入渲染树的时候调用，这个函数在生命周期中只调用**一次**。这里可以做一些初始化工作，比如初始化State的变量。



- didChangeDependencies
state依赖的对象发生变化时调用


- didUpdateWidget 
当组件的状态改变的时候就会调用didUpdateWidget,比如调用了`setState()`.

实际上这里flutter框架会创建一个新的Widget,绑定本State，并在这个函数中传递老的Widget。

这个函数一般用于比较新、老Widget，看看哪些属性改变了，并对State做一些调整。

需要注意的是，涉及到controller的变更，需要在这个函数中移除老的controller的监听，并创建新controller的监听。

![](https://segmentfault.com/img/bVbbY2H?w=1308&h=850)

- build
构建Widget时调用


- deactivate
当移除渲染树的时候调用


- dispose
组件即将销毁时调用

非常熟悉的流程，估计就是借鉴的React内容来写的

`didChangeDependencies`有两种情况会被调用。

1. 创建时候在initState 之后被调用
2. 在依赖的InheritedWidget发生变化的时候会被调用

依赖的widget就是上边这个：

	class AppHomePage extends StatefulWidget {
	  @override
	  _AppHomePageState createState() => _AppHomePageState();
	}

	class _AppHomePageState extends State<AppHomePage> {}

正常的退出流程中会执行deactivate然后执行dispose。
但是也会出现deactivate以后不执行dispose，直接加入树中的另一个节点的情况。
这里的状态改变包括两种可能：

1. 通过setState内容改变 
2. 父节点的state状态改变，导致孩子节点的同步变化。
 


# ref

https://segmentfault.com/a/1190000015211309
https://www.imooc.com/article/details/id/68585