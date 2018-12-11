title: Flutter入门系列10---关于状态管理BLoC

date: 2018-04-28 23:14:46

tags: [android,Flutter,BLoC]

categories: flutter

------------------------------------------


做过web的同学应该速度React+Redux这个东西，到了Flutter，我们也需要有个东西来做不同页面的跳转逻辑，数据的分发管理等问题。

因为一个程序大了，界面很多的，逻辑很复杂，这时候需要有个东西来做管理啊

目前我们工程有尝试用到了[BLoC](https://hk.saowen.com/a/fbb6e484de022173fe85248875286060ce40d069c97420bc0be49d838e19e372)这个东西，在这里做简单介绍下



BLoC的全称是 业务逻辑组件（Business Logic Component）。
如果你对响应式编程熟悉的话，应该很好懂这个到底是什么

![](https://i.imgur.com/xt3xilA.png)


BLoC模式由来自Google的Paolo Soares和Cong Hui设计，并在2018年DartConf期间（2018年1月23日至24日）首次展示。 在YouTube上观看此视频。

 

简而言之，业务逻辑（Business Logic ）需要：

- 转移到一个或几个BLoC，
- 尽可能从表现层中删除。 换句话说，UI组件应该只关心UI事物而不关心业务，
- 依赖Streams独家使用输入（Sink）和输出（流），
- 保持平台独立，
- 保持环境独立。

事实上，BLoC模式最初被设想为允许独立于平台重用相同的代码：Web应用程序，移动应用程序，后端。

<!--more-->

# 起航

我们看回上面的图，就是像事件总线一样来做解耦工作

1. Widgets通过Sinks向BLoC发送事件，
2. BLoC通过Stream通知Widgets，
3. 由BLoC实现的业务逻辑不是他们关注的问题。

从上面来看，我们可以直接看到使用BLoC的一个巨大的好处。
感谢业务逻辑与UI的分离：

我们可以随时更改业务逻辑，对应用程序的影响最小，
我们可能会更改UI而不会对业务逻辑产生任何影响，
现在，测试业务逻辑变得更加容易。

#ref 

- https://hk.saowen.com/a/fbb6e484de022173fe85248875286060ce40d069c97420bc0be49d838e19e372

- https://cloud.tencent.com/developer/article/1345645

- https://yq.aliyun.com/articles/640862