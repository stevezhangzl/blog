---
title: WebRTC-第5章实现高性能网络服务器
date: 2023-07-11 05:42:42
tags:
---



[toc]



# 高性能网络服务器



- 通过fork实现高性能网络服务器
- 通过select实现高性能网络服务器
- 通过epoll实现高性能网络服务器



## 1 fork

### 1.1 以fork方式实现高性能网络服务器

- 每收到一个连接就创建一个子进程
- 父进程负责接收连接
- 通过fork创建子进程



```cpp
#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <string.h>
#include <unistd.h>

#define MESSAGE_LEN 1024
int main(int argc,char *argv[])
{
#if 0
    if(daemon(0,0)==-1){
        std::cout << "error" << std::endl;
        exit(-1);
    }
#endif
    pid_t pid;
    int socket_fd,accept_fd;
    int on= 1;
    int ret = -1;
    int backlog = 10;
    struct sockaddr_in server_address,remoteaddr;

    bzero(&server_address,sizeof(server_address));
    char in_buff[MESSAGE_LEN] = {0};

    socket_fd = socket(PF_INET,SOCK_STREAM,0);
    if(socket_fd == -1){
        std::cout<< "Failed to create socket!" << std::endl;
        exit(-1);
    }
    ret = setsockopt(socket_fd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on));
    if(ret == -1){
        std::cout << "Failed setsocketopt" << std::endl;
    }
    server_address.sin_family = AF_INET;
    server_address.sin_port = 8111;
    server_address.sin_addr.s_addr = INADDR_ANY;
    ret = bind(socket_fd, (struct sockaddr *)&server_address,sizeof(struct sockaddr_in));
    if(ret==-1){
        std::cout << "Failed to bind addr!" << std::endl;
        exit(-1);
    }
    ret = listen(socket_fd,backlog);
    if(ret==-1){
        std::cout << "Failed to listen socket!" << std::endl;
        exit(-1);
    }

    for(;;){
        socklen_t addr_len = sizeof(struct sockaddr);
        accept_fd = accept(socket_fd,(struct sockaddr *)&remoteaddr,&addr_len);

        pid = fork();
        if(pid == 0){//子进程处理，父进程跳过
            for(;;){
                ret = recv(accept_fd,(void *)in_buff,MESSAGE_LEN,0);
                if(ret == 0){//断开连接
                    std::cout  << "断开连接" << std::endl;
                    break;
                }
                if(ret < 0){
                    perror("recv");
                    break;
                }
                std::cout << "recv:" << in_buff << std::endl;
                ret = send(accept_fd,(void *)in_buff,MESSAGE_LEN,0);
                if(ret <= 0){
                    perror("send");
                    break;
                }
            }

            close(accept_fd);
        }



    }
    if(pid!=0) //父进程关掉socket_fd
        close(socket_fd);
    return 0;
}
```



### 1.2 fork方式带来的问题

- 资源被长期占用（子进程一直被占用，如果是长连接）
- 分配子进程花费时间长（如果是大并发，子进程创建时间过长）





## 2 select 



### 2.1 select 实现高性能网络服务器

#### 2.1.1 什么是异步IO

> 所谓异步IO，是指以事件触发的机制来对IO操作进行处理
>
> 与多进程和多线程技术相比，异步IO技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。





- 遍历文件描述符集中的所有描述符，找出有变化的描述符（在上层把找开的文件描述符都遍历一遍，本质上是半自动）
- 对于侦听的socket和数据处理的socket要区别对待
- socket必须设置为非阻塞方式工作



#### 2.1.2 重要API

- **FD_ZERO、FD_SET、FD_ISSET**
- **flag fcntl(fd,F_SETFL/F_GETFL,flag)  设置非阻塞/获取标志**
- **events select(nfds,readfds,writefds,exceptfds,timeout)**
  - ndfs: 最大文件描述符+1
  - readfds：读的文件描述符集
  - writefds：写的文件描述符集
  - exceptfds：异常的文件描述符集





