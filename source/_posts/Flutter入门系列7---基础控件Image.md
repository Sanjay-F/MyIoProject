title: Flutter系列7---基础控件Image

date: 2018-03-15 23:14:46

tags: [android,Flutter,Image]

categories: flutter

------------------------------------------

显示图片当然是最基础和重要的轮子，我们来看下Flutter的Image对应于安卓的Imageview是有多么的好用，封装的多实在的库。。
 
 
-- **Image：**通过ImageProvider来加载图片

-- **Image.asset：**用来加载本地资源图片

-- **Image.file：**用来加载本地（File文件）图片

-- **Image.network：**用来加载网络图片

-- **Image.memory：**用来加载Uint8List资源（byte数组）图片 

相比安卓早期需要自己弄一个库去加载，实在厚道，自动封装了对应的加载方法了。怎么使用我们看下下面代码案例

图片格式上支持： JPEG , PNG ,GIF , 动态 GIF , WebP , 动态WebP , BMP WBMP .

<!--more-->


我们先看下demo

	import 'package:flutter/material.dart';
	
	void main() =>
	    runApp(ImageDemoApp());
	
	class ImageDemoApp extends BoxStateLessWidget{
	 
	  @override
	  Widget build(BuildContext context) { 
	    return new Scaffold(
	      appBar: new AppBar(title: new Text("Container Test"),
	        centerTitle: true,),
	
	      body: new ListView(
	        children: <Widget>[
	          // 资源图片
	          new Image.asset('imgs/logo.jpeg'),
	          //网络图片
	          new Image.network(
	              'https://flutter.io/images/homepage/header-illustration.png'),
	          //使用ImageProvider加载图片
	          new Image(image: new NetworkImage(
	              "https://flutter.io/images/homepage/screenshot-2.png"),)
	        ],
	      ),
	    );
	  }
	}

