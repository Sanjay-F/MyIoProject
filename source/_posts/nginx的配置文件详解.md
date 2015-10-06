title: nginx的配置文件详解
date: 2015-10-06 13:33:06
tags: [nginx,conf]
---



这是一篇介绍关于nginx的配置文件的介绍文章

主要有负载均衡策略，重试策略 ，备机策略 三个模块.


<!--more-->

## 1.配置文件的全部内容如下

	#运行nginx所在的用户名和用户组
	#user  www www;
	   
	#启动进程数
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
	  worker_connections 65535;
	}
	#设定http服务器，利用它的反向代理功能提供负载均衡支持
	http
	{
	  #设定mime类型
	  include       mime.types;
	  default_type  application/octet-stream;
	  include /usr/local/nginx/conf/proxy.conf;
	  #charset  gb2312;
	  #设定请求缓冲   
	  server_names_hash_bucket_size 128;
	  client_header_buffer_size 32k;
	  large_client_header_buffers 4 32k;
	  #client_max_body_size 8m;
	         
	  sendfile on;
	  tcp_nopush     on;
	   
	  keepalive_timeout 60;
	   
	  tcp_nodelay on;
	   
	#  fastcgi_connect_timeout 300;
	#  fastcgi_send_timeout 300;
	#  fastcgi_read_timeout 300;
	#  fastcgi_buffer_size 64k;
	#  fastcgi_buffers 4 64k;
	#  fastcgi_busy_buffers_size 128k;
	#  fastcgi_temp_file_write_size 128k;
	   
	#  gzip on;
	#  gzip_min_length  1k;
	#  gzip_buffers     4 16k;
	#  gzip_http_version 1.0;
	#  gzip_comp_level 2;
	#  gzip_types       text/plain application/x-javascript text/css application/xml;
	#  gzip_vary on;
	   
	  #limit_zone  crawler  $binary_remote_addr  10m;
	upstream tomcat{                      # 负载均衡站点的名称为tomcat，可以自己取
	   # ip_hash;                    # 可选，根据来源IP方式选择web服务器，省略的话按默认的轮循方式选择web服务器
	    server 127.0.0.1:8080;       # web服务器的IP地址及tomcat发布端口
	    server 127.0.0.1:8081;
	    
	    ＃server localhost:8080 weight=1;  
       ＃server localhost:9999 weight=5;
       ＃如果你的两台机器性能不一样，一台牛逼，一台配置渣渣，那么给牛逼的那台分配多点任务，使用weight＝5这样的大一些的权重，来让他多干点活。 
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