```cpp
#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/select.h>



#define FD_SIZE 1024
#define MESSAGE_LEN 1024
int main(int argc,char *argv[])
{
#if 0
    if(daemon(0,0)==-1){
        std::cout << "error" << std::endl;
        exit(-1);
    }
#endif
    int max_fd = -1;
    int events = 0;
    int flags;
    int socket_fd,accept_fd;
    int on= 1;
    int ret = -1;
    int backlog = 10;
    struct sockaddr_in server_address,remoteaddr;
    fd_set fd_sets;
    bzero(&server_address,sizeof(server_address));
    char in_buff[MESSAGE_LEN] = {0};
    int accept_fds[FD_SIZE] = {-1,};
    int curpos = -1;
    socket_fd = socket(PF_INET,SOCK_STREAM,0);
    if(socket_fd == -1){
        std::cout<< "Failed to create socket!" << std::endl;
        exit(-1);
    }
    ret = setsockopt(socket_fd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on));
    if(ret == -1){
        std::cout << "Failed setsocketopt" << std::endl;
    }
    flags = fcntl(socket_fd,F_GETFL,0);
    fcntl(socket_fd,F_SETFL,flags|O_NONBLOCK);

    max_fd = socket_fd;
    server_address.sin_family = AF_INET;
    server_address.sin_port = 8111;
    server_address.sin_addr.s_addr = INADDR_ANY;
    ret = bind(socket_fd, (struct sockaddr *)&server_address,sizeof(struct sockaddr_in));
    if(ret==-1){
        std::cout << "Failed to bind addr!" << std::endl;
        exit(-1);
    }
    ret = listen(socket_fd,backlog);
    if(ret==-1){
        std::cout << "Failed to listen socket!" << std::endl;
        exit(-1);
    }

    for(;;){
        FD_ZERO(&fd_sets);
        FD_SET(socket_fd,&fd_sets);

        for(int i=0;i<FD_SIZE;i++){
            if(accept_fds[i]!=-1){
                if(accept_fds[i] > max_fd){
                    max_fd = accept_fds[i];
                }
                FD_SET(accept_fds[i],&fd_sets);
            }
        }

        events = select(max_fd+1,&fd_sets,NULL,NULL,NULL);
        if(events<0){
            std::cout <<"Failed to use select" << std::endl;
            break;
        }else if(events == 0){
            std::cout << "timeout....!" <<std::endl;
            continue;
        }else if(events){
            if(FD_ISSET(socket_fd,&fd_sets)){
                for(int i = 0;FD_SIZE;i++){
                    if(accept_fds[i] == -1){
                        curpos = i;
                        break;
                    }
                }
                socklen_t addr_len = sizeof(struct sockaddr);
                accept_fd = accept(socket_fd,(struct sockaddr *)&remoteaddr,&addr_len);

                flags = fcntl(accept_fd,F_GETFL,0);
                fcntl(accept_fd,F_SETFL,flags|O_NONBLOCK);

                accept_fds[curpos] = accept_fd;


            }

            for(int i=0;i<FD_SIZE;i++){
                if(accept_fds[i]!=-1&&FD_ISSET(accept_fds[i],&fd_sets)){
                    memset(in_buff,0,MESSAGE_LEN);
                    ret = recv(accept_fds[i],(void *)in_buff,MESSAGE_LEN,0);
                    if(ret == 0){//断开连接
                        std::cout  << "断开连接" << std::endl;
                        close(accept_fds[i]);
                        break;
                    }
                    if(ret < 0){
                        perror("recv");
                        break;
                    }
                    std::cout << "recv:" << in_buff << std::endl;
                    ret = send(accept_fds[i],(void *)in_buff,MESSAGE_LEN,0);
                    if(ret <= 0){
                        perror("send");
                        break;
                    }

                }
            }
        }
    }
    close(socket_fd);
    return 0;
}
```





#### 2.1.3 设置select超时时间

```c
struct timeval{
  long tv_sec; //秒
  long tv_usec; //微秒
}
```



#### 2.1.4 select函数输入参数的意义

- 我们关心的文件描述符
- 对每个文件描述符我们关心的状态（读、写、异常）
- 我们要等待的时间（永远、一段时间、不等待）



#### 2.1.5 从select函数得到的信息

- 已经做好准备的文件描述符的个数
- 对于读、写、异常，哪些文件描述符准备好了



### 2.2 理解select模型

- 理解select模型的关键在于理解fd_set类型，一个整形字的集合
- fd_set就是多个整型字的集合，每个bit代表一个文件描述符
- FD_ZERO 表示将所有位置0
- FD_SET 是将fd_set中的某一位置 1
- select 函数执行后，系统会修改fd_set中的内容
- select 函数执行后，应用层要重新设置fd_set中的内容



### 2.3 理解selec模型示意图



![image-20230717132635873](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230717132635873.png)

发生改变时，发现 3 6位置发生变化 5没有变化 ，调用完成后，重新把感兴趣的描述符置为1



## 3 epoll

### 3.1 epoll实现高性能网络服务器

#### 3.1.1 使用Epoll的好处

- 没有文件描述符的限制
- 工作效率不会随着文件描述符的增加而下降，epoll只返回发生改变的文件描述符，不用全遍历
- Epoll经过系统内核优化更高效



#### 3.1.2 Epoll事件的触发模式

- Level Trigger 水平触发：没有处理反复发送，epoll默认是水平触发，如果需要边缘触发，需要设置
- Edge Trigger 边缘触发：只发送一次，效率更高，开发更难写



#### 3.1.3 Epoll重要API

- int epoll_create() 参数无意义，可忽略
- int epoll_ctl(epfd,op,fd,struct epoll_event *event); 增、删、改 efd
- int epoll_wait(epfd,events,maxevents,timeout); 
  - events 事件集合
  - maxevents 最大返回多少个，这个可以小于返回的集合的的大小  如果events的个数是100 maxevents是10 那就是每次只返回10个



