---
title: WebRTC-第4章实现TCP/UDP服务器
date: 2023-07-10 04:39:55
tags:
---







[toc]

# TCP Server

## 1.TCP Server 网络编程基本步骤

1. **创建socket，指定使用的TCP协议**
2. **将socket与地址和端口绑定**
3. **侦听端口**
4. **创建新的socket**
5. **使用recv接收数据**
6. **使用send发送数据**
7. **使用close关闭连接**



## 2.TCP常见套接字选项

- SO_REUSEADDR 端口处于WAIT_TIME仍然可以启动
  - 当网络服务器关闭时，它有一个TIME_WATI，这个是当使用socket的时候，谁主动发起关闭，就会有一个WAIT_TIME，等待其他用户的最终的FIN，最后的消息过来才会真正的关闭，如果使用SO_REUSEADDR 就不用等待WAIT_TIME 可以启另外一个服务，复用地址和端口
- SO_RCVBUF 接收的缓冲区大小
- SO_SNDBUF 发送的缓冲区大小





## 3.TCP通信流程图

![image-20230710044658165](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230710044658165.png)

## 4.重要结构体

```c
struct sockaddr_in{
  sa_family_t			sin_family;//ipv4/ipv6
  unint16_t				sin_port;//端口
  struct in_addr	sin_addr;//ip地址，也是一个结构体
  char						sin_zero[8];//补位，都设为0
}

struct in_addr{
  in_addr_t				s_addr;//转成整型
}
```



程序里的函数内部使用，用字节来实现，和上面存的内容是一样的

```c
struct sockaddr{
  sa_family_t			sin_family;
  char						sin_zero[14]; 
}
```



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
    int socket_fd,accept_fd;
    int on= 1;
    int ret = -1;
    int backlog = 10;
    struct sockaddr_in localaddr,remoteaddr;


    char in_buff[MESSAGE_LEN] = {0};

    socket_fd = socket(AF_INET,SOCK_STREAM,0);
    if(socket_fd == -1){
        std::cout<< "Failed to create socket!" << std::endl;
        exit(-1);
    }
    ret = setsockopt(socket_fd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on));
    if(ret == -1){
        std::cout << "Failed setsocketopt" << std::endl;
    }
    localaddr.sin_family = AF_INET;
    localaddr.sin_port = 8111;
    localaddr.sin_addr.s_addr = INADDR_ANY;
    bzero(&(localaddr.sin_zero),8);
    ret = bind(socket_fd, (struct sockaddr *)&localaddr,sizeof(struct sockaddr_in));
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

        for(;;){
            ret = recv(accept_fd,(void *)in_buff,MESSAGE_LEN,0);
            if(ret == 0){//断开连接
                break;
            }
            std::cout << "recv:" << in_buff << std::endl;
            send(socket_fd,(void *)in_buff,MESSAGE_LEN,0);
        }
        close(accept_fd);
    }
    close(socket_fd);
    return 0;
}
```







# TCP Client



## TCP Client 网络编程基本步骤

1. 创建socket，指定使用TCP协议
2. 使用connect连接服务器
3. 使用recv/send接收/发送 数据
4. 关闭socket



![image-20230711044230469](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230711044230469.png)



# UDP Server



### UDP Server网络编程基本步骤

1. 创建socket，指定使用UDP协议
2. 将socket与地址和端口绑定
3. 使用recv/send接收/发送数据
4. 使用close关闭连接



## UDP通信流程图

![image-20230711052212555](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230711052212555.png)







