---
title: Hexo部署到VPS自动发布
date: 2018-07-26 15:02:30
category : Linux
tag : 
    - Linux
---

![Linux](/images/linux-1.jpg)

> Hexo部署到github访问的速度较慢，所以想着把Hexo直接丢在自己VPS上去，部署一套git环境以后方便自动发布更新

## 1.Git 安装
``` bash
#通常使用的方法下载git
yum -y install git
#查看版本 这种下载一般不是最新的版本
yum --version
```
发现并不是最新版本逼死强迫症啊，只能通过下载最新git源码自行编译安装。
Git 的工作需要调用 `curl`，`zlib`，`openssl`，`expat`，`libiconv` 等库的代码，所以需要先安装这些依赖工具。在有 `yum` 的系统上（比如 Fedora）或者有 `apt-get` 的系统上（比如 Debian 体系），可以用   下面的命令安装：

``` bash
#卸载旧版本git
yum remove git
#安装依赖环境
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
#安装编译工具
yum install  gcc perl-ExtUtils-MakeMaker
#下载最新版git
wget https://github.com/git/git/archive/v2.18.0.tar.gz
#解压
tar -zxvf v2.18.0.tar.gz
#进入解压文件夹
cd git-2.18.0
#编译代码 perfix这里为赋值变量
make prefix=/usr/local/git all
#安装软件 
make prefix=/usr/local/git install
#清除编译数据
make clean all
#环境变量配置
echo export PATH=$PATH:/usr/local/git/bin >>/etc/bashrc
#生效环境变量
source /etc/bashrc
```
>/etc/profile，/etc/bashrc 是系统全局环境变量设定 ~/.profile，~/.bashrc用户家目录下的私有环境变量设定

## 2.创建git仓库
创建一个git库用来存放Hexo生成的html静态文件和相关资源，然后通过post-receive 钩子函数进行自动执行脚本讲生成的资源checkout发布到nginx达到自动发布更新的功能。
``` bash
#创建git用户
adduser git
#设置密码
passwd git
#创建Hexo博客库 目录自行选择
mkdir laochenpiBlog && chown git:git laochenpiBlog
#laochenpiBlog目录下创建blog.git  --bare裸仓库 没有工作空间
git init --bare blog.git && chown git:git -R blog.git 
#laochenpiBlog 目录下创建静态网页库 
mkdir blog.site && chown git:git blog.site
#进入钩子函数目录
cd hooks/
#创建钩子函数文件
touch post-service && chown git:git post-receive && chmod 755 post-service
```
为Hexo编写自动化脚本在仓库hooks创建脚本 `vi post-receive` ,脚本会在git有收发的时候就会调用执行
```
git --work-tree=/var/laochenpiBlog/blog.site --git-dir=/var/laochenpiBlog/blog.git checkout -f
```

## 3.Hexo配置发布测试
终于把Git环境弄好了，现在就需要修改配置文件`_config.yml` 中的发布项
``` xml
#Deployment
## repo  为你的vps创建的库地址
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo : git@45.77.87.214:/var/laochenpiBlog/blog.git
  branch: master
```
修改完毕，见证奇迹的时候到了，找到自己博客目录下用 `git bash`  发布
 ```
 #清除缓存  重编译  发布
 hexo clean && hexo g && hexo d
 ```
 ![OpenSSH](/images/passwd.png)
 输入密码发布完毕，然后远程上你的VPS查看下你的 `blog.site` 是否自动 `check out`了最新发布的内容了。

## 4.Git 免密发布
每次发布都需要输入密码实在是太麻烦了而且在有可能泄露密码引起安全问题，有什么比较方面安全的方式呢，通过google一波发现可以通过秘钥的形式实现无密码发布登录。

秘钥方式通过RSA加密生成公有秘钥，然后把公有秘钥提交到VPS 上的秘钥认证文件中 `authroized_keys`，修改 OpenSSH 客户端的配置 `sshd_config`  实现RSA秘钥认证方式。

