---
layout: post
keywords: 分布式 GPU 解密
title: "重定向管道"
categories: [代码, 工作]
tags: [OpenCL, 工作]
group: archive
icon: leaf
---
{% include codepiano/setup %}

这一个月以来一直在忙着搞分布式解密系统的项目。前后花了三天的时间研究了一下基于OpenCL的GPU编程，优化了一下从同事那里拿来的md5解密程序，在AMD HD7950显卡上，运行速度从之前的50M/s提升到2000M/s，不过与老外的6000M/s还是有不少差距。
还是领导有远见，这个项目目标很伟大，号称要支持上百种解密算法，如果每种算法都要自己研究甚至优化，需要投入很多人力，于是直接拍板先集成别人的已有的工具，再着重研究几个本土化的软件破解算法即可，这样能迅速占领市场。

# 调用框架

于是为计算节点编写一个通用的调用框架的任务落到我头上，框架的核心功能无非是能够运行解密软件并且获取解密软件的解密过程和结果。方案很简单，fork+pipe+exec可以轻松搞定，首先利用fork派生出子进程，然后在子进程中利用dup2将输出输入流重定向到管道，
最后在子进程中调用exec执行调用程序即可，这样在父进程中从管道里就可以随时读取程序的输出。需要特别留意是:

* 防止僵死进程的产生，必须在父进程中随时waitpid。

* 由于管道是半双工的，因此如果想要同时实现父子进程的读写，需要pipe两次。

{% highlight cpp %}
class Parent
{
protected:
struct param
{
	int pid;
	int read_fd;
	int write_fd;
};
map<int, struct param> tables;

struct thread_param
{
	int uid;
	Parent* inst;
};

//uid是标志符，用以区分每次调用
int Exec(int uid, const char* path, const char* args[], void* (*monitor)(void*))
{
	int fd1[2], fd2[2];
	int pid;
	
	//两次pipe实现全双工
	pipe(fd1);
	pipe(fd2);
	
	if((pid = fork()) < 0)
	{
		return -1;
	}
	else if(pid == 0)//子进程
	{
		close(fd1[1]);
		close(fd2[0]);
		close(0);
		close(1);
		close(2);
		dup2(fd1[0], 0);
		dup2(fd2[1], 1);
		dup2(fd2[1], 2);
		close(fd1[0]);  
		close(fd2[1]);

		execv(path, args);
	}
	else//父进程
	{
		close(fd1[0]);
		close(fd2[1]);   
		int flag = fcntl(fd2[0], F_GETFL, 0);
		fcntl(fd2[0], F_SETFL, flag|O_NONBLOCK);
	
		struct param p = {pid, fd2[0], fd1[1]};
		tables[uid] = p;
	
		thread_param* pp = (thread_param*)malloc(sizeof(*pp));
		pthread_t tid;
		pp->uid = uid;
		pp->inst = this;
		pthread_create(&tid, NULL, monitor, (void *)pp);
		return pid;
	}
}
public:
//调用接口
int Start(int uid, void* other_params)
{
	//...
	return this->Lauch(uid, other_params);
}

//从输出流中读取
int Read(int uid, char* buf, int n)
{
	//获取该uid相关的子进程id
	map<int, param>::iterator it = tables.find(uid);
	int pid = it->second.pid;

	//检测子进程是否结束，这样可以避免僵死进程
	int status = -1;
	int rv = waitpid(pid, &status, WNOHANG);
	if(rv > 0){
		CleanUp(uid);	//从tables中清除该entry
		return ERR_CHILDEXIT;//子进程已经结束		
	}
	
	//读操作
	int fd = it->second.read_fd;
	return read(fd, buf, n);
}

//往输入流中写
int Write(int uid, const char* buf, int n)
{
	//获取该uid相关的子进程id
	map<int, param>::iterator it = tables.find(uid);
	int pid = it->second.pid;

	//检测子进程是否结束，这样可以避免僵死进程
	int status = -1;
	int rv = waitpid(pid, &status, WNOHANG);
	if(rv > 0){
		CleanUp(uid);
		return ERR_CHILDEXIT;
	}
	
	//写操作
	int fd = it->second.write_fd;
	return write(fd, buf, n);
}

//由派生类具体实现
virtual int Lauch(int uid, void* other_params) = 0;		
}

class ChildA : Parent
{
public:
	int Lauch(int uid, void* other_params)
	{
		//以下根据具体每个Child和相应的params做一些特定的初始化
		...
		
		return this->Exec(uid, path, args, MonitorThread);
	}
	
	//派生类需要实现这样一个线程，用以操控子进程的输出输入流
	static void*MoitorThread(void* p)
	{
		thread_param* param = (thread_param*)p;
		ChildA* inst = (Child*)param->inst;
		int uid = param->uid;
		char buffer[4096];
		int n;
		
		while(1)
		{
			n = inst->Read(uid, buffer, sizeof(buffer));
			if(n == ERR_CHILDEXIT) {
				printf("Detected child exit\n");
				break;
			}else if(n <= 0){
				continue;
			} 
			
			//对输出buffer进行解析
			
			//必要时可以操控输入流
			n = inst->Write(uid, "\n", 1);
			if(n == ERR_CHILDEXIT)
			{			
				printf("Detected child exit2\n");
				break;
			}
		}
		
		return NULL;
	}
};
{% endhighlight %}

接下来的事情就很简单了，只需要具体就某个解密软件的输入输出进行解析和分析就可以了。剩下来的就是资源调度方面的事情了，这个逻辑也很直观，从资源池中取可用的计算资源，然后从服务端取解密任务，然后调用上述框架调用某个解密软件即可，在MonitorThread里面读取解密进度，随时上报服务端。
解密结束之后将结果上报服务端，同时资源池将资源回收。

# 其他模块
这只是计算节点的主要开发工作，至于服务端和用户控制端的逻辑更加负责，真是任重道远啊，

# 持续
只能寄希望于其他几个小伙伴更加给力点，争取尽快联调好，将原型做出来。