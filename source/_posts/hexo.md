---
layout: post
title: "Hexo+GitHub 第一次搭建笔记"
date: 2018-05-24 14:48
categories: [技术]
comments: true
tags: 
	- 心得 
---

![Hexo](/images/hexo.jpg)

> Hexo+GitHub 搭建踩坑行动，平时有什么代码心得或者遇到一些奇葩BUG、都没有记下来，后来遇到类似的问题居然又忘记了，所以想自己搭建一个博客记录下一些平时遇到的问题和需要解决的一些技术问题记录下来以便以后回来还可以查阅，就用Hexo搭建一个静态的博客。

## 1.Hexo 环境准备
 * [Node.js](http://nodejs.cn/) hexo依赖环境
 * [Git Bash](https://git-scm.com/) 根据OS下载安装包 用于发布和更新微博
 
##### 安装 Hexo
``` bash
#1.安装hexo环境
npm install hexo-cli -g  
#2.初始化hexo blog 文件夹和相关带代码 bolgName为文件夹名称
hexo init [blogName]
#3.进入博客文件夹
cd blog
#4.进行依赖更新安装
npm install
 ```
 

##### 常用指令
```bash
#新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。
$ hexo init [folder]
#新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 
#default_layout 参数代替。如果标题包含空格的话，请使用引号括起来。
$ hexo new [layout] <title>
#生成静态文件。
$ hexo g
#发表草稿
$ hexo publish [layout] <filename>
# 启动服务器。默认情况下，访问网址为： http://localhost:4000/
$ hexo s
# 部署网站
$ hexo deploy
# -p， --port	重设端口
# -s， --static	只使用静态文件
# -l， --log	启动日记记录，使用覆盖记录格式
 ```
 
 ## 2.GitHub Page 准备
* 登录Github创建一个reqo，名称为 `` [yourname].github.io `` (这里注意下yourname最好跟你库的用户名一样)

* 本地使用git设置username 和email 
        
```
git config --global user.name [username]
git config --global user.email [email]
```

* GitHub SSH KEY 设置
![GitHub SSH key设置](/images/ssh-key.jpg) 

``` bash
    ssh-keygen -t rsa -C [email]
```
秘钥 `` C:\Users\serwer\.ssh\id_rsa.pub `` 复制添加到Github SSH Key中

在 **Git Bash** 中验证是否添加成功：``ssh -T git@github.com``

* 配置_config.yml 发布静态文件到github，修改_config.yml进行github发布设置

``` yml
deploy:
  type: git
  repo: git@github.com:[username]/[username].github.io.git
  branch: master
 ```
 通过 **Git Bash** `` hexo d `` 进行发布更新到github 然后访问你的reqo page即可看到属于你自己的静态微博    
 
 可能遇到的问题：
  ![缺少发布插件](/images/error.jpg)
  
  解决方法:`` npm install --save hexo-deployer-git `` 安装hexo git发布插件然后执行``hexo d`` 

 
    