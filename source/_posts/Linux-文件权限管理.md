---
title: Linux-文件权限管理
date: 2018-07-25 15:07:09
category : Linux
tags:
    - 技术
    - Linux
---

![Linux](/images/linux-1.jpg)

>为了保证文件系统的安全隐私，对文件进行权限控制，防止非法用户查看、修改、删除等操作。只有在指定用户或用户组才能进行操作，例如一些隐私文件或者文件夹不想被其他人进行访问查看可对文件进行权限控制。

## 1. ls 命令查询文件属性
``` bash
[root@vultr ~]# ls -al
total 44
dr-xr-x---  4     root     root    4096      Jul 19 05:05 .
dr-xr-xr-x 18    root     root    4096      Jun  5 21:42 ..
-rw-------  1     root     root    4369       Jul 25 07:02 .bash_history
-rw-r--r--  1     root     root    18           Dec 29  2013 .bash_logout
-rw-r--r--  1     root     root     176         Dec 29  2013 .bash_profile
-rw-r--r--  1     root     root     176         Dec 29  2013 .bashrc
drwx------ 3     root     root     4096       Jul 18 07:35 .cache
-rw-r--r--  1     root     root      100         Dec 29  2013 .cshrc
drwxr----- 3     root     root      4096       Jun  5 21:45 .pki
-rw-r--r--  1     root     root      129         Dec 29  2013 .tcshrc
[权限]    [连接数][所有者][用户组][文件容量][修改时间] [文件名]
```
- ##### [权限]  
第一个字符代表文件是 "目录、文件或链接文件等"
    `[d]` 代表是目录，例如 .pki
    `[-]` 代表是文件，例如 .tcshrc
    `[l]` 代表为链接文件(linkfile)
    `[b]` 代表设备文件里的可以供存储的接口设备
    `[c]` 代表设备文件里的串行端口，例如键盘、鼠标
接下来的3个为位一组，均为 "rwx" 3个参数组合
    `[r]` 代表read 可读
    `[w]` 代表write 可写
    `[x]` 代表execute 可执行
    `[-]` 代表没有权限
    第一组代表 "文件所有者的权限"，第二组代表 "同用户组的权限"，第三组代表 "其他非本用户组的权限"
- __[连接数]__ 文件的硬链接个数
-  __[所有者]__ 文件的所有者账号
- __[用户组]__ 文件的所有用户组
- __[文件容量]__ 文件的容量 单位/B
- __[修改时间]__ 文件的创建时间或最近的一次修改时间
- __[文件名]__     文件的名称 带 "." 则表示当前文件为`隐藏文件`
    
## 3.改变文件属性和权限命令
- ##### chgrp 改变文件所属用户组
改变的用户组必须存在于`/etc/group`，对于不存在的用户组改变会执行失败
``` bash
#示例 [-R] 递归 文件或者目录下所有的的文件
chgrp [-R] [文件或目录]
#更新install.log用户组为user
chgrp user install.log
```
- ##### chown 改变文件所有者
改变的用户必须存在于`/etc/passwd`，对于不存在的用户改变会执行失败
``` bash
#示例 [-R] 递归 文件或者目录下所有的的文件
chown [-R] [文件或目录]
#更新install.log用户所属为test
chown test install.log
#可用.[用户组] 改变用户组 将install.log所属用户组改为groupTest
chown .groupTest install.log
```
- ##### chmod 改变文件的权限
改变rwx 读写执 3个权限，3个身份`owner`，`group`，`others`，组合9个权限。  
  - 数字类型改变权限:
  权限rwx按分数 `r : 4` ` w : 2`  `x : 1`，改变权限的组合方式按分数来决定权限rwxrwxrwx 对应777，rw--wx--- 对应610
``` bash
#示例 [-R] 递归 文件或者目录下所有的的文件
chmod [-R] [分数组合] [文件或目录]
#改变install.log的权限 763代表了 rwxrw---x
chown 763 install.log
```
  - 符号类型改变权限:
权限rwx按符号 u(user)，g(group)，o(others)，a(all)，+(加入)，-(除去)，=(设置)组合。
``` bash
#用户拥有读写，用户组读，其他执行 u=rw-,g=r--,o=--x
chmod u=rw-,g=r--,o=--x install.log
#所有身份都去除写权限 
chmod a-r install.log
#所有身份都添加执行权限
chmod a+x install.log
```

## 2.RWX 对于文件和目录的差别
##### 对于文件来说：
- r  (read)  可以读取文件的实际内容
- w (write) 可以编辑、新增、或修改文件的内容，但是不能删除文件
- x (execute) 可以执行，可执行并非由文件的后缀来决定例如常见的`.exe` `.bat` `.com` 等，而是由x 属性来决定

##### 对于文件目录来说：
- r  (read contents in directory)  可以查询该目录下的文件名数据既可使用`ls`查询
- w (modify contents of directory) 可以新建新的文件和目录、删除已存在的文件和目录（无视改文件的权限控制）、转义目录内的文件和文件夹
- x (access directory) 可以进入该目录文件 既可使用`cd`进入该目录
