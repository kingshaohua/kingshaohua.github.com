---
layout: post
title: "[填坑大作战]使用SFTP上传文件，并支持proxy"
description: "use sftp with proxy"
category: python
tags: [sftp,proxy,netcat,nc,curl,libssh2]
---
## 需求 ##

前段时间让实现一个上报功能的模块，要求很简单：使用SFTP上传文件，并支持proxy。一开始感觉分分钟搞定的事情，而且又有python这个大杀器。结果前前后后折腾了2周多，踩了不少坑，特此记录，以警后人。=_=  

## 初步方案 ##
使用python + pexpect + 系统自带的sftp程序。用python调用sftp程序来登陆远端并上传文件。sftp程序是交互式的，所以使用python的pexpect库来实现自动化交互。关于pexpect库的使用参考以下文章。

[http://www.ibm.com/developerworks/cn/linux/l-cn-pexpect1/](http://www.ibm.com/developerworks/cn/linux/l-cn-pexpect1/)  
[http://www.ibm.com/developerworks/cn/linux/l-cn-pexpect2/](http://www.ibm.com/developerworks/cn/linux/l-cn-pexpect2/)  
[http://pexpect.readthedocs.org/en/latest/](http://pexpect.readthedocs.org/en/latest/)  

python很好用，pexpect也很好用，一切都很美好，直到开始实现proxy机制（下文会详细阐述）。由于种种原因，最终放弃了此方案，不过以下代码除了不能完美支持proxy(可以支持免密码登陆的proxy)，其他功能都OK。所以虽然最终没有用上，但还是很有借鉴意义。>_<

    
	#sftp for pexpect，python
	sftp_prompt='sftp>'
	
	class MY_SFTP(object):
		__server=''
		__port=''
		__user=''
		__password=''
		__home_path=''
		__child_sftp=''
		__logger=''
		__proxy_config={}
	
		__proxy_type=''
		__proxy_server=''
		__proxy_port=''
		__proxy_user=''
		__proxy_pwd=''
		"""sftp operation """
		def __init__(self):
			self.__logger = logging.getLogger()
			pass
		#connect to sftp
		#proxy_config = ('yes', '192.168.206.130', 'http', '8080', '', '')
		def login(self,server,port,user,password,home_path,proxy_config):
			bChanged = False
			if(self.__server != server or self.__port != port or \
				self.__user != user or self.__password != password or \
				self.__home_path != home_path or \
				self.__proxy_config != proxy_config):
				bChanged = True
			self.__server = server
			self.__port = port
			self.__user = user
			self.__password = password
			self.__home_path = home_path
			self.__proxy_config = proxy_config
	
			if(False == bChanged and True == self.isConnect()):
				#skip...
				return 0;
			#disconnect first
			self.disConnect()
			try:
				cmd = 'sftp '
				cmd += ' -oPort='+self.__port+' '
				if(self.__proxy_config[0] != 'no'):
					cmd += '-o "ProxyCommand nc '
					if(self.__proxy_config[2] == 'http'):
						cmd += ' -X connect '
					elif(self.__proxy_config[2] == 'socks4'):
						cmd += ' -X 4 '
					elif(self.__proxy_config[2] == 'socks5'):
						cmd += ' -X 5 '
					cmd += ' -x '+self.__proxy_config[1]+':'+self.__proxy_config[3]+' %h %p " '
	
				#TODO:proxy
				cmd += self.__user+'@'+self.__server
				self.__logger.info('SFTP=['+cmd+'],it will take a few minutes.')
				self.__child_sftp = pexpect.spawn(cmd,timeout=100,)
				self.__child_sftp.isalive()
				while True:
					index = self.__child_sftp.expect(['.*yes/no.*','.*password.*',pexpect.EOF,'.+'])
					#logging sftp output
					self.__logger.info('SFTP=[ '+str(self.__child_sftp.after)+' ]')
					if index == 0:
						'''
						for example:
	
						'''
						self.__child_sftp.sendline('yes')
					elif index == 1:
						'''
						for example:
							root@192.168.206.130's password:
						'''
						self.__child_sftp.sendline(self.__password)
						if 0 == self.__child_sftp.expect([sftp_prompt,'Permission denied']):
							#change sftp home path
							if(self.sftp_change_dir(self.__home_path) != 0):
								self.__child_sftp.close()
								return -1;
						else:
							self.__child_sftp.close()
							self.__logger.error('pwd[%s] is not correct for %s@%s',self.__password,self.__user,self.__server)
							return -1
						break
					elif (index == 2):
						return -1
					elif (index == 3):
						'''
						some other info
						'''
						pass
			except Exception as e:
				self.__logger.error(e, exc_info=True)
				return -1
			self.__logger.info('Finish init_sftp')
			return 0
	
		#change sftp dir
		def sftp_change_dir(self,path):
			try:
				if(False == self.isConnect()):
					self.__logger.err("sftp not alive")
					return -1
	
				self.__child_sftp.sendline('cd '+path)
				i = self.__child_sftp.expect(['.*No such file or directory.*','.*is not a directory.*',sftp_prompt])
				if (0== i or 1==i):
					self.__logger.error('cd path['+str(path)+'] failed.SFTP=[ '+str(self.__child_sftp.after)+' ]')
					return -1
			except Exception as e:
				self.__logger.error(e,exc_info=True)
				return -1
			return 0
	
		#put zip file via sftp
		def sftp_put(self,filename):
			try:
				if(False == self.isConnect()):
					self.__logger.error("sftp not connect...")
					return -1
	
				self.__child_sftp.sendline('put '+filename)
				self.__child_sftp.expect('sftp>')
				self.__logger.info('SFTP=[ '+str(self.__child_sftp.before)+' ]')
			except Exception as e:
				self.__logger.error(e,exc_info=True)
				return -1
			return 0
	
	
	
		#if connect or not
		def isConnect(self):
			return ("" !=self.__child_sftp and True == self.__child_sftp.isalive())
	
		#disconnect sftp
		def disConnect(self):
			if(self.isConnect()):
				self.__logger.info("sftp disconnect...")
				self.__child_sftp.close()
			return 0

## SFTP添加代理支持 ##

这里首先说明下SFTP,FTP,FTPS,这三个是不同的东西！参考：[http://blog.csdn.net/cuker919/article/details/6403925](http://blog.csdn.net/cuker919/article/details/6403925 "Sftp和ftp 区别、工作原理等（汇总ing）")  
主要注意几点：  
- SFTP不等于FTPS，虽然它们都提供了安全型的FTP机制。  
- FTPS是基于FTP协议的提升方案。而SFTP却是基于SSH的，作为SSH的一个工具，并兼容FTP的交互命令。因此FTP的默认端口是21，而SFTP的是22。SFTP也没有什么主动模式和被动模式，它其实就是个SSH链接。  

sftp程序可以通过Proxycommand选项来配置proxy链接。ssh也提供此选项，由此可以看出他两一脉相承。  
好了，开始坑爹了~，撇开网上各种不靠谱的解决方案。最终结论如下：
Proxycommand 中可以使用netcat(简写成nc)来连接proxy。

比如有SFTP服务端（ip=192.168.206.128,user=root,password=123456,使用默认端口22）

**HTTP Proxy,proxy=192.168.206.130:8080,没有密码**  

    sftp -o "ProxyCommand nc -v -X connect -x 192.168.206.130:8080 %h %p" root@192.168.206.128  


**Socks4 Proxy,proxy=192.168.206.130:10081,没有密码**  

    sftp -o "ProxyCommand nc -v -X 4 -x 192.168.206.130:10081 %h %p" root@192.168.206.128  

**Socks5 Proxy,proxy=192.168.206.130:10080,user=test,pwd=123456**  

    sftp -o "ProxyCommand nc -v -X 5 -P test -x192.168.206.130:10080 %h %p" root@192.168.206.128  

nc由于写的比较早，现在有很多版本，有些版本已经不更新了。有些不支持代理功能，有些不支持密码认证。（说多了都是泪），我的机器上没有netcat-openbsd，没有找到安装包。折腾了好几天之后，快到版本期限了，**赶紧放弃此方案**。 因此上述3个命令，只有第一个http proxy的命令验证过。
 
**netcat-traditional**  
> 不提供代理支持

	[root@localhost bin]# nc -h
	GNU netcat 0.7.1, a rewrite of the famous networking tool.
	Basic usages:
	connect to somewhere:  nc [options] hostname port [port] ...
	listen for inbound:    nc -l -p port [options] [hostname] [port] ...
	tunnel to somewhere:   nc -L hostname:port -p port [options]
	
	Mandatory arguments to long options are mandatory for short options too.
	Options:
	  -c, --close                close connection on EOF from stdin
	  -e, --exec=PROGRAM         program to exec after connect
	  -g, --gateway=LIST         source-routing hop point[s], up to 8
	  -G, --pointer=NUM          source-routing pointer: 4, 8, 12, ...
	  -h, --help                 display this help and exit
	  -i, --interval=SECS        delay interval for lines sent, ports scanned
	  -l, --listen               listen mode, for inbound connects
	  -L, --tunnel=ADDRESS:PORT  forward local port to remote address
	  -n, --dont-resolve         numeric-only IP addresses, no DNS
	  -o, --output=FILE          output hexdump traffic to FILE (implies -x)
	  -p, --local-port=NUM       local port number
	  -r, --randomize            randomize local and remote ports
	  -s, --source=ADDRESS       local source address (ip or hostname)
	  -t, --tcp                  TCP mode (default)
	  -T, --telnet               answer using TELNET negotiation
	  -u, --udp                  UDP mode
	  -v, --verbose              verbose (use twice to be more verbose)
	  -V, --version              output version information and exit
	  -x, --hexdump              hexdump incoming and outgoing traffic
	  -w, --wait=SECS            timeout for connects and final net reads
	  -z, --zero                 zero-I/O mode (used for scanning)
	
	Remote port number can also be specified as range.  Example: '1-1024'


**netcat in centos**  
> centos6上的最新版本，不提供密码认证，nc-1.84-22.el6.src.rpm

	[root@localhost lib64]# nc -h
	usage: nc [-46DdhklnrStUuvzC] [-i interval] [-p source_port]
	          [-s source_ip_address] [-T ToS] [-w timeout] [-X proxy_version]
	          [-x proxy_address[:port]] [hostname] [port[s]]
	        Command Summary:
	                -4              Use IPv4
	                -6              Use IPv6
	                -D              Enable the debug socket option
	                -d              Detach from stdin
	                -h              This help text
	                -i secs         Delay interval for lines sent, ports scanned
	                -k              Keep inbound sockets open for multiple connects
	                -l              Listen mode, for inbound connects
	                -n              Suppress name/port resolutions
	                -p port         Specify local port for remote connects
	                -r              Randomize remote ports
	                -S              Enable the TCP MD5 signature option
	                -s addr         Local source address
	                -T ToS          Set IP Type of Service
	                -C              Send CRLF as line-ending
	                -t              Answer TELNET negotiation
	                -U              Use UNIX domain socket
	                -u              UDP mode
	                -v              Verbose
	                -w secs         Timeout for connects and final net reads
	                -X proto        Proxy protocol: "4", "5" (SOCKS) or "connect"
	                -x addr[:port]  Specify proxy address and port
	                -z              Zero-I/O mode [used for scanning]
	        Port numbers can be individual or ranges: lo-hi [inclusive]  

**netcat-openbsd**  
> 提供代理+密码认证，ubuntu上自带的。

	NAME
	     nc - arbitrary TCP and UDP connections and listens
	
	SYNOPSIS
	     nc [-46DdFhklNnrStUuvz] [-I length] [-i interval] [-O length]
	        [-P proxy_username] [-p source_port] [-s source] [-T toskeyword]
	        [-V rtable] [-w timeout] [-X proxy_protocol] [-x proxy_address[:port]]
	        [destination] [port]  

## CURL方案 ##

在此祭出大名鼎鼎的libcurl库。其实我也是第一次听说，不然早就这么干了(-_-!).关于curl基本介绍，大家就自行查阅吧。虽然它主要是用来抓取网页，但也提供了对一些其他协议的封装。使用`curl -V`（大写V）可以查询本机上curl所支持的协议。php和python都

	[root@localhost bin]# curl -V
	curl 7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.34.0 OpenSSL/1.0.0 zlib/1.2.3 libidn/1.18 libssh2/1.4.3
	Protocols: dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp scp sftp smtp smtps telnet tftp 
	Features: IDN IPv6 Largefile NTLM SSL libz  

可以你所查到的协议支持没有这么多，因为如果想要支持sftp的话，curl就需要libssh2的支持，编译libssh2时也需要openssl的支持。因此首先讲下如何编译出支持sftp协议的curl。  

**编译ssl** 

    ./config --prefix=/build/openssl_64 threads shared
    make clean
    make
    make install  

**编译libssh2**

	./configure --prefix=/build/ssh_64  --with-openssl  --with-libssl-prefix=/build/openssl_64
	make
	make install

**编译curl**  

	./configure --prefix=/build/curl_64 --with-ssl=/root/Downloads/openssl-1.0.1h  --with-libssh2=/build/ssh_64
	make
	make install

> PS:这边--with-ssl没有使用/build/openssl_64而使用原来ssl的编译目录，因为直接使用ssl的输出目录，编译会报错。（问题暂时未查）

好了，现在curl支持了sftp了。附上几段示例代码：

**php测试是否能链接sftp**

	$ch = curl_init();
	
	curl_setopt($ch, CURLOPT_URL, "sftp://192.168.206.128/root/my_dir/);
	curl_setopt($ch, CURLOPT_USERPWD,"root:123456");
	curl_setopt($ch, CURLOPT_PORT, 22);
	curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_SFTP);
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
	$proxy_type='http'; //set your proxy type
	switch ($proxy_type) {
		case 'http':
			curl_setopt($ch, CURLOPT_HTTPPROXYTUNNEL, 1);
			curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_HTTP);
			break;
		case 'socks4':
			curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_SOCKS4);
			break;
		case 'socks5':
			curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_SOCKS5);
			break;
		default:
			break;
	}
	curl_setopt($ch, CURLOPT_PROXY, 192.168.206.128);
	curl_setopt($ch, CURLOPT_PROXYPORT,8080);
	$proxy_auth=''// $proxy_auth="test:123456";
	if($proxy_auth != ''){
		curl_setopt($ch, CURLOPT_PROXYUSERPWD,$proxy_auth);
	}
	curl_exec($ch);
	$error='';
	if (curl_errno($ch) == 0) {
	    $error = 'ok';
	} else {
	    $error = curl_error($ch);
	}

**python测试是否能连接sftp**

	def test_link():
		try:
			output = StringIO.StringIO()
			sftp = pycurl.Curl()
			sftp.setopt(pycurl.URL, 'sftp://192.168.206.128/root/mydir/')
			sftp.setopt(pycurl.USERPWD,"root:123456")
			sftp.setopt(pycurl.PORT,22)
			sftp.setopt(pycurl.WRITEFUNCTION, output.write)
			sftp.setopt(pycurl.UPLOAD,0)
			#proxy setting
			sftp.setopt(pycurl.PROXY,"192.168.206.130")
			sftp.setopt(pycurl.PROXYPORT, 8080)
			proxy_type='http' #set your proxy type
			if(proxy_type == 'http'):
				sftp.setopt(pycurl.HTTPPROXYTUNNEL, 1)
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_HTTP)
			elif(proxy_type == 'socks4'):
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_SOCKS4)
			elif(proxy_type == 'socks5'):
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_SOCKS5)
			proxy_auth='' #proxy_auth='test:123456'
			if(proxy_auth!=''):
				sftp.setopt(pycurl.PROXYUSERPWD,proxy_auth)
	
			sftp.perform()
		except pycurl.error, error:
				errno, errstr = error
				return (-1,errstr)
	
		return (0,'link ok')


