---
layout:         page
title:         【Linux】Multiplexed I/O——select的性能问题以及poll
subtitle:       poll与select的对比
date:           2018-10-27
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

### Multiplexed I/O with select
一个Linux程序经常需要处理多个文件描述符，这种情况下，采用阻塞I/O 或者非阻塞I/O都不行：阻塞I/O可能一直阻塞在一个文件描述符上，尽管此时其他文件描述符已经可以读取或者写入；而非阻塞I/O要求程序轮询，这是浪费系统时间。

Multiplexed I/O 可以很方便地处理多个文件描述符的情况。其中可以使用的有select、poll（以及带信号屏蔽的 pselect、ppoll）。

先来看看select：
```cpp
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);

```
select返回：
1. 返回一个N（N>0），表示已经准备好的文件描述符数（至少一个文件描述符已经准备好才返回）。 什么是“准备好”？文件描述符集readfds中的至少一个文件描述符可读，或者writefds中的至少一个文件描述符可写，或者系统检测到exceptfds中的至少一个文件描述符发生异常条件。这里提一下，Linux系统中，只有socket会发生这种异常条件（接收到 out-of-band 数据）。
2. 返回0，表示已经超时。设置timeout=NULL，则select无限期阻塞，直到有描述符准备好。
3. 返回-1，表示出错。errno指示出错原因，比如 errno==EINTR 表示select被异步信号中断了才返回。

#### 1、使用select时的注意事项
select有几个地方值得注意：
1. fd_set这个结构的缓冲区大小是预设的，并非动态增长。也就是说select的文件描述符集参数不能包含无限个描述符。通过一个负整数参数或者一个 >= FD_SETSIZE 的参数调用FD_CLR() 和 FD_SET() 都会导致 **未定义的行为**。
2. **select会更新它的参数**，包括文件描述符集以及超时。select返回时，三个参数readfds、writefds、exceptfds就已经被更新，包含当前准备好的可读，可写，发生异常条件的文件描述符集。这个特性会影响我们使用select的方式，比如处理完一批描述符后，要重新设置select的几个描述符集参数，才能继续阻塞在这些描述符上，等待它们准备好被处理。
3. **nfds是一个很重要的参数**，这个参数值一般都要设置成我们要处理的文件描述符中的最大值，再加1。为什么？因为select就是通过遍历 0 ~ nfds-1 之间的描述符，来判断哪些描述符准备好了。这个特性使得在我们要处理的描述符值比较大的情况下，select的性能很差。后面会再次说这个问题。

#### 2、代码
这是一个测试小程序，包括server和client。server可以接受多个client的连接并返回client的echo内容。没什么实际用处，只是演示一下（另外，这里用FIFO来实现进程通信）。
~~~cpp
// server


#define MAX(a,b) \
    ((a) >= (b) ? (a) : (b))

int main(int argc, char* arg[]) {
    std::string serverfifo = "serverfifo";
    int not_exist = access(serverfifo.c_str(), F_OK);
    if (not_exist != 0) {
        int err = mkfifoat(AT_FDCWD, serverfifo.c_str(), S_IRUSR |S_IWUSR);
        if (err == -1) {
            std::cout << "mkfifoat " << serverfifo << "  failed." << std::endl;
            return 1;
        }
    }
    
    int fifo = openat(AT_FDCWD, serverfifo.c_str(),O_RDWR);
    if (fifo == -1) {
        std::cout << "openat " << serverfifo << "  failed." << std::endl;
        return 1;
    }

    std::map<std::string, int> clients;
    auto do_clean = [&fifo, &serverfifo, &clients] {
        close(fifo);
        unlink(serverfifo.c_str());

        for (auto var : clients) {
            close(var.second);
        }
    };
    
    fd_set read_set;
    FD_ZERO(&read_set);
    // FD_SET(STDIN_FILENO, &read_set[0]);

    // FD_SET(fifo, &read_set[0]);


    // fd_set write_set;

    // FD_ZERO(&write_set);

    // FD_SET(fifo, &write_set);


    char buf[4096] = { 0 };
    int max_fd = fifo;
    
    while(true) {
        FD_ZERO(&read_set);
        FD_SET(STDIN_FILENO, &read_set);
        FD_SET(fifo, &read_set);

        int fds = select(max_fd + 1, &read_set, NULL, NULL, NULL);
        if (fds == -1) {
            std::cout << "select failed." << std::endl;
            do_clean();
            return 1;
        }
        else if(fds > 0) {
            // fd_sets are updated by select

            if (FD_ISSET(fifo, &read_set)) {
                memset(buf,0,sizeof(buf));
                int n = read(fifo, buf, sizeof(buf));
                if (n == 0) {
                    // The other end point of this fifo was closed

                    std::cout << "fifo was closed by client." << std::endl;
                    continue;
                }
                std::cout << "recv: " << buf << std::endl;

                std::string data(buf, strlen(buf));
                if (data.find("connect") == 0) {
                    // data has a '\n' at the end.

                    std::string client_pid = data.substr(8, data.length() - 8).c_str();
                    int client_fifo = open(client_pid.c_str(),O_WRONLY);
                    if (client_fifo == -1) {
                        std::cout << "open fifo of " << client_pid << "  failed. errno=" << errno << std::endl;
                    }
                    else {
                        clients[client_pid] = client_fifo;
                        max_fd = MAX(max_fd, client_fifo);

                        // FD_SET(client_fifo, &read_set[index_of_set ^ 1]);

                    }
                }
                else if (data.find("echo ") == 0) {
                    std::string client_pid = data.substr(data.find_last_of(' ') + 1);
                    std::string message = data.substr(5, data.find_last_of(' ') - 4).c_str();
                    write(clients[client_pid], message.c_str(), message.length());
                }
            }

            if (FD_ISSET(STDIN_FILENO, &read_set)) {
                memset(buf,0,sizeof(buf));
                int n = read(STDIN_FILENO, buf, sizeof(buf));
                if (n > 0 ) {
                    if(strcmp(buf, "q\n") == 0 || strcmp(buf, "quit\n") == 0) {
                        break;
                    }
                    else {
                        //std::cout << buf << std::endl;

                    }
                }
            }          
        }
    }

    do_clean();

    return 0;
}

