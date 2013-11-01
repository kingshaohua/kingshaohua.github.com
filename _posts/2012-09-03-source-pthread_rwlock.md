---
layout: post
title: "读写锁pthread_rwlock源码分析"
description: "读写锁pthread_rwlock源码分析"
category: 锁,源码分析
tags: [rwlock,Linux]
---

## 写在之前 ##

本来想写一个读写锁源码的详细注释，后来想想还是算了。原因是工作量有点大，肯定会很长很长，这样会让人不想看下去（因为会有很多废话），而且更建议读者自己去分析源码，这样才能收获更多。所以，本文就传达一些自己之前并不知道的东西。

## 源码说明 ##

读写锁在pthread.h中。它的实现和定义是在glibc中而不是在内核中。下载glibc源码研究之。本文分析基于glibc-2.16.0。

关于线程模块的源码位于 glibc-2.16.0/nptl目录下


## 目录说明 ##

glibc-2.16.0/nptl 下都是.c的文件，很自然的就想到这些都是实现。

glibc-2.16.0/nptl/Sysdeps下的一些目录i386,powerpc,X86_64等等这些都是基于不同平台的一些特殊处理。

glibc-2.16.0/nptl/sysdeps/pthread目录是总的头文件目录。其中的pthread.h应该就是我们程序中要包含的头文件。

## 读写锁属性设置 ##

优先级设置：这边的优先指的是读写的优先。就是说当writer想加锁时，是否能阻塞住后面的reader（不参与竞争）。如果不能阻塞可能会引起等饿死（reader一直加锁，writer只能阻塞在那了）。

通过pthread_rwlockattr_setkind_np可以设置读写锁类型(优先级)，有以下几种


    enum
    {
      PTHREAD_RWLOCK_PREFER_READER_NP,//读者优先
      PTHREAD_RWLOCK_PREFER_WRITER_NP,
      PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP,//写者优先
      PTHREAD_RWLOCK_DEFAULT_NP = PTHREAD_RWLOCK_PREFER_READER_NP
    };


PTHREAD_RWLOCK_PREFER_READER_NP和PTHREAD_RWLOCK_PREFER_WRITER_NP效果一样：都是reader优先，PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP是writer优先。**默认设置是reader优先。**

**PS:AIX和solaris下都没有设置优先级的函数。暂时还没研究。**


## 溢出与最大读者 ##

之前一篇文章提到如果先unlock,在rdlock或者wrlock就会报错。参看源码:

    
    /* Unlock RWLOCK.  */
    int
    __pthread_rwlock_unlock (pthread_rwlock_t *rwlock)
    {
    //省略
      if (rwlock->__data.__writer)
    rwlock->__data.__writer = 0;
      else
    --rwlock->__data.__nr_readers;//如果没有writer就直接递减readers,若开始便调用unlock这时__nr_readers = -1
    //省略
      return 0;
    }
    /* Acquire read lock for RWLOCK.  */
    int
    __pthread_rwlock_rdlock (rwlock)
     pthread_rwlock_t *rwlock;
    {
      int result = 0;
    //省略
      /* Get the rwlock if there is no writer...  */
      if (rwlock->__data.__writer == 0
    	  /* ...and if either no writer is waiting or we prefer readers.  */
    	  && (!rwlock->__data.__nr_writers_queued
    	  || PTHREAD_RWLOCK_PREFER_READER_P (rwlock)))
    	{//若属性不是设置为PTHREAD_RWLOCK_PREFER_READER_NP，则写者并不优先，可能会等饿死
    	  /* Increment the reader counter.  Avoid overflow.  */
    	  if (__builtin_expect (++rwlock->__data.__nr_readers == 0, 0))
    	{//刚才的__nr_readers =-1,所以++后就为0了。逻辑就认为溢出了然后就返回EAGAIN。
    	//这个和所谓的&quot;最大读者&quot;所导致的效果相同
    	  /* Overflow on number of readers.	 */
    	  --rwlock->__data.__nr_readers;
    	  result = EAGAIN;
    	}
    	  else
    	LIBC_PROBE (rdlock_acquire_read, 1, rwlock);
    
    	  break;
    	}
    //省略
      return result;
    }


PS：实际上，达到“最大读者”的时候，后面reader会阻塞在那，而不会返回错误。这个之前的实验有验证过。

## futex ##

进入rwlock都有lll_lock和lll_unlock底层锁保证锁的互斥，所以虽然是读写锁，但从某种层面上说还是会互斥的（有点乱就去看源码吧：）。

lll_lock和lll_unlock是通过futex来实现的。具体可以google “futex”。

推荐几篇文章：

[http://blog.csdn.net/nellson/article/details/5400360](http://blog.csdn.net/nellson/article/details/5400360 "http://blog.csdn.net/nellson/article/details/5400360")

[http://blog.chinaunix.net/uid-7190305-id-3051059.html](http://blog.chinaunix.net/uid-7190305-id-3051059.html "http://blog.chinaunix.net/uid-7190305-id-3051059.html")

关于futex的几个特点：

1、futex通过在用户态的检测，如果有竞争就不用陷入内核提高low-contention的效率。

2、直接使用futex系统调用来实现进程间的同步是有问题的。要结合futex同步机制来实现同步。

3、对于用户态的竞争判断是通过CAS来实现的，各平台实现不同。
