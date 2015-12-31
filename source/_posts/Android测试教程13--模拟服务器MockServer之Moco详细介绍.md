title: Android测试教程13--模拟服务器MockServer之Moco详细介绍
date: 2015-12-07 15:33 
tags: [android,测试]
categories: android
---


 前面一篇介绍了如何用mockito来测试我们的一些异步任务，例如网络请求时候的异步回调。
现在做进一步的介绍，一个模拟我们的服务器的东东--[moco](https://github.com/dreamhead/moc)

<!--more-->

# 先运行起来
请先[下载这个文件moco.jar](http://download.csdn.net/detail/f112122/9333217)，接着，在我们的桌面新建一个文件data.json，用记事本打开，粘贴下面的东东 ，具体意思后面介绍

	[{ "request" : { "uri" : "/hello" }, "response" : { "text" : "Hello  World !!!" } } ]
保存好，然后打开我们的终端或者CMD。
输入下面的命令

	java -jar moco-runner-0.10.2-standalone.jar start -p 5638 -c data.json

我比较简单粗暴，直接把桌面文件拉过去，CMD自动填好地址，所以看到都是Desktop上的。 
回车，确认后，会有下面的信息 

	INFO Server is started at 5638
	INFO Shutdown port is 52384

![这里写图片描述](http://img.blog.csdn.net/20151207134614445)

 所以现在，我们打开我们的游览器， 输入

	http://localhost:5638/hello
理论上没有出错，那么返回的就是`hello world!!!`
**如下：** 
![这里写图片描述](http://img.blog.csdn.net/20151207135247014)

## 解释：

	java -jar moco-runner-0.10.2-standalone.jar start -p 5638 -c data.json

这个命令的意思是要mock监听本地的5638端口，对应的请求返回的数据在我们的data.json里面。
当有网络请求到来的时候，我们的mock就会去查`data.json`，把`request`为`hello`对应的数据返回，
我们写的是 `Hello  World !!!`，所以游览器返回这个。

如果没有这个请求，就返回空。

如果要**添加新的请求**的，可以这样写

	[{"request":{ "uri": "/hello"}, "response" : { "text": "Hello World" }}  ,
	 {"request":{ "uri": "/go"}, "response": { "text" : "go away !!!" }}  ]
加个逗号`，`接着复制多一份，改下请求的uri。ok，就这样，是不是很简单的感觉？？
那么更复杂的请求怎么办呢？例如有post数据的情况。
 

# 复杂的请求
显然我们的请求确实可能这么简单，最少都是带参数的，例如要获取`jack`用户信息，请求像下面这样

	http://localhost:5638/user/getUser?name=jack

那么我们的数据该怎么写呢？

	{ 
	 "request" : { 
		"uri" : "/user/getUser",
		 "queries": { "name":"jack" } 
	  },
	  "response" : {
		   "text" : "Hey. I'm jack" 
		 }  
    }

## 正则表达式

对于rest风格的url，Moco支持正则匹配。

	{ 
	  "request": { 
		   "uri": { 
	       "match": "/getUser/\\w+" } 
	    },
	  "response": {
		   "text": "Find a boy."
	   } 
	}


## POST , GET , PUT , DELETE 等方法

除了Get外，Post，Put，Delete等请求模式自然是支持的，格式如下，加多`method`和`Forms`参数

	{
	  "request" :{
	      "method" : "post",
	       "uri": "/getUser",
	      "forms" :{
	          "name" : "jack"
	       }
	    },
	  "response" : {
	      "text" : "Hi. you get jack data"
	    }
	} 

我们用postman来做模拟，返回数据正确！

![这里写图片描述](http://img.blog.csdn.net/20151207143746997)
 

## Headers ,  Cookies  , StatusCode
这个是支持特定的头的和cookies是支持的，我们需要加的就是`headers`，`cookies` 和`status`属性，参考如下

	{
	  "request" :{
	      "method" : "post",
	      "headers" : {
	        "content-type" : "application/json"
	      }
	    },
	  "response" :{
    	  "status" : 200,
	      "text" : "bar",
	       "cookies" :{
	          "login" : "true"
          },
	       "headers" :{
              "content-type" : "application/json"
           }
	    }
	}

## Json
我们对返回的数据，可以定义为Json的，格式如下

	{
		  "request" :{
	              "uri": "/getJson",
			      "method" : "post",			      
		   },
		  "response" :{
	    	  "status" : 200,
	    	   "headers" :{
	              "content-type" : "application/json"
	           },
			  "json": {
	                "foo": "bar"
	               }		       
		  }
	}


## 重定向

介绍一个redirect

	{ 
	  "request" : { "uri" : "/redirect" }, 
	  "redirectTo" : "http://www.XXX.com" 
	  }
	
 
## 延迟
我觉得这个还是个重要的属性，因为移动手机的网络环境很复杂，高RTT不是盖的，网络延迟个几十秒的那也是正常的，所以我们需要一个`latency`

	{
	  "request" :{
	      "text" : "foo"
	    },
	  "response" :{
	      "latency": {
	          "duration": 1,
	          "unit": "second"
	        }
	    }
	}



##template
上面的请求参数的值和返回值都是固定的，这自然太过死板。
好在从0.8版本开始，Moco提供了`template`功能，可以动态的返回一些参数值，虽然是beta版本。

例如：

	  {
	    "request": {
	        "uri": "/getUser2"
	        },
	    "response": {
	          "text": {
	             "template": "${req.queries['name']}"
	                 }
            }
	 }
这样，但当我们的请求是`localhost:5638/getUser2?name=nameIsJack`,那么返回的数据是`nameIsJack`

![这里写图片描述](http://img.blog.csdn.net/20151207145001447)

。。。。这个CSDN真是，写一半就又崩溃了。传图片又不可以，高可用的服务器啊。

除了上面的，还有

	"template": "${req.version}"
	"template": "${req.method}"
	"template": "${req.content}"
	"template": "${req.headers['foo']}"
	"template": "${req.queries['foo']}"
	"template": "${req.forms['foo']}"
	"template": "${req.cookies['foo']}"

这些方法上面都用过了，就不一  一解释了。

##Event事件
从0.9版本开始，mock提供了event方法，什么意思呢
有时候，我们请求一些特定接口的时候，我们可能需要去请求别的地址，从而才能完成请求。
例如OAuth等，牵涉到第三方的情况。这时候，Event就派上大用场了
 
	 {
	    "request": {
	        "uri" : "/event"
	    },
	    "response": {
	        "text": "event"
	    },
	    "on": {
	        "complete": {
	            "get" : {
	                "url" : "http://another_site/OAuth?xxx=xxxx"
	            }
	        }
	    }
	}
这样我们就可以等去验证完权限部分，才返回结果。



## 异步请求 Asynchronous

前面的请求默认都是同步的，这意味着只有等到服务器处理完，结果才会返回给客户端
如果你的实际情况不是这样，需要来点异步的，那么从0.9版本开始，有这个功能了，另外你还可以延迟个5秒，像这样

	{
		    "request": {
		        "uri" : "/event"
		    },
		    "response": {
		        "text": "event"
		    },
		    "on": {
		        "complete": {
		        "async" : "true",
			    "latency" : 5000,
		            "post" : {
		                "url" : "http://www.baidu.com",
		                "content": "content"
		            }
		        }
		    }
		}
	
 
	 
---
# 分模块

我们的请求很显然是有很多的，例如有u
ser模块的userApi业务，聊天模块的chatApi业务等等，
如果把这些不同的模块的业务都写在一个文件里面，那就太难看了，为此，需要一些明智点的解决方案
下面以一个`TodoList`的作为介绍

	[ { "context": "/user", "include": "user.json" }, { "context": "/todo", "include": "todo.json" } ]

这样，当我们访问
 `http://localhost:12306/user/create` 和 `http://localhost:12306/todo/getAll`时候，
 会跳到后面对应的json再处理一遍

	//user
	[ { "request" : { "uri" : "/create" }, "response" : { "text" : "这是创建用户请求" } } ]
	//todo
	[ { "request" : { "uri" : "/getAll" }, "response" : { "text" : "这是获取用户所有Todo的请求" } } ]
	 


 Moco支持在全局的配置文件中引入其他配置文件，这样就可以分服务定义配置文件，便于管理。
 配置好文件，在全局文件中引入即可： 
 全局配置如下：
 
	java -jar moco-runner-standalone.jar start -p 5638 -g data.json

**注意，此时需要通过参数  `-g`  在加载全局配置文件。**,使用的不是`-c`了

否则配置文件解析会报错。
  
---
 
# 后记
再github上star了这个项目，发发现还是大神郑晔写的，CSDN也曾做过采访
[【开源专访】郑晔谈Moco框架的前世今生以及Java编程之道](http://www.csdn.net/article/2013-08-23/2816684-Moco-Java-framework)

 

---

来源：

[Mock Server- Moco 使用指南](http://www.coderli.com/mock-server-moco-guide/) 
[更详细的介绍，看官方文档吧](https://github.com/dreamhead/moco/blob/master/moco-doc/apis.md)

 