---
layout: post
title: "Sun平台下的可变参数宏问题"
description: "Sun平台下的可变参数宏问题"
category: Solaris
tags: [sun,Solaris]
---
最近修改了一个项目中的日志组件，在打印函数的地方添加了文件名，行数。  
原来的日志宏是这样的：

    #define TADD_NORMAL Log

log函数的实现是

    void Log(const char * fmt, ...){ ....}

使用方式：

    TADD_NORMAL("记录日志");
    
    TADD_NORMAL("记录日志[%d]",1);

现在要添加文件名和行数。


方案一：

    TADD_NORMAL("[%s %d]:记录日志",__FILE__,__LINE__);

这样的方式无疑是弱爆了，需要修改所有用到日志组件的地方。

为了保证上层模块代码无需改变。可以使用可变参数宏：

方案二：

修改后的宏如下：

    #define TADD_ NORMAL(FMT,...) Log("[%s:%d]: " FMT, __FILE__,__LINE__,##__VA_ARGS__)

到这边一切似乎都很好，linux编译通过，程序输出正常。很开心的去过周末了。

周一过来后，同事还是叫了,你修改的日志组件，sun编译报错啦,(结尾逗号出现在参数列表中.).
所以重新回顾了下关于C++可变参数宏的使用,可以参考这篇博文：  
[http://www.cppblog.com/fwxjj/archive/2010/12/03/135364.html]([http://www.cppblog.com/fwxjj/archive/2010/12/03/135364.html "[http://www.cppblog.com/fwxjj/archive/2010/12/03/135364.html")

## 总结 ##

可变参数有以下几种形式

    #define ADD_LOG(…) printf(__VA_ARGS__)
    
    #define ADD_LOG(fmt,…) printf(fmt,##__VA_ARGS__)
    
    #define ADD_LOG(fmt,args…) printf(fmt,##args)

而我这边就是使用的第二种。为啥会报错呢？  

我的初步猜测是##__VA_ARGS__中的##没有起作用.因为这个是GNU CPP中进行了特别处理：  

GNU CPP还有两种更复杂的宏扩展，在标准C里，你不能省略可变参数，但是你却可以给它传递一个空的参数。例如，下面的宏调用在ISO C里是非法的，因为字符串后面没有逗号：debug ("A message")  

GNU CPP在这种情况下可以让你完全的忽略可变参数。在上面的例子中，编译器仍然会有问题（complain），因为宏展开后，里面的字符串后面会有个多余的逗号。
为了解决这个问题，CPP使用一个特殊的`##`操作。书写格式为：
    #define debug(format, ...) fprintf (stderr, format, ## __VA_ARGS__)

这里，如果可变参数被忽略或为空，`##`操作将使预处理器（preprocessor）去除掉它前面的那个逗号。如果你在宏调用时，确实提供了一些可变参数，GNU CPP也会工作正常，它会把这些可变参数放到逗号的后面。象其它的pasted macro参数一样，这些参数不是宏的扩展。  

在SUN环境中使用的是CC编译器：(cc`小写`编译C代码，CC`大写`编译C++)  

在sun的技术手册中
[http://docs.oracle.com/cd/E19205-01/821-0387/6nleno8eq/index.html](http://docs.oracle.com/cd/E19205-01/821-0387/6nleno8eq/index.html "http://docs.oracle.com/cd/E19205-01/821-0387/6nleno8eq/index.html")

D.1.16 具有可变数目的参数的宏

6.10.3 宏替换

C 编译器接受以下形式的#define预处理程序指令：

    #define identifier (...) replacement_list
    
    #define identifier (identifier_list, ...) replacement_list

如果宏定义中的identifier_list以省略号结尾，则意味着调用中的参数比宏定义中的参数（不包括省略号）多。否则，宏定义中参数的数目（包括由预处理标记组成的参数）与调用中参数的数目匹配。对于在其参数中使用省略号表示法的#define预处理指令，在其替换列表中使用标识符__VA_ARGS__。以下示例说明可变参数列表宏工具。

    #define debug(...) fprintf(stderr, __VA_ARGS__)
    
    #define showlist(...) puts(#__VA_ARGS__)
    
    #define report(test, ...) ((test)puts(#test):\
    
    printf(__VA_ARGS__))
    
    debug(“Flag”);
    
    debug(“X = %d\n”,x);
    
    showlist(The first, second, and third items.);
    
    report(x=y, “x is %d but y is %d”, x, y);

其结果如下：

    fprintf(stderr, “Flag”);
    
    fprintf(stderr, “X = %d\n”, x);
    
    puts(“The first, second, and third items.”);
    
    ((x=y)puts(“x=y”):printf(“x is %d but y is %d”, x, y));

可以看到，SUN中没有提到用##解决多一个逗号问题。所以编译就报错了。所以只能使用

    #define TADD_ NORMAL(...) Log(__VA_ARGS__)

这样的形式了。

解决方案如下：

编写一个函数
    
    MyLog(const * sName,int iLine,const char * fmt,….)
    
    {
    
    char sTemp[10240];
    
    va_list ap;
    
    va_start(ap,fmt);
    
    vsnprintf(sLogTemp, sizeof(sTemp), fmt, ap);
    
    va_end (ap);
    
    Log("[%s:%d]: %s", sName, iLine, sTemp);
    
    }

这样就能复用Log(const char * fmt,….)

对于sun下面使用如下的宏

    #define TADD_NORMAL(...) MyLog (__FILE__,__LINE__,__VA_ARGS__)

再次编译，运行，问题解决了。