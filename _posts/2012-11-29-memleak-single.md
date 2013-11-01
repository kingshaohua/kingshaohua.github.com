---
layout: post
title: "单例模式下的析构与内存泄露"
description: "单例模式下的析构与内存泄露"
category: memleak
tags: [memleak]
---

最近系统改造了一段时间，以防一些误修改，该做一次性能优化了。首先做了下内存泄露测试。用的是valgrind。具体用法自己google吧。  

测试结果如下：  

    ==8953== LEAK SUMMARY:
    ==8953==definitely lost: 0 bytes in 0 blocks
    ==8953==indirectly lost: 0 bytes in 0 blocks
    ==8953==  possibly lost: 0 bytes in 0 blocks
    ==8953==still reachable: 60,328 bytes in 224 blocks
    ==8953== suppressed: 0 bytes in 0 blocks

Possibly，indirectly，definitely如果有lost。就要注意了，解决起来也很简单。Valgrind打印出的信息足够定位问题了。  

“still reachable: 60,328 bytes in 224 blocks”是指内存指针还在 还有机会使用或者释放。  

对此，参考overstackflow上的回复：  

[http://stackoverflow.com/questions/3840582/still-reachable-leak-detected-by-valgrind](http://stackoverflow.com/questions/3840582/still-reachable-leak-detected-by-valgrind "http://stackoverflow.com/questions/3840582/still-reachable-leak-detected-by-valgrind")

(回复较长就截取了一段)  

**In general, there is no need to worry about "still reachable" blocks.** They don't pose the sort of problem that true memory leaks can cause. For instance, there is normally no potential for heap exhaustion from "still reachable" blocks. This is because these blocks are usually one-time allocations, references to which are kept throughout the duration of the process's lifetime. While you could go through and ensure that your program frees all allocated memory, there is usually no practical benefit from doing so since the operating system will reclaim all of the process's memory after the process terminates, anyway. Contrast this with true memory leaks which, if left unfixed, could cause a process to run out of memory if left running long enough, or will simply cause a process to consume far more memory than is necessary.  

虽说不用考虑，但还是好奇的看了下。  

多加了粗体部分的参数就可以看到哪些地方触发了"still reachable" blocks
valgrind --tool=memcheck --leak-check=yes --leak-check=full --show-reachable=yes  

结果发现是使用单例模式地方，仔细想想，确实不曾考虑过单例模式的析构问题。例如下面的样例代码：  


    #include <iostream>
    using namespace std;
    class CSingleton
    {
    public:
    	static CSingleton* GetInstance();
    private:
    CSingleton(){};
    	static CSingleton * m_pInstance;
    };
    
    CSingleton * CSingleton::m_pInstance = NULL;
    CSingleton* CSingleton::GetInstance()
    {
    	if(NULL == m_pInstance)
    	{
    		m_pInstance = new CSingleton();
    	}
    	return m_pInstance;
    }
    int main()
    {
    	CSingleton::GetInstance();
    	return 0;
    }

m_pInstance这个指针指向的内存会在程序退出后由操作系统去回收内存。在程序退出前不能销毁也不需要销毁。所以我们也没考虑过这边的析构。但是我如果想要程序结束前，触发其析构（可能有时候不得不这么做。）  

上网搜了些资料确实有这方面的文章：  

[http://www.cnblogs.com/my_life/articles/2356709.html](http://stackoverflow.com/questions/3840582/still-reachable-leak-detected-by-valgrind "http://stackoverflow.com/questions/3840582/still-reachable-leak-detected-by-valgrind")  

主要思想是利用：嵌套类和静态类成员，利用程序在结束时析构全局变量的特性，选择最终的释放时机。（不过不得不吐槽网上copy来copy去的这些文章。自己也不尝试一下就贴到自己的博客里。文章里的代码根本跑不通的。）  

下面是我的测试代码：  

    #include <iostream>
    using namespace std;
    class CSingleton
    {
    	//其他成员
    public:
    	static CSingleton* GetInstance();
    	static CSingleton * m_pInstance;
    private:
    CSingleton(){};
    public:
    	class CGarbo //它的唯一工作就是在析构函数中删除CSingleton的实例
    	{
    	public:
    		~CGarbo()
    		{
    			printf("delete\n");
    			if( CSingleton::m_pInstance )
    				delete CSingleton::m_pInstance;
    		}
    	};
    private:
    	static  CGarbo Garbo; //定义一个静态成员，程序结束时，系统会自动调用它的析构函数
    };
    CSingleton::CGarbo CSingleton::Garbo;
    CSingleton * CSingleton::m_pInstance = NULL;
    CSingleton* CSingleton::GetInstance()
    {
    	return NULL;
    }
    int main()
    {
    	CSingleton::GetInstance();
    	printf("my\n");
    	return 0;
    }  

注意点：  

1、嵌套类CGarbo定义需要是public的不然，外部无法初始化。  

2、static CSingleton * m_pInstance;也要是public的，不然嵌套类中无法调用。  


修改了系统中的单例模式，再跑下Valgrind。果然之前报“still reachable”的地方就没有了。还有其他的地方，就“no need to worry”，接着还忙着要做性能测试的。  

另外性能测试推荐的工具是：vtune (只能适用在intel架构上，windows下收费，Linux下免费)。  

中午和同事讨论了下进程间的锁是到底是个什么样的东西。是在用户态的还是核心态的。最近要做一个内存镜像导出的工具。内存中有共享锁，所以使用镜像文件恢复内存的话不知道能不能跨机器恢复。目前感觉锁信息是保存在用户态的。还待考证。  

