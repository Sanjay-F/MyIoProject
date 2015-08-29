title: 基于ApacheServer+SpringMvc+Tomcat+JK搭建的简单集群教程
date: 2015-08-29 15:16:19
tags: [tomcat,集群,Apache,SpringMvc]
---

# 基础知识
在深入了解前，有必要先把了解下一些基础的知识，这样才能更好的理解。

 Web客户访问Tomcat服务器上JSP组件的两种方式如下图所示。
  
![Web客户访问Tomcat服务器上的JSP组件的两种方式](http://img.blog.csdn.net/20150828161212922)
                    
  在图中，
  Web客户1直接访问Tomcat服务器上的JSP组件，他访问的URL为http://localhost:8080 /index.jsp。
  Web客户2通过HTTP服务器访问Tomcat服务器上的JSP组件。假定HTTP服务器使用的HTTP端口为默认的80端口，那么Web客户2访问的URL为http://localhost:80/index.jsp 或者 http://localhost/index.jsp。

　　下面，介绍Tomcat与HTTP服务器之间是如何通信的。

<!--more-->

　　`Tomcat`服务器通过`Connector`连接器组件与客户程序建立连接，
　　Connector组件负责接收客户的请求，以及把Tomcat服务器的响应结果发送给客户。
     默认情况下，Tomcat在server.xml中配置了两种连接器：
	
	 <!-- Define a non-SSL Coyote HTTP/1.1
	　　Connector on port 8080 -->
	　　<Connector port="8080"
	　　maxThreads="150"
	　　minSpareThreads="25"
	　　maxSpareThreads="75"
	　　enableLookups="false"
	　　redirectPort="8443"
	　　acceptCount="100"
	　　debug="0"
	　　connectionTimeout="20000"
	　　disableUploadTimeout="true" />
	
	　<!-- Define a Coyote/JK2 AJP 1.3
	　　Connector on port 8009 -->
	　　<Connector port="8009"
	　　enableLookups="false"
	　　redirectPort="8443" debug="0"
	　　protocol="AJP/1.3" />
 
**解释：**

1. 第一个连接器监听8080端口，负责建立HTTP连接。

　　在通过浏览器访问Tomcat服务器的Web应用时，使用的就是这个连接器。

2. 第二个连接器监听8009端口，负责和其他的HTTP服务器建立连接。

　　在把Tomcat与其他HTTP服务器集成时，就需要用到这个连接器。

　　 
## JK插件

　　Tomcat提供了专门的JK插件来负责`Tomcat`和`HTTP`服务器的通信。应该把JK插件安置在对方的HTTP服务器上。当HTTP服务器接收到客户请求时，它会通过JK插件来过滤URL，JK插件根据预先配置好的URL映射信息，决定是否要把客户请求转发给Tomcat服务器处理。

　　假定在预先配置好的URL映射信息中，所有"/*.jsp"形式的URL都由Tomcat服务器来处理，那么在图22-1的例子中，JK插件将把客户请求转发给Tomcat服务器，Tomcat服务器于是运行index.jsp，然后把响应结果传给HTTP服务器，HTTP服务器再把响应结果传给Web 客户2。

　　对于不同的HTTP服务器，Tomcat提供了不同的JK插件的实现模块。本章将用到以下JK插件：
　　与Windows下的Apache HTTP服务器集成：mod_jk_2.0.46.dll
　　与Linux（RedHet）下的Apache HTTP服务器集成：mod_jk.so-ap2.0.46-rh72..46-rh72
　　与IIS服务器集成：isapi_redirect.dll
　　
## AJP协议

　　AJP是为Tomcat与HTTP服务器之间通信而定制的协议，能提供较高的通信速度和效率。
　　在配置Tomcat与HTTP服务器集成中，读者可以不必关心AJP协议的细节。关于AJP的知识也可以参考网址：
　　http://jakarta.apache.org/builds/jakarta-tomcat-connectors/jk2/doc/common/AJPv13.html

# 在Windows下Tomcat与Apache服务器集成

有了上面的基础知识后，我们就可以开始工作了,先下载需要的东西

1. Apache Server 
http://pan.baidu.com/s/1eQ98aVC

2. mod_jk.so ( 主要作用是建立 Apache Server 与 Tomcat 之间的连接 )
http://pan.baidu.com/s/1dDuHaST

3. Tomcat
http://tomcat.apache.org/download-70.cgi

4. Jdk1.7
下载地址： http://java.sun.com

---

我们在本地的集群配置效果图；

![配置效果](http://img.blog.csdn.net/20150828172905843)
  
## 下载安装，配置文件

新建一个文件夹，叫devel，然后把 Jdk 、 tomcat 、 apache 等都安装在这个目录下,
然后把tomcat**复制多一份**，命名做`tomcat1`和`tomcat2`
（因为我们需要连接两个不同的tomcat，每个tomcat上跑我们的项目，正如图片上显示的）

1. 把下载的 mod_jk- 1.2.31 -httpd-2.2.3.so 文件拷贝到你的`apache\modules` 下。
2. 在` apache\conf `目录下新建 `mod_jk.conf` 和 `workers.properties` 文件。

### 修改mod_jk.conf，配置Apache-Server和Jk两者的连接

	# Load mod_jk2 module
	LoadModule jk_module modules/mod_jk-1.2.31-httpd-2.2.3.so
	#注意，这句 modules/mod_jk-1.2.31-httpd-2.2.3.so就是我们刚才放到modules目录下的
	#mod_jk-1.2.31-httpd-2.2.3.so文件，请保证两者名字一样。
	 
	# Where to find workers.properties( 引用 workers 配置文件 )
	# 通过这个配置文件来告诉jk有哪些服务器可以连接的。
	# 以后如果加入新的服务器,就在workers.properties中加入新的服务器地址就可以了。
	JkWorkersFile conf/workers.properties
	 
	# Where to put jk logs(log 文件路径 )
	JkLogFile logs/mod_jk2.log
	 
	# Set the jk log level [debug/error/info](log 级别 )
	JkLogLevel info
	      
	# Select the log format(log 格式 )
	JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
	# JkOptions indicate to send SSL KEY SIZE,
	JkOptions +ForwardKeySize +ForwardURICompat -ForwardDirectories
	 
	# JkRequestLogFormat set the request format
	JkRequestLogFormat "%w %V %T"
	 
	# Send JSPs for context / to worker named loadBalancer(URL 转发配置，匹配的 URL 才转发到 tomcat 进行处理 )
	JkMount /*.jsp controller
	# JkMount /*.* loadBalancer

### 修改`workers.properties`文件，告诉Server有哪些服务器可连接的
	#server  列表
	worker.list =  controller
	 
	# tomcat1(ajp13  端口号，在tomcat下server.xml配置,默认8009)
	worker.tomcat1.port=8009
	#tomcat 的主机地址，如不为本机，请填写ip地址
	worker.tomcat1.host=localhost
	worker.tomcat1.type=ajp13
	#server 的加权比重，值越高，分得的请求越多，如果你的其中一台配置高，就把这个数值按性能调大些.
	worker.tomcat1.lbfactor = 1
	 
	# tomcat2
	worker.tomcat2.port=9009
	worker.tomcat2.host=localhost
	worker.tomcat2.type=ajp13
	worker.tomcat2.lbfactor = 1
	 
	# controller( 负载均衡控制器)
	worker.controller.type=lb
	#  指定分担请求的tomcat
	worker.controller.balanced_workers=tomcat1,tomcat2
	#worker.controller.sticky_session=true

通过这个配置文件，我们告诉Apache-Server,去哪里连接我们的2 个 tomcat 服务器，然后进行负载均衡处理，把请求分配到两个服务器上。

### 在 httpd.conf 里面的最后加上：
	# JK module settings
	Include conf/mod_jk.conf   
说明：以上表示将 mod_jk.conf 配置文件包含进来。这个文件也在当前的`apache/modules`里面.

## 修改Tomcat的端口
上面配置完后，我们基本配置好了，不过还需要改下我们的tomcat配置，因为我们在`workers.properties`告诉Apache-Server了。两个服务器的地址分别是在`localhost:8009`和`localhost:9009`,所以我们需要修改了tomcat的端口配置。

### 修改Tomcat1
打开`Tomcat1/conf/server.xml`,在里面找到

    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
这个就是我们在 workers.properties中申明的tomcat1的接口,应为我们用的是默认配置，所以prot属性就不用改了，不过下面的tomcat2就要了。
**把他改成下面这长串**

	    <!-- Define an AJP 1.3 Connector on port 8009 -->
	    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" 
		  URIEncoding="UTF-8"  minSpareThreads="25" maxSpareThreads="75" 
	       enableLookups="false" disableUploadTimeout="true" connectionTimeout="20000"
			acceptCount="300"  maxThreads="300" maxProcessors="1000" minProcessors="5"
			useURIValidationHack="false"
	        compression="on" compressionMinSize="2048"
			compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain" 
			/>

这里面有些小的点需要强调的，`URIEncoding="UTF-8"`这个显示的申明，我们用的是UTF-8,这可以避免我们以后数据出现乱码问题。其余的一些配置，你可以根据实际的需要修改.

**接着**找到
`    <Engine name="Catalina" defaultHost="localhost"> `,
改成这样
`<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat1">`

就是在屁股加多`jvmRoute="tomcat1"。`

**然后**，我们在这句话的下面，插入多这么长串的内容：

	<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster" 
	                  channelSendOptions="6"> 
	              <Manager className="org.apache.catalina.ha.session.BackupManager" 
	                    expireSessionsOnShutdown="false" 
	                    notifyListenersOnReplication="true" 
	                    mapSendOptions="6"/> 
	           <Channel className="org.apache.catalina.tribes.group.GroupChannel"> 
	             <Membership className="org.apache.catalina.tribes.membership.McastService" 
	                         bind="127.0.0.1" 
	                         address="228.0.0.4" 
	                         port="45564" 
	                         frequency="500" 
	                         dropTime="3000"/> 
	             <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"  
	                       address="auto"  
	                       port="4001"  
	                       selectorTimeout="100" 
	                       maxThreads="6"/> 
	             <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter"> 
	               <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" timeout="60000"/>  
	             </Sender> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/> 
	           </Channel> 
	              <Valve className="org.apache.catalina.ha.tcp.ReplicationValve" 
	                  filter=".*\.gif;.*\.js;.*\.jpg;.*\.png;.*\.htm;.*\.html;.*\.css;.*\.txt;"/> 
	              <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/> 
	         </Cluster>

具体的含义你可以慢慢看，就不解释了。
### 修改Tomcat2
接着我们来改下tomcat2吧
打开`Tomcat2/conf/server.xml`,在里面找到

`<Server port="8005" shutdown="SHUTDOWN">`
改成
`<Server port="9005" shutdown="SHUTDOWN">`
因为我们是单机啊！这个端口已经被我们的Tomcat1霸占了啊，所以我们需要修改下啊！

**接着**

    <!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
 
**把他改成下面这长串**

	    <!-- Define an AJP 1.3 Connector on port 8009 -->
	    <Connector port="9009" protocol="AJP/1.3" redirectPort="9443" 
		  URIEncoding="UTF-8"  minSpareThreads="25" maxSpareThreads="75" 
	       enableLookups="false" disableUploadTimeout="true" connectionTimeout="20000"
			acceptCount="300"  maxThreads="300" maxProcessors="1000" minProcessors="5"
			useURIValidationHack="false"
	        compression="on" compressionMinSize="2048"
			compressableMimeType="text/html,text/xml,text/javascript,text/css,text/plain" 
			/>

注意，我们的`port="9009"`，`redirectPort="9443"`这个是修改后的

接着找到`    <Engine name="Catalina" defaultHost="localhost"> `,改成这样
`<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat2">`
在屁股加多`jvmRoute="tomcat2"`。
然后，我们在这句话的下面，插入多这么长串的内容：

	<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster" 
	                  channelSendOptions="6"> 
	              <Manager className="org.apache.catalina.ha.session.BackupManager" 
	                    expireSessionsOnShutdown="false" 
	                    notifyListenersOnReplication="true" 
	                    mapSendOptions="6"/> 
	           <Channel className="org.apache.catalina.tribes.group.GroupChannel"> 
	             <Membership className="org.apache.catalina.tribes.membership.McastService" 
	                         bind="127.0.0.1" 
	                         address="228.0.0.4" 
	                         port="45564" 
	                         frequency="500" 
	                         dropTime="3000"/> 
	             <Receiver className="org.apache.catalina.tribes.transport.nio.NioReceiver"  
	                       address="auto"  
	                       port="4002"  
	                       selectorTimeout="100" 
	                       maxThreads="6"/> 
	             <Sender className="org.apache.catalina.tribes.transport.ReplicationTransmitter"> 
	               <Transport className="org.apache.catalina.tribes.transport.nio.PooledParallelSender" timeout="60000"/>  
	             </Sender> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.TcpFailureDetector"/> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.MessageDispatch15Interceptor"/> 
	             <Interceptor className="org.apache.catalina.tribes.group.interceptors.ThroughputInterceptor"/> 
	           </Channel> 
	              <Valve className="org.apache.catalina.ha.tcp.ReplicationValve" 
	                  filter=".*\.gif;.*\.js;.*\.jpg;.*\.png;.*\.htm;.*\.html;.*\.css;.*\.txt;"/> 
	              <ClusterListener className="org.apache.catalina.ha.session.ClusterSessionListener"/> 
	         </Cluster>

 此处有一个Receiver port=”xxxx”，两个tomcat中此处的端口号必须唯一，即tomcat中我们使用的是port=4001，那么我们在tomcat2中将使用port=4002

---

这样我们基本配置完毕了，现在开启我们的intellij（用过的人都说好，请抛弃Eclipse，太破了），新建两个模板的springMvc项目！

	@Controller
	@RequestMapping("/")
	public class HelloController {
	
	    @RequestMapping(method = RequestMethod.GET)
	    public String printWelcome(ModelMap model) {
	        model.addAttribute("message", "Hello world!--from tomcat1");
	        return "hello";
	    }
	}
**然后**
布署工程前先确保你把工程中的WEB-INF\we b.xml文件做了如下的修改，
在web.xml文件的最未尾即“</web-app>”这一行前加入如下的一行：

	<distributable/>

使该工程中的session可以被tomcat的集群节点进行轮循复制。

 这样我们基本就做完了所有，是时候来试下我们的效果了！运行下我们的项目，
分别访问http://localhost:8080/与http://localhost:9090/
看下返回的是否为我们的标准语句`Hello world!--from tomcatX`
然后启动Apache_server
 ![ 启动apache](http://img.blog.csdn.net/20150829115428074)
 
然后访问直接用http://localhost/  
不加端口的形式访问,如果返回了我们需要的信息，恭喜你，成功了，搭建好一个简单的集群了！！！！如果不成功，请看下面的总结


---

# 总结

1. 启动apache，错误代码1；
  这个的错误原因可能是我们的apache使用的80端口被我们的另外程序被霸占了，需要修改下，打开`apche/conf/httpd.conf`文件，然后找到listen字段，修改它后面的80端口为一个新的端口。
  另外一种可能是我们的配置文件写错了，请打开看下是否和我写的一样，还是说出现了乱码，导致了bug。
2. 为何每次都返回来自同一个服务器的信息？
这个我也苦闷了很久，为何每次刷新，返回的都是from tomcat1!!,后来想可能是因为我们刷新的速度太慢，达不到并发的程度，所以就试着关闭tomcat1.结果真的就返回了 from tomca2!!。
另外一个原因可能端口没配置对，你自检查下。
如果你看到另外一个intellij的后台打印了一堆的红色的字，说没找到对`/favicon`的请求，那么显然，你明白你的是成功的了。
![这里写图片描述](http://img.blog.csdn.net/20150829150723161)
还有就是需要等待下同步，这样apache_Server才反应得过来啊，因为他会隔一段时间，检测这两个服务器是否在运行，没有就不会转发请求过来，所以我们需要注意这个问题啦。

3. 环境配置问题
 把系统环境变更中的CATALINA_HOME与TOMCAT_HOME这两个变量去除掉

