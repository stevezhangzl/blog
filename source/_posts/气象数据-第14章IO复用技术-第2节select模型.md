---
title: 气象数据-第14章IO复用技术-第2节select模型
date: 2023-04-11 05:32:34
tags:
---



select 模型是基于事件的

事件分为两类，可读事件  可写事件

可读事件

1. 新客户端连接请求 accept
2. 新客户端有报文到recv
3. 客户端连接已断开



可写事件

1. 可以向客户端发送报文send，可以写



select()的功能是等待事件的发生。包括可读事件和可写事件。

等待可读事件的发生很好理解，比如新客户端连接请求，有报文接收，客户端断开。但可写事件比较难理解，什么叫等待可写事件，我们习惯性的思维是有需要写的东西，想写就send，往 socket里写。`实际上并不是想写就一定能写的，也可能会阻塞`，会等待。下面做一个测试。测试写也会阻塞的情况。



改造demo07.cpp 

```cpp
//testiowriteclient.cpp

#include "../_public.h"
 
int main(int argc,char *argv[])
{
  if (argc!=3)
  {
    printf("Using:./demo07 ip port\nExample:./demo07 127.0.0.1 5005\n\n"); return -1;
  }

  CTcpClient TcpClient;

  // 向服务端发起连接请求。
  if (TcpClient.ConnectToServer(argv[1],atoi(argv[2]))==false)
  {
    printf("TcpClient.ConnectToServer(%s,%s) failed.\n",argv[1],argv[2]); return -1;
  }

  char buffer[102400];
 
  // 与服务端通讯，发送一个报文后等待回复，然后再发下一个报文。
  for (int ii=0;ii<100000;ii++)
  {
    SPRINTF(buffer,sizeof(buffer),"这是第%d个超级女生，编号%03d。",ii+1,ii+1);
    if (TcpClient.Write(buffer)==false) break; // 向服务端发送请求报文。
    printf("发送：%s\n",buffer);

    //memset(buffer,0,sizeof(buffer));
    //if (TcpClient.Read(buffer)==false) break; // 接收服务端的回应报文。
    //printf("接收：%s\n",buffer);

    //sleep(1);  // 每隔一秒后再次发送报文。
  }
}
```

我们注释掉读的部分，只往服务端写，写100000个报文



改造 demo08.cpp

```cpp
//testiowriteserver.cpp

#include "../_public.h"
 
int main(int argc,char *argv[])
{
  if (argc!=2)
  {
    printf("Using:./demo08 port\nExample:./demo08 5005\n\n"); return -1;
  }

  CTcpServer TcpServer;

  // 服务端初始化。
  if (TcpServer.InitServer(atoi(argv[1]))==false)
  {
    printf("TcpServer.InitServer(%s) failed.\n",argv[1]); return -1;
  }

  // 等待客户端的连接请求。
  if (TcpServer.Accept()==false)
  {
    printf("TcpServer.Accept() failed.\n"); return -1;
  }

  printf("客户端（%s）已连接。\n",TcpServer.GetIP());

  char buffer[102400];

  // 与客户端通讯，接收客户端发过来的报文后，回复ok。
  while (1)
  {
    memset(buffer,0,sizeof(buffer));
    if (TcpServer.Read(buffer)==false) break; // 接收客户端的请求报文。
    printf("接收：%s\n",buffer);

    //usleep(300);

    //strcpy(buffer,"ok");
    //if (TcpServer.Write(buffer)==false) break; // 向客户端发送响应结果。
    //printf("发送：%s\n",buffer);
  }
}
```

我们注释掉写的部分，只从客户端里读



结果发现

![image-20230413061609692](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230413061609692.png)



![image-20230413061705337](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230413061705337.png)

服务端和客户端都很快的接收和发送完了100000个报文，没有问题

我们把服务端程序的usleep(300) 打开，让服务端读慢下来，客户端的写快于服务端的读，我们发现客户端往服务发送的时候，暂停了好几次，服务端一直在读，只不会比较慢，客户端要等服务端读一会，才会继续写下去



因为TCP存有缓冲区，缓冲区填满的时候，send会阻塞，所以客户端并没有像之前很快的写完，而是每次都要暂停一会，等待缓冲区有空间，才会继续往里写。这个就是是否可写事件的触发







- TCP有缓冲区，如果缓冲区已填满，send函数会阻塞。
- 如果发送端关闭了socket，缓冲区中的数据会继续发送给接收端，不会丢失。



可写事件可以简单的理解为，只要TCP的缓冲区没有满，就可以继续往里写





select() 等待事件的发生（监视哪些socket发生了事件）

所以用单进程也可以监视并发的功能



如图所示select的函数

![image-20230414052709894](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230414052709894.png)



