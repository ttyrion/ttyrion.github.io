---
layout:     page
title:      【Linux】自动信号屏蔽和 longjmp的问题
subtitle:   Linux信号处理过程
date:       2018-09-12
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 前言

>信号处理函数中常会使用跳转函数跳转到应用程序某处，而不是从信号处理函数返回，例如可以使用 longjmp & setjmp，但是跳转却有可能导致问题。

## Linux 怎么处理信号？
1. 系统给进程递送信号signo，并调用信号处理函数sig_handler。
2. 进入sig_handler后，signo**信号被自动加入进程的信号屏蔽字中**。这阻止再次产生的信号signo中断sig_handler的调用。
3. 执行sig_handler内的代码。
4. sig_handler内的代码执行完毕，**系统恢复进程的信号屏蔽字**，信号signo不再被阻塞。
5. 进程回到产生信号的地方继续执行。

如下是一段测试代码：  
```cpp
#define write_to_terminal(msg) { \
		std::stringstream stream;\
		stream << msg; \
		write(STDOUT_FILENO,stream.str().c_str(),stream.str().length()); \
	}

std::string read_from_terminal() {
	char buf[80] = { 0 };
	int bytes = read(STDIN_FILENO, buf, sizeof(buf));
	std::string msg;
	if(bytes > 0) {
		msg.append(buf);
	}
	return msg;
}

bool is_sig_blocked(int signo) {
	sigset_t cur_sig_mask;
	if(sigprocmask(SIG_SETMASK, nullptr, &cur_sig_mask) == 0) {
		if(sigismember(&cur_sig_mask, signo)) {
			return true;
		}
	}

	return false;	
}

static void on_sigint(int signo) {
	switch(signo) {
		case SIGINT: {
			write_to_terminal("[on_sigint]:SIGINT catched. " << std::endl);
			bool sigint_blocked = is_sig_blocked(SIGINT);
			write_to_terminal("[on_sigint]:SIGINT blocked? " << sigint_blocked  << std::endl);
			write_to_terminal("[on_sigint]>");
			std::string msg = read_from_terminal();
			write_to_terminal("[on_sigint]:" << msg);
			break;
		}
		default: {
			write_to_terminal("[on_sigint]:An unknown signal catched." << std::endl);
			break;
		}
	}
}
int main(int argc, char** argv) {
	bool sigint_blocked = is_sig_blocked(SIGINT);
	write_to_terminal("[main]:SIGINT blocked? " << sigint_blocked  << std::endl);

	if (signal(SIGINT, on_sigint) == SIG_ERR) {
		write_to_terminal("[main]:Try  catching SIGINT failed." << std::endl);
		return 1;
	}
	
	for(;;) {
		write_to_terminal("[main]> ");
		std::string msg = read_from_terminal();
		write_to_terminal("[main]:" << msg);
	}

	return 0;
}
```
上述代码生成的可执行文件为sample。下面在 TerminalA 中执行sample。所有的**":"**表示进程输出的提示信息，而**">"**表示进程等待输入。  
```cpp
//TerminalA

marting@192:~/signal/sample/bin> ./sample 
[main]:SIGINT blocked? 0
[main]> Hello
[main]:Hello
[main]> 
```
从上面输出可知，启动进程的时候，SIGINT信号并没有被阻塞。下面通过另一个 TerminalB 给进程发送一个SIGINT信号：  
```cpp
//TerminalB

marting@192:~/signal/sample/bin> ps -eH | grep sample 
 7551 pts/1    00:00:00         sample
marting@192:~/signal/sample/bin> kill -s SIGINT 7551
marting@192:~/signal/sample/bin> 
```

再看 TerminalA 输出，以下只列出每个Terminal内容的更新。  
```cpp
//TerminalA

...
[main]> [on_sigint]:SIGINT catched. 
[on_sigint]:SIGINT blocked? 1
[on_sigint]>
```
从 TerminalA 的输出可以看出，进程收到了 SIGINT 信号，并且进入信号处理函数后，SIGINT信号已经被加入进程的信号屏蔽字。
信号处理函数被终端读取操作阻塞，我们再次给进程发送SIGINT信号看看会怎么样。  
```cpp
//TerminalB

marting@192:~/signal/sample/bin> kill -s SIGINT 7551
marting@192:~/signal/sample/bin> kill -s SIGINT 7551
marting@192:~/signal/sample/bin> kill -s SIGINT 7551
marting@192:~/signal/sample/bin> kill -s SIGINT 7551
marting@192:~/signal/sample/bin> 
```
在 TerminalB 中，一次性给进程发送了四个 SIGINT 信号。但是 TerminalA 并没有任何输出，因为信号处理函数 on_sigint 仍被阻塞。
现在在 TerminalA 中输入Hello，恢复信号处理函数 on_sigint 的执行。  
```cpp
//TerminalA

...
[on_sigint]>Hello
[on_sigint]:Hello
[on_sigint]:SIGINT catched. 
[on_sigint]:SIGINT blocked? 1
[on_sigint]>
```
从 TerminalA 的输出可以看出，信号处理函数 on_sigint 刚执行完，又被再次调用并且阻塞。因为在 on_sigint 第一次阻塞的过程中，
我们给进程发送了四个 SIGINT 信号，此时SIGINT信号被进程屏蔽，处于未决状态，等待内核将信号递送给进程。所以 on_sigint 第一次
执行完毕后，进程再次被信号中断，进入信号处理函数中（同时可以看到，SIGINT信号此时也再次被加入进程信号屏蔽字）。在 TerminalA 
中输入Hello2,再次恢复 on_sigint 的执行：  
```cpp
//TerminalA

...
[on_sigint]>Hello2
[on_sigint]:Hello2
//光标此时停留在这一行

```
从上面的输出可以看出，信号处理函数执行完毕之后，进程并没有再次被SIGINT信号中断。也就是说，我们发给进程的四个SIGINT信号，系统
并没有将这几个信号加入队列，然后逐个递送给进程。**系统只是保留并且递送了一个SIGINT信号**。上面光标的位置也说明，进程再次执行到
for循环，等待用户输入。

## longjmp and setjmp
信号处理函数中常会使用跳转函数跳转到应用程序某处，而不是从信号处理函数返回。那么我们使用 longjmp 来试试看会怎么样。
把上面的测试代码稍作更改：    
```cpp
static jmp_buf sigint_env;
//信号处理函数break语句之前加入一行

longjmp(sigint_env, 1);

//main函数入口后加入几行

if(setjmp(sigint_env) == 0) {
	std::cout << "[main]first setjmp." << std::endl;
}
else {
	std::cout << "[main]not first setjmp." << std::endl;
}
}
```
将上面的代码生成可执行文件sample，同样在 TerminalA 中进行类似验证。TerminalA中的输出内容如下：   
```cpp
//TerminalA

marting@192:~/signal/sample/bin> ./sample 
[main]first setjmp.
[main]:SIGINT blocked? 0
[main]> [on_sigint]:SIGINT catched. 
[on_sigint]:SIGINT blocked? 1
[on_sigint]>Hello
[on_sigint]:Hello
[main]not first setjmp.
[main]:SIGINT blocked? 1
[main]> 
``` 
从上面可以看出，sample进程只收到第一个SIGINT信号并调用信号处理函数，信号处理函数执行完毕后，进程跳转到main函数入口之后。
此时SIGINT屏蔽字仍然在进程信号屏蔽字中（[main]:SIGINT blocked? 1）。之后再给sample进程发送SIGINT信号，进程却再也收不到该信号了。

所以信号处理函数里面的跳转，不能用 longjmp，可以改用 siglongjmp。
