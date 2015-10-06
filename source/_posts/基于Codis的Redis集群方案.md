title: 基于Codis的Redis集群方案
date: 2015-10-01 17:55:46
tags: [redis,codis,cluster,集群,cache]
---


 
## 安装go
首先按照golang，下载地址： https://golang.org/dl/，最新的1.4.2版本。 
如果被墙使用golang中国下载： http://golangtc.com/download。
有mac版的apk，直接安装完事。 

## 设置环境变量
	
打开终端，输入`sudo su` ,化身炒鸡管理员，输入下面命令。

	export GOROOT=/usr/local/go
	export PATH=$PATH:$GOROOT/bin
	export GOPATH=/usr/local/codis 
然后再输入下`go env` .查看下打印的内容里面的GOPATH对不对。	
## 下载codis,安装codis

```
cd /usr/local
go get -u -d github.com/wandoulabs/codis

cp -R /usr/local/codis/pkg/ /usr/local/codis/cmd/ /usr/local/codis/src/github.com/wandoulabs/codis 

cd $GOPATH/src/github.com/wandoulabs/codis
 
sh bootstrap.sh 
#然后顺着安装完的提示，运行下测试
make gotest
```
建议只通过go get命令来下载codis，除非你非常熟悉go语言的目录引用形式从而不会导致代码放错地方。该命令会下载master分支的最新版，我们会确保master分支的稳定。
当然你也可以去下载Release版本的，然后解压到` $GOPATH/src/github.com/wandoulabs/codis`

安装codis，需要下载依赖包。比较慢 然后就等一下，安装过程还是挺久了，大概有十多分钟,不要以为死机，然后取消。。。
<!--more-->

