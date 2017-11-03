---
layout: post
title: "Something about fork"
description: ""
category: "linux"
tags: [linux]
---

在大学参加的一场笔试里，出现了一道关于fork函数的题目，当时都不知道这个是什么，题目自然是没做对。现在工作了，虽然具体工作没有和它打交道，但是我看的一些博文，书里，有的知识点里它不断的出现，有的部分如果不搞懂它就不明白这部分是内容。为了打破这种局限，我决定去看看它到底是什么。

要理解fork是什么就要知道它的为什么而来。说道这个，就得说道操作系统层面了，在我们感知的同一时间里，电脑可以一边放音乐，一边处理文字，这个得得益于多进程以及系统的进程调度。这个多进程的实现，就得靠fork函数了。

举个实际的例子，在linux中，系统创建的第一个进程及0号进程的名字是init，init启动完毕后会接着启动系统需要的其它进程，这些进程就是fork创建的，因此，通过fork命令，整个系统中的进程是一个树的结构。在linux系统中可以通过pstree这个命令来查看。

# 基本用法

下面来说说fork这个函数，linux的manual中是这样介绍的，fork()通过拷贝调用它的进程来创建新的进程。新的进程，称为子进程。虽说是拷贝，但也有些内容是例外，暂时先不计较这些，先来实际用用看，细节可以参考函数文档[fork()][1]。

```c
#include <unistd.h>
#include <stdio.h>

int main(){
	pid_t fpid;
	int count = 0;
	fpid=fork();
	if(fpid<0){
		printf("error in fork!");
	}else if(fpid == 0){
		printf("I am the child,the pid is %d\n" , getpid());
		count++;
	}else{
		printf("I am the parent,the pid is %d\n",getpid());
		count++;
	}
	printf("count result is %d\n",count);
	return 0;
}

```

这个代码要说也没太多的花样，只是个选择分支，而选择的依据就是fork的返回值。

fork的返回值分为三种：

  - 0：当前进程为子进程时
  - pid： 当前进程为父进程，返回创建的子进程的id
  - 负数：意味着创建失败，有错误发生

执行成功后会有两个进程（父/子进程）。根据返回值，执行相应分支的代码。由于每个进程中的变量是相对独立的，所以，count的值也都是1。至于是谁先执行，这个由系统的调度决定。


# 进阶

```c
#include <unistd.h>
#include <stdio.h>

int main(){
	int i = 0;
	printf("i son/pa ppid pid fpid\n");

	for(i=0;i<2;i++){
		pid_t fpid = fork();
		if(fpid == 0)
			printf("%d child %4d %4d %4d\n",i,getppid(),getpid(),fpid);
		else
			printf("%d parent %4d %4d %4d\n",i,getppid(),getpid(),fpid);
	}
	return 0;
}

```

上面这段代码加入了循环，让执行的流程稍微复杂了那么一点点，这个值得一步一步分析。


```bash
#某次代码的执行结果
i son/pa ppid pid fpid
0 parent 3966 4866 4867
0 child  4866 4867    0
1 parent 3966 4866 4868
1 parent 4866 4867 4869
1 child    1  4869    0
1 child    1  4868    0
```

一：在父进程中，i=0时，执行fork函数，出现了父子两个进程，分别是p4373和p4378。可以看出他们之间的关系是3966->4866->4867。父子两个进程分别执行，打印相关信息。

```bash
0 parent 3966 4866 4867
0 child  4866 4867    0
```

二：当i=1时，从输出可以看出，p4866被系统调度执行，它执行fork，创建了进程p4868。然后是p4867被调用，创建了进程p4869。两个父进程相继执行，打印相关信息。

```bash
1 parent 3966 4866 4868
1 parent 4866 4867 4869
```

三：第二步，创建的两个子进程执行完后就结束了，执行return 0。打印相关信息。

```bash
1 child    1  4869    0
1 child    1  4868    0
```

到后面p4868,p4869的父进程执行完死亡了，此时这两个子进程的父进程被置为p1。





参考文章：
[linux中fork（）函数详解][2]
  


  [1]: http://man7.org/linux/man-pages/man2/fork.2.html
  [2]: http://blog.csdn.net/jason314/article/details/5640969
