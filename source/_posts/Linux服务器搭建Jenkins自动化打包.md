---
title: Linux服务器搭建Jenkins自动化打包
date: 2018-2-28 22:38
categories: Jenkins
tags: [Jenkins, Android]
---
> 本文作为一次折腾服务器自动化配置的一次“纪念帖”... ...

## Jenkins Install 
安装Jenkins之前，需要安装jdk，此处就不细说，不会请google
### yum 安装 Jenkins
yum的repos中默认是没有Jenkins的，需要先将Jenkins存储库添加到yum repos
```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```
### 安装Jenkins
```bash
yum install jenkins
```
Jenkins安装好之后，需要修改一下默认配置。默认情况是Jenkins是使用Jenkins用户启动的，但这个用户目前系统并没有赋予权限，这里我们将启动用户修改为root；另外Jenkins默认端口是8080，这个跟tomcat的默认端口冲突，我们也修改一下默认端口。
输入命令进入Jenkins配置文件：
```bash
vi /etc/sysconfig/jenkins
```
找到相关属性，配置：
```python
JENKINS_USER="root"
JENKINS_PORT="8081"
```
保存退出 wq
然后，启动Jenkins：
```bash
service jenkins start
```
如图，启动成功
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins1.png)

## Jenkins Setting
打开浏览器，输入服务器的域名，端口8081，go ~
### 第一进入，需要密码
进入登录页面后，Jenkins提示我们需要输入超级管理员密码进行解锁。根据提示，我们可以在/var/lib/jenkins/secrets/initialAdminPassword文件里找到密码 
```bash
tail /var/lib/jenkins/secrets/initialAdminPassword
```
复制，粘贴，go ~
### 插件安装：Install suggested plugins
后面的过程比较简单，略过，进入到Jenkins主页
### 全局配置
侧边栏：系统管理 -> 全局工具配置
* JDK
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins2.png)
* Git
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins3.png)
* Gradle
gradle 使我们自己下载的，可以将本地的gradle zip 用 scp 直接上传至服务器，解压至相应目录
此处，我们配置多个gradle版本，4.1 和 3.5
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins4.png)

> 我们可能会忘记 以上配置的路径，在这里备用下命令

```bash
which java
which git
```
## New Task
上面配置完基本的设置后，我们开始配置Android打包的任务
* 新建任务
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins5.png)
* 配置项目url
这里选择github的项目为例子：
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins6.png)
* 配置git url
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins7.png)
* 配置gradle
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins8.png)
* 配置构建后操作
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins9.png)
配置结束，save, 我们执行下:
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins10.png)
打包结果，输出apk，mapping:
![](http://7xr1vo.com1.z0.glb.clouddn.com/jenkins11.png)

参考：
https://www.jianshu.com/p/c517f09df025
https://www.jianshu.com/p/eb3cbb34be97