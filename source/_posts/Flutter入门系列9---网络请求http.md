title: Flutter入门系列9---网络请求http

date: 2018-03-28 23:14:46

tags: [android,Flutter,http]

categories: flutter

------------------------------------------


这次我们来做个简单的demo，去执行网络请求，加载回来数据后，在listView去显示。由于没有什么现成的api可以调用，这次只能借鉴了别人的demo来说了。。
效果类似这样的： 
![](https://i.imgur.com/JGF7W7c.png)
<!--more-->


# 起航

	import 'dart:convert';
	import 'package:flutter/material.dart';
	import 'dart:async';
	import 'dart:ui';
	import 'package:http/http.dart' as http;   //这个flutter允许加别名，做过web同学一定很熟悉
	
	void main() => runApp(new MyApp());
	class MyApp extends StatelessWidget {
	  @override
	  Widget build(BuildContext context) {
	    return new MaterialApp(
	      title: 'Flutter Demo',
	      home: new AppHomePage(),
	    );
	  }
	}
 
上面你这些是模板代码一样的，可以跳过。我们看下下面的内容

	class AppHomePage extends StatefulWidget {
	  @override
	  _AppHomePageState createState() => _AppHomePageState();
	}
	
	class _AppHomePageState extends State<AppHomePage> {
	  List movies;
	
	  Future getMovies({String type = 'anime', int page = 1, String subtype = 'movie'}) async {

		//flutter 发送网络请求还是很简单的。就这么简单粗暴的写
		//但实际工程肯定会在包一层，这里是demo

	    final String url = "https://api.jikan.moe/top/$type/$page/$subtype";
	    final response = await http.get(url);
		//发请求，然后await等待结果回来


	    if (response.statusCode == 200) {
	      List top = json.decode(response.body)['top'];
	      setState(() {
	        movies = top.map((json) => Animate.fromJson(json)).toList();
	      });
	    } else {
	      print("err code $response.statusCode");
	    }
	  }
	
	  @override
	  void initState() {
	    super.initState();
	    getMovies(); //下一篇讲下这个state的生命周期，简单理解成activity的onCreate
	  }
	
	  @override
	  Widget build(BuildContext context) {
	    return Scaffold(
	      appBar: AppBar(
	        title: Text('Top Animate Movies'),
	      ),
		//flutter不提供设置view的可见性，只通过这个判断来显示不同的view
		//做过web的同学应该也很熟悉这样的套路，对于纯安卓开发会有点不习惯

	      body: movies == null
	          ? Center(child: CircularProgressIndicator()) //数据没回来前显示进度条
	          : Padding(
	              padding: EdgeInsets.symmetric(horizontal: 6.0, vertical: 10.0),
	              child: ListView.builder(
	                itemCount: movies.length,
	                itemBuilder: (BuildContext context, int index) {
	                  return AnimateCard(movies[index]);  //数据回来后就调用这个AnimateCard去生成list的item
	                },
	              ),
	            ),
	    );
	  }
	}
然后我们看下card的布局


	class AnimateCard extends StatelessWidget {
	  Animate animate;
	
	  AnimateCard(this.animate);
	
	  @override
	  Widget build(BuildContext context) {
	    return Card(
	      child: InkWell(
	        child: ListTile(
				//头部图片，
	          leading: Image.network(animate.imgUrl,width: 45.0,height: 45.0),
				//中间文字
	          title: Text(
	            animate.title,
	            maxLines: 2,
	            textAlign: TextAlign.start,
	            overflow: TextOverflow.ellipsis,
	            style: TextStyle(color: Colors.deepPurple, fontSize: 16.0),
	          ),
				//尾部评分![](https://i.imgur.com/J9SNf1k.png)
	          trailing: Text(
	            animate.score.toString(),
	          ),
	        ),
	      ),
	    );
	  }
	}

这部分也很好理解，就不细说了

最后把实体bean放这里，整个就完成了

	class Animate {
	  final String imgUrl;
	  final String title;
	  final double score;
	
	  Animate({
	    this.imgUrl,
	    this.title,
	    this.score,
	  });
	
	  factory Animate.fromJson(Map<String, dynamic> json) {
	    return Animate(
	      imgUrl: json['image_url'] as String,
	      title: json['title'] as String,
	      score: json['score'] as double,
	    );
	  }
	}


#ref 

https://juejin.im/post/5b7ac50d6fb9a01a0b318e71