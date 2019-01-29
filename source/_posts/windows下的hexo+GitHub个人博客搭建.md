---
title: windows下的hexo+GitHub个人博客搭建
comments: true
toc: true
categories: 
- hexo
tags: 
- hexo
abbrlink: ae3b3943
---



# windows下的hexo+GitHub个人博客搭建

## 1、node.js的下载与安装

### 1)、下载node.js

去node的官方网站下载windows版本的zip文件[node.js的官方下载连接](https://nodejs.org/en/download/)

<!--more-->

下载后解压至任意目录

在解压后目录中新建两个文件夹(用来存放npm全局模块的安装目录,也可以指定其他目录

node-cache
node-global

### 2)、配置环境变量

#### 新建NODE_HOME

```
变量名:NODE_HOME
变量值:C:\Java\node-v10.15.0-win-x64(路径为解压node后存放的根目录)
```

#### 配置path

```java
%NODE_HOME%
%NODE_HOME%\node-global
```

### 1)、设置node

打开cmd,输入命令设置node

```
npm config set prefix "C:\Java\node-v10.15.0-win-x64\node-global"
npm config set cache "C:\Java\node-v10.15.0-win-x64\node-cache"
```

此处地址为安装目录,根据你们实际情况修改

### 2)、设置npm国内源

```
npm config set registry "https://registry.npm.taobao.org"
```

### 3)、测试node是否安装成功

```
node -v
npm -v
```

```
C:\Users\siworae>node -v
v10.15.0
C:\Users\siworae>npm -v
6.4.1
```

出现这个则说明安装成功

## 2、安装git

git下载地址:https://www.git-scm.com/download/

去git官网下载相对于的版本进行安装

安装成功后可以通过git --version测试是否安装成功

```
C:\Users\siworae>git --version
git version 2.10.2.windows.1
```

## 3、注册GitHub账号和创建GitHub仓库

在[GitHub官网](https://github.com/)注册GitHub账号

1)、创建仓库

{% asset_img 1.jpg 图片上传失败%}

创建git仓库时候，仓库的名称有格式要求，例如我的GitHub仓库用户名是siworae,仓库名称就是**siworae.github.io**

然后点击下面的create repository就可以成功创建GitHub仓库了,创建完之后会跳转到一个新的界面,不要关闭.留着它

## 4、安装hexo

hexo官方中文文档:<https://hexo.io/zh-cn/docs/index.html>

hexo中的命令介绍

```
hexo g #完整命令为hexo generate，用于生成静态文件 
hexo s #完整命令为hexo server，用于启动服务器，主要用来本地预览 
hexo d #完整命令为hexo deploy，用于将本地文件发布到github上 
hexo n #完整命令为hexo new，用于新建一篇文章
```

在任意目录新建hexo文件夹(名字任意),我在C盘中新建了一个hexo文件夹,进入这个文件夹,鼠标右键选择"Git Bash here"进入git命令行窗口,执行以下命令设施node、安装hexo和git依赖.之后所有的操作都需要在这个目录下完成

### 1)、安装

文中所有路径都尽量不要包含中文路径,不然很容易出现各种稀奇古怪的错误

```
npm  install  hexo-cli -g
npm  install  hexo-deployer-git  --save
```

### 2)、初始化hexo

```
hexo init
```

### 3)、安装依赖

```
npm insatll
```

### 4)、生成静态文件

```
hexo g
```

启动本地服务,就可以进行本地预览了,启动之后这个窗口不能关闭,不然博客无法访问

```
hexo s
```

这个时候我们就可以在浏览器输入:[http://127.0.0.1:4000](http://127.0.0.1:4000/)进行访问我们的博客了,但是此时博客还是只在我们本地上,想要在外网可以访问还需要将博客发布到GitHub上

## 5、部署本地网站到GitHub

### 1)、设置GitHub和git

首先,需要将你的git连接上你的GitHub

在Git Bash here命令行窗口输入GitHub中设置的用户名和关联邮箱进行本地git设置

```
git config  --global  user.name  "your name"  
git config  --global  user.email  your_email@youremail.com
```

在本机上生成ssh的密钥用于GitHub登陆(your_email@youremail.com改为自己在GitHub绑定的邮箱)

```
ssh-keygen -t rsa -C your_email@youremail.com
```

然后一路回车,系统会自动将生成的公钥和私钥文件保存在C:\Users\siworae\.ssh文件夹里面(siworae为自己的电脑用户名)

.ssh为隐藏文件夹,需要开启显示隐藏文件才能看到

在文件夹里面找到id_rsa.pub文件,用记事本或者notepad打开,复制.

打开GitHub登陆,点击settings

{% asset_img 2.jpg 图片上传失败 %}

{% asset_img 3.jpg 图片上传失败 %}

在SSH Keys/Add new界面中,title填写标题,可随意,key一栏填写刚刚复制的id_rsa.pub文件里面的内容.点击页面下面的Add SSH key就添加成功了

### 2)、配置hexo配置文件

在hexo安装根目录下找到_config.yml文件.

用记事本或者notepad打开,在最后面添加GitHub的信息

注意在hexo所有的配置文件中,冒号后面都需要跟一个空格,不然会报错

repository填写你的仓库地址,在创建完仓库之后就会跳转到这个界面,选择SSH.复制路径

{% asset_img 4.jpg 图片上传失败 %}

```
deploy:
  type: git
  repository: git@github.com:siworae/siworae.github.io.git
  branch: master
```

这个时候你的GitHub已经和hexo关联起来了

```
hexo g
hexo d
```

将本地代码推送到GitHub上,然后你可以到GitHub仓库看看代码是否已经推送成功.成功之后你就可以通过**siworaer.GitHub.io**网址来访问自己的博客了

## 6、主题安装

进入hexo目录.打开git命令行窗口,执行以下命令将主题文件下载至hexo/themes目录,这里我已安装BlueLake主题为例,具体主题可以到官网下载<https://hexo.io/themes/>

```
git clone https://github.com/chaooo/hexo-theme-BlueLake.git
```

hexo安装根目录下找到_config.yml文件,将landscape改为BlueLake

```
#theme: landscape
theme: BlueLake
```

清除缓存并重新生成静态文件

```
hexo clean && hexo g && hexo s
```

打开浏览器就可以查看新的主题效果了

具体的主题设置可以参照官方文档https://github.com/chaooo/hexo-theme-BlueLake#bluelake

## 7、修改站点配置信息

hexo安装根目录下找到_config.yml文件

1)、设置语言

```
language: zh-CN
```

2)、设置个人信息

```
title: ##标题
subtitle: ##副标题
description: ##简要描述
keywords: ##关键字
author: ##作者信息
```

然后重新推送到GitHub上,然后你的个人博客基本上就大功告成了.

## 8、安装问题中出现的问题

如果安装hexo后出现bash: hexo: command not found,将hexo安装目录下的...\node_modules\hexo\bin配置到path环境变量中