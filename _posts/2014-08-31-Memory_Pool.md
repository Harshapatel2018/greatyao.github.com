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
    
    if (__bytes_left >= __total_bytes)//空闲内存 >= n*20，对应于情况1
      {
	__result = _S_start_free;
	_S_start_free += __total_bytes;
	return __result ;
      }
    else if (__bytes_left >= __n)//n <= 可用内存 < n*20，对应于情况2
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
相对而言，__pool_alloc分配器的内存释放过程比较简单，主要工作由deallocate函数完成，它接受两个参数，一个是指向要释放的内存块的指针p，另外一个表示要释放的内存块的大小n。如果n超过128bytes，则交由std::delete操作去处理；否则将该内存块加到相应的空闲链表中

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
应用程序在某个时候申请了32字节的大小，发现free_list[4]链表头为空，于是STL将会调用refill操作申请32\*20=640个字节空间，然后allocate_chunck操作发现内存池中没有空闲内存可用，将由new操作符申请32\*20\*2=1280个字节，其中的640字节由给free_list[4]管理，尚余的640字节由start_free和end_free两个游标管理；这时候假设应用程序又申请了16个字节内存，同样发现free_list[2]链表头为空，于是与上面类似refill-->allocate_chunck调用序列将会执行，这时候发现start_free和end_free管理的内存空间比较充足，直接划分出16\*20个字节给free_list[2]，start_free游标继续下移；然后，应用程序释放了一块16字节的大小(注意不是上一步而是在此之前申请到的)，Allocator将其插入到free_list[2]的链表表头节点中，形成如下的状态示意图。
![image](/assets/post-images/stl.jpg)

不难发现，该内存池分配器存在一个很大的缺点，allocate_chunk函数中由std::new申请大内存永远无法得到释放，除非程序生命周期结束。如果应用程序不断的向STL申请小块内存直至系统内存枯竭，然后一次性释放，将会看到释放到的内存将会全部交给free_list链表数组进行管理而不是操作系统，这时候再申请超过128字节的内存或者其他非STL库申请内存将会失效！！！

## nginx内存池技术 ##
nginx中与内存相关的操作主要在文件 os/unix/ngx_alloc.{h,c} 和 core/ngx_palloc.{h,c} 中实现，我们首先看一下与内存池相关的数据结构定义：

{% highlight cpp %}
typedef struct {    //内存池的数据结构模块  
    u_char               *last;    //块内存的可分配的内存的起始位置  
    u_char               *end;     //块内存的结束位置  
    ngx_pool_t           *next;    //链接到下一个内存块，内存池的很多块内存就是通过该指针连成链表的  
    ngx_uint_t            failed;  //记录内存分配不能满足需求的失败次数  
} ngx_pool_data_t;   //结构用来维护内存池的数据块，供用户分配之用。

struct ngx_pool_t {  //内存池的管理分配模块  
    ngx_pool_data_t       d;         //内存池的数据块（参见上面），设为d  
    size_t                max;       //数据块大小，小块内存的最大值  
    ngx_pool_t           *current;   //指向当前或本内存池  
    ngx_chain_t          *chain;     //该指针挂接一个ngx_chain_t结构  
    ngx_pool_large_t     *large;     //指向大块内存分配（nginx中大块内存分配直接采用malloc)  
    ngx_pool_cleanup_t   *cleanup;   //析构函数，挂载内存释放时需要清理资源的一些必要操作  
    ngx_log_t            *log;       //内存分配相关的日志记录  
};  

struct ngx_pool_large_t {  //大块数据分配的结构体
    ngx_pool_large_t     *next;  //链表指针
    void                 *alloc; //大块内存地址
}; 
{% endhighlight %} 

### 内存池创建和销毁 ###
nginx内存池的创建非常简单，由函数ngx_create_pool来完成，申请一块size大小的内存，把它分配给 ngx_poo_t。nginx的内存对齐按16位对齐。

{% highlight cpp %}
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;
    }

	//计算内存池的数据区域，初始化内存池数据数据的相关数据
    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;

    size = size - sizeof(ngx_pool_t);
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;

    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
{% endhighlight %}

下图是内存池初始化时刻的示意图
![image](/assets/post-images/nginx_create.jpg)

