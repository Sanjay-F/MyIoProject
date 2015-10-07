title: nginx的配置文件详解
date: 2015-10-06 13:33:06
tags: [nginx,conf]
---



这是一篇介绍关于nginx的配置文件的介绍文章

主要有负载均衡策略，重试策略 ，备机策略 三个模块.


<!--more-->

## 1.配置文件的内容如下

	#运行nginx所在的用户名和用户组
	#user  www www;
	   
	 
	##工作的子进程数量（通常等于CPU数量或者2倍于CPU）
	worker_processes 8;
	#全局错误日志及PID文件
	error_log  /usr/local/nginx/logs/nginx_error.log  crit;
	   
	pid        /usr/local/nginx/nginx.pid;
	   
	#Specifies the value for maximum file descriptors that can be opened by this process.
	   
	worker_rlimit_nofile 65535;
	
	#工作模式及连接数上限
	events
	{
	
	  use epoll;
	  #use 设置用于复用客户端线程的轮询方法。
	  #如果你使用Linux 2.6+，你应该使用epoll。
	  #如果你使用*BSD，你应该使用kqueue。	   
      #值得注意的是如果你不知道Nginx该使用哪种轮询方法的话，它会选择一个最适合你操作系统的。
	
	  worker_connections 65535;
	  #允许最大连接数  
	}
	
	#设定http服务器，利用它的反向代理功能提供负载均衡支持
	http
	{
	  #设定mime类型
	  include       mime.types;
	  #设置文件使用的默认的MIME-type。
	  default_type  application/octet-stream;
	   
	  include /usr/local/nginx/conf/proxy.conf;#只是一个在当前文件中包含另一个文件内容的指令。这里我们使用它来加载稍后会用到的一系列的MIME类型。
	  #charset  gb2312;#设置我们的头文件中的默认的字符集。还有UTF-8版本的，charset UTF-8;
	   
	  #设定请求缓冲   
	  server_names_hash_bucket_size 128;
	  client_header_buffer_size 32k;
	  large_client_header_buffers 4 32k;
	  #client_max_body_size 8m;
	         
	  sendfile on;
	  #可以让sendfile()发挥作用。sendfile()可以在磁盘和TCP socket之间互相拷贝数据(或任意两个文件描述符)。Pre-sendfile是传送数据之前在用户空间申请数据缓冲区。之后用read()将数据从文件拷贝到这个缓冲区，write()将缓冲区数据写入网络。sendfile()是立即将数据从磁盘读到OS缓存。因为这种拷贝是在内核完成的，sendfile()要比组合read()和write()以及打开关闭丢弃缓冲更加有效(更多有关于sendfile)
	  tcp_nopush     on; #告诉nginx在一个数据包里发送所有头文件，而不一个接一个的发送
	   
	   
	   
	  keepalive_timeout 60;
	  #keepalive_timeout 给客户端分配keep-alive链接超时时间。
	  #服务器将在这个超时时间过后关闭链接。我们将它设置低些可以让ngnix持续工作的时间更长。
	   
	  tcp_nodelay on;
	  #告诉nginx不要缓存数据，而是一段一段的发送–当需要及时发送数据时，就应该给应用设置这个属性，这样发送一小块数据信息时就不能立即得到返回值。
	  
	  
	  
	  access_log  off #设置nginx是否将存储访问日志。关闭这个选项可以让读取磁盘IO操作更快(aka,YOLO)。当然调试的时候还是开一下，看下有什么问题没
	  
	   
	   
	  
  	#fastcgi_cache_path /usr/local/nginx/fastcgi_cache levels=1:2 keys_zone=TEST:10m #inactive=5m; #这个指令为FastCGI缓存指定一个路径，目录结构等级，关键字区域存储时间和非活动删除时间。 
	#  fastcgi_connect_timeout 300;#指定连接到后端FastCGI的超时时间。 
	#  fastcgi_send_timeout 300;#向FastCGI传送请求的超时时间，这个值是指已经完成两次握手后向FastCGI传送请求的超时时间。 
	#  fastcgi_read_timeout 300;#接收FastCGI应答的超时时间，这个值是指已经完成两次握手后接收FastCGI应答的超时时间。 
	
   
	#  fastcgi_buffer_size 64k;	#指定读取FastCGI应答第一部分需要用多大的缓冲区，这里可以设置为fastcgi_buffers指令指定的缓冲区大小，上面的指令指定它将使用1 个16k的缓冲区去读取应答的第一部分，即应答头，其实这个应答头一般情况下都很小（不会超过1k），但是你如果在fastcgi_buffers指令中 指定了缓冲区的大小，那么它也会分配一个fastcgi_buffers指定的缓冲区大小去缓存。
	
	#  fastcgi_buffers 4 64k;
	#指定本地需要用多少和多大的缓冲区来缓冲FastCGI的应答，如上所示，如果一个php脚本所产生的页面大小为256k，则会为其分配16个16k的缓 冲区来缓存，如果大于256k，增大于256k的部分会缓存到fastcgi_temp指定的路径中，当然这对服务器负载来说是不明智的方案，因为内存中 处理数据速度要快于硬盘，通常这个值的设置应该选择一个你的站点中的php脚本所产生的页面大小的中间值，比如你的站点大部分脚本所产生的页面大小为 256k就可以把这个值设置为16 16k，或者4 64k 或者64 4k，但很显然，后两种并不是好的设置方法，因为如果产生的页面只有32k，如果用4 64k它会分配1个64k的缓冲区去缓存，而如果使用64 4k它会分配8个4k的缓冲区去缓存，而如果使用16 16k则它会分配2个16k去缓存页面，这样看起来似乎更加合理。
	
	
	#  fastcgi_busy_buffers_size 128k;#这个指令我也不知道是做什么用，只知道默认值是fastcgi_buffers的两倍。
	#  fastcgi_temp_file_write_size 128k;
	#在写入fastcgi_temp_path时将用多大的数据块，默认值是fastcgi_buffers的两倍。

    ##!!gzip这个很重要，提升性能的重点啊
	##!!gzip这个很重要，提升性能的重点啊
	##!!gzip这个很重要，提升性能的重点啊

	#  gzip on;
	#  gzip_disable "msie6"; #为指定的客户端禁用gzip功能。我们设置成IE6或者更低版本以使我们的方案能够广泛兼容。****
	# gzip_static on; ##告诉nginx在压缩资源之前，先查找是否有预先gzip处理过的资源。这要求你预先压缩你的文件（在这个例子中被注释掉了），从而允许你使用最高压缩比，这样nginx就不用再压缩这些文件了（想要更详尽的gzip_static的信息，请点击这里）。
	#  gzip_min_length  1k; #设置对数据启用压缩的最少字节数。如果一个请求小于1000字节，我们最好不要压缩它，因为压缩这些小的数据会降低处理此请求的所有进程的速度。
	#gzip是告诉nginx采用gzip压缩的形式发送数据。这将会减少我们发送的数据量
	#  gzip_buffers     4 16k;
	#  gzip_http_version 1.0;
	#  gzip_comp_level 4;  #设置数据的压缩等级。这个等级可以是1-9之间的任意数值，9是最慢但是压缩比最大的。我们设置为4，这是一个比较折中的设置。
	#  gzip_types       text/plain application/x-javascript text/css application/xml; #设置需要压缩的数据格式。上面例子中已经有一些了，你也可以再添加更多的格式。
	#  gzip_vary on;



	  #limit_zone  crawler  $binary_remote_addr  10m;
	  
	  
	upstream tomcat{                      # 负载均衡站点的名称为tomcat，可以自己取
	   # ip_hash;                    # 可选，根据来源IP方式选择web服务器，省略的话按默认的轮循方式选择web服务器
	    server 127.0.0.1:8080;       # web服务器的IP地址及tomcat发布端口
	    server 127.0.0.1:8081;

	    ＃server localhost:8080 weight=1;
       ＃server localhost:9999 weight=5;
       ＃如果你的两台机器性能不一样，一台牛逼，一台配置渣渣，那么给牛逼的那台分配多点任务，使用weight＝5这样的大一些的权重，来让他多干点活。当然具体数值应该根据实际的性能比例来调这个数值
	        }
	server {
	     listen       80;                   # 站点侦听端口80
	     server_name  localhost;            # 站点名称

	  location / {
	        root   html;
	        index  index.html index.htm;
	        proxy_pass http://tomcat;      # 负载均衡指向的发布服务tomcat
	          }
	}
	}


