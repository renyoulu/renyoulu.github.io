---
layout: post
title: 在Linux上利用Github+jekyll建立个人博客（1）
key: 2018-04-30
tags:
  - jeykll
  - github
  - Linux
category: blog
---

一直以来都想自己建立一个个人网站，将自己的技术经验和想法分享到网上，今天终于实现了。本人从技术小白的角度分享一下自己在Linux环境下建立个人博客的经验，有讲错的欢迎指正。<!--more-->
*注：本博客搭建的系统是 Ubuntu 16.04 LTS 长期支持版本,本文搭建的为最简洁的 **jekyll** 原生静态页面* 

### 大概的步骤如下
1.  搭建本地jekyll的运行环境和安装jekyll
2.  配置git环境，在github上注册一个你自己的账号，新建仓库
3.  将本地博客push（上传）到github的仓库中

## 一、github搭建博客的机制（你需要知道的）
github是一个具有版本管理功能的全球开源平台，为了方便使用者，在github上你可以建立自己的仓库，给你免费提供一个G的存储，一个仓库对应一个项目，一个项目可以设置好几个分支。github人性化的在于他给每一个仓库都对应了一个域名，允许用户自定义项目首页，用来替代默认的源码列表。在每一个项目里，设置一个解析的文件，github会根据解析文件代码解析项目内容，并在此基础之上显示在你项目对应的github-page上，也就是一个具体的托管在github上的静态网页。不过这里需要注意的是，github解析项目文件的时候有两种方式
*注：你在github上的用户名为 `username`，你有一个仓库`username.github.io`或一个仓库`blog`*

> 1.解析名为`username.github.io`仓库下的`master`分支

> 2.解析名为`blog`仓库下的`gh-pages`分支

jekyll是什么，jekyll是一个静态站点生成器，换句话说，它可以将你建立的网站源码生成一个静态页面。恰巧，github可以提供给你一个网站的域名和网站源码托管平台，所以就可以利用这两者搭建你的个人网站了。当然，静态站点生成器不止这一个，例如还有`Hexo`,利用它同样可以生成炫酷的网站，但就个人而言，选择更加小巧简洁的jekyll。你也可以查看大牛[搭建一个最简单的博客的理解](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)

## 二、配置本地环境jekyll运行环境
在配置之前，建议将ubuntu的软件安装源更换为网易源[有需要请点击这里,忽略无关内容](https://blog.csdn.net/shenlan18446744/article/details/51492451)
打开终端 开始之前，先更新下软件,jekyll依赖于ruby，我们先安装ruby,官方需要ruby>2.0,我们安装2.3版本

**1. 安装ruby**
```bash
sudo apt-get update
sudo apt-get install ruby2.3    //可以输入ruby -v 查看安装版本
```
**2. 安装jekyll**
2.3版本的ruby是集成了gem的，我们需要使用gem 安装jekyll
```bash
gem install jekyll            //可以输入jekyll -v 查看安装版本
```
* 如果提示`缺少gem` ，可以输入

```bash
sudo apt-get install rubygems

```

* 如果提示 `rb can't find header files for ruby`

```bash
sudo apt-get install ruby2.3-dev

```
* 如果提示`缺少依赖rdiscount`

```bash
gem install rdiscount -V

```
其他报错，请根据提示尝试解决～问度娘
**3. 安装bunler**
bundler是在jekyll预览的时候用的，这里先安装

```bash
gem install bundler

```
## 三、使用jekyll建立本地站点
万事具备！开干！
打开终端，进入你的用户根目录，直接输入

```bash
jekyll new blog_demo     //blog_demo是你的博客项目名称

```
然后你打开你的主文件夹图形界面就可以看到一个blog_demo文件，打开如图所示
![](https://s1.ax1x.com/2018/04/29/CGQ5CQ.png)

回到终端
```bash
cd blog_demo    //进入项目目录
```
启动jekyll预览本地站点
```bash
jekyll serve

```
提示如下图，你可以在本地4000端口查看到站点已经建立好，在终端上右键`Server address`
![ip](https://s1.ax1x.com/2018/04/29/CGQI3j.png)

大吉大利，你的第一个博客建立好了！surprise！如下图

![](https://s1.ax1x.com/2018/04/29/CGQogs.png)

然后你回到图形界面主文件夹你可以看到多了一个`_site`的文件夹，此文件下是用来存放你在本地预览的时候建立的文件，但是我们一般上传源代码的时候，将它忽略，让git不再去管理这个文件。
回到终端，`ctrl+c`结束jekyll预览，新建`.gitignore`文件
```bash
touch .gitignore
```
把_site添加到文件内
```bash
vim .gitignore          //若没有安装vim，没事就问问度娘
```
在弹出的界面输入`i`进入插入模式，输入_site/*,按`esc`后输入`:wq`回车保存退出

*注：新版本的jekyll默认添加`.gitignore`文件，需要终端输入`ll -l`查看，若有，这一步跳过*
## 四、配置Git 连接Github
以上都做好了的话，还配置一下git命令的运行环境，具体相关请查阅[git中文手册](https://git-scm.com/book/zh/v2)
打开终端
```bash
sudo apt-get install git       //安装GIT
```
**1. 在github上申请一个账号**[申请地址](https://github.com/)
**2. 然后如下在终端配置你Github注册的用户名和邮箱**，例如我的是`renyoulu`,email：`Renyou_Lu@yeah.net`
```bash
git config --global renyoulu      //更换成你的用户名
git config --global Renyou_Lu@yeah.net    //更换成你的邮箱地址
```
**3. 新建SSH-Key**
在新建SSH-key之前需要首先安装SSH,因为linux和mac已经有已经安装了SSH了，如果要查看是否已经安装，在终端输入
```bash
ssh
```
出现如下提示说明已经安装
![](https://s1.ax1x.com/2018/04/30/CGtnFx.png)
生成SHH-Key，在终端输入
```bash
ssh-keygen -t rsa   //需要连续输入三次回车
```
会看到如下提示，则说明已经生成成功
![](https://s1.ax1x.com/2018/04/30/CGt16e.png)

生成的key默认存放在.ssh文件夹下，我们需要找到这个文件，并把公钥复制出来
```bash
cd ~/.ssh
vim id_rsa.pub
```
输入`i`，选中复制`esc`退出`：wq`保存。

**4. 添加SSH Key到GitHub**
接下来需要添加公钥到github上，打开你的github，点击头像,选择setting
![](https://s1.ax1x.com/2018/04/30/CGtB6g.png)

接着
![](https://s1.ax1x.com/2018/04/30/CGtDXQ.png)

点击New SSH Key按钮后进行Key的填写操作，将复制的公钥填入key中，完成SSH-Key的添加。如下图：
![](https://s1.ax1x.com/2018/04/30/CGt67n.png)

添加SSH Key成功之后，继续输入命令进行测试
```bash
ssh -T git@github.com
```
如果出现如下提示，则说明已经添加成功
> Hi renyoulu! You've successfully authenticated, but GitHub does not provide shell access.

## 五、上传本地源代码
好了，现在我们只剩下把本地的站点上传到github上了，继续！进入项目blog_demo根目录，打开终端
```bash
git init
git checkout origin gh-pages
git add .
git commit -m "first post"
git remote add origin https://github.com/renyoulu/blog_demo.git
git push origin gh-pages
```
等待大概1-2分钟之后，再次刷新https://renyoulu.github.io/blog_demo/，就能看到我们的blog。
还有一种方式是建立一个名为username.github.io的仓库，本地新建username.github.io的博客，上传到master分支即可。
