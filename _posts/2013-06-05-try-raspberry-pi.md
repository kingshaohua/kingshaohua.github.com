---
layout: post
title: "raspberry pi试用小结"
description: "树莓派试用"
category: linux
tags: [raspberry pi]
---

**购买**  
我是在淘宝上入手了一个红色树莓派（512内存,700Mhz），权当是六一礼物吧 ：）。上图
![](/media/img/raspberry_1.png)
我买了一个板子（送壳子，电源头），SD卡(8G class4),电源线，带电源的usb集线器，HDMI线，HDMI转VDI转接头，散热铜片。  
买回来后还是有点小后悔，我对接显示器，接电视需求不大，树莓派完全可以远程登录，并且用xmanager远程桌面，所以也不需要外接鼠标和键盘。淘宝上一并买的配件性价比也不高。    
所以，准备入手的同学可以先买个板子(大陆有授权官网，也送壳子)，电源线（若有就不用买了），电源头（苹果的插座就可以），SD卡（这方面兼容性没研究过）。其他配件可以等试用之后，看自己需要再购买，买回来不用就浪费了:(  。另外板子用了一会还是蛮烫的，散热片也可以买个，至于效果就不知道了。

----------

**不外接显示器、键盘、鼠标，远程登录Raspberry pi **  
1、首先向SD卡上写上系统镜像，网上教程多的是，这边就不赘述了。  
2、写好镜像，接上电源和网线，上图：
![](/media/img/raspberry_2.png)
3、搜索IP:在电脑上用ipscan网段扫描工具，扫描局域网内的主机，如果看见一个叫Raspberrypi的设备，就是它了。如果是能登陆上路由器的话，在路由器的主机列表里也很容易看到。  
4、知道ip后直接用ssh登陆就可以，使用putty登陆，用户名pi,密码raspberrypi。  
5、登陆OK后，先进行基本配置  
    sudo raspi-config  
选择expand_rootfs，然后把整个系统的可用空间扩展到储存卡的大小。然后reboot重启就可以了。  
6、网上有人说用VNC进行远程桌面，那个还要装vncserver。而且如果没有网络的话，还没法装。这边我使用的xmanager来远程桌面。在自己的电脑上安装好xmanager.使用xstart，进行如下配置
![](/media/img/raspberry_3.png)
点击Run,就远程登陆上啦。上图：
![](/media/img/raspberry_4.png)
7、个人感觉512内存跑桌面系统太勉强了，稍微点个东西，cpu内存就飚的老高。所以，我不准备接显示器用桌面系统。直接后台命令行了。
PS：debian下设置apt-get代理的方法 
    sudo vi /etc/apt/apt.conf  
添加下列内容  
    Acquire {
    http::proxy "http://user:pass@yourProxyAddress:port"
    }  
没有用户名密码可以填http://yourProxyAddress:port  

----------

**下一步计划**  
准备把板子连入公网，直接在网络上访问，打造成一个网络主机，并且可以定时发送监控邮件。