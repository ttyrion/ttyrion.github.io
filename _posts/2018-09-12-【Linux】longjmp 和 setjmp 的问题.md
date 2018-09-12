---
layout:     page
title:      【Linux】longjmp 和 setjmp 的问题
subtitle:   longjmp引起的自动屏蔽信号
date:       2018-09-12
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

>信号处理函数中常会使用跳转函数跳转到应用程序某处，而不是从信号处理函数返回。这里说一下如果使用longjmp & setjmp 会带来的问题。

## 自动屏蔽信号
信号处理函数中调用longjmp的问题：  
如果产生了一个相应的信号，进入对应的信号处理函数中，此时当前信号被自动加到进程的 **信号屏蔽字** 中，并且在跳出信号处理函数后，**信号屏蔽字** 也不会恢复，该信号会继续被屏蔽。也就是说，这不仅是在调用信号处理函数的过程中阻止了该信号再次中断当前信号处理函数，即便是执行完信号处理函数后（跳出信号处理函数），**进程也无法再次接收到该信号**。  
## 代码验证
有一段如下代码：  
```cpp
static jmp_buf sigint_env;
static void on_sigint(int signo) {
	switch(signo) {
		case SIGINT: {
			std::cout << "SIGINT catched." << std::endl;
			longjmp(sigint_env, 1);
			break;
		}
		default: {
			std::cout << "An unknown signal catched." << std::endl;
			break;
		}
	}
}

int main(int argc, char** argv) {
	const int MAXLINE = 80;
	char buf[MAXLINE];

	// if(signal(SIGINT, &Handler::OnSignal) == SIG_ERR) {
	// 	std::cout << "signal error!" << std::endl;
	// }

	if (signal(SIGINT, on_sigint) == SIG_ERR) {
		std::cout << "Try  catching SIGINT failed." << std::endl;
		return 1;
	}

	if(setjmp(sigint_env) == 0) {
		std::cout << "first setjmp." << std::endl;
	}
	else {
		std::cout << "not first setjmp." << std::endl;
	}

	std::cout << "> ";
	//read from file stream "stdin"
	while (fgets(buf, MAXLINE, stdin) != NULL) {
		fputs(buf,stdout);
		std::cout << "> ";
	}

	return 0;
}
```
将上面的代码生成可执行文件sample，做如下测试。   
在第一个终端中执行sample，在第二个终端中多次给sample进程发送SIGINT信号，结果分别如下：  
第一个终端显示内容如下：  
```
***@***> ./sample
first setjmp.
> Hello
Hello
> SIGINT catched.
not first setjmp.
>
```
第二个终端显示内容如下：  
```
***@***> ps -eH | grep sample
12316 pts/2    00:00:00         sample
***@***> kill -s SIGINT 12316
***@***> kill -s SIGINT 12316
***@***> kill -s SIGINT 12316
***@***>
```
从上面可以看出，sample进程只收到第一个SIGINT信号并调用信号处理函数跳转到main函数while循环前面。之后再给sample进程发送SIGINT信号，sample进程就再也收不到该信号了。
