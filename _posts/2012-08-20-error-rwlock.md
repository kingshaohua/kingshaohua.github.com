---
layout: post
title: "读写锁、解决pthread_rwlock_rdlock返回错误码11 EAGAIN错误"
description: "读写锁、解决pthread_rwlock_rdlock返回错误码11 EAGAIN错误"
category: 错误解决
tags: [rwlock,Linux]
---

最近项目中有用到linux下的读写锁。本来以为挺简单的：不就是可以多读单写么？实际上真正用起来还是遇到些问题了，堵了我一下午的时间。

关于读写锁的使用，可以参考下面的文章或者google搜索 **linux 读写锁**：

[http://blog.csdn.net/dai_weitao/article/details/1752843](http://blog.csdn.net/dai_weitao/article/details/1752843 "http://blog.csdn.net/dai_weitao/article/details/1752843")


我在项目中使用的时候遇到这样的问题：

在调用`int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); `这个接口的时候没有堵塞，却返回错误码11。

然后man pthread_rwlock_rdlock看错误码介绍：

> 
> ERRORS
> 
> The pthread_rwlock_tryrdlock() function shall fail if:
> 
> **EBUSY** The read-write lock could not be acquired for reading because a writer holds the lock or a writer with the appropriate priority was blocked on it.
> 
> The pthread_rwlock_rdlock() and pthread_rwlock_tryrdlock() functions may fail if:
> 
> **EINVAL** The value specified by rwlock does not refer to an initialized read-write lock object.
> 
> **EAGAIN** The read lock could not be acquired because the maximum number of read locks for rwlock has been exceeded.
> 
> The pthread_rwlock_rdlock() function may fail if:
> 
> **EDEADLK** The current thread already owns the read-write lock for writing.
> 
> 
> These functions shall not return an error code of **EINTR**.
> 

关于错误码的值可以参考[http://blog.chinaunix.net/uid-12555930-id-2929771.html](http://blog.chinaunix.net/uid-12555930-id-2929771.html "http://blog.chinaunix.net/uid-12555930-id-2929771.html")，当然也可以用strerror打印出描述信息。

错误码**11**表示的是Resource temporarily unavailable也就是**EAGAIN**

对照man里的描述来看，原因就是：读锁加的太多了，超出最大可读者数了。

然后上网查了下关于“linux下读写锁的最大可读数”。在此不得不吐槽一下，网上把文章copy来copy去的，直接贴到自己的blog中，这种做法实在太容易误导人了。最后查出来的结论是：最大可读数一般为逻辑CPU的个数（还不止一处查到）。当时就囧了：像我们现在的机器也就双核。要是以前可都是单核。这读写锁的最大可读数要是为逻辑CPU的个数，那岂不是和互斥锁差不了多少嘛？

然后我就写了段小代码测试了一下（代码在下面，大家可以自行修改测试）：

我的测试环境是：虚拟机，单核，redhat5 32位

最后结论是：

（1）在单线程内，锁可以`pthread_rwlock_rdlock()`无数次。

（2）最大可读数为300。（网上说的不能全信，copy-copy很容易三人成虎）

（3）当达到最大可读数的时候，后面的加读锁的线程会阻塞。

（4）当被写锁锁住时，后面的读锁线程会被阻塞。

测试过程中并没有出现返回错误码的情况。我想肯定是我哪边用错了。于是我在项目的加锁解锁的地方加上当前读锁和写锁的计数器。再次跑了下，发现问题了：开始正常加锁解锁没有问题，**但当对没有加（读写）锁的锁调用pthread_rwlock_unlock后，再调用pthread_rwlock_rdlock就返回错误码11**。于是，我在测试案例中也测试了下，果然报错了。修复后，程序便能正常运行了。

关于为什么会报错，还需要深入源码。以后有研究再发出来。

测试代码：

    //main.c
    #include &lt;pthread.h&gt;
    #include &lt;stdio.h&gt;
    
    pthread_rwlock_t	mutex;
    pthread_rwlockattr_t mattr; 
    //写线程
    void * write_thread(void *arg)
    {
    	int iRet = 0;
    	if((iRet = pthread_rwlock_wrlock(&mutex)) != 0)
    	{
    		printf("write end[%d]\n",iRet);
    	}
    	printf("write finish[%d]\n",iRet);
    	sleep(2);
    }
    //读线程
    void * reader_thread(void *arg)
    {
    	int iRet = 0;
    	if((iRet = pthread_rwlock_rdlock(&mutex)) != 0)
    	{
    		printf("reader end[%d]\n";,iRet);
    	}
    	printf("reader finish[%s]\n";,strerror(iRet));
    	
    	sleep(1);
    }
    //测试
    int TestMaxRWLockNum()
    {
    	int iRet = 0;
    	int i = 0;
    	iRet = pthread_rwlockattr_init(&mattr);
    	
    	if(iRet != 0)
    	{
    		return iRet;	
    	}
    	//进程间使用	
    	iRet = pthread_rwlockattr_setpshared(&mattr, PTHREAD_PROCESS_SHARED);
    	if(iRet != 0)
    	{
    		return iRet;	
    	}
    	iRet = pthread_rwlock_init(&mutex, &mattr);
    	if(iRet != 0)
    	{
    		return iRet;	
    	}
    	pthread_rwlock_unlock(&mutex);//测试先解锁会导致错误码11
    	pthread_t reader;
    	
    	for(i = 0; i &lt; 10; i++)	  
    	{
    		if(i % 2)
    		{
    			pthread_create(&reader, NULL, reader_thread, NULL);//创建读者
    		}
    		else
    		{
    			pthread_create(&reader, NULL, write_thread, NULL);//创建写者
    		}
    	}
    	sleep(30);
    	printf("end[%d]\n",iRet);
    	return iRet;
    }
    
    int main(int argc,char * argv[])
    {
    	TestMaxRWLockNum();
    	return 0;
    }


Makfile如下：

    all:
    gcc -g -c main.c
    gcc -g -o main -lpthread main.o
