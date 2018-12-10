title: Flutter系列5---基础控件Stack

date: 2018-03-10 23:14:46

tags: [android,Flutter]

categories: flutter

------------------------------------------

Flutter还提供了一个叫stack的轮子，他有点像安卓的RelatvieLayout，不过安卓被用得最少的，可能是绝对布局了。
基本没派上用处。


	import 'package:flutter/material.dart';
	
	void main() {
	  runApp(
	    new MaterialApp(
	      title: 'Demo',
	      home: new Scaffold(
	        appBar: new AppBar(
	          title: new Text('Demo'),
	        ),
	        body: new Center(
	          child: new Stack(//第一个子控件最下面
	            // alignment: new Alignment(0.6, 0.6),
	            // alignment: new Alignment(0.6, 0.6), 堆放在x（宽度）*0.6 y（高度）*0.6处。
	            //默认不赋值的话，那么就再屏幕的左上角位置开始绘制 

	            //statck
	            children: <Widget>[
	              new Align(
	                alignment: FractionalOffset.center,
	//          heightFactor: 40.0,
	//          widthFactor: 40.0,
	                child: new Image.network(//加载网络图片
	                  'http://had5e1.jpg',
	                  height: 300.0,
	                  width: 300.0,
	                  repeat: ImageRepeat.repeat,//图片重复方式
	                ),
	              ),
	              new Opacity(
	                opacity: 0.5,//不透明度
	                child: new Container(
	                  width: 300.0,
	                  height: 400.0,
	                  color: Colors.blue,
	                ),
	                new Positioned(//方法二
		                  right: 15.0,
		                  top: 15.0,
		                  child: new Icon(
		                    Icons.share,
		                    color: Colors.white,
		                  ),
	                ),
	              )
	            ],
	          ),
	        ),
	      ),
	    ),
	  );
	}
	 
除了上面的alignment用法，我们还可以用另外的方法来做控制

1. 使用alignment配合FractionalOffset：对于FractionalOffset的参数，我是这么理解的：相当于比例，第一个代表横向的权重，第二个代表竖向的权重，横0.9代表在横向十分之九的位置，竖0.1代表在竖向十分之一的位置


2. 使用position，我们看上面的例子的第三个，就是使用position来做一些定位的逻辑的。top、right、buttom、left，直接代表，距离该方向的距离



Stack的布局行为，根据child是positioned还是non-positioned来区分。

- 对于positioned的子节点，它们的位置会根据所设置的top、bottom、right以及left属性来确定，这几个值都是相对于Stack的左上角；
- 对于non-positioned的子节点，它们会根据Stack的aligment来设置位置。

对于绘制child的顺序，则是第一个child被绘制在最底端，后面的依次在前一个child的上面，类似于web中的z-index。如果想调整显示的顺序，则可以通过摆放child的顺序来进行。

我们来看下构造函数

	Stack({
	  Key key,
	  this.alignment = AlignmentDirectional.topStart,
	  this.textDirection,
	  this.fit = StackFit.loose,
	  this.overflow = Overflow.clip,
	  List<Widget> children = const <Widget>[],
	})

- alignment：对齐方式，默认是左上角（topStart）。
- textDirection：文本的方向，绝大部分不需要处理。
- fit：定义如何设置non-positioned节点尺寸，默认为loose。
  
  其中StackFit有如下几种：
	•	loose：子节点宽松的取值，可以从min到max的尺寸；
	•	expand：子节点尽可能的占用空间，取max尺寸；
	•	passthrough：不改变子节点的约束条件。
- overflow：超过的部分是否裁剪掉（clipped）。
 
 

