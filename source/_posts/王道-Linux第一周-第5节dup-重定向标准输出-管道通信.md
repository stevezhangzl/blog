---
title: 王道-Linux第一周-第5节dup-重定向标准输出-管道通信
date: 2023-07-13 06:04:25
tags:
---



[TOC]



## 1.文件描述符的复制

- 系统调用函数dup和dup2可以实现文件描述符的复制
- dup返回一个新的文件描述符（是自动分配的，数值是没有使用文件描述符的最小编号）。这个新的描述符是旧文件描述符的拷贝。这意味着两个描述符共享同一个数据结构
- dup2允许调用者用一个有效描述符(oldfd)和目标描述符(newfd)。函数成功返回时，目标描述将变成旧描述符的复制品，此时两个文件描述符现在都指赂同一个文件，并且是函数第一个参数（也就是oldfd）指向的文件（执行完成以后，如果newfd已经打开了文件，该文件将会被关闭），原型为：

```c
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd,int newfd);
```

- 如果是fd=fd1 在此情况下，两个文件描述符变量值相同，指向同一个打开的文件，但是内核的文件打开引用计数还是1，所以close(fd)和close(fd1)都会导致文件立即关闭
- 使用文件描述符的复制，情况会有所区别
- dup的原理：当使用文件的时候，为了和硬件（比如磁盘）建立联系，进程地址空间中应当分配一片空间存放各个已经打开文件inode的的信息（此时文件的信息已经存放在内存，和实际磁盘内容无关了），linux当中是采用链表的方式将它们组织起来，称为inode表。除此以外，系统为了高效管理文件，需要一个额外的数据结构来管理文件，称为文件表。文件表里面存放了文件的状态标志（典型的比如引用计数）、偏移量以及inode表的指针。文件表和inode表是在进程地址空间里面，是存放内核区域中的，并且这些内容是所有进程共享的（所以多进程同时写入会有问题）
- 而我们所使用的文件描述符与内核区中的另一个数据结构文件指针表有关，文件指针表的索引就是描述符，而数组的内容就是文件表项的指针。当执行dup函数以后，在文件指针表当中，会有两个不同的描述符来描述同一个文件，而在文件表当中，该文件的引用计数会自增1。当关闭文件时，文件指针表会移除该文件相关的项，并且文件表中该文件的计数会自减1，当引用计数为0的时候，文件表以及inode表的表项会被释放
- 使用dup函数可以实现输出的重定向

```c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd = open(argv[1],O_RDWR);
    ERROR_CHECK(fd,-1,"open");
    printf("fd=%d\n",fd);


    //关闭之前一般要求刷新一下
    printf("\n");
    //关闭标准输出
    close(STDOUT_FILENO);
    int fd1 = dup(fd);//dup复制的位置是，没有使用的文件描述符的最小编号，所以这里fd1就变成了1
    printf("fd1=%d\n",fd1);
    close(fd);
    printf("the out of stdout\n");
    return 0;
}
```

- 该程序首先打开了一个文件，返回一个文件描述符，因为默认的就打开了0，1，2表示标准输入、标准输出、标准错误输出。而且close(STDOUT_FILENO) 则表示关闭标准输出，此时文件描述符1就空着，然后dup(fd)则会复制一个文件描述符到当前未打开的最小描述符，此时关闭fd自身，然后在用标准输出的时候，发现标准输出重定向到你指定的文件了。那么printf所输出的内容也就直接输出到文件（因为printf的原理就是将内容输出入到描述符为1的文件里面）
- dup2(int oldfd,int newfd)也是进行描述符的复制，只不过采用此种复制，新的描述符由用户用参数newfd显式指定。对于dup2，如果newfd已经指向一个已经打开的文件，内核会首先关闭掉newfd所指向的原来的文件。此时再针对于newfd文件描述符操作的文件，则采用的是oldfd的文件描述符。如果成功dup2的返回值与newfd相同，否则为-1

```c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd = open(argv[1],O_RDWR);
    ERROR_CHECK(fd,-1,"open");
    printf("fd=%d\n",fd);

    printf("\n");
    int fd1 = dup2(fd,STDOUT_FILENO);//dup2第二个参数是指要复制到什么位置，不用先close
    printf("fd1=%d\n",fd1);
    close(fd);
    printf("the out of stdout\n");
    return 0;
}
```





## 2.文件描述符和文件指针

- fopen函数实际在运行的过程中也获取了文件描述符。使用fileno函数可以得到文件指针的文件描述符。当使用fopen获取文件指针以后，依然是可以使用文件描述符来执行IO，例如：

```c
//fileno.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    FILE *fp= fopen(argv[1],"rb+");
    ERROR_CHECK(fp,NULL,"fopen");
    int fd = fileno(fp);
    printf("fd=%d\n");
    char buf[128]={0};
    read(fd,buf,5);
    printf("buf=%s\n",buf);
    return 0;
}
```