**python上传文件到sftp**

	def upload_file():
		try:
			filename='/usr/local/file_to_sftp/1.txt'
			sftp = pycurl.Curl()
			sftp.setopt(pycurl.URL, 'sftp://192.168.206.128/root/mydir/1.txt')
			sftp.setopt(pycurl.USERPWD,"root:123456")
			sftp.setopt(pycurl.PORT,22)
			sftp.setopt(pycurl.READFUNCTION, open(filename, 'rb').read)
			filesize = os.path.getsize(filename)
			sftp.setopt(pycurl.INFILESIZE, filesize)
			sftp.setopt(pycurl.UPLOAD,1)
			#proxy setting
			sftp.setopt(pycurl.PROXY,"192.168.206.130")
			sftp.setopt(pycurl.PROXYPORT, 8080)
			proxy_type='http' #set your proxy type
			if(proxy_type == 'http'):
				sftp.setopt(pycurl.HTTPPROXYTUNNEL, 1)
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_HTTP)
			elif(proxy_type == 'socks4'):
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_SOCKS4)
			elif(proxy_type == 'socks5'):
				sftp.setopt(pycurl.PROXYTYPE, pycurl.PROXYTYPE_SOCKS5)
			proxy_auth='' #proxy_auth='test:123456'
			if(proxy_auth!=''):
				sftp.setopt(pycurl.PROXYUSERPWD,proxy_auth)
	
			sftp.perform()
		except pycurl.error, error:
				errno, errstr = error
				print errstr
		return  

有几点需要注意的地方:  

- 如果不设置 CURLOPT_RETURNTRANSFER 选项（python中没有此选项，使用pycurl.WRITEFUNCTION实现相同效果），perform或者curl_exec之后会把结果输出到标准输出上。可能这并不是想要的结果。因此通过此选项可以截获结果数据。  

- 对于http类型的代理，如果想要在其代理上传过sftp/ssh协议，需要设置HTTPPROXYTUNNEL为1。使其变成http隧道。不然,http代理只支持http的访问方式。  


- curl的使用都是一堆setopt之后最后调用一次“执行”。这和交互式的FTP不通，用交互式访问FTP，获取上传文件分为两步：1、链接。2、切换目录。3、上传文件。因此用curl上传文件时，在setopt设置URL时需要**全路径**（比如"sftp://192.168.206.128/root/mydir/1.txt",就是上传结束后，远端服务器上该文件的全路径）  

## 写在最后 ##
一个简单的功能，折腾到这么久，归根结底还是对用“脚本来实现功能”这一模式不熟练。对第三方开源库了解甚少。在解决各种坑上面走了很多弯路。  

> 因此，填坑大作战正式开始！