- readfds：需要监视的可读事件的socket的集合
- writefds：需要监视的可写事件的socket的集合
- timeout：超时时间，如果监视的socket什么事情都没有发生，select将超时返回



socket的集合是什么意思呢

我们看下它的声明 vim /usr/include/sys/select.h



![image-20230414054301005](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230414054301005.png)



替换后

fds_bits[16]         它是一个位图  bitmap  1024个位



![image-20230414054655393](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230414054655393.png)

用位图表示socket集合是这样的，有新的socket连接，就把对应位置置为1 如果socket释放了，就把位置置为0



linux提供了4个宏来操作`位图`

![image-20230414054933843](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230414054933843.png)

- FD_ZERO:用于初始化位图，把全部的位置清0
- FD_SET:用于把某一个位置置为1，参数fd是需要置位的位置，意思是向集合中增加了一个新的socket
-  FD_CLR:用于把某一个位置置为空





再来说select函数的第一个参数

![image-20230416095748150](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230416095748150.png)

它的取值是，后面三个socket，readfds,writefds,exceptfds 这三个最大的socket的id再加1

意思是告诉select位图有多大



select的第三个参数，在网络编程中用不到，固定填空就行了，后面说的poll和epoll中就没这个了

最后一个是时间结构体





```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>

//初始化服务端的监听端口
int initserver(int port);

int main(int argc,char *argv[]){
  if(argc!=2){ printf("usage:./tcpselect port\n"); return -1;}

  //初始化服务端用于监听socket.
  int listensock = initserver(atoi(argv[1]));
  printf("listensock=%d\n",listensock);

  if(listensock<0){printf("initserver() failed.\n"); return -1;}

  fd_set readfds;       //读事件socket的集合，包括监听socket和客户端连接上来的的socket

  FD_ZERO(&readfds);    //初始化读事件socket的集合
  FD_SET(listensock,&readfds);    //把listensock添加到读事件socket的集合中。

  int maxfd = listensock;

  while (true){
    //事件 1）新客户端的连接请求 accept 2) 客户端有报文到达recv，可以读； 3）客户端连接已断开
        //4) 可以向客户端发送报文send，可以写。
    //可读事件  可写事件

    fd_set tmpfds = readfds;
    int infds = select(maxfd+1,&tmpfds,NULL,NULL,NULL);

    if (infds<0)
    {   
      printf("select() failed.\n");perror("select failed");break;
    }
    
    if (infds == 0)
    {
      printf("select() timeout.\n"); continue;
    }

    //如果infds>0 表示有事件发生的socket的数量
    for(int eventfd=0;eventfd<=maxfd;eventfd++){
      if (FD_ISSET(eventfd,&tmpfds)<=0) continue; //如果没有事件,continue

      //如果发生事件的是listensock,表示有新的客户端连上来了。
      if (eventfd==listensock)
      {
        struct sockaddr_in client;
        socklen_t len = sizeof(client);
        int clientsock = accept(listensock,(struct sockaddr*)&client,&len);
        if(clientsock<0) {perror("accept() failed");continue;}

        printf("accept client(socket=%d) ok.\n",clientsock);

        //把新客户端的socket加入可读 socket的集合。
        FD_SET(clientsock,&readfds);//注意这里加入的不是临时的，而是外成创建的
        if(maxfd < clientsock) maxfd = clientsock; //更新maxfd的值
      }else{
        //如果客户端连接的socket有事，表示有报文发过来或者连已断开
        char buffer[1024]; //存放从客户端读取的数据。
        memset(buffer,0,sizeof(buffer));
        if (recv(eventfd,buffer,sizeof(buffer),0)<=0)
        {
          //哪果客户端的连接已断开。
          printf("client(eventfd=%d) disconnected.\n",eventfd);
          close(eventfd); //关闭客户端socket连接
          FD_CLR(eventfd,&readfds); //把已关闭的客户端的socket从可读socket的集合中删除。

          //重新计算maxfd的值，注意，只有当eventfd==maxfd时才需要计算
          if (eventfd == maxfd)
          {
            for(int ii = maxfd;ii>0;ii--){ //从后面往前找
              if (FD_ISSET(ii,&readfds))
              {   
                maxfd = ii; break;
              }
              
            }
          }
          
        }else{
          //如果客户端有报文发过来。
          printf("recv(eventfd)=%d:%s\n",eventfd,buffer);
          //把接收到的报文内容原封不动的发回去
          send(eventfd,buffer,strlen(buffer),0);
        }
        
      } 
    }
  }
  

  return 0;
}


//初始化服务端的临听端口
int initserver(int port){
  int sock = socket(AF_INET,SOCK_STREAM,0);
  if(sock<0){
    perror("socket() failed"); return -1;
  }

  int opt = 1;unsigned int len = sizeof(opt);
  setsockopt(sock,SOL_SOCKET,SO_REUSEADDR,&opt,len);

  struct sockaddr_in servaddr;
  servaddr.sin_family = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port = htons(port);

  if(bind(sock,(struct sockaddr *)&servaddr,sizeof(servaddr)) < 0){
    perror("bind() failed"); close(sock); return -1;
  }

  if(listen(sock,5)!=0){
    perror("listen() failed"); close(sock); return -1;
  }


  return sock;

}
```





