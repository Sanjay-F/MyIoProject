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


## ListTile
我们看下构造函数

	const ListTile({
	    Key key,
	    this.leading,              // item 前置图标
	    this.title,                // item 标题
	    this.subtitle,             // item 副标题
	    this.trailing,             // item 后置图标
	    this.isThreeLine = false,  // item 是否三行显示
	    this.dense,                // item 直观感受是整体大小
	    this.contentPadding,       // item 内容内边距
	    this.enabled = true,
	    this.onTap,                // item onTap 点击事件
	    this.onLongPress,          // item onLongPress 长按事件
	    this.selected = false,     // item 是否选中状态
	}) 

常用功能都提供了，自动提供了很好的封装接口。

# list.Builder
除了上面的直接生成childer方式，还可以用build的方式
 
	class HomePage extends StatefulWidget {

	  @override
	  State<StatefulWidget> createState() => new  ListState();
	}
	
	//用build得到一个ListView
	class ListState extends State {
	  @override
	  Widget build(BuildContext context) {
	    return new ListView.builder(itemCount: 40, itemBuilder: buildItem);
	  }
	
	  //ListView的Item
	  Widget buildItem(BuildContext context, int index) {
	    return new Text("demoText$index");
	  }
	} 

通过传入的itemBuilder去构建view,相对于前面的用系统自带的，这个方式可以达到更高的自定义View的效果。

有时我们会需要分隔符，可以这么来：

	child: new ListView.separated(
	    itemCount: iconItems.length,
	    separatorBuilder: (BuildContext context, int index) => new Divider(),  // 添加分割线
	    itemBuilder: (context, item) {
	        return buildListData(context, strItems[item], iconItems[item]);
	    },
	)