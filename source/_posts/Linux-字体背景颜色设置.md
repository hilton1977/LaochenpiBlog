---
title: Linux 字体背景颜色设置
tags:
  - Linux
categories:
  - Linux
toc: false
date: 2019-08-08 15:18:38
---

![](images/linux-1.jpg)

### 颜色范围
![](/images/color-scope.png)

例如我们打印红色警告字体 `echo -e '\033[31m[警告]红色警告提示\031[0m'`其中`\033[31m`为红色颜色


### 颜色设置
![](/images/color-setting.png)
``` bash
echo -e "\033[40;37m oldboy trainning \033[0m"    黑底白字（字体颜色匹配上面的，自己更改）
echo -e "\033[41;37m oldboy trainning \033[0m"    红底白字
echo -e "\033[42;37m oldboy trainning \033[0m"    绿底白字
echo -e "\033[43;37m oldboy trainning \033[0m"    黄底白字
echo -e "\033[44;37m oldboy trainning \033[0m"    蓝底白字
echo -e "\033[45;37m oldboy trainning \033[0m"    紫底白字
echo -e "\033[46;37m oldboy trainning \033[0m"    天蓝白字
echo -e "\033[47;30m oldboy trainning \033[0m"    白底黑字
```