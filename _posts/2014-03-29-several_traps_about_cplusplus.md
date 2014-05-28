---
layout: post
keywords: 陷阱 网络编程 C++
title: "开发陷阱几则"
categories: [C++]
tags: [代码, 工作]
group: archive
icon: leaf
---
{% include codepiano/setup %}

这几天和别人联调程序的时候，出现了很多意想不到的问题，其实有些问题之前不是没碰到过并且已经解决了，只是时间长了又抛诸脑后了，我看还是有必要将其“持久化”，记录在这里，便于查阅。

# read/recv和write/send

在使用原始socket进行网络编程时，recv和send这两个函数并不能确保接受或者发送的数据的大小就是你指定的长度。
send函数通常把数据放到一个内核缓冲区里，如果对方接受了该数据，则会将相应的数据从缓冲区释放掉。由于对方可能没有来得及接受数据，
缓冲区会越积越大，在某个时间段缓冲区可能接近满负荷状态了，这时候你再往缓冲区里面写数据，缓冲区没有足够多的容量来存放你的数据了，
因此send只会返回给你一部分数据。解决方法很简单，通过一个循环即可。

{% highlight c %}
int Send(int fd, void* data, int size)
{
	int total = 0;
	do{
		int n = send(fd, data+total, size-total, 0);
		if(n < 0) return -1;
		total += n;
		if(total == size) break;
 	}while(1);
	return total;
}
{% endhighlight %}

# 32位和64位的sizeof(long)

我一直在Windows 32位机器上进行开发，一天同事在Ubuntu 64位机器上运行时发现并不能有效的从服务端接受数据，
由于设计的协议是将所有的数据压缩之后再在网络上进行传输，跟踪了一下发现数据的确是收到了，只是解压缩时出现了错误。
代码中用的是zlib库，我百思不得其解，又跑到Ubuntu 32位上重新编译发现一切正常，因此首先怀疑可能是zlib对64位的支持不够。继续调查终于发现了问题的所在。

{% highlight c %}
int uncompress(Bytef *dest, uLongf *destLen, const Bytef *source, uLong sourceLen);
{% endhighlight %}

平时基本上不用long类型，都是直接用int32或者int64，实际编码的时候我给第2个参数传递的是一个int指针，由于编译不通过我将其强制转换成long的指针，
在32位的机器上由于long和int的sizeof都是4，所以没有任何问题，但是64位下sizeof(long)是8，从而导致uncompress对dest缓冲区的大小识别错误。