- fopen的原理：fopen函数在执行的时候，会先调用open函数，打开文件并且获取文件对象的信息（通过文件描述符可以获取文件对象的具体信息），然后fopen函数会在用户态空间申请一片空间作为缓冲区，不仅在内核态有缓冲区，保护磁盘，高频读取，用fopen 尽可能在用户态读取
- fopen的优势：因为read和write是系统调用，需要频繁地切换用户态和内核态，所以比较耗时。借助用户态缓冲区，可以减少read和write的次数，使用fdopen函数可以根据文件描述符生成用户态缓冲区

```c
//fdopen.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd = open(argv[1],O_RDWR);
    ERROR_CHECK(fd,-1,"open");
    FILE *fp = fdopen(fd,"rb+");
    ERROR_CHECK(fp,NULL,"fdopen");
    char buf[128]={0};
    fread(buf,1,sizeof(buf),fp);
    printf("buf=%s\n",buf);
    return 0;
}
```



- 总结一下：通过fileno可以把FILE对象转成fd  通过fdopen 可以把fd转成FILE对象

- 如果需要高效地使用不带缓冲IO，为了和存储体系匹配，最好是一次读/写入一个块（通常是4K）大小的数据。另外如果需要使用内存映射，也应当使用open函数来打开文件



**使用场景**

> ​	每次如果读1个字节，频烦读写用fopen 如果  用open就尽量以一个块的大小来读写数据

- 如果获取了文件指针，就不要通过文件描述符的方式来关闭文件

```c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    FILE *fp= fopen(argv[1],"rb+");
    ERROR_CHECK(fp,NULL,"fopen");
    int fd = fileno(fp);
    close(fd);//如果使用fileno(fp) 那么close以后fd的数值不会发生改变
    printf("fd=%d\n",fd);
    char buf[128]={0};
    char *ret = fgets(buf,sizeof(buf),fp);
    ERROR_CHECK(ret,NULL,"fgets");
    printf("buf=%s\n",buf);
    return 0;
}
//出现报错  fgets: Bad file descriptor
```





## 3.标准输入输出文件描述符

- 与标准的输入输出流对应，在更底层的实现是用标准输入、标准输出、标准错误输出文件描述符表示的。它们分别用STDIN_FILENO、STDOUT_FILENO和STDERR_FILENO三个宏表示，值分别是0、1、2三个整型数字



## 4.管道

- （有名）管道文件是用来数据通信的一种文件，经是半双工通信，它在ls-l 命令中显示为p，它不能存储数据

| 传输方式 | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| 全双工   | 双方可以同时向另一方发送数据                                 |
| 半双工   | 某个时刻只能有一方向另一方发送数据，其他时刻的传输方向可以相反 |
| 单工     | 永远只能由一方向另一方发送数据                               |

```shell
$mkfifo[管道名字]
使用cat打开管道可以打开管道的读端
$cat[管道名字]
打开另一个终端，向管道当中输入内容可以实现写入内容
$echo "string" > [管道名字]
此时读端也会显示内容
```

- 当然也可以使用C程序来分别实现读端和写端

```c
//readpipe.c
#include <func.h>

int main(int argc,char* argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_RDONLY);
    ERROR_CHECK(fd,-1,"open");
    printf("I am reader\n");
    //读端读
    char buf[128] = {0};
    read(fd,buf,sizeof(buf));//管道没数据，就会阻塞
    printf("read=%s\n",buf);
    close(fd);
    return 0;
}
```

./readpipe 1.pipe

```c
//writepipe.c
#include <func.h>

int main(int argc,char* argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_WRONLY);
    ERROR_CHECK(fd,-1,"open");
    printf("I am writter\n");
    write(fd,"hello",5);//写是不阻塞的
    close(fd);
    return 0;
}
```

./writepipe 1.pipe



> 有一个问题，如果写端while循环，读端阻塞，这时关闭写端，读端read会收到什么，还是阻塞的么？

```c
//readpipe.c
#include <func.h>

int main(int argc,char* argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_RDONLY);
    ERROR_CHECK(fd,-1,"open");
    printf("I am reader\n");
    //读端读
    char buf[128] = {0};
    int ret = read(fd,buf,sizeof(buf));//管道没数据，就会阻塞
    printf("ret = %d,read=%s\n",ret,buf); //这里打印read的返回值
    close(fd);
    return 0;
}
```

写端阻塞，然后通过ctrl + c 关闭写端

```c
//writepipenowrite.c
#include <func.h>

int main(int argc,char* argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_WRONLY);
    ERROR_CHECK(fd,-1,"open");
    //printf("I am writter\n");
    //write(fd,"hello",5);//写是不阻塞的
    while(1); //不能close close对端也会close
    close(fd);
    return 0;
}
```



- 写端关闭，读端不会卡住，会得到零，就知道对方关闭