那么我们开始吧！

- ##### 服务器端 
修改 OpenSSH 认证 ` vi /etc/ssh/sshd_config ` 
开启公钥认证 `PubkeyAuthentication yes`   
认证Keys文件目录 用户/.ssh/文件名 `AuthorizedKeysFile      .ssh/authorized_keys`  
RSA加密认证 `RSAAuthentication yes` 
``

这里要提示一点 Centos 7 和 Centos 6 遇到的问题，Centos 7 由于OpenSSH版本原因 RSAAuthentication 已经弃用，无需添加修改.
```
#用户提交的git用户的秘钥文件夹创建和权限分配#
——————————————————————————————————————
#创建认证文件authorized_keys
touch /home/git/.ssh/authorized_keys
#.ssh权限 700 authorize_keys 权限600
chmod 700 /home/git/.ssh && chmod 600 /home/git/.ssh/authorize_keys
```
这里要注意 `.ssh` 和 `authorize_keys` 的权限问题，可能在加密认证的时候由于权限导致失败,SSH登录日志可以用 `tail /var/log/secure` 查看，`sshd -t`进行查看配置是否正常 需要在~目录下执行，执行`systemctl restart sshd` 重启 `SSH`服务

- ##### 客户端 
`ssh-keygen -t rsa -C userName`  生成秘钥文件，地址一般在 `~/.ssh` 中。
`id_rsa` 加密公钥 ` id_rsa.pub` 加密公钥  多用户用`cat` 追加秘钥到认证文件中

``` bash
#上传认证秘钥到服务器上 对应用户的authorized_keys中
cat ~/.ssh/id_rsa.pub | ssh git@45.77.87.214 "cat - >> /home/git/.ssh/authorized_keys"
```
配置完毕后使用 `ssh -vvT git@45.77.87.214` 看看是不是不用密码就可以登录VPS了，然后发布就再也不用密码了，一条命令就OK。

## 5.  Nginx配置映射
终于到最后一步了，就差最后一步配置 Nginx 服务映射静态文件了。
``` bash
#Centos yum源安装
yum install nginx
#启动nginx服务
systemctl start nginx
#查看服务状态
systemctl status nginx -l
```
这里有可能出现的问题：
1.无法从外网访问 检查下80端口是否开启,添加80端口`firewall-cmd --permanent --zone=public --add-port=80/tcp --permanent` 和 `firewall-cmd --reload` 重载配置
2.服务可能没有启动成功，排查下配置问题

修改80端口默认映射库地址，`nginx -t`查看nginx配置文件地址  

![nginx configuration file path](/images/nginx.png)
``` bash
#停止Nginx服务
systemctl stop nginx
#修改Nginx的配置文件root
vi /etc/nginx/nginx_conf
#修改 root 配置hexo静态文件地址，即之前创建的静态文件地址
root [hexo静态文件地址]
#修改完毕退出 重启Nginx服务
systemctl start nginx
```
修改完毕启动好服务然后通过外网访问你 VPS IP地址即可访问，大功告成以后可以在任意地方通过git提交的方式进行自动发布。请记得随时备份自己的重要文件以免丢失！

## 遇到的问题
已配置秘钥但是SSH还是需要密码，相信很多小伙伴都遇到过，下面是可能原因
1. 查看sshd_config 配置文件是否正确开启了3个认证配置，更改后重启OpenSSH服务
2. 查看下ssh登录日志 排查下原因，可能是认证文件目录权限问题，`.shh 700 ` `authorized_key 600 ` 过大或者过小的权限都有可能导致认证是失败。
3. `authorized_key` 中秘钥千万千万不要直接从客户端直接复制过来，可能会有空格和其他转义一些特殊情况导致秘钥不正确。可通过 `cat`或`scp` 命令远程进行上传秘钥保证正确性。
4. Centos 7 版本的 `OpenSSH` RSAAuthentication已经弃用无需设置、添加该设置可能导致启动异常。