## 2、负载均衡策略

Nginx 提供轮询（round robin）、用户 IP 哈希（client IP）和指定权重 3 种方式。

默认情况下，Nginx 会为你提供轮询作为负载均衡策略。但是这并不一定能够让你满意。比如，某一时段内的一连串访问都是由同一个用户 Michael 发起的，那么第一次 Michael 的请求可能是 backend2，而下一次是 backend3，然后是 backend1、backend2、backend3…… 在大多数应用场景中，这样并不高效。当然，也正因如此，Nginx 为你提供了一个按照 Michael、Jason、David 等等这些乱七八糟的用户的 IP 来 hash 的方式，这样每个 client 的访问请求都会被甩给同一个后端服务器。具体的使用方式如下：

	 upstream backend {
	    ip_hash;
	    server backend1.example.com;
	    server backend2.example.com;
	    server.backend3.example.com;
	}
 
这种策略中，用于进行 hash 运算的 key，是 client 的 C 类 IP 地址（C 类 IP 地址就是范围在 192.0.0.0 到 223.255.255.255 之间，前三段号码表示子网，第四段号码为本地主机的 IP 地址类别）。这样的方式保证一个 client 每次请求都将到达同一个 backend。当然，如果所 hash 到的 backend 当前不可用，则请求会被转移到其他 backend。
 