内存池销毁时，会释放掉所有与给该内存池相关的大块和小块内存。在内存池销毁之前，还必须做一些数据的清理工作，如文件的关闭等。前面说到cleanup指向了一个清理函数，因此只需要对内存池中的析构函数遍历调用即可。  

{% highlight cpp %}
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;

    //清理工作
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }

	//释放大块内存
    for (l = pool->large; l; l = l->next) {

        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);

        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }

	//释放小块内存
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);

        if (n == NULL) {
            break;
        }
    }
}
{% endhighlight %}

###内存的申请
nginx向内存池中申请内存由ngx_palloc和ngx_pnalloc完成，这两个函数作用类型，唯一的区别在于是否对内存进行对齐操作。

内存池中分配内存的具体流程如下：

1. 对于大块内存（即ngx_create_pool中的size参数，也就是内存池创建时指定的内存大小）直接由ngx_palloc_large分配；对于小块内存则转2；
2. 遍历pool链表，通过last和end两个游标寻找是否有合适的空间，有则返回该内存，同时更新游标；否则转3；
3. 通过ngx_palloc_block重新申请一个内存块

{% highlight cpp %}
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    ngx_pool_t  *p;

    if (size <= pool->max) {

        //遍历pool链表寻找合适大小的空间
        p = pool->current;
        do {
            m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT);

			//有合适的空间
            if ((size_t) (p->d.end - m) >= size) {
                p->d.last = m + size;

                return m;
            }

            p = p->d.next;

        } while (p);

		//没有合适的，重新申请一块内存块
        return ngx_palloc_block(pool, size);
    }

    //大块内存申请
    return ngx_palloc_large(pool, size);
}
{% endhighlight %}

大块内存的申请即ngx_palloc_large直接通过ngx_alloc（其实是malloc的简单封装）向操作系统申请，然后将其链接到内存池中的large链表中。
{% highlight cpp %}
void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;

    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    n = 0;

    //将分配到的内存链入pool的large链中，  
    //首先考虑原始pool在之前已经分配过large内存的情况 
    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    //如果该pool之前并未分配large内存或者链表深度超过3，则新建一个ngx_pool_large_t来管理
    //大块内存  
    large = ngx_palloc(pool, sizeof(ngx_pool_large_t));
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
{% endhighlight %}

紧接着，我们来剖析ngx_palloc_block()函数，新建一个与pool同等大小的内存块，将其链入到pool的链表末尾节点，同时更新pool中的current节点。
{% highlight cpp %}
void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new, *current;

	//计算内存块大小
    psize = (size_t) (pool->d.end - (u_char *) pool);

    //与内存池创建时类似，更新相关游标和字段
    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;
    //字段初始化结束

    
    current = pool->current;

    for (p = current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
			//失败次数4次以上，表明当前内存块中的可用内存被再次分配到的几率相当低了
			//因此将节点往前进
            current = p->d.next;
        }
    }

    p->d.next = new;//将分配的block链入内存池链表的尾节点  

    pool->current = current ? current : new;

    return m;
}
{% endhighlight %}

### 内存的释放 ###
在nginx中，小块内存除了在内存池销毁之外是不能释放的，但是大块内存却可以。ngx_pfree函数就是用来控制大块内存的释放，代码非常简单，遍历large链表中的alloc字段来寻找相对应的large结构，然后调用ngx_free（等同于free）来释放内存。
{% highlight cpp %}
ngx_int_t
ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p", l->alloc);
            ngx_free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
}
{% endhighlight %}

下图是内存池运行一段时间的示意图
![image](/assets/post-images/nginx_snapshot.jpg)

### 总结 ###
前面说到，nginx并没有提供释放内存的接口，那么nginx是如何完成内存释放的呢？总不能一直申请，用不释放啊。针对这个问题，nginx利用了web server应用的特殊场景来完成：一个web server总是不停的接受connection和request，所以nginx就将内存池分了不同的等级，有进程级的内存池、connection级的内存池、request级的内存池。也就是说，创建好一个worker进程的时候，同时为这个worker进程创建一个内存池，待有新的连接到来后，就在worker进程的内存池上为该连接创建起一个内存池；连接上到来一个request后，又在连接的内存池上为request创建起一个内存池。这样，在request被处理完后，就会释放request的整个内存池，连接断开后，就会释放连接的内存池。

## Python对象的内存池技术