~~~

~~~cpp
// client


int main(int argc, char* arg[]) {
    pid_t pid = getpid();
    std::cout << "pid=" << pid << std::endl;
    std::cout << "Enter 'connect' to connect to server." << std::endl;
    std::cout << "Enter 'echo **' to request the remote echo server." << std::endl;

    std::string client_fifo = std::to_string(pid);
    int err = mkfifoat(AT_FDCWD, client_fifo.c_str(), S_IRUSR |S_IWUSR);
    if (err == -1) {
        std::cout << "mkfifo " << client_fifo << "  failed." << std::endl;
        return 1;
    }

    std::string server_fifo_name = "serverfifo";
    int server_fifo = openat(AT_FDCWD, server_fifo_name.c_str(),O_WRONLY);
    if (server_fifo == -1) {
        std::cout << "openat " << server_fifo_name << "  failed." << std::endl;
        return 1;
    }

    // An open for read-only FIFO without O_NONBLOCK flag would block

    // until some other process opens the FIFO for writing.

    int fifo = openat(AT_FDCWD, client_fifo.c_str(),O_RDWR);
    if (fifo == -1) {
        std::cout << "openat " << client_fifo << "  failed." << std::endl;
        close(server_fifo);

        return 1;
    }

    auto do_clean = [&fifo, & client_fifo,&server_fifo,&server_fifo_name] {
        close(fifo);
        unlink(client_fifo.c_str());

        close(server_fifo);
        unlink(server_fifo_name.c_str());
    };

    fd_set read_set;
    FD_ZERO(&read_set);

    char buf[4096] = { 0 };
    int max_fd = fifo;
    while(true) {
        FD_ZERO(&read_set);
        FD_SET(STDIN_FILENO, &read_set);
        FD_SET(fifo, &read_set);

        int fds = select(max_fd + 1, &read_set, NULL, NULL, NULL);
        if (fds == -1) {
            std::cout << "select failed." << std::endl;
            do_clean();
            return 1;
        }
        else if(fds > 0) {
            if (FD_ISSET(fifo, &read_set)) {
                memset(buf,0,sizeof(buf));
                int n = read(fifo, buf, sizeof(buf));
                if (n == 0) {
                    std::cout << "fifo was closed by client." << std::endl;
                    do_clean();
                    return 1;
                }
                std::cout << buf << std::endl;
            }

            if (FD_ISSET(STDIN_FILENO, &read_set)) {
                memset(buf,0,sizeof(buf));
                int n = read(STDIN_FILENO, buf, sizeof(buf));
                if (n > 0 ) {
                    if(strcmp(buf, "q\n") == 0 || strcmp(buf, "quit\n") == 0) {
                        break;
                    }
                    else {
                        std::string request(buf, strlen(buf));
                        // remove the last '\n'
                        request.pop_back();
                        request.push_back(' ');
                        request += std::to_string(pid);
                        write(server_fifo, request.c_str(), request.length());
                    }
                }
            }          
        }
    }

    do_clean();
    
    return 0;
}

~~~

#### select的性能问题
再次说一下select的性能问题。

首先：假设我们想要处理两个文件描述符1001和1002，那么我们应该给select传的第一个参数nfds的值就至少要是1003，nfds==1003使得select会遍历0~1002这些描述符来检测它们的状态。那么问题就很明显了，0~1000之间的这么多描述符，其实我们并不关心，也不需要处理，但是select依然会去遍历它们，这里就导致了性能损耗。