## select模型的细节 



- 写事件
- 超时机制
- 水平触发
- 性能测试
- 存在的问题



### 写事件

- 如果tcp缓冲区没有满，那么socket连接是可写的
- tcp发送缓冲区2.5M，接收缓冲区1M;



```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <error.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netdb.h>


int main(int argc,char *argv[]){
  if (argc!=3)
  {
    printf("Using:./demo01 ip port\nExample:./demo01 127.0.0.1 5005\n\n"); return -1;
  }

  // 第1步：创建客户端的socket。
  int sockfd;
  if ( (sockfd = socket(AF_INET,SOCK_STREAM,0))==-1) { perror("socket"); return -1; }
 
  // 第2步：向服务器发起连接请求。
  struct hostent* h;
  if ( (h = gethostbyname(argv[1])) == 0 )   // 指定服务端的ip地址。
  { printf("gethostbyname failed.\n"); close(sockfd); return -1; }
  struct sockaddr_in servaddr;
  memset(&servaddr,0,sizeof(servaddr));
  servaddr.sin_family = AF_INET;
  servaddr.sin_port = htons(atoi(argv[2])); // 指定服务端的通讯端口。
  memcpy(&servaddr.sin_addr,h->h_addr,h->h_length);
  if (connect(sockfd, (struct sockaddr *)&servaddr,sizeof(servaddr)) != 0)  // 向服务端发起连接清求。
  { perror("connect"); close(sockfd); return -1; }

  printf("connect ok.\n");
  char buf[1024];
  int bufsize = 0;
  socklen_t optlen = sizeof(bufsize);
  getsockopt(sockfd,SOL_SOCKET,SO_RCVBUF,&bufsize,&optlen);//获取接收缓冲区的大小
  printf("recv bufsize=%d\n",bufsize);

  getsockopt(sockfd,SOL_SOCKET,SO_SNDBUF,&bufsize,&optlen);//获取发送缓冲区的大小
  printf("send bufsize=%d\n",bufsize);
  return 0;


  for(int ii=0;ii<10000;ii++){
    memset(buf,0,sizeof(buf));
    printf("please input:");scanf("%s",buf);

    if (send(sockfd,buf,strlen(buf),0) <=0) 
    {
      printf("write() failed.\n"); close(sockfd); return -1;
    }
    
  }
  
  
}
```





>   printf("connect ok.\n");
>   char buf[1024];
>   int bufsize = 0;
>   socklen_t optlen = sizeof(bufsize);
>   getsockopt(sockfd,SOL_SOCKET,SO_RCVBUF,&bufsize,&optlen);//获取接收缓冲区的大小
>   printf("recv bufsize=%d\n",bufsize);
>
>   getsockopt(sockfd,SOL_SOCKET,SO_SNDBUF,&bufsize,&optlen);//获取发送缓冲区的大小
>   printf("send bufsize=%d\n",bufsize);
>   return 0;
>
> 我们通过这段代码可以获得接收和发送缓冲区的大小，这个因为不同的系统而不同

![image-20230417054328470](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230417054328470.png)





### 超时机制

select 第三个参数，设置超时时间



```cpp

    fd_set tmpfds = readfds;
    struct timeval timeout;  timeout.tv_sec = 10; timeout.tv_usec =0;
    int infds = select(maxfd+1,&tmpfds,NULL,NULL,&timeout);

    if (infds<0)
    {   
      printf("select() failed.\n");perror("select failed");break;
    }
    
    if (infds == 0)
    {
      printf("select() timeout.\n"); continue;
    }
```



```cpp
struct timeval
{
  __time_t tv_sec;		/* Seconds.  */
  __suseconds_t tv_usec;	/* Microseconds.  */
};
```

timeval 第一个参数是秒 ，第二个参数是微秒

如程序，把超时时间设置成10s  这样

每10秒就会返回select =0 超时

![image-20230423050521077](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230423050521077.png)



利用select函数的超时机制，可以对读写函数的超时做改造

<img src="/Users/zhanglong/Library/Application Support/typora-user-images/image-20230423051223525.png" alt="image-20230423051223525" style="zoom:50%;" />



发现recv是没有超时时间的参数的





### 水平触发

- 如果事件和数据已经在缓冲区里，程序调用select()时会报告事件，数据也不会丢失
- 如果select()已经报告了事件，但是程序没有处理它，下次调用select()的时候会重新报告















