---
layout: post
title: "使用awk提取代码中的错误码信息"
description: "使用awk提取代码中的错误码信息"
category: 工具
tags: [Linux,awk]
---

使用awk提取代码中的错误码信息
最近给了个任务，整理错误码，要有“错误码，简单说明，详细说明，解决方案”，我们的错误码是定义在一个.h的文件中用#define 宏来定义的。一个个的copy到execl中肯定太废了。所以直接用awk处理文件那是再合适不过了，第一次用awk，现学现卖了。
脚本如下：  

    !/bin/sh
    #错误码定义文件转csv文件
    #格式:#define [空格]  [宏以ERROR_开头]  [空格]  (错误码)  //错误说明//解决方法
    Echo "Error_code,Error_brief,Error_detail,Error_solution" > error_code.csv
    grep '#define ERROR' mdbErr.h | awk -F '//' ' 
    function getErrCode(code_desc)
    {
    	split(code_desc,parts," ")
    	return 0-substr(parts[3],2,6)
    }
    {OFS=",";gsub("\t"," ",$2);gsub("\t"," ",$3);print getErrCode($1),$2,$2,$3}
    '  >>  error_code.csv

关键几点是: 
 
1、“错误说明，解决方法”这些之间可能用空格，所以不能单纯的用空格分隔。所以我采用的方式是先按`//`分隔。  

将$1(第一列)用getErrCode函数处理。`split(code_desc,parts," ")`，按空格分割。`0-substr(parts[3],2,6)`，从括号中取出错误码并取反。  

2、`gsub("\t"," ",$2)`对第二列进行\t替换成空格。

**主要使用了awk的用户自定义函数和其内建字符串处理函数。真是强大。**
