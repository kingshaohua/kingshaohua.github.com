---
layout: post
title: "question about cache miss"
description: ""
category: 
tags: []
---
要高命中率就要知道哪些数据需要做缓存
哪些是热数据


其实高级操作系统，基本上都不需要内存池了。如果你需要的是cpu高速缓存的命中率，那么注意内存行对齐，还有内存最好统一分一大块，这样有助于cache预取。