#### 3.1.4 Epoll的事件

- EPOLLET  边缘触发
- EPOLLIN  
- EPOLLOUT
- EPOLLPRI 出现中断的时候
- EPOLLERR 出现问题的时候
- EPOLLHUP 出现挂起的时候



#### 3.1.5 epoll_ctl 相关操作

- EPOLL_CTL_ADD
- EPOLL_CTL_MOD 修改
- EPOLL_CTL_DEL 将已经加入的epfd 进行删除



#### 3.1.6 Epoll重要的结构体

```c
typedef union epoll_data{ //这里的是联合体，只能用ptr或 fd或 u32等，但不能同时
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
}epoll_data_t;

struct epoll_event{
  uint32_t events； //Epoll events  这里是EPOLLIN EPOLLOUT EPOLLERR 这些事件
  epoll_data_t data; //user data variable   
}
```





#### 3.1.7 Epoll Server 代码

```cpp
#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <fcntl.h>

#define TIMEOUT 500
#define MAX_EVENTS 20
#define MESSAGE_LEN 1024
int main(int argc,char *argv[])
{
#if 0
    if(daemon(0,0)==-1){
        std::cout << "error" << std::endl;
        exit(-1);
    }
#endif

    int socket_fd,accept_fd;
    int on= 1;
    int ret = -1;
    int backlog = 10;
    struct sockaddr_in server_address,remoteaddr;
    int flags;
    bzero(&server_address,sizeof(server_address));
    char in_buff[MESSAGE_LEN] = {0};

    socket_fd = socket(PF_INET,SOCK_STREAM,0);
    if(socket_fd == -1){
        std::cout<< "Failed to create socket!" << std::endl;
        exit(-1);
    }
    flags = fcntl(socket_fd,F_GETFL,0);
    fcntl(socket_fd,F_SETFL,flags|O_NONBLOCK);

    ret = setsockopt(socket_fd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on));
    if(ret == -1){
        std::cout << "Failed setsocketopt" << std::endl;
    }
    server_address.sin_family = AF_INET;
    server_address.sin_port = 8111;
    server_address.sin_addr.s_addr = INADDR_ANY;
    ret = bind(socket_fd, (struct sockaddr *)&server_address,sizeof(struct sockaddr_in));
    if(ret==-1){
        std::cout << "Failed to bind addr!" << std::endl;
        exit(-1);
    }
    ret = listen(socket_fd,backlog);
    if(ret==-1){
        std::cout << "Failed to listen socket!" << std::endl;
        exit(-1);
    }

    struct epoll_event ev,events[MAX_EVENTS];
    int epfd = epoll_create(256);
    ev.events = EPOLLIN;
    ev.data.fd = socket_fd;
    int event_number;
    epoll_ctl(epfd,EPOLL_CTL_ADD,socket_fd,&ev);
    for(;;){

        event_number = epoll_wait(epfd,events,MAX_EVENTS,TIMEOUT);
        for(int i = 0; i<event_number;i++){
            if(events[i].data.fd == socket_fd){
                socklen_t addr_len = sizeof(struct sockaddr);
                accept_fd = accept(socket_fd,(struct sockaddr *)&remoteaddr,&addr_len);
                flags = fcntl(accept_fd,F_GETFL,0);
                fcntl(accept_fd,F_SETFL,flags|O_NONBLOCK);

                ev.events = EPOLLIN|EPOLLET;
                ev.data.fd = accept_fd;
                epoll_ctl(epfd,EPOLL_CTL_ADD,accept_fd,&ev);
            }else if(events[i].events & EPOLLIN){
                do{
                    memset(in_buff,0,MESSAGE_LEN);
                    ret = recv(events[i].data.fd,(void *)in_buff,MESSAGE_LEN,0);
                    if(ret == 0){//断开连接
                        std::cout  << "断开连接" << std::endl;
                        close(events[i].data.fd);
                        break;
                    }
                    if(ret == MESSAGE_LEN){
                        std::cout << "maybe have data..." << std::endl;
                    }
                }while(ret<0  && errno == EINTR);
                if(ret < 0){
                    switch(errno){
                    case EAGAIN://要等待下一次事件的到来，现在没有数据了
                        break;
                    default:
                        break;
                    }
                }
                if(ret >0){
                    std::cout << "recv:" << in_buff << std::endl;
                    send(events[i].data.fd,(void*)in_buff,MESSAGE_LEN,0);
                }
            }
        }
    }
    close(socket_fd);
    return 0;
}
```





### 3.2 epoll+fork实现高性能网络服务器

尽量把cpu都利用上，让它们合理的均摊

使用fork 的小技巧：提前fork 

进一步优化：把主核和fork进行绑定





## 4 异步IO处理库





















