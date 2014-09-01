---
layout: post
title: "浅谈内存池技术"
tagline: null
categories: [技术]
tags: [内存池, STL, nginx, Python]
icon: leaf
published: true
---

许多C/C++库的开发者为了提高库的使用性能，同时为了更方便高效的管理内存和减少内存碎片，都在内部实现了一套内存池技术。这样就不需要跟操作系统打交道，而是从预先申请内存中直接获取。本文主要剖析以下几种内存储技术：

* 类STL的内存池技术
* nginx内存池技术
* Python对象的内存池技术

<h2>类STL内存池技术</h2>  
阅读gcc 4.9.1中的有关libstdc++的STL的源代码，会发现STL提供了很多种Allocator的实现。在libstdc++-v3\include\ext目录下，STL中实现了好多种不同的allocator：

    * __gnu_cxx::new_allocator: 简单地封装了new和delete操作符，通常就是std::allocator
    * __gnu_cxx::malloc_allocator: 简单地封装了malloc和free函数
    * __gnu_cxx::array_allocator: 申请一堆内存
    * __gnu_cxx::debug_allocator: 用于debug
    * __gnu_cxx::throw_allocator: 用于异常
    * __gnu_cxx::__pool_alloc: 基于内存池
    * __gnu_cxx::__mt_alloc: 对多线程环境进行了优化
	  * ......

其中__pool_alloc就是基于内存池实现的一套allocator，该内存池的结构与侯捷在《STL源码剖析》中对SGI STL的内存池技术的描述大体类似。


