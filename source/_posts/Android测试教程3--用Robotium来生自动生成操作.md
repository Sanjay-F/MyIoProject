title: Android测试教程3--用Robotium来生自动生成操作
date: 2015-11-19 12:16
tags: [android,测试]
categories: android
---
 
前面的教程2。
我们自己手写了一些简单的测试案例，你发现需要各种绑定界面的。
现在作为偷懒的人，想像以前用按键精灵一样，自动给我们做好各种点击操作，
来看下运行是否正常。那么该怎么做呢？
下面教程告诉你，用`Robotium`如何做到。
 
<!--more-->

# 安装插件
在开始前，我们需要先安装下插件。
![这里写图片描述](http://img.blog.csdn.net/20151119111331639)
如果没出来，点击提示的"Browse repositories.."去搜索下吧。 安装后重启下。

# 开始测试
装好后，在我们的tool->Robitum。
![这里写图片描述](http://img.blog.csdn.net/20151119111505638)

进入这个界面，我们一般选择我们需要测试的模块，例如我们项目默认的app模块。 

 ![这里写图片描述](http://img.blog.csdn.net/20151119112032086)
在设置设立，我们有一些配置的东西。
![这里写图片描述](http://img.blog.csdn.net/20151119112228445)

具体的解释如下：

> - **Use sleeps** - choose if sleeps should be used to playback test cases in the same speed as they were recorded. Can be useful for
> slower apps like bandwidth intensive or hybrid apps.
> 
> - **Keep app data** - choose if app data is to be kept when starting a new recording session.
>  
> - **Identify class over string** - default View identifier is always the resource ID. In the event a resource ID is missing it's possible
> to choose if a View class identifier is to be used over a String
> identifier (the text displayed by the View).

好了，现在我们还是来New一个新的Robotium Test吧。点击那个按钮，然后开始测试吧
点击后，他会开始building￥……%@！￥%# 
然后把app安装到手机或虚拟器上，你就可以开始你的各种操作啦，Robotium都会记录下来
![这里写图片描述](http://img.blog.csdn.net/20151119112815158)
 
如果按错了，也可以点击`DeleteStep` 来删除。
测试完，就点击下面的 `Stop`结束就可以了，然后再点旁边的`save`，保存操作。
在结束后，他会自动把所有操作自动保持到项目的文件里面去。
如果你是第一次测试，需要`Sync`下项目！
![这里写图片描述](http://img.blog.csdn.net/20151119121144319)

可以看到，我们的所有操作他都是记录下来的

![这里写图片描述](http://img.blog.csdn.net/20151119121447938)

我们也可以在这个基础上做一些改动，完全看自己的需要。

### 参考内容：
[User Guide Android Studio](http://robotium.com/pages/user-guide-android-studio)
 
  