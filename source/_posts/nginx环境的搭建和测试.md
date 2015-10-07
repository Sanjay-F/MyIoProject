title: nginx环境的搭建和测试
date: 2015-10-06 13:32:07
tags: [nginx]
---

# 安装Tomcat

Tomcat官网：http://tomcat.apache.org/
我使用的版本：apache-tomcat-7.0.42.tar.gz
在桌面新建多一个文件叫Tomcat，解压到目录下（可以自己选）
复制多一分，分别叫tomcat1和tomcat2
打开tomcat2里面的`conf/server.xml`文件，然后编辑如下


注意：
要这么修改，是因为这两个都是跑在我自己的电脑上啊，如果你的是在不同的服务器下，可以不用的.

### 第一处端口修改：

	<!--  修改port端口为8002，另外tomcat的配置不动-->  
	<Server port="8002" shutdown="SHUTDOWN">   
注：端口随意，不重复就好，只是为了方便区别，我把这个叫8002；


###   第二处端口修改：
	<!-- port="8082" tomcat监听端口-->  
	<Connector port="8082" protocol="HTTP/1.1"   
	               connectionTimeout="20000"   
	               redirectPort="9443" />  
  注意，我这里把`redirectPort`修改了.
   
 
<!-- ###   第三处端口修改：  
	Engine元素增加 jvmRoute  属性：
	<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat2">  
注：另外一个tomcat1的server.xml文件，请在这个位置请加多`jvmRoute="tomcat1"` -->
	 
<!--more-->

# 安装nginx
   下载地址在：`http://nginx.org/en/download.html`，建议下载 Stable version版(我下载的是V1.8.0，用1.9.5好像有点小问题)安装在桌面叫`nginx`的文件夹内。
   
打开`nginx.conf`文件，对其进行配置，大慨在在中间点的地方，加入upstram，然后在server模块内加多一句话`proxy_pass http://tomcat;`。如下
	
	upstream tomcat{  # 负载均衡站点的名称为tomcat，可以自己取
	   # ip_hash;  # 可选，根据来源IP方式选择web服务器，
	               #省略的话按默认的轮循方式选择web服务器
	    server localhost:8080;       # web服务器的IP地址及tomcat发布端口
	    server localhost:8082;       #如果不是本地单机，修改下地址和端口。
	}
	        
	server {
	     listen       80;                   # 站点侦听端口80
	     server_name  localhost;            # 站点名称
	 
	     location / {
	        root   html;
	        index  index.html index.htm;
			
			#添加多下面这句
	        proxy_pass http://tomcat;      # 负载均衡指向的发布服务tomcat,这个要和上面的upstream的tomcat这个保持一致
	     }	  
	} 

### 一些讲解：

 你应该注意到，在我们的upstrem localhost里面有一个注释掉的一行代码`#ip_hash`,这个是nginx的负载方法之一，如果想开启这方法，取消注释就好了。 
 
## NGINX负载的几种方式

nginx 的 upstream目前支持`4` 种方式的分配
 
1. 轮询（默认） 
      每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。    
2. weight 
      指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。 
3. ip_hash 
      每个请求按访问ip的hash结果分配，这样每个ip固定访问一个后端服务器.      
4. fair（第三方） 
      按后端服务器的响应时间来分配请求，响应时间短的优先分配。  
4. url_hash（第三方）
      每个请求按访问url的hash结果分配，这样每个url固定指向一个后端服务器.
      
      
  很多人以为，使用ip_hash来实现，可以解决session问题，因为一个固定ip对应一个session一直都访问一个web。但ip_hash其实上不能够完全解决ip问题，因为有很多用户的ip随时都可能在变动，特别是移动终端，ip由他的附近的基站分配，一值在变，最好还是用memcached之类的缓存框架来存取来实现session共享。
  
  
#测试

 验证配置与测试负载均衡

   首先测试nginx配置是否正确，测试命令：
   `nginx -t`
   (默认验证:conf\nginx.conf),也可以指定配置文件路径。
 如果打印显示ok，那就是没问题。有问题就自己看下哪里弄错了。
    
上面我们算配置好了，利用intellij创建多两个模版springMvc项目，分别叫tomcat1Test和tomcat2Test。

配置他们的Application Server为我们一开始弄好的tomcat1和tomcat2.
一个简单的controller。
	
	@Controller
	@RequestMapping("/")
	public class HelloController {
		@RequestMapping(method = RequestMethod.GET)
		public String printWelcome(ModelMap model) {
			model.addAttribute("message", "Hello world!--from 1");
			return "hello";
		}
	}

然后shift＋f10运行这两个程序。确保两个程序正常运行。
在我们的终端启动: `sudo nginx`.
一切正常的话，访问`localhost`
刷新一下，就会看到来自不同页面的的结果。

#后记
 
 至此nginx+tomcat负载均衡配置结束，关于tomcat Session的问题通常是采用memcached，或者采用nginx_upstream_jvm_route ，他是一个 Nginx 的扩展模块，用来实现基于 Cookie 的 Session Sticky 的功能。如果tomcat过多(>=5台)，不建议session同步，server间相互同步session很耗资源，高并发环境容易引起Session风暴。特别是当系统崩溃重现恢复时候，简直可以把系统卡死。
 所以请根据自己应用情况合理采纳session解决方案。例如建一个集群，保存session和一些别的缓存等。
 