select还有另一个问题。fd_set这个结构得多大？假设它至少能处理一个进程能打开的所有描述符, Linux进程一般至少能打开1024个文件描述符。那么假设fd_set至少能包含这么多描述符，它占用的空间大小就会很大。而处理一个这么大的结构体，显然，效率也会降低。

后面可以看到poll能解决select的问题。

### Multiplexed I/O with poll
看看poll的定义：
```cpp
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
int ppoll(struct pollfd *fds, nfds_t nfds,
               const struct timespec *tmo_p, const sigset_t *sigmask);

struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};

```
poll返回：
1. 返回一个N（N>0），表示 revents!=0 的pollfd结构数，也就是对应的 events 发生了，或者发生错误的文件描述符的数目。
2. 返回0，表示已经超时，timeout是定时毫秒数。timeout的值为0，可以使得poll立即返回；timeout的值如果是一个负数，则如果没有任何描述符准备好，poll就无限期阻塞。
3. 返回-1，表示出错。errno指示出错原因，比如 errno==EINTR 表示select被异步信号中断了才返回。

#### 1、使用poll时的注意事项
poll和select不同，**poll不会更新传入的参数值**。我们通过 events 告诉poll我们关心的事件（POLLIN 或者 POLLOUT 等等），而poll返回时通过 revents 返回对应描述符发生的事件。events 的值也可以设成0，此时revents中能返回的事件就只有 POLLHUP, POLLERR 和 POLLNVAL这三个。

#### 2、代码
下面是改为使用poll的server代码，client依然使用select，代码如上。
```cpp

// server


#define MAX(a,b) \
    ((a) >= (b) ? (a) : (b))

int main(int argc, char* arg[]) {
    std::string serverfifo = "serverfifo";
    int not_exist = access(serverfifo.c_str(), F_OK);
    if (not_exist != 0) {
        int err = mkfifoat(AT_FDCWD, serverfifo.c_str(), S_IRUSR |S_IWUSR);
        if (err == -1) {
            std::cout << "mkfifoat " << serverfifo << "  failed." << std::endl;
            return 1;
        }
    }
    
    int fifo = openat(AT_FDCWD, serverfifo.c_str(),O_RDWR);
    if (fifo == -1) {
        std::cout << "openat " << serverfifo << "  failed." << std::endl;
        return 1;
    }

    char buf[4096] = { 0 };
    std::map<std::string, int> clients;

    auto do_clean = [&fifo, &serverfifo, &clients] {
        close(fifo);
        unlink(serverfifo.c_str());

        for (auto var : clients) {
            close(var.second);
        }
    };

    pollfd pfds[2];
    
    pfds[0].fd = STDIN_FILENO;
    pfds[0].events = POLLIN;

    pfds[1].fd = fifo;
    pfds[1].events = POLLIN;

    while(true) {
        int readable_fds = poll(pfds, sizeof(pfds)/sizeof(pollfd), -1);
        if(readable_fds == -1) {
            std::cout << "poll failed." << std::endl;
            do_clean();
            return 1;
        }
        else if (readable_fds > 0) {
            if(pfds[0].revents & POLLIN) {
                memset(buf,0,sizeof(buf));
                int n = read(pfds[0].fd, buf, sizeof(buf));
                if (n > 0 ) {
                    if(strcmp(buf, "q\n") == 0 || strcmp(buf, "quit\n") == 0) {
                        break;
                    }
                    else {
                        //std::cout << buf << std::endl;
                    }
                }
            }
            
            if(pfds[1].revents & POLLIN) {
                memset(buf,0,sizeof(buf));
                int n = read(fifo, buf, sizeof(buf));
                if (n == 0) {
                    // The other end point of this fifo was closed
                    
                    std::cout << "fifo was closed by client." << std::endl;
                    continue;
                }
                std::cout << "recv: " << buf << std::endl;

                std::string data(buf, strlen(buf));
                if (data.find("connect") == 0) {
                    std::string client_pid = data.substr(8, data.length() - 8).c_str();
                    int client_fifo = open(client_pid.c_str(),O_WRONLY);
                    if (client_fifo == -1) {
                        std::cout << "open fifo of " << client_pid << "  failed. errno=" << errno << std::endl;
                    }
                    else {
                        clients[client_pid] = client_fifo;
                    }
                }
                else if (data.find("echo ") == 0) {
                    std::string client_pid = data.substr(data.find_last_of(' ') + 1);                    
                    if(clients.find(client_pid) == clients.end()) {
                        std::cout << "Please connect to the server first. " << std::endl;                        
                        continue;
                    }

                    std::string message = data.substr(5, data.find_last_of(' ') - 4).c_str();
                    write(clients[client_pid], message.c_str(), message.length());
                }
            }
        }
    }
    
    do_clean();

    return 0;
}
```