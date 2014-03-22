---
layout: post
keywords: 分布式 GPU 解密
description: happy,new,year
title: ""
categories: [代码, 工作]
tags: [OpenCL, 工作]
group: archive
icon: leaf
---
{% include codepiano/setup %}

这一个月以来一直在忙着搞分布式解密系统的项目。前后花了三天的时间研究了一下基于OpenCL的GPU编程，优化了一下从同事那里的md5解密程序，在AMD HD7959显卡上，运行速度从之前的50M/s提升到2000M/s，不过与老外的6000M/s还是有不少差距。
还是领导有远见，这个项目目标很伟大，号称要支持上百种解密算法，如果每种算法都要自己研究甚至优化，需要投入很多人力，于是直接拍板先集成别人的已有的工具，再着重研究几个本土化的软件破解即可，这样能迅速占领市场。

于是编写一个通用的调用框架的任务落到我头上，框架的核心功能无非是能够运行解密软件并且获取解密软件的解密过程和结果。方案很简单，fork+pipe+exec可以轻松搞定，首先利用fork派生出子进程，然后在子进程中利用dup2将输出输入流重定向到管道，
最后在子进程中调用exec执行调用程序即可，这样在父进程中从管道里就可以随时读取程序的输出。需要特别留意是:
× 防止僵死进程的产生，必须在父进程中随时waitpid。
× 由于管道是半双工的，因此如果想要同时实现父子进程的读写，需要pipe两次。

class Parent
{
protected:
	int Exec(const char* path, const char* args[])
	{
		int fd1[2], fd2[2];
		int pid;
		
		pipe(fd1);
		pipe(fd2);
		
		if((pid = fork()) == 0)
		{
			close(fd1[0]);
			close(fd2[1]);
			close(0);
			close(1);
			close(2);
			dup2()
		}
		else if(pid > 0)
		{
		}
	}
public:
	int StartCrack();

	virtual int Lauch() = 0;		
}

class ChildA : Parent
{
public:
	int Lauch();	
	
	static void*Moitor(void* p)
	{
		Parent* instance = (Parent*)p;
		char buffer[4096];
		int n;
		
		while(1)
		{
			n = instance->Read(buffer, sizeof(buffer));
			if()
		}
		
		return NULL;
	}
};

下面只需要具体就某个解密软件的输入输出进行解析和分析。

剩下来的就是资源调度方面的事情了，锁啊，