这个简单的案例.我们看下构造函数

	/// A widget that displays an image.
	///
	/// Several constructors are provided for the various ways that an image can be
	/// specified:
	///
	///  * [new Image], for obtaining an image from an [ImageProvider].
	///  * [new Image.asset], for obtaining an image from an [AssetBundle]
	///    using a key.
	///  * [new Image.network], for obtaining an image from a URL.
	///  * [new Image.file], for obtaining an image from a [File].
	///  * [new Image.memory], for obtaining an image from a [Uint8List].
	///
	/// The following image formats are supported: {@macro flutter.dart:ui.imageFormats}
	///
	/// To automatically perform pixel-density-aware asset resolution, specify the
	/// image using an [AssetImage] and make sure that a [MaterialApp], [WidgetsApp],
	/// or [MediaQuery] widget exists above the [Image] widget in the widget tree.
	///
	/// The image is painted using [paintImage], which describes the meanings of the
	/// various fields on this class in more detail.
	///
	/// See also:
	///
	///  * [Icon], which shows an image from a font.
	///  * [new Ink.image], which is the preferred way to show an image in a
	///    material application (especially if the image is in a [Material] and will
	///    have an [InkWell] on top of it).
	class Image extends StatefulWidget {
	  /// Creates a widget that displays an image.
	  ///
	  /// To show an image from the network or from an asset bundle, consider using
	  /// [new Image.network] and [new Image.asset] respectively.
	  ///
	  /// The [image], [alignment], [repeat], and [matchTextDirection] arguments
	  /// must not be null.
	  ///
	  /// Either the [width] and [height] arguments should be specified, or the
	  /// widget should be placed in a context that sets tight layout constraints.
	  /// Otherwise, the image dimensions will change as the image is loaded, which
	  /// will result in ugly layout changes.
	  ///
	  /// If [excludeFromSemantics] is true, then [semanticLabel] will be ignored.
	  const Image({
	    Key key,
	    @required this.image,
	    this.semanticLabel,
	    this.excludeFromSemantics = false,
	    this.width,
	    this.height,
	    this.color,
	    this.colorBlendMode,
	    this.fit,
	    this.alignment = Alignment.center,
	    this.repeat = ImageRepeat.noRepeat,
	    this.centerSlice,
	    this.matchTextDirection = false,
	    this.gaplessPlayback = false,
	  }) : assert(image != null),
	       assert(alignment != null),
	       assert(repeat != null),
	       assert(matchTextDirection != null),
	       super(key: key);


对应字段

- width & height
用来指定显示图片区域的宽高（并非图片的宽高）


- fit
设置图片填充，类似于Android中的ScaleType

	- BoxFit.contain
	全图居中显示但不充满，显示原比例	
	- BoxFit.cover
	图片可能拉伸，也可能裁剪，但是充满容器		
	- BoxFit.fill
	全图显示且填充满，图片可能会拉伸		
	- BoxFit.fitHeight
	图片可能拉伸，可能裁剪，高度充满		
	- BoxFit.fitWidth
	图片可能拉伸，可能裁剪，宽度充满		
	- BoxFit.scaleDown
	效果和contain差不多， 但是只能缩小图片，不能放大图片



- color & [colorBlendMode](https://docs.flutter.io/flutter/dart-ui/BlendMode-class.html)
这两个属性需要配合使用，就是颜色和图片混合，就类似于Android中的Xfermode。一般少用到，贴链接，有需要时候去看下就可以了。


- alignment
用来控制图片摆放的位置


- repeat
用来设置图片重复显示（repeat-x水平重复，repeat-y垂直重复，repeat两个方向都重复，no-repeat默认情况不重复）.这个做过web都明白，就是填充view时候的不同方式。


- centerSlice
设置图片内部拉伸，相当于在图片内部设置了一个.9图，但是需要注意的是，要在显示图片的大小大于原图的情况下才可以使用这个属性，要不然会报错.理由是下面这个源码：
`assert(sourceSize == inputSize, 'centerSlice was used with a BoxFit that does not guarantee that the image is fully visible.');`


- matchTextDirection
这个需要配合Directionality进行使用


- gaplessPlayback
当图片发生改变之后，重新加载图片过程中的样式（1、原图片保留）
 


因此我们可以这么修改图片的显示

	new Image.network(
            'https://gw.alicdn.com/tfs/TB1CgtkJeuSBuNjy1XcXXcYjFXa-906-520.png',
            fit: BoxFit.contain,
            width: 150.0,
            height: 100.0,
          ),

 
	 new Image.asset(
              'images/logo.png',
              width: 250,
              height: 250,
              fit: BoxFit.contain,
              centerSlice:
                  new Rect.fromCircle(center: const Offset(20, 20), radius: 1),
            )
 
## FadeInImage
在实际开发中，考虑到图片加载速度可能不能达到预期。所以希望能增加渐入效果&增加placeHolder的功能。Flutter同样提供的这样的组件——FadeInImage。 
 

## 提示
唯一有点不友好的是，添加本地图片有点麻烦，希望官方后续优化好这个问题，就是`asset`文件需要把添加的图片自己手动加到`pubspec.yaml`

	flutter:
	  assets:
	    - assets/my_icon.png
	    - assets/background.png
	    
不像安卓，添加文件的drawable目录，自动有个R的id来使用。同时加载asset是自己写路径，容易出错啊。特别是图片多了之后，成千上百，那简直受不了




网络请求Image是大家最常见的操作。这里重点说明两个点：

- 缓存

ImageCache是ImageProvider默认使用的图片缓存。ImageCache使用的是LRU的算法。默认可以存储1000张图片。如果觉得缓存太大，可以通过设置ImageCache的maximumSize属性来控制缓存图片的数量。也可以通过设置maximumSizeBytes来控制缓存的大小（默认缓存大小10MB）。

- CDN优化

如果想要使用cdn优化，可以通过url增加后缀的方式实现。默认实现中没有这个点，但是考虑到cdn优化的可观收益，建议大家利用好这个优化。
 

## ref

https://flutterchina.club/assets-and-images/
https://www.jianshu.com/p/9e6c470ea5bf