title: Flutter入门系列3---常用基础控件Container

date: 2018-01-18 22:45:46

tags: [android,Flutter,Container]

categories: Flutter
 

------------------------------------------

  
这一篇基于官方的案例，说下基础控件TexT，IconButton，Column，Row.着重Container的使用

先上下效果图

![](https://i.imgur.com/RYjmJfo.png)

顶部蓝色三个元素，一个汉堡图，一个title，一个右边的搜索图。
然后下面是一个简单的text。

<!--more-->

先上代码

	import 'package:flutter/material.dart';
	
	//运行的入口，就像java执行main方法一样
	void main() => runApp(MaterialApp(
	      title: 'My app', // used by the OS task switcher
	      home: MyScaffold() //绘制内容为MyScaffold
	    )); 

	
	class MyScaffold extends StatelessWidget {
	  @override
	  Widget build(BuildContext context) {
	    // Material is a conceptual piece of paper on which the UI appears.
		
		//这个返回的内容是个列表Column，上面是MyAppBar,下一行是Center里面的内容
	    return Material(
	      // Column is a vertical, linear layout.
	      child: Column(
	        children: <Widget>[
	          MyAppBar(
	            title: Text(
	              'Example title',
	              style: Theme.of(context).primaryTextTheme.title,
	            ),
	          ),

	          Center(
	            child: Text('Hello, world!'),
	          ),
	        ],
	      ),
	    );
	  }
	}
	
	
	class MyAppBar extends StatelessWidget {
	
	  MyAppBar({this.title});
	  final Widget title;
	
	  @override
	  Widget build(BuildContext context) {
	    return Container(

	      height: 56.0, // in logical pixels
	      padding: const EdgeInsets.symmetric(horizontal: 8.0),
	      decoration: BoxDecoration(color: Colors.blue[500]),

	      // Row is a horizontal, linear layout.
	      child: Row( 
			//row就是水平的线性布局空间。
	        children: <Widget>[
	          IconButton(
	            icon: Icon(Icons.menu),
	            onPressed: null, // 按钮的点击事件，这里没加
	          ),

	          // Expanded expands its child to fill the available space.
	          Expanded(
	            child: title,
	          ),
				//有点像LineLayout里面的weight=1操作，撑满所有

	          IconButton(
	            icon: Icon(Icons.search),
	            tooltip: 'Search',
	            onPressed: null,
	          ),
	        ],
	      ),
	    );
	  }
	}

看我简单注释和效果，其实也就明白了Column，Row类似于安卓的LineLayout在方向的两个拆分。然后Container是个容器
TexT，IconButton这是基础的小控件。

# Container

官方给出的简介，是一个结合了绘制（painting）、定位（positioning）以及尺寸（sizing）widget的widget。
它是一个组合的widget，内部有绘制widget、定位widget、尺寸widget。后续看到的不少widget，都是通过一些更基础的widget组合而成的。

	Object > Diagnosticable > DiagnosticableTree > Widget > StatelessWidget > Container

	Container({
	    Key key,
	    this.alignment,
	    this.padding,
	    Color color,
	    Decoration decoration,
	    this.foregroundDecoration,
	    double width,
	    double height,
	    BoxConstraints constraints,
	    this.margin,
	    this.transform,
	    this.child,
	  }) 

稍微说下就是，你可能会发现，只有contaienr那一次有height，padding等属性的设置，而在IconButton就没有，因为后者却是没有哈哈。
所以才说container有什么定位，尺寸能力，原本我们在Android熟悉的对TextView做的一堆设置，在这里通通没有，你需要拿多Container去包一层，从而获得对应的能力

- alignment：控制child的对齐方式，如果container或者container父节点尺寸大于child的尺寸，这个属性设置会起作用，有很多种对齐方式。
- padding：decoration内部的空白区域，如果有child的话，child位于padding内部。padding与margin的不同之处在于，padding是包含在content内，而margin则是外部边界，设置点击事件的话，padding区域会响应，而margin区域不会响应。
- color：用来设置container背景色，如果foregroundDecoration设置的话，可能会遮盖color效果。
- decoration：绘制在child后面的装饰，设置了decoration的话，就不能设置color属性，否则会报错，此时应该在decoration中进行颜色的设置。
- foregroundDecoration：绘制在child前面的装饰。
- width：container的宽度，设置为double.infinity可以强制在宽度上撑满，不设置，则根据child和父节点两者一起布局。
- height：container的高度，设置为double.infinity可以强制在高度上撑满。
- constraints：添加到child上额外的约束条件。
- margin：围绕在decoration和child之外的空白区域，不属于内容区域。
- transform：设置container的变换矩阵，类型为Matrix4。
- child：container中的内容widget。

 
![我们来看下盒子模型](https://i.imgur.com/i2QR8OM.jpg)
 

Container首先有padding围绕着子部件 (图中深绿色部分) , 将宽高作为约束 . 然后 container 被额外的空白控件围绕, 叫做 margin.

其绘制顺序大致为:

- 先应用给定的变换 transform
- 然后绘制 decoration
-  再绘制子部件 child
- 最后绘制 foregroundDecoration

transform 是指对widget在原有基础上做一些类似旋转、平移之类的变换 .
decoration 以及 foregroundDecoration 是部件的 背景/前景 ‘装饰’ , 比如绘制部件的边框,背景图片等都是 decoration 和 foregroundDecoration中设置的.

看下下面示例：

	class DemoContainer extends StatelessWidget {
	
	  @override
	  Widget build(BuildContext context) {
	    var imgUrl = "https://ws1.sinaimg.cn/large/006tNc79gy1fpa5bvsqskj3044048mx5.jpg";
	    return new Container(
	        padding: const EdgeInsets.all(16.0),  // 内边距
	        color: new Color(0xFFF2F2F2),         // 背景色
	        alignment: Alignment.center,          //子部件对齐方式
	        child: new Container(                 // 子部件
	          width: 400.0,                           //宽
	          height: 400.0,                          //高
	          // color 与 decoration 互斥 .如需设置decoration 和 color , 可在decoration中设置color
	          // color: Colors.blueGrey,
	          padding: const EdgeInsets.all(16.0),
	          alignment: Alignment.center,
	          decoration: new BoxDecoration(
	            color: Colors.blueGrey,
	            border: new Border.all(
	              color: Colors.blue,
	              width: 8.0,
	            ),
	            image: new DecorationImage(image: new NetworkImage(imgUrl))
	          ),
	          child: new Text('Halcyon',style: const TextStyle(color: Colors.blue,fontSize: 24.0),),
	        )
	
	    );
	  }
	}

![](https://i.imgur.com/X0C8SHw.png)

首先导航栏下面整个是一个内边距padding 16 的灰色(#F2F2F2)背景的Container ,
 
其唯一子部件是居中对齐的, 然后我们主要说明的就是这个子部件,同样是个Container 
这个子 Container 宽高均为400,也设置了内边距** padding 16 **.

其 decoration 设置为拥有四周宽度为**8**的**蓝色边框**,并且有一个表情包图片作为容器背景, 由于没指定图片拉伸方式 , 此时图片以原始大小居中显示 。同时这个容器拥有一个内容为 ‘Halcyon’ 的文本子控件居中。


## 尺寸的调节
类似view的onMeasure，而Container自身尺寸的调节分两种情况：

1. Container在没有子节点（children）的时候，会试图去变得足够大。除非constraints是unbounded限制，在这种情况下，Container会试图去变得足够小。
2. 带子节点的Container，会根据子节点尺寸调节自身尺寸，但是Container构造器中如果包含了width、height以及constraints，则会按照构造器中的参数来进行尺寸的调节。

也很好理解，默认就是match_parent的状态，如果有子view，就去wrap_content, 写死宽高就按照写的走。 


## 绘制
类似onLayout操作，其实Container类似于android中的ViewGroup。里面包含了很多的基础widget，Flutter是通过组合基础widget的方式弄了个container出来的。

因此Container的布局行为有时候是比较复杂的。一般情况下，Container会遵循如下顺序去尝试布局：

1. 对齐（alignment）；
2. 调节自身尺寸适合子节点；
3. 采用width、height以及constraints布局；
4. 扩展自身去适应父节点；
5. 调节自身到足够小。 


上面这些听起来有点晕，我们可以直接看源码的角度来解决问题：
# 源码

	  @override
	  Widget build(BuildContext context) {
	    Widget current = child;
	
	    if (child == null && (constraints == null || !constraints.isTight)) {
	      current = new LimitedBox(
	        maxWidth: 0.0,
	        maxHeight: 0.0,
	        child: new ConstrainedBox(constraints: const BoxConstraints.expand())
	      );
	    }
	
	    if (alignment != null)
	      current = new Align(alignment: alignment, child: current);
	
	    final EdgeInsetsGeometry effectivePadding = _paddingIncludingDecoration;
	    if (effectivePadding != null)
	      current = new Padding(padding: effectivePadding, child: current);
	
	    if (decoration != null)
	      current = new DecoratedBox(decoration: decoration, child: current);
	
	    if (foregroundDecoration != null) {
	      current = new DecoratedBox(
	        decoration: foregroundDecoration,
	        position: DecorationPosition.foreground,
	        child: current
	      );
	    }
	
	    if (constraints != null)
	      current = new ConstrainedBox(constraints: constraints, child: current);
	
	    if (margin != null)
	      current = new Padding(padding: margin, child: current);
	
	    if (transform != null)
	      current = new Transform(transform: transform, child: current);
	
	    return current;
	  } 

 最里层的是child，如果为空或者其他约束条件，则最里层包含的为一个LimitedBox，然后依次是Align、Padding、DecoratedBox、前景DecoratedBox、ConstrainedBox、Padding（实现margin效果）、Transform。

Container的源码本身并不复杂，复杂的是它的各种布局表现。我们谨记住一点，如果内部不设置约束，则按照父节点尽可能的扩大，如果内部有约束，则按照内部来。 