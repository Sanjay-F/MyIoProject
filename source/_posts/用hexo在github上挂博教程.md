title: 用hexo在github上挂博教程
date: 2015-08-22 13:36:14
tags: [hexo,github]
#description: 简述:这是一篇讲解如何在windows上用hexo在github上挂博客的教程  ![这里写图片描述](http://7xl9zd.com1.z0.glb.clouddn.com/ic_hexo.png)  
categories: hexoSerial

---



![这里写图片描述](http://7xl9zd.com1.z0.glb.clouddn.com/ic_hexo.png)

## 1.下载 msysgit ，并安装
https://git-for-windows.github.io/


## 2.安装Node.js
在 Windows 环境下安装 Node.js 非常简单，仅须下载安装文件并执行即可完成安装。
下载地址：https://nodejs.org/download
因为我用的Windows的64位，所以下载地址在这：
https://nodejs.org/dist/v0.12.7/x64/node-v0.12.7-x64.msi

安装好后，打开cmd，输入下面命令，看下是否正常
```
	node -v 
	npm -v
```
正常的话，效果如下，不行就百度下原因吧。
![这里写图片描述](http://img.blog.csdn.net/20150821161739705)

## 3.安装hexo
利用 npm 命令即可安装。（**先在任意位置点击鼠标右键，选择Git bash**）
```
npm install -g hexo
```
在打印的日志里面，请记下安装的hexo的位置，然后再系统的环境变量`PATH`里面加多这个库的lib地址；


<!--more-->

## 4.创建hexo文件夹
安装完成后，在你喜爱的文件夹下（如H:\hexo），执行以下指令(在H:\hexo内点击鼠标右键，选择Git bash)，Hexo 即会自动在目标文件夹建立网站所需要的所有文件。
```
hexo init
```
## 5.安装依赖包

	npm install

## 6.本地查看

现在我们已经搭建起本地的hexo博客了，执行以下命令(在H:\hexo)，然后到浏览器输入ocalhost:4000看看。

	hexo generate
    hexo server
好了，至此，本地博客已经搭建起来了，只是本地哦，别人看不到的。下面，我们要部署到Github。
## 7.Github账号
已有账号可以跳过，没有的，请自行注册，很简单，这里就不介绍了。
![这里写图片描述](http://zipperary.com/img/repo.png)
## 创建repository
在自己Github主页右下角，创建一个
`New repository`。比如我的Github账号是Sanjay，那么我应该创建的repository名字应该是`Sanjay.github.io`

>[这个是github官方规定的](https://help.github.com/articles/user-organization-and-project-pages/ ),必须要名字和账号名保持一致！要不然到
>后面部署完毕后，你会发现找不到这个页面的。
>User & Organization Pages live in a special repository dedicated to 
>GitHub Pages files. You will need to name  this repository with the 
>account name.
>
>1. You must use the `username.github.io` naming scheme.
>2. Content from the master branch will be used to build and
>publish your GitHub Pages site.
> 
>You can only use your own account name for a User or Organization Page repository. A repository like `joe/bob.github.io` will not build a User Pages site.When User Pages are built, they are available at `http(s)://<username>.github.io.`


## 8.部署
编辑_config.yml，拉倒**最下面的一行**(在H:\hexo下)。你在部署时，要把下面的`Sanjay`都换成你的账号名。
 

编写在`_config.yml`需要的代码如下
```
 deploy:
	  type: git   #最新版3.X把这个github缩写成git了,这句注视可以不复制的.
	  repository: https://github.com/Sanjay/Sanjay.github.io.git
	  branch: master
```
  
据说最新版本的hexo 中，这里的 type 要写成 git，而不是 github。
执行下列指令即可完成部署。

	hexo g  #新版本支持的缩写，可以看最后面的tips了解情况 
	hexo d
注意：有些新用户需要设置 ssh，否则上述命令会失败。ssh 的介绍和设置方法请看官方教程，不用担心，很简单。
连接：
https://help.github.com/articles/generating-ssh-keys/

## 9.记住：
每次修改本地文件后，需要`hexo g`才能保存。每次使用命令时，都要在H:\hexo目录下。
Okay,我们的博客已经完全搭建起来了，在浏览器访问Sanjay.github.io就能看到你的成就了！

# 后记
 1. **同步问题**；我们可以看到，我们成功的挂博客到了github上去了，但是，需要注意的是，我们的项目是用hexo来生成，然后部署到github上面去的，这个带来的问题就是，部署的内容和我们本地的项目目录里面的内容不一样，**所以**，如果我们换一台电脑写，想从github上面同回来，再继续写，那是不可能的，github上面的是hexo编译过后的内容，所以我们最好也在github上面新建一个项目，把我们的整个项目也同步上去，这样就可以在不同电脑上写文章啦。
 2.  同步的配置问题；我们想用` git pull`来同步项目的时候，我们会遇到
 
```
fatal: No remote repository specified.  Please, specify either a URL or a  
remote name from which new revisions should be fetched.  
```
这个需要我们配置下从哪里取回项目，打开我们的h:\hexo目录里面的`.git`文件（这是一个隐藏文件）。然后修改里面的config文件，在屁股哪里加多这个,`注意`,请把下面的两条url改成你自己在github上对应的地址，原因我已经在上面一条说明了，我们需要同步这个项目，好换台电脑后也可以继续写。

```
[remote "origin"]  
        url = https://github.com/sanjay/xxx.git  #换成你自己的地址
        fetch = +refs/heads/*:refs/remotes/origin/*  
        pushurl = https://github.com/sanjay/xxx.git #换成你自己的地址 
[branch "master"]  
        remote = origin  
        merge = refs/heads/master  

```
 

# bugs

 1. 有网友反应右键菜单中没有git bash选项，可以进入开始菜单找到git bash，然后通过cd进入相应目录执行命令。
 2. 在github部署完成之后，马上访问可能出现404错误，这是正常的，（最多）等待十分钟左右就可以访问了。如果还不行，那很可能是 github 发送给你的验证邮件你没有打开看，据多方反映，验证后就没问题了。
 
 3. 如果在hexo d之后出现fatal:
  **'username.github.io' does not appear to be a git repository**，一是检查 repo 的名字是否合乎规范、是否含有`大写字母`、config.yml 中的 deploy 配置是否正确，二是把 git bash 关掉，重新打开再执行命令。
 
 4. 有的同学可能不是 IT 界的，或者对shell 命令不太了解。在要求输入密码时，你输入之后密码是不显示的，这是为了安全，并非是你没输上。
出现乱码的，不要使用 windows 中的「记事本」打开并编辑文件，推荐使用 sublime text，很简单。如果已经在「记事本」中编辑过，需要使用 sublime text 转码为「utf8」。

 5. 安装 hexo 时卡在那儿不动，很可能是网络不给力，能全局 break wall 就好了。
遇到什么其他的问题，不妨删除.deploy 和db.json 再重新生成试一试。

# tips
hexo现在支持更加简单的命令格式了，比如：
hexo g == hexo generate
hexo d == hexo deploy
hexo s == hexo server
hexo n == hexo new

  
 ---
 
如果觉得上面过程麻烦，你可以直接用简书这个网站写，逼格很好。http://www.jianshu.com
人家的显示效果是这样的，![截自简述的一张图](http://img.blog.csdn.net/20150820185826259)
 
 参考质料: http://zipperary.com/2013/05/28/hexo-guide-2/
