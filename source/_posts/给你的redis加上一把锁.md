title: 给你的redis加上一把锁
date: 2015-10-07 17:55:46
tags: [redis]
---

 一个段子
 曾经有个实习生，对redis没有加密，
 被一名黑客知道了地址，一条命令`FLUSHALL`，清掉了缓存。
 瞬间数据库被压崩，导致整个业务崩溃。

 ![这里写图片描述](http://img.blog.csdn.net/20151007171421076)
 
 那么，我们该如何保护我们的redis呢？

<!--more-->
# redis的保护方案

 

1. 比较安全的办法是采用绑定IP的方式来进行控制。

		bind 127.0.0.1
表示仅仅允许通过127.0.0.1这个ip地址进行访问。那么其实只有自己才能访问自己了，其他机器都无法访问他。

		# By default Redis listens for connections from all the network interfaces
		# available on the server. It is possible to listen to just one or multiple
		# interfaces using the "bind" configuration directive, followed by one or
		# more IP addresses.
		#
		# Examples:
		#
		# bind 192.168.1.100 10.0.0.1
		# bind 127.0.0.1

  这个方法有一点不太好，我难免有多台机器访问一个redis服务，老修改配置有点麻烦，虽然redis支持运行期间修改bind参数。

2. 设置密码，以提供远程登陆
    聪明的redis早已经看穿了一切，在配置文件的security里面，有下面这个；

		################################## SECURITY ###################################
		
		# Require clients to issue AUTH <PASSWORD> before processing any other
		# commands.  This might be useful in environments in which you do not trust
		# others with access to the host running redis-server.
		#
		# This should stay commented out for backward compatibility and because most
		# people do not need auth (e.g. they run their own servers).
		# 
		# Warning: since Redis is pretty fast an outside user can try up to
		# 150k passwords per second against a good box. This means that you should
		# use a very strong password otherwise it will be very easy to break.
		#
		# requirepass foobared
     这个我们可以只需要取消到注释，然后变成我们自己的密码就好了。
	
	 	 requirepass  zx092;3lmvz_0##!1aB20asiASDF"Q@34t  #像这样的密码基本无敌啊
     
     **注意：**
     
     因为redis并发能力极强，仅仅搞密码，攻击者可能在短期暴力破解，所以建议密码越长越好，比如128位（密码在conf文件里是明文，所以不用担心自己会忘记）。暴力破解到这个级别根本就自动放弃啦。
     使用了后，我们就需要用密码登录啦

		./redis-cli -h 192.168.1.121 -a 密码
如果没有密码，那么他得到的就是

		(error) NOAUTH Authentication required
      **更注意的**
      这年代做开发的，redis集群是标配，如果你有slave，记得也要配置好密码，
            
		# If the master is password protected (using the "requirepass" configuration
		# directive below) it is possible to tell the slave to authenticate before
		# starting the replication synchronization process, otherwise the master will
		# refuse the slave request.
		#
		# masterauth <master-password>
   如配置文件里面说的，如果你的Master有设置密码了，记得同slave也配置下，要不然会拒绝slave访问的。
3.  指令重命名 

        # Command renaming.
		#
		# It is possible to change the name of dangerous commands in a shared
		# environment. For instance the CONFIG command may be renamed into something
		# hard to guess so that it will still be available for internal-use tools
		# but not available for general clients.
		#
		# Example:
		#
		# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
		#
		# It is also possible to completely kill a command by renaming it into
		# an empty string:
		#
		# rename-command CONFIG ""
		#
		# Please note that changing the name of commands that are logged into the
		# AOF file or transmitted to slaves may cause problems.
你看，聪明的redis已经在配置里面说了，对一些指令，你可以把它过滤掉，不给执行，想flushdb这种清库的指令，直接过滤掉。当然这个也导致后来接手的人不知道过滤这件事，导致指令没法执行啊，redis也表示了一些要注意的事：
`changing the name of commands that are logged into the AOF file or transmitted to slaves may cause problems.`
4. 内网
把你的redis服务器设在内网，外网禁止访问，这个是相对安全的方法。
