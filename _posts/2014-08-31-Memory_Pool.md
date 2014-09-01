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

## 类STL内存池技术  
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

__pool_alloc内存分配器只适用于小块内存的管理，如果申请的内存块大于128字节，就将申请的操作移交std::new去处理；如果申请的区块大小小于128字节时，就从本分配器维护的内存池中分配内存。

该Allocator最小的分配单位为8Byte，其内部维护了一个长度为维护128/8=16的自由链表数组，每个链表管理着固定大小的内存

-  _S_free_list[0] --------> 8 byte
-  _S_free_list[1] --------> 16 byte
-  _S_free_list[2] --------> 24 byte
-  _S_free_list[3] --------> 32 byte
-  ... ...
-  _S_free_list[15] -------> 128 byte


{% highlight cpp %} 
    class __pool_alloc_base 
    {
    protected:

      enum { _S_align = 8 };
      enum { _S_max_bytes = 128 };
      enum { _S_free_list_size = (size_t)_S_max_bytes / (size_t)_S_align };
      
      union _Obj
      {
	union _Obj* _M_free_list_link; 
	char        _M_client_data[1];    // The client sees this.
      };
      
      static _Obj* volatile         _S_free_list[_S_free_list_size]; /* 自由链表 */

      // Chunk allocation state.
      static char*                  _S_start_free;/* 内存池起始位置 */
      static char*                  _S_end_free;/* 内存池结束位置 */
      static size_t                 _S_heap_size;     /* 内存池大小 */
	  ......
	 };
{% endhighlight %} 

这种内存池的分配工作流程大致如下：

1. 通过内存对齐的方式在空闲链表数组里找到合适的内存块链表头；
2. 判断链表头是否为空：
    - 如果不为空，则返那块内存地址，并将此块内存块从它对应的链表中移除出去；
    - 如果为空，则表示没有此规格空闲的内存，调用refill在freelist上重新挂载20个此规格的内存空间（形成链表），也就是保证此规格的内存空间下次请求时够用 

{% highlight cpp %}
    template<typename _Tp>
    _Tp*
    __pool_alloc<_Tp>::allocate(size_type __n, const void*)
    {
      pointer __ret = 0;
      if (__builtin_expect(__n != 0, true))
	{
	  ......
	  const size_t __bytes = __n * sizeof(_Tp);	      
	  if (__bytes > size_t(_S_max_bytes) || _S_force_new > 0)
	    __ret = static_cast<_Tp*>(::operator new(__bytes));/* 超过128字节的直接new*/
	  else
	    {
	      _Obj* volatile* __free_list = _M_get_free_list(__bytes);/* 找到合适的空闲链表*/
	      
	      __scoped_lock sentry(_M_get_mutex());
	      _Obj* __restrict__ __result = *__free_list;
	      if (__builtin_expect(__result == 0, 0))
		__ret = static_cast<_Tp*>(_M_refill(_M_round_up(__bytes)));/* 链表为空，表明该规格的内存已经售罄，需要refill重新购买 */
	      else
		{
		  /* 该链表尚有可用空间，交付使用，并将其从freelist链表中移除*/
		  *__free_list = __result->_M_free_list_link;
		  __ret = reinterpret_cast<_Tp*>(__result);
		}
	      if (__ret == 0)
		std::__throw_bad_alloc();
	    }
	}
      return __ret;
    }
{% endhighlight %} 


