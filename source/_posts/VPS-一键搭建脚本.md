---
title: VPS 一键搭建脚本
tags:
  - 心得
categories:
  - 技术
toc: false
date: 2019-08-07 10:49:11
---

![](/images/linux-1.jpg)

> 每次换VPS我都要重新手打下脚本去装V2Ray2，然后装各种软件环境还要把自己的博客搬过来真的麻烦，重复的动作让人枯燥所以想起编写个脚本一键执行，懒人就用懒方法

### 脚本
``` bash 
start_menu(){
	
echo ">>>>>>>>>>>>>>>>>>>>>>>> 小明 VPS 管理一键脚本 >>>>>>>>>>>>>>>>>>>>>>>>>>"
echo "1 - V2Ray管理脚本"
echo "2 - BBR 加速脚本"
echo "3 - 老陈皮 Blog 环境搭建"
echo "4 - 退出脚本"

echo
read -p "请输入数字:" num
case "$num" in 
1)
	v2ray_manager
	;;
2)
	bbr_manager
	;;
3)
	install_blog
	;;
4)
	exit 0
	;;
*)	
	echo&&echo "无法找到对应脚本执行3s后回到主界面"&&sleep 3s&&start_menu
	;;
esac
	
}


# 安装V2Ray
v2ray_manager(){
	echo
	echo '>>>>>>>>>>>>>>>>>>>>>>>>> V2Ray管理脚本 >>>>>>>>>>>>>>>>>>>>>>>>>>>>>'
	echo '1 - 启动服务'
	echo '2 - 停止服务'
	echo '3 - 重启服务'
	echo '4 - 查看配置'
	echo '5 - 退出脚本'
	echo	
	read -p "请输入数字:" num
	case $num in
		1)
			systemctl start v2ray || v2ray_manager
			;;
		2)
			systemctl stop v2ray || v2ray_manager
			;;
		3)
			systemctl start v2ray || v2ray_manager
			;;
		4)  
			echo$(cat /etc/v2ray/config.json) || v2ray_manager
			;;
		5)
			start_menu
			;;
		*)	
			echo&&echo "无法找到对应脚本执行3s后回到主界面" && sleep 3s &&	v2ray_manager
			;;
	esac
}

bbr_manager(){
	echo '>>>>>>>>>>>>>>>>>>>>> BBR加速模块配置 >>>>>>>>>>>>>>>>>>>>>'
	echo
	bash <(curl -s https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh)
	start_menu
}

install_blog(){
	echo '安装老陈皮'
}

start_menu


```