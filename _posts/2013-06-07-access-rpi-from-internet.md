---
layout: post
title: "把raspberry pi抛到internet上"
description: "从internet访问树莓派"
category: raspberry
tags: [raspberry]
---
**驱动安装**  

我买的是TP-LINK TL-WN725N。树莓派的兼容指南中说，这个725N有两个版本：v1版本可以即插即用，v2版本是无需外接电源的，但是需要手动安装驱动。本着折腾的心理，就在亚马逊买了个。收到货拆开一看，USB口上直接印了个v2。（喜忧参半吧）。  
首先回来插上去之后，敲命令lsusb。表示果然不识别，ifconfig也看不到wlan0.然后就按国内一论坛上放出来的方法。下载了人家编译好的ko。然后重启什么的。果然不好用。再次无力吐槽一下，有些人给的方案说的言简意赅，基本没用，害的我还重刷了一下SD卡。 
其实如果lsusb看不到产品型号，ifconfig能看见wlan0就可以了。按照国人论坛上的方法折腾了两个小时，最后找到国外的一篇文章：
[http://blog.pi3g.com/2013/05/tp-link-tl-wn725n-nano-wifi-adapter-v2-0-raspberry-pi-driver/](http://blog.pi3g.com/2013/05/tp-link-tl-wn725n-nano-wifi-adapter-v2-0-raspberry-pi-driver/ "http://blog.pi3g.com/2013/05/tp-link-tl-wn725n-nano-wifi-adapter-v2-0-raspberry-pi-driver/")  
大体意思和之前论坛里的差不多，但是人家的方法说明很有条理。一路顺着做下来，ifconfig 果然看到了waln0。  
这边说明一下配置。那个文章里给的配置是这样的：

    auto lo
    
    iface lo inet loopback
    #iface eth0 inet dhcp
    
    allow-hotplug wlan0
    auto wlan0
    iface wlan0 inet dhcp
    wpa-ssid "your-ssid"
    wpa-psk "your-password"
    #wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
    iface default inet dhcp
我的配置是

    auto lo
    
    iface lo inet loopback
    iface eth0 inet dhcp
    
    allow-hotplug wlan0
    auto wlan0
    iface wlan0 inet dhcp
    wpa-ssid "TP-LINK_6C1DB8"
    wpa-psk  "123456"
    #iface wlan0 inet manual
    
    #wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
    iface default inet dhcp
区别就是我保留了iface eth0 inet dhcp，这个是有线网卡的配置，我担心无线配置错误,远程登录不了，又没有显示器，就只能重新刷SD卡了。之前按论坛上的配置，最后就是的。

----------

**域名绑定动态IP **
 
1、我的树莓派是连在无线路由器上的，所以得先在路由器里将其配置成DMZ主机，就可以对外网暴露了。  
2、路由器重启之后，公网IP就会变化。所以得使用域名绑定动态IP的方法，有一种是用“花生壳”，我用的是dnspod，我参考的是：  
[http://nfplayer.com/rpi/211.html](http://nfplayer.com/rpi/211.html "http://nfplayer.com/rpi/211.html")
（这个里面有个错误，第二步获取record_id应该是用curl -k https://dnsapi.cn/Record.List -d "login_email=xxx&login_password=xxx&domain_id=xxx"） 
最后把那个python脚本放在开机启动项里就行了。  
注意:  
远程连接树莓派可能不支持中文，所以那个python脚本里有可能中文，比如"默认"，所以直接在本地编辑好脚本后，用sftp上传到树莓派中就可以了，不用远程上去编辑中。  
所以，每次只要查看dnspod里绑定域名的ip就知道我们的树莓派目前在公网的IP了。

----------

**远程访问**  
本来呢，直接用SSH访问就OK了，可惜如果在一些特定区域，自己没法直接连入公网（比如在公司里需要挂代理才能访问外网），尝试了在WinScp里代理访问SSH也不好用。最后想到了使用web ssh方式，其实这已经不是一个ssh，就是一个简易的web交互平台，利用了node.js强大的后端交互能力。我用的是tty.js。具体方法可以参考 [ http://www.fritz-hut.com/raspberry-pi-tty-js/]( http://www.fritz-hut.com/raspberry-pi-tty-js/ " http://www.fritz-hut.com/raspberry-pi-tty-js/")  
需要注意的是：  
（1）安装好之后，node的启动命令是nodejs不是node。所以启动tty.js的命令是

    nodejs tty.js
（2）tty.js启动好之后，ps看只有tty.js。  
（3）tty.js也不稳定，有时候在网页上打开不了终端，暂时不知道怎么回事，可以用手机远程到树莓派上把这个tty.js进程kill掉，重启就好了。  
（4）在手机上装个ssh客户端，应急的时候还是相当管用啊。

----------
**最后**

就先写到这了，困死了，接下来准备考虑在上面搭个爬虫程序。一直想玩爬虫的说。