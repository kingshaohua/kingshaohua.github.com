---
layout: post
title: "使用Jenkins搭建云构建平台"
description: "使用Jenkins搭建云构建平台"
category: jenkins
tags: [jenkins,持续构建]
---
**Why 需要云构建?**  

最近要对一个开发中的项目进行开发调试，该项目至少需要部署在4个节点上。因此每次代码便跟之后，手动上传编译代码变得极为无趣。因此，需要使用持续构建工具来解放双手。Jenkins（前身是hudson）就是这样一个工具。

**Why 用Jenkins?**  

1、方便！方便！方便！  

a)使用Master/slave模式构建云构建平台。只需要在Master上运行Jenkins。Slave上可以无需安装任何东西。（当然一些基础工具还是要安装的，另外高阶用法这边不做讨论）  

b)Jenkins无需数据库支持，无需web 容器支持，只要一条命令java –jar Jenkins.war。剩下的就是界面配置jobs了。

详细介绍与使用可以查看：
[http://www.cnblogs.com/itech/archive/2011/11/23/2260009.html](http://www.cnblogs.com/itech/archive/2011/11/23/2260009.html "http://www.cnblogs.com/itech/archive/2011/11/23/2260009.html")

**How 用Jenkins?**  

环境要求：  
（1）java 版本要高于1.5  
（2）ant网上都说要装，我没装也可以构建。  
（3）我是在4台RedHat上构建。  

机器划分：
10.45.16.39作为Master机，10.45.16.41, 10.45.16.42, 10.45.16.44这三台作为Slave机。  

步骤：  
（1）上传jenkins.war到39机器的jenkins目录。执行java –jar Jenkins.war。  
（2）浏览器中访问http://10.45.16.39:8080/。就可以看到控制面板了。  
（3）在管理节点中添加3个slave机器：举例如下  
 ![](/media/img/use_jenkins_1.png)
PS：
网上说要配什么SSH的公钥密钥，我这边直接用（用户名+密码）也可以的。只要配置

Credentials就可以了。  
配置好之后如下所示  
![](/media/img/use_jenkins_2.png)

这边要注意的是：在slave 上执行jobs的时候会找不到该用户下的环境变量。所以在执行脚本之前要先载入用户环境变量。(source $HOME/.bash_profile)。  
（4）下面开始配置job。我的方案是，四台机器上执行相同的job。所以只要配置一个job，然后关联到四个节点上就可以了。  

**1、点击(新建job)**  
![](/media/img/use_jenkins_3.png)
  我们选择“构建多个配置项目”，因为我们想在构建的时候可以在多个节点上都执行。详细原因可以参考：  
[http://tonybai.com/2012/02/15/intergating-on-multiple-platforms-simultaneously-using-jenkins/](http://tonybai.com/2012/02/15/intergating-on-multiple-platforms-simultaneously-using-jenkins/ "http://tonybai.com/2012/02/15/intergating-on-multiple-platforms-simultaneously-using-jenkins/")  

**2、添加svn**    
![](/media/img/use_jenkins_4.png)
注意：
（1）如果第一次添加，会报认证失败。在错误信息里有个Add Credentials.填入用户名密码之后，就不报错了。如果还是报错，换个浏览器吧（我把遨游浏览器切换了下浏览器模式就好了）  
（2）Svn URL后面加了个@head,表示取最新版本的代码。如果不加，jenkins会取当前机器时间点的代码（如果主机上时间点不对的话，就会取到老版本的代码了）。网上有其他办法，但我感觉我的办法最好，窃喜。  
（3）Check-out Strategy，这边选择的是每次获取最新拷贝，这个看个人喜好了。  
（4）这边local module directory:这边我就写当前目录，待会下面会copy到用户目录下去。  

**3、将job绑定到主机节点**  
![](/media/img/use_jenkins_5.png)
注意：  
（1）要先选择下拉框中的Slaves才出现上面的框框。  
（2）选择要绑定的节点（这边我把四个节点都选上了。）  

**4、增加构建步骤（添加脚本）**
![](/media/img/use_jenkins_6.png)
注意：  
（1）. $HOME/.bash_profile ：用来载入用户环境变量，因为解决在slave节点上环境变量获取问题。（在配置节点的时候也可以设置环境变量，但感觉还是我的方法好，哈哈）。  
（2）cp -rf ./* $QuickMDB_HOME/：将jenkins各个job区的svn checkout出的代码拷贝到用户指定的目录。（还记得刚才的local module directory）。  
（3）Win2Unix.sh：是一个将window下文件编码转换成unix编码的脚本。(详见附录)  
（4）这边除了编译，也可以启停进程，能自动就自动，争取达到一键完成。  

**5、点击保存,job创建完毕了**。  

**6、执行脚本**
![](/media/img/use_jenkins_7.png)
这边我创建了两个job。点击立即构建就行了。因为我是作为开发测试用。就不配置定时构建任务了，每次需要的时候手动构建就行了。  
OK,生活如此美好~~。  

附录：Win2Unix.sh
	Usage()
	{
    	echo "Param Number can't be 0 \n"
        echo "usage:Win2Unix  Param1, Param2 ......\n"
        exit 1
	}
	Dos2Unix()
	{       
		cat $1 | perl -pe '~s/\r//g' > file_name_bak;
		mv file_name_bak $1;
	}
	Multi_Process()
	{
	cd $1
	DIR=`ls`
	for file_name in $DIR ; do
	    if [ -f "$file_name" ];then
	                Dos2Unix $file_name
	        elif [ -d "$file_name" ];then
	            Multi_Process $file_name
	        else
	            echo " $file_name is unkonwn type \n"
	        fi
	done
	cd ..
	}
	
	if [ $# -eq 0 ] ; then
	   Usage
	else 
	    echo "Transfor begin,Please wait...\n"
	    while [ "$1" ]
	        do
	       if [ -f "$1" ];then
	           Dos2Unix $1
	       else
	          Multi_Process $1
	                  echo "OK, $1: Transfor Windows format to Unix format succsssfully!"
	       fi;
	        shift
	        done
	fi  

