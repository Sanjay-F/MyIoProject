title: 源码探索系列31---插件化基础之Service组件管理
date: 2016-04-15 19:53:46
tags: [android,源码,DroidPlugin,AMS,AMN,service]
categories: android

------------------------------------------
 
前面辛辛苦苦，终于分析完启动插件包的Activity的全部过程。
现在我们来看下四大金刚之一的另外一个Sevice。他和Activity一样，也需要提前占坑，然后HOOK才可以使用。就让我们来看性下整个过程会涉及到的过程，顺便温习下Service的启动流程吧。

<!--more-->

#起航
API:23

为了能够成功的HOOK，我们老规矩的，先找下HOOK点。
我们可以预测的是整个HOOK的过程和Activity一样，都有欺上瞒下，然后启动目标Servcie的过程。根据前面在[源码探索系列7---四大金刚之Service](http://sanjay-f.github.io/2015/12/18/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%977---%E5%9B%9B%E5%A4%A7%E9%87%91%E5%88%9A%E4%B9%8BService/)记录 