![这里写图片描述](http://img.blog.csdn.net/20150905185418142)
 


执行全部指令后，会在`$GOPATH/src/github.com/wandoulabs/codis/bin` 文件夹生成 codis-config, codis-proxy 两个可执行文件, (另外, bin/assets 文件夹是 codis-config 的 dashboard http 服务需要的前端资源, 需要和 codis-config 放置在同一文件夹下)

![这里写图片描述](http://img.blog.csdn.net/20150905191432922)
 
 
－－－－－－－－－－－－－－－－－

安装测试成功，就可以配置了。 
编译后的二进制文件在/usr/local/codis/bin目录下面。 

## zookeeper

启动codis之前需要安装zookeeper。 
下载地址：http://zookeeper.apache.org/releases.html#download 
安装到目录/usr/local/zookeeper

	mkdir /usr/local/zookeeper/logs

修改下配置文件
	
	vi /usr/local/zookeeper/conf/zoo.cfg配置文件

修改连接的端口为2181
	
	tickTime=2000
	clientPort=2181
	dataDir=/usr/local/zookeeper/data
	dataLogDir=/usr/local/zookeeper/logs
 我的版本是3.4.6，在该目录下自带了zoo_sample.cfg的文件，已经把我们需要的写好，你偷懒也可以直接复制，然后改下名字的。
 
### 启动zookeeper

	sh /usr/local/zookeeper/bin/zkServer.sh start

### 原因
启动Codis服务，之前必须启动zookeeper，为什么呢？

因为在上一段的codis-config 和 codis-proxy 在不加 -c 参数的时候, 
默认会读取当前目录下的 [config.ini 文件.关于该文件的配置，请点击这里](https://github.com/wandoulabs/codis/blob/master/config.ini) 
在这个文件的最前面的两行，标记了zk的地址是`192.168.0.123`，端口为`2181`, 具体内容如下
 

	# zookeeper or etcd
	coordinator=zookeeper
	
	# Use comma "," for multiple instances. If you use etcd, you should also use this property.
	zk=192.168.0.123:2181
如果不先启动，会一直显示连接不上的。如果你启动了也连接不上，那就改下地址为`localhost`吧
 
---
## 启动

有的新版本没有这个目录。。。
那么请 跳过这个看下面的一步步手动的启动方式2吧。

sample目录有简单的集群配置。 
startall.sh脚本会同时将redis启动。
 
启动之后就可以看到zookeeper里面的数据了。

	[zk: localhost:2181(CONNECTED) 8] ls /zk/codis/db_test 
	[migrate_manager, fence, migrate_tasks, LOCK, slots, actions, servers, proxy]


## 2.启动流程
手动启动方式，如果你没有上面提到的sample这个目录的话。。。

0. 启动 dashboard, 执行` bin/codis-config dashboard`, 该命令会启动 dashboard
![这里写图片描述](http://img.blog.csdn.net/20150923074205332)
如果显示连接不上192.168.0.123之类的信息，那么回到config里面修改这个为localhost:2181吧。
或者需要确定你是否按我上面说的，改了zookeeper的配置接口了，同时启动了。

   **这里有一些重要的点需要说的，请看结尾后记3。**

1. 初始化 slots , 执行 `bin/codis-config slot init`,该命令会在zookeeper上创建slot相关信息
![这里写图片描述](http://img.blog.csdn.net/20150923074937402)
还是那句话，启动不了就改下ip地址为localhost

2. 启动 Codis Redis , 和官方的Redis Server参数一样, 
  e.g: `bin/codis-server XXX.conf`
  在这里有个小坑，请在配置文件里面设置maxmemory为某个GB大小，如果是独立一台服务器可以设置为内存的3/4大小。我的8g，还开4台，所以就设置了1GB。
 为何要这么做呢，看最后面的`后记5`。

3. 添加 Redis Server Group , 每一个 Server Group 作为一个 Redis 服务器组存在, 只允许有一个 master, 可以有多个 slave, group id 仅支持大于等于1的整数

  **不过你也可以跳过步骤4和5,在启动dashboard后配置redis的。**
  **不过你也可以跳过步骤4和5,在启动dashboard后配置redis的。**
    **不过你也可以跳过步骤4和5,在启动dashboard后配置redis的。**
    说了三遍了。

		$ bin/codis-config server -h                                                                                                                                                                                                                   usage:
			    codis-config server list
			    codis-config server add <group_id> <redis_addr> <role>
			    codis-config server remove <group_id> <redis_addr>
			    codis-config server promote <group_id> <redis_addr>
			    codis-config server add-group <group_id>
			    codis-config server remove-group <group_id>

 如: 添加两个 server group, 每个 group 有两个 redis 实例，group的id分别为1和2， redis实例为一主一从。

 添加一个group，group的id为1， 并添加一个redis master到该group

		$ bin/codis-config server add 1 localhost:6379 master
添加一个redis slave到该group

		$ bin/codis-config server add 1 localhost:6380 slave
类似的，再添加group，group的id为2

			$ bin/codis-config server add 2 localhost:6479 master

			$ bin/codis-config server add 2 localhost:6480 slave
		
4. 设置 server group 服务的 slot 范围 Codis 采用 Pre-sharding 的技术来实现数据的分片, 默认分成 1024 个 slots (0-1023), 对于每个key来说, 通过以下公式确定所属的 Slot Id : SlotId = crc32(key) % 1024 每一个 slot 都会有一个且必须有一个特定的 server group id 来表示这个 slot 的数据由哪个 server group 来提供.

		$ bin/codis-config slot -h                                                                                                                                                                                                                     
		usage:
		    codis-config slot init
		    codis-config slot info <slot_id>
		    codis-config slot set <slot_id> <group_id> <status>
		    codis-config slot range-set <slot_from> <slot_to> <group_id> <status>
		    codis-config slot migrate <slot_from> <slot_to> <group_id> [--delay=<delay_time_in_ms>]
	
  如:

  设置编号为[0, 511]的 slot 由 server group 1 提供服务, 编号 [512, 1023] 的 slot 由 server group 2 提供服务

		$ bin/codis-config slot range-set 0 511 1 online
		$ bin/codis-config slot range-set 512 1023 2 online
5. 启动 codis-proxy

		 bin/codis-proxy -c config.ini -L ./log/proxy.log  --cpu=8 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000

  刚启动的 codis-proxy 默认是处于 offline状态的, 然后设置 proxy 为 online 状态, 只有处于 online 状态的 proxy 才会对外提供服务

		 bin/codis-config -c config.ini proxy online <proxy_name>  <---- proxy的id, 如 proxy_1
这里的proxy_1是配置文件config.ini里面的默认配置名字，你可以打开里面拉到最下面看下id，或者用`bin/codis-config proxy list`,查看下。

	![这里写图片描述](http://img.blog.csdn.net/20150923075617026)
	终于可以运行啦！！

   **备注：**这时候到dashboard去看下最下面的Proxy Status 那栏是否有内容，如果只显示No proxies，那么请打开到你在`启动5. codes—proxy`的时候配置的log的地方，查看下显示什么。我遇到的是ulimit大小不够1024的问题。

6. 开启主从自动切换

		/usr/local/codis/codis-ha/codis-ha --codis-config=localhost:18087 --productName=proxy_1   < --productName 为config.ini中product >

7. 打开浏览器 http://localhost:18087/admin

	现在可以在浏览器里面完成各种操作了， 玩得开心
 
 	![这里写图片描述](http://img.blog.csdn.net/20151001170323914)
	
---


## 3，管理界面显示

启动之后就可以访问admin页面了。 
 ![这里写图片描述](http://img.blog.csdn.net/20151001165831553)
 
数据迁移界面 
![这里写图片描述](http://img.blog.csdn.net/20151001170128509)
整个桶结构 
![这里写图片描述](http://img.blog.csdn.net/20151001170018516)

# 4，总结

Codis可以完美的解决Redis集群问题，在目前Redis 3.0版本还不是很稳定的情况下，是非常不错的解决方案。支持数据扩展。 
但是并不是所有的redis命令都支持。

如果你使用以下命令:

	KEYS, MOVE, OBJECT, RENAME, RENAMENX, SORT, SCAN, BITOP,MSETNX, BLPOP, BRPOP, BRPOPLPUSH, PSUBSCRIBE，PUBLISH, PUNSUBSCRIBE, SUBSCRIBE, UNSUBSCRIBE, DISCARD, EXEC, MULTI, UNWATCH, WATCH, SCRIPT EXISTS, SCRIPT FLUSH, SCRIPT KILL, SCRIPT LOAD, AUTH, ECHO, SELECT, BGREWRITEAOF, BGSAVE, CLIENT KILL, CLIENT LIST, CONFIG GET, CONFIG SET, CONFIG RESETSTAT, DBSIZE, DEBUG OBJECT, DEBUG SEGFAULT, FLUSHALL, FLUSHDB, INFO, LASTSAVE, MONITOR, SAVE, SHUTDOWN, SLAVEOF, SLOWLOG, SYNC, TIME

是无法直接迁移到 Codis 上的. 你需要修改你的代码, 用其他的方式实现.这点确实有点点遗憾



---
资料来源：

1. [本文一部分参考自官方的Codis引导文档](https://github.com/wandoulabs/codis/blob/master/doc/tutorial_zh.md#build-codis-proxy–codis-config)
2. [Redis集群方案，Codis安装测试](http://blog.csdn.net/freewebsys/article/details/44100919)
3. [Redis解决方案Codis安装使用](http://blog.csdn.net/dc_726/article/details/47052607) 
4. [codis集群搭建](http://www.cenhq.com/2015/09/15/codis集群搭建/)
5. [codis其中一个作者自己写的配置](http://navyaijm.blog.51cto.com/4647068/1637688?utm_source=tuicool)
## 后记


1. 这个过程如果你遇到没找到$GOPATH的问题的话，我找到的解决方案是这条命令	
	
		sudo env GOPATH=/usr/local/codis  sh ./bootstrap.sh 
就是在前面加多`env GOPATH=/usr/local/codis`这么一个参数

2. 真的觉得安装在／usr/local这样的位置很麻烦！！！一堆的权限问题！！！

3.  **关于dashboard的关闭。**，我们在启动了dashboard后，他打印了一堆内容，如下：
	```
 2015/09/26 11:18:41 dashboard.go:160: [INFO] dashboard listening on addr: :18087
2015/09/26 11:18:41 dashboard.go:143: [INFO] dashboard node created: /zk/codis/db_test/dashboard, {"addr": "localhost:18087", "pid": 1701}
2015/09/26 11:18:41 dashboard.go:144: [WARN] ********** Attention **********
2015/09/26 11:18:41 dashboard.go:145: [WARN] You should use `kill {pid}` rather than `kill -9 {pid}` to stop me,
2015/09/26 11:18:41 dashboard.go:146: [WARN] or the node resisted on zk will not be cleaned when I'm quiting and you must remove it manually
2015/09/26 11:18:41 dashboard.go:147: [WARN] *******************************
	```
   
   是的，如内容说说的，如果要关闭，记得要`kill -{pid}`,要不然突然电脑没电之类的bug，导致异常退出的时候，就得手动关闭，要不然在下次启动的时候，就会遇到下面的内容：
	```
	bin/codis-config dashboard
	2015/09/26 11:14:10 dashboard.go:160: [INFO] dashboard listening on addr: :18087
	2015/09/26 11:14:10 dashboard.go:234: [PANIC] create zk node failed
	[error]: dashboard already exists: {"addr": "192.168.0.123:18087", "pid": 30155}
	[stack]: 
	    3   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/cconfig/dashboard.go:234
	            main.runDashboard
	    2   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/cconfig/dashboard.go:54
	            main.cmdDashboard
	    1   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/cconfig/main.go:84
	            main.runCommand
	    0   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/cconfig/main.go:151
	            main.main
	        ... ...
	```
[手动关闭的办法如下](https://github.com/wandoulabs/codis/issues/314):
跳到安装`zookpeer`的目录下，然后连接到服务器，关闭dashboard。

	$ ./bin/zkCli.sh -server 127.0.0.1:2181
	[zk: 127.0.0.1:2181(CONNECTED) 10] ls /zk/codis/db_test
	[slots, migrate_tasks, actions, ActionResponse, dashboard]
	[zk: 127.0.0.1:2181(CONNECTED) 7] rmr /zk/codis/db_test/dashboard 
	删除后再启动就能启动了\
	
4. [Ulimit 的大小问题](http://unix.stackexchange.com/questions/108174/how-to-persist-ulimit-settings-in-osx-mavericks)

	启动proxy的时候
	
	bin/codis-proxy -c config.ini -L ./log/proxy.log  --cpu=8 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000
	  打印的log显示了下面的问题：	
	
		2015/09/28 23:39:13 main.go:102: [PANIC] ulimit too small: 256, should be at  least 1024
		[stack]: 
		1   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/proxy/main.go:102
				            main.checkUlimit
		0   /usr/local/codis/src/github.com/wandoulabs/codis/cmd/proxy/main.go:166
				            main.main
				        ... ...
	 这时候需要我们调大一下ulimit：
	
		$ ulimit -n 1024

5. 关于redis的配置maxmemory
	    为何要设置这个maxmemory呢，因为codes在做主从切换的时候，用的是codis-ha;
    codis-ha实现codis-server的主从切换，codis-server主库挂了会提升一个从库为主库，从库挂了会设置这个从库从集群下线。 而这个codes－ha需要我们明确每个redis可以使用的最大内存。不能是NAN GB，所以需要我们配置这个属性。拉到redis自带的配置文件的中间地方，有下面这段，我们取消maxmemory 的注释就好了。
		    
		# Don't use more memory than the specified amount of bytes.
		# When the memory limit is reached Redis will try to remove keys
		# according to the eviction policy selected (see maxmemory-policy).
		#
		# If Redis can't remove keys according to the policy, or if the policy is
		# set to 'noeviction', Redis will start to reply with errors to commands
		# that would use more memory, like SET, LPUSH, and so on, and will continue
		# to reply to read-only commands like GET.
		#
		# This option is usually useful when using Redis as an LRU cache, or to set
		# a hard memory limit for an instance (using the 'noeviction' policy).
		#
		# WARNING: If you have slaves attached to an instance with maxmemory on,
		# the size of the output buffers needed to feed the slaves are subtracted
		# from the used memory count, so that network problems / resyncs will
		# not trigger a loop where keys are evicted, and in turn the output
		# buffer of slaves is full with DELs of keys evicted triggering the deletion
		# of more keys, and so forth until the database is completely emptied.
		#
		# In short... if you have slaves attached it is suggested that you set a lower
		# limit for maxmemory so that there is some free RAM on the system for slave
		# output buffers (but this is not needed if the policy is 'noeviction').
		#
		maxmemory 1GB

6. [数据迁移migrate的问题](https://github.com/wandoulabs/codis/issues/463)

   人在江湖飘，哪能不会有迁移数据的一天，因为填错了组，所以导致这个迁移任务一致显示出错，reblance卡住不能动，对此的解决方法是，到zk下面的migrate_task里面删掉任务
```
sh /usr/local/zookeeper/bin/zkCli.sh
ls /zk/codis/db_test/migrate_tasks
＃在显示的里面最小的那个任务号码。放到下面这一个去
delete /zk/codis/db_test/migrate_tasks/XXXXX任务号码XXXX

	```