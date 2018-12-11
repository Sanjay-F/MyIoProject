title: Flutter入门系列3---基础列表控件Row和Column

date: 2018-01-28 20:45:46

tags: [android,Flutter]

categories: Flutter
 

------------------------------------------

 上一篇说了Container，流水的讲了Row和Column。这里来进一步的说下.
做过web的估计很好理解这个，因为这个和Web中Flex布局很像，但坑的是不带滚动属性，不像ScrollView。如果超出了一行，在debug下面则会显示溢出的提示。 



 ![](https://i.imgur.com/HY4o4wN.png)

这个顶部有三个控件，红黄蓝，通过设置flex来设置了占比。

	 class MyAppBar extends StatelessWidget {
	  MyAppBar({this.title});
	
	  final Widget title;
	
	  @override
	  Widget build(BuildContext context) {
	    return Container(
	        height: 56.0, // in logical pixels
	       
	        decoration: BoxDecoration(color: Colors.blue[500]),
	        // Row is a horizontal, linear layout.
	        child: Row(
	          children: <Widget>[
	            Expanded(
	              child: Container(
	                color: Colors.red,
	                padding: EdgeInsets.all(5.0),
	              ),
	              flex: 1,
	            ),
	            Expanded(
	              child: Container(
	                color: Colors.yellow,
	                padding: EdgeInsets.all(5.0),
	              ),
	              flex: 2,
	            ),
	            Expanded(
	              child: Container(
	                color: Colors.blue,
	                padding: EdgeInsets.all(5.0),
	              ),
	              flex: 1,
	            ),
	          ],
	        )
	    );
	  }
	}

<!--more-->

我们看下row的构造函数内容：

	class Row extends Flex {
	  /// Creates a horizontal array of children.
	  ///
	  /// The [direction], [mainAxisAlignment], [mainAxisSize],
	  /// [crossAxisAlignment], and [verticalDirection] arguments must not be null.
	  /// If [crossAxisAlignment] is [CrossAxisAlignment.baseline], then
	  /// [textBaseline] must not be null.
	  ///
	  /// The [textDirection] argument defaults to the ambient [Directionality], if
	  /// any. If there is no ambient directionality, and a text direction is going
	  /// to be necessary to determine the layout order (which is always the case
	  /// unless the row has no children or only one child) or to disambiguate
	  /// `start` or `end` values for the [mainAxisAlignment], the [textDirection]
	  /// must not be null.
	  Row({
	    Key key,
	    MainAxisAlignment mainAxisAlignment = MainAxisAlignment.start,
	    MainAxisSize mainAxisSize = MainAxisSize.max,
	    CrossAxisAlignment crossAxisAlignment = CrossAxisAlignment.center,
	    TextDirection textDirection,
	    VerticalDirection verticalDirection = VerticalDirection.down,
	    TextBaseline textBaseline,
	    List<Widget> children = const <Widget>[],
	  }) : super(
	    children: children,
	    key: key,
	    direction: Axis.horizontal,
	    mainAxisAlignment: mainAxisAlignment,
	    mainAxisSize: mainAxisSize,
	    crossAxisAlignment: crossAxisAlignment,
	    textDirection: textDirection,
	    verticalDirection: verticalDirection,
	    textBaseline: textBaseline,
	  );
	}



![](https://i.imgur.com/bgfgrWb.png)

1. MainAxisAlignment：主轴方向上的对齐方式，会对child的位置起作用，默认是start。
其中MainAxisAlignment枚举值：
	- center：将children放置在主轴的中心；
	- end：将children放置在主轴的末尾；
	- spaceAround：将主轴方向上的空白区域均分，使得children之间的空白区域相等，但是首尾child的空白区域为1/2；
	- spaceBetween：将主轴方向上的空白区域均分，使得children之间的空白区域相等，首尾child都靠近首尾，没有间隙；
	- spaceEvenly：将主轴方向上的空白区域均分，使得children之间的空白区域相等，包括首尾child；
	- start：将children放置在主轴的起点；
	
	其中spaceAround、spaceBetween以及spaceEvenly的区别，就是对待首尾child的方式。其距离首尾的距离分别是空白区域的1/2、0、1。
2. MainAxisSize：在主轴方向占有空间的值，默认是max。
  MainAxisSize的取值有两种：
	- max：根据传入的布局约束条件，最大化主轴方向的可用空间；
	- min：与max相反，是最小化主轴方向的可用空间；

3. CrossAxisAlignment：children在交叉轴方向的对齐方式，与MainAxisAlignment略有不同。
  CrossAxisAlignment枚举值有如下几种：
	- baseline：在交叉轴方向，使得children的baseline对齐；
	- center：children在交叉轴上居中展示；
	- end：children在交叉轴上末尾展示；
	- start：children在交叉轴上起点处展示；
	- stretch：让children填满交叉轴方向；

4. TextDirection：阿拉伯语系的兼容设置，一般无需处理。

5.  VerticalDirection：定义了children摆放顺序，默认是down。
	VerticalDirection枚举值有两种：
	- down：从top到bottom进行布局；
	- up：从bottom到top进行布局。
	
	top对应Row以及Column的话，就是左边和顶部，bottom的话，则是右边和底部。TextBaseline：使用的TextBaseline的方式，有两种，前面已经介绍过。


## Column

Row和Column都是Flex的子类，只是direction参数不同。Column各方面同Row，因此在这里不再另行讲解。

 	Object > Diagnosticable > DiagnosticableTree > Widget > RenderObjectWidget > MultiChildRenderObjectWidget > Flex > Row
	Object > Diagnosticable > DiagnosticableTree > Widget > RenderObjectWidget > MultiChildRenderObjectWidget > Flex > Column

我们来看下类源码的地方
	class Column extends Flex {
	  /// Creates a vertical array of children.
	  ///
	  /// The [direction], [mainAxisAlignment], [mainAxisSize],
	  /// [crossAxisAlignment], and [verticalDirection] arguments must not be null.
	  /// If [crossAxisAlignment] is [CrossAxisAlignment.baseline], then
	  /// [textBaseline] must not be null.
	  ///
	  /// The [textDirection] argument defaults to the ambient [Directionality], if
	  /// any. If there is no ambient directionality, and a text direction is going
	  /// to be necessary to disambiguate `start` or `end` values for the
	  /// [crossAxisAlignment], the [textDirection] must not be null.
	  Column({
	    Key key,
	    MainAxisAlignment mainAxisAlignment = MainAxisAlignment.start,
	    MainAxisSize mainAxisSize = MainAxisSize.max,
	    CrossAxisAlignment crossAxisAlignment = CrossAxisAlignment.center,
	    TextDirection textDirection,
	    VerticalDirection verticalDirection = VerticalDirection.down,
	    TextBaseline textBaseline,
	    List<Widget> children = const <Widget>[],
	  }) : super(
	    children: children,
	    key: key,
	    direction: Axis.vertical,
	    mainAxisAlignment: mainAxisAlignment,
	    mainAxisSize: mainAxisSize,
	    crossAxisAlignment: crossAxisAlignment,
	    textDirection: textDirection,
	    verticalDirection: verticalDirection,
	    textBaseline: textBaseline,
	  );
	}

