---
title: Git 服务器搭建与 Python 建站环境搭建
date: 2016-08-23 23:47:56
categories: 工作总结
tags: [Android, Git，Python]
keywords: Android, Git，Python
comments: true
---

搭建环境为阿里云 CenterOS 7.0 64位,
Git服务器自带版本号为 1.8.3.1

### Git服务器搭建与配置
1. 创建Git管理员账号并设置密码：
`useradd -m git`
`passwd git`
*执行此步操作后，系统会在会在/Home/下创建一个以用户名命名的文件夹*
*删除用户的命令为`userdel username`*
*删除组的命令为`groupdel groupname`*
2. 导入客户端生成的SSH公钥文件：
服务端在/Home/git目录下创建.ssh文件夹，创建出来的文件夹是隐藏文件，注意使用ls命令是无法查看的，在.ssh文件夹下创建authorized_keys文件，将客户端生成的公钥文件(.pub文件)内容导入文件中。
*客户端生成SSH公钥方法请参考[Generating a new SSH key and adding it to the ssh-agent](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)*
*客户端如果生成有多个SSH Key, 请本地配置config文件*
3. 禁止git账号的SSH登录：
修改 /etc/passwd 文件中git账号的配置。如果有 /bin/bash 修改为 /usr/bin/git-shell，否则增加此配置。
4. 修改SSH认证配置文件:
命令行中执行`vi /etc/ssh/sshd_config`，将配置文件中的
`RSAAuthentication yes`
`PubkeyAuthentication yes`
这两个选项的注释打开。
5. 初始化Git仓库：
在/home/git/目录下创建repositories目录，同时修改权限。
`sudo mkdir /home/git/repositories`
`sudo chown git:git /home/git/repositories`
`chmod 700 /home/git/repositories`
在repositories目录创建git仓库，并同时初始化。
`cd /home/git/repositories/`
`mkdir mytest.git`
`sudo chown git:git mytest.git/`
`cd mytest.git/`
`git init --bare`
至此，整个Git服务器搭建工作完成。
7. 若干可能遇到的问题说明：
  - 使用SSH登录如果提示此异常：fatal: Interactive git shell is not enabled. hint: ~/git-shell-commands should exist and have read and execute access.
  解决方法如下：
  `cp /usr/share/doc/git/contrib/git-shell-commands /home/git -R`
  `chown git:git /home/git/git-shell-commands/ -R`
  - 如果使用SSH登录进入，显示结果为：Run 'help' for help, or 'exit' to leave.  Available commands:
list
  [参考文档](https://git-scm.com/docs/git-shell)解决方法如下：
  基于上一步操作，
  `vi /home/git/git-shell-commands/no-interactive-login`
  `cat >/home/git/git-shell-commands/no-interactive-login <<\EOF`
  输入：`printf '%s\n' "Hi $USER! You've successfully authenticated, but I do not"`
  输入：`printf '%s\n' "provide interactive shell access."`
  输入：`exit 128`
  `EOF`
  `chmod +x /home/git/git-shell-commands/no-interactive-login`
  - 如果在创建仓库后，发现客户端提交异常：remote: error: insufficient permission for adding an object to repository database ./objects
  remote: fatal: failed to write object
  error: unpack failed: unpack-objects abnormal exit
  请修改仓库读写权限如下：
  `sudo chown -R git:git mytest.git`
  - 如果第一次提交时提示异常：No refs in common and none specified; doing nothing.
  Perhaps you should specify a branch such as 'master'.
  请使用如下的提交命令：
  `git push --set-upstream origin master`

***

### Nginx安装：
1. 下载对应版本的nginx包
`# wget http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm`
2. 建立nginx的yum仓库
`# rpm -ivh nginx-release-centos-7-0.el7.ngx.noarch.rpm`
3. 下载并安装nginx
`# yum install nginx`
4. 启动nginx服务
`systemctl start nginx`
5. 测试
在浏览器地址栏中输入部署nginx环境的机器的IP，如果一切正常，应该能看到"Welcome to nginx!"字样的内容。

### Django 安装
1. 首先安装pip包管理工具：
`wget https://bootstrap.pypa.io/get-pip.py`
`python get-pip.py`
2. 使用pip安装django:
`pip install django＝＝1.7`
3. 安装后检测是否安装完成：
`python`
`python install django｀
`django.get_version()｀
输出1.7，证明安装成功。

### GUnicorn安装
[参考文档](http://docs.gunicorn.org/en/latest/install.html)使用`pip install unicorn`,全部安装完成


### MySQL安装：
在CenterOS 7.0中的yum源没有mysql-server,所以我们采用如下方式安装：
`# wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm`
`# rpm -ivh mysql-community-release-el7-5.noarch.rpm`
`# yum install mysql-community-server`
成功安装之后重启mysql服务
`# service mysqld restart`


### MySQLdb 模块依赖安装：
直接尝试运行python项目报错缺少MySQLdb，使用yum安装
`yum install MySQL-python -y`
*尝试使用pip安装，但是安装失败。如果有成功的同学欢迎分享经验。*

*PS:昨天服务器刚刚搭好，今天阿里云就报警说服务器被黑了，看后台是用git用户登录的。原来git配置时考虑文件权限读取，本来还把git用户sudoers文件中的。后来格完盘考虑到安全性就没在加了。附上操作：
`visudo`在`root ALL=(ALL)ALL`下面一行添加 `git ALL=(ALL)ALL`*

>如果有建站经验丰富的同学，欢迎砸砖留言指导：）
