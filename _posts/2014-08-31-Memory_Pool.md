---
layout: post
title: "浅谈内存池技术"
tagline: null
categories: [技术]
tags: [内存池, STL, nginx, Python]
icon: leaf
published: true
---

许多C/C++库的开发者为了提高库的使用性能，同时为了更方便高效的管理内存和减少内存碎片，都在内部实现了一套内存池技术。这样就不需要跟操作系统打交道，而是从预先申请内存中直接获取。本文主要剖析以下几种内存池技术：

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

该Allocator最小的分配单位为8Byte，其内部维护了一个长度为维护128/8=16的空闲链表数组，每个链表管理着固定大小的内存

-  _S_free_list[0] --------> 8 byte
-  _S_free_list[1] --------> 16 byte
-  _S_free_list[2] --------> 24 byte
-  _S_free_list[3] --------> 32 byte
-  ... ...
-  _S_free_list[15] -------> 128 byte

下面是有关该Allocator的一些定义

{% highlight cpp %} 
    class __pool_alloc_base 
    {
    protected:

      enum { _S_align = 8 };
      enum { _S_max_bytes = 128 };
      enum { _S_free_list_size = (size_t)_S_max_bytes / (size_t)_S_align };
      
      union _Obj
      {
	   union _Obj* _M_free_list_link;   /* 将同规格的空闲内存串接起来，形成链表 */
	   char        _M_client_data[1];    // The client sees this.
      };
      
      static _Obj* volatile         _S_free_list[_S_free_list_size]; /* 空闲自由链表 */

      // Chunk allocation state.
      static char*                  _S_start_free; /* 内存池起始位置 */
      static char*                  _S_end_free;   /* 内存池结束位置 */
      static size_t                 _S_heap_size;  /* 内存池大小 */
	  ......
	 };
{% endhighlight %} 

### 内存申请 ###
这种内存池的分配工作流程大致如下：

1. 通过内存对齐的方式在空闲链表数组里找到合适的内存块链表头（比如13bytes会向上对齐到16bytes，对应于链表_S_free_list[1]）；
2. 判断链表头是否为空：
    - 如果不为空，则返那块内存地址，并将此块内存块从它对应的链表中移除出去；
    - 如果为空，则表示没有此规格空闲的内存，调用refill在freelist上重新挂载20个此规格的内存空间（形成链表），也就是保证此规格的内存空间下次请求时够用 

