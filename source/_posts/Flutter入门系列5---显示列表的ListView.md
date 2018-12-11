title: Flutter入门系列5---显示列表的ListView

date: 2018-02-13 22:45:46

tags: [android,Flutter,ListView]

categories: Flutter
 

------------------------------------------

上一篇说了row和Column，但不带滚动。
这次我们说下带滚动效果的ListView

![](https://i.imgur.com/9nL2QDe.png)

<!--more-->

# 起航

我们先写个demo在这里

	import 'package:flutter/material.dart';
	
	void main() =>
	    runApp(MyListApp());
	
	class MyListApp extends StatelessWidget {
	  @override
	  Widget build(BuildContext context) {
	    final title = 'list';
	    return new MaterialApp(
	      title: title,
	      home: new Scaffold(
	        appBar: new AppBar(
	          title: new Text(title),
	        ),
	        body: new Center(
	          child: new ListView(

	            //控制方向 默认是垂直的
				//scrollDirection: Axis.horizontal,

	            children: <Widget>[
	              _getListItem('Maps', Icons.map),
	              _getListItem('phone', Icons.phone),
	              _getListItem('Maps', Icons.adb),
	            ],
	          ),
	        ),
	      ),
	    );
	  }
	
	  /**
	   * 抽取item项
	   */
	  Widget _getListItem(String title, IconData icon) {
	    return new Container( 
	//      ListTile
	      child: new ListTile(
	//       显示在title之前
	        leading: new Icon(icon),
	//        显示在title之后
	        trailing: new Icon(icon),
	        title: new Text(title),
	        subtitle:new Text("我是subtitle") ,
	      ),
	    );
	  }
	}

不得不说这个flutter封装的控件更好，当初安卓没提供水平方向的Listview，各自造轮子，现在只需要一个属性设置下就支持了。`scrollDirection: Axis.horizontal`
另外自带的内置ListTile也挺好的，不用像Android一样，去弄个adapter，然后viewHolder，自己inflate不同的view做处理。相对友好，只需要自己去new个view出来就好了。