再介绍一个和 ip_hash 配合使用的关键字：`down`。

当某个一个 server 暂时性的宕机（down）时，你可以使用“down”来标示出来，并且这样被标示的 server 就不会接受请求去处理。具体如下：
	
	 upstream backend {
	    server blog.csdn.net/poechant down;
	    server 145.223.156.89:8090;
	    server unix:/tmp/backend3;
	}
 
还可以使用指定权重（weight）的方式，如下：
	
	 upstream backend {
	    server backend1.example.com;
	    server 123.321.123.321:456 weight=4;
	} 

默认情况下 weight 为 1，对于上面的例子，第一个 server 的权重取默认值 1，第二个是 4，所以相当于第一个 server 接收 20% 的请求，第二接收 80% 的。要注意的是 weight 与 ip_hash 是不能同时使用的，原因很简单，他们是不同且彼此冲突的策略。

## 3、重试策略

可以为每个 backend 指定最大的重试次数，和重试时间间隔。所使用的关键字是 max_fails 和 fail_timeout。如下所示：

	upstream backend {
		server backend1.example.com weight=5;
		server 54.244.56.3:8081 max_fails=3 fail_timeout=30s;
	} 
在上例中，最大失败次数为 3，也就是最多进行 3 次尝试，且超时时间为 30秒。max_fails 的默认值为 1，fail_timeout 的默认值是 10s。传输失败的情形，由 proxy_next_upstream 或 fastcgi_next_upstream 指定。而且可以使用 proxy_connect_timeout 和 proxy_read_timeout 控制 upstream 响应时间。

有一种情况需要注意，就是 upstream 中只有一个 server 时，max_fails 和 fail_timeout 参数可能不会起作用。导致的问题就是 nginx 只会尝试一次 upstream 请求，如果失败这个请求就被抛弃了 : ( ……解决的方法，比较取巧，就是在 upstream 中将你这个可怜的唯一 server 多写几次，如下：

	upstream backend {
		server backend.example.com max_fails fail_timeout=30s;
		server backend.example.com max_fails fail_timeout=30s;
		server backend.example.com max_fails fail_timeout=30s;
	} 
## 4、备机策略

从 Nginx 的 0.6.7 版本开始，可以使用“backup”关键字。当所有的非备机（non-backup）都宕机（down）或者繁忙（busy）的时候，就只使用由 backup 标注的备机。必须要注意的是，backup 不能和 ip_hash 关键字一起使用。举例如下：

	upstream backend {
		server backend1.example.com;
		server backend2.example.com backup;
		server backend3.example.com;
	}