{% highlight cpp %}
    template<typename _Tp>
    _Tp* __pool_alloc<_Tp>::allocate(size_type __n, const void*)
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
	      	     /* 链表头为空，表明该规格的内存已经售罄，需要调用refill重新进货 */
		     __ret = static_cast<_Tp*>(_M_refill(_M_round_up(__bytes)));
	      else
		{
		  /* 该链表尚有可用空间，直接交付使用，并将其从对应的freelist链表中移除*/
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

再来看看refill操作，该操作调用allocate_chunk函数在原对应空闲链表头处挂载20个此规格的内存空间（之所以是20个大小，是为了保证下次请求此规格的内存空间时够用），然后将这些同等规格的内存用链表串接起来。

{% highlight cpp %}
    void* __pool_alloc_base::_M_refill(size_t __n)
    {
    int __nobjs = 20;
    char* __chunk = _M_allocate_chunk(__n, __nobjs);//申请20块同等规格大小的内存块
    _Obj* volatile* __free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    
    if (__nobjs == 1)
      return __chunk;//内存不够或其他原因，只申请到一块，直接返回

    __free_list = _M_get_free_list(__n);
    
    // Build free list in chunk.
    __result = (_Obj*)(void*)__chunk;
    *__free_list = __next_obj = (_Obj*)(void*)(__chunk + __n);
    for (int __i = 1; ; __i++)//将__nobjs个同等规格大小的内存串接起来，形成链表
      {
	__current_obj = __next_obj;
	__next_obj = (_Obj*)(void*)((char*)__next_obj + __n);
	if (__nobjs - 1 == __i)
	  {
	    __current_obj->_M_free_list_link = 0;
	    break;
	  }
	else
	  __current_obj->_M_free_list_link = __next_obj;
      }
    return __result;
  }
{% endhighlight %} 


chunk_alloc()的具体过程为：

1. 如果start_free和end_free之间的空间足够分配n\*20大小的内存空间，则从这个空间中取出n\*20大小的内存空间，同时更新start_free游标；否则转到2；
2. 如果start_free和end_free之间的空间足够分配大于n的内存空间，则分配n的整数倍的内存空间，更新start_free游标，同时更新nobj以返回这个整数；否则转到3；
3. OK，现在内存池中连一块大小为n的内存都没有了，为了最大限度的利用内存而不浪费，此时如果内存池中还有一些内存（这些内存大小肯定小于n），则将这些内存插入到相应的的空闲链表中；
4. 调用new向操作系统申请大小为（2\*n\*20 + 附加量）的内存空间， 如果申请成功，更新start_free, end_free和heap_size，并重新调用chunk_alloc()；否则转到5；
5. new操作符分配内存失败，依次遍历16个空闲链表，只要有一个空闲链表中可用的内存大小超过n，就释放该链中的一个节点，重新调用chunk_alloc()

{% highlight cpp %}
    char* __pool_alloc_base::_M_allocate_chunk(size_t __n, int& __nobjs)
    {
    char* __result;
    size_t __total_bytes = __n * __nobjs;
    size_t __bytes_left = _S_end_free - _S_start_free;
    
    if (__bytes_left >= __total_bytes)//空闲内存 >= n\*20，对应于情况1
      {
	__result = _S_start_free;
	_S_start_free += __total_bytes;
	return __result ;
      }
    else if (__bytes_left >= __n)//n <= 可用内存 < n\*20，对应于情况2
      {
	__nobjs = (int)(__bytes_left / __n);
	__total_bytes = __n * __nobjs;
	__result = _S_start_free;
	_S_start_free += __total_bytes;
	return __result;
      }
    else
      {
	// Try to make use of the left-over piece.
	if (__bytes_left > 0)// 可用内存 < n，对应于情况3
	  {
	    _Obj* volatile* __free_list = _M_get_free_list(__bytes_left);
	    ((_Obj*)(void*)_S_start_free)->_M_free_list_link = *__free_list;
	    *__free_list = (_Obj*)(void*)_S_start_free;
	  }
	
	//情况4，只能向操作系统申请了，这样为什么申请2倍的空间，主要还是为了下一步考虑
	size_t __bytes_to_get = (2 * __total_bytes
				 + _M_round_up(_S_heap_size >> 4));
	__try
	  {
	    _S_start_free = static_cast<char*>(::operator new(__bytes_to_get));
	  }
	__catch(const std::bad_alloc&)
	  {
	    // 情况5，new也分配失败了，说明操作系统没这么多的内存了，这时候可以遍历16个空闲链表，
		// 只要其可用大小超过n，就可以拿来使用
	    size_t __i = __n;
	    for (; __i <= (size_t) _S_max_bytes; __i += (size_t) _S_align)
	      {
		_Obj* volatile* __free_list = _M_get_free_list(__i);
		_Obj* __p = *__free_list;
		if (__p != 0)
		  {
		    *__free_list = __p->_M_free_list_link;
		    _S_start_free = (char*)__p;
		    _S_end_free = _S_start_free + __i;
		    return _M_allocate_chunk(__n, __nobjs);
		    // Any leftover piece will eventually make it to the
		    // right free list.
		  }
	      }
	    // What we have wasn't enough.  Rethrow.
	    _S_start_free = _S_end_free = 0;   // We have no chunk.
	    __throw_exception_again;
	  }
	_S_heap_size += __bytes_to_get;
	_S_end_free = _S_start_free + __bytes_to_get;
	return _M_allocate_chunk(__n, __nobjs);
      }
  }　　
{% endhighlight %} 

### 内存释放 ###
相对而言，__pool_alloc分配器的内存释放过程比较简单，主要工作由deallocate函数完成，它接受两个参数，一个是指向要释放的内存块的指针p，另外一个表示要释放的内存块的大小n。如果n超过128bytes，则交由delete操作去处理；否则将该内存块加到相应的空闲链表中

{% highlight cpp %}
    template<typename _Tp>
    void __pool_alloc<_Tp>::deallocate(pointer __p, size_type __n)
    {
      if (__builtin_expect(__n != 0 && __p != 0, true))
	{
	  const size_t __bytes = __n * sizeof(_Tp);
	  if (__bytes > static_cast<size_t>(_S_max_bytes) || _S_force_new > 0)
	    ::operator delete(__p); //大小超过128字节，直接由delete处理
	  else
	    {
	      _Obj* volatile* __free_list = _M_get_free_list(__bytes);//找到相对应的空闲链表
	      _Obj* __q = reinterpret_cast<_Obj*>(__p);

	      __scoped_lock sentry(_M_get_mutex());
		  //将该内存嫁接到原始链表头
	      __q ->_M_free_list_link = *__free_list;
	      *__free_list = __q;
	    }
	}
    }
{% endhighlight %} 

### 总结 ###
应用程序在某个时候申请了32字节的大小，发现free_list[4]链表头为空，于是STL将会调用refill操作申请32\*20=640个bytes空间，然后allocate_chunck操作发现内存池中没有空闲内存可用，将由new操作符申请32\*20\*2=1280个字节，其中的640bytes由给free_list[4]管理，尚余的640bytes由start_free和end_free两个游标管理；这时候假设应用程序又申请了16个字节内存，同样发现free_list[2]链表头为空，于是与上面类似也会refill-->allocate_chunck调用序列将会执行，这时候发现start_free和end_free管理的内存空间比较充足，直接划分出16\*20个bytes给free_list[2]，start_free游标继续下移；然后，应用程序释放了一块16bytes的大小，Allocator将其插入到free_list[2]的链表表头节点中，形成如下的状态示意图。
![image](/assets/post-images/stl.jpg)

不难发现，该内存池分配器存在一个很大的缺点，allocate_chunk函数中由std::new申请大内存永远无法得到释放，除非程序生命周期结束。如果应用程序不断的向STL申请小块内存直至系统内存枯竭，然后一次性释放，将会看到释放到的内存将会全部交给free_list链表数组进行管理而不是操作系统，这时候再申请超过128字节的内存或者其他非STL库申请内存将会失效！！！
