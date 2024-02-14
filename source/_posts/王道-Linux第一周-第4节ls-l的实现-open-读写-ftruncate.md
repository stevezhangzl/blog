---
title: 王道-Linux第一周-第4节ls-l的实现-open-读写-ftruncate
date: 2023-06-30 14:45:25
tags:
---

[toc]



## 模拟ls -l命令

```c
//ls.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    DIR *dir;

    dir = opendir(argv[1]);
    if(NULL == dir){
        perror("opendir");
        return -1;
    }
    struct dirent *p;
    struct stat buf;
    int ret;
    while((p = readdir(dir))){
        ret = stat(p->d_name,&buf);
        printf("mode=%x %3ld %s %s %6ld %s %s\n",\
           buf.st_mode,buf.st_nlink,getpwuid(buf.st_uid)->pw_name,getgrgid(buf.st_gid)->gr_name,buf.st_size,ctime(&buf.st_mtime),p->d_name);
    }
    closedir(dir);
    return 0;
}
```

注意ctime函数的使用

```c
char *ctime(const time_t *timep);//把time_t 长整型转成string
```

- 注意getpwuid和getgrgid需要包含关文件pwd.h和grp.h 这两个函数是把uid和gid转成对应的结构体，结体体里有name这种字符串信息，可以输出对应的用户名和所属组的string name



- ls / 根目录里的dirent是超级块  dirent是有备份的   
- mv的过程本质就是在另一个目录新建了一个硬连接，把当前目录的硬连接删除的过程



## 基于文件描述符的文件操作

### 文件描述符



- 上一节所讨论的文件都是使用FILE类型来管理，根据FILE类型的结构可知它的本质是一个缓冲区。与FILE类型相关的文件操作（比如fopen，fread等）称为带缓冲的IO，它们是ISO C的组成部分
- POSIX 标准支持另一类不带缓冲区的IO。在这里，文件是使用文件描述符来描述。文件描述符是一个非0的整数。从原理上来说，每次打开文件的时候，进程地址空间的内核部分里面会维护一个已经打开的文件的数组，文件描述符的作用就是这个数组的索引。因此，文件描述符可以实现进程和打开文件之间的交互

### 打开、创建和关闭文件

使用open函数可以打开或创建并打开一个文件，使用creat函数可以创建一个文件

```c
#include <sys/types.h>//头文件
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname,int flags);//文件名 打开方式
int open(const char *pathname,int flags,mode_t mode);//文件名 打开方式 权限  fopen创建的文件的权限只能是664 但这个可以指定权限  注意这里是八进制0666
int creat(const char *pathname,mode_t mode);//文件名 权限
//creat 现在已经不常用了，它等价于
open(pathname,O_CREAT|O_TRUNC|O_WRONLY,mode);
```

- 执行成功时，open函数返回一个文件描述符，表示已经打开的文件；执行失败是，open函数返回-1，并设置相应的errno
-  flags和mode都是一组掩码的合成值，flags表示打开或创建的方式，mode表示文件的访问权限
- flags的呆选项有

| 掩码       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| O_RDONLY   | 以**只读**的方式打开                                         |
| O_WRONLY   | 以**只写**的方式打开                                         |
| O_RDWR     | 以**读写**的方式打开                                         |
| O_CREAT    | 如果文件不存在，则创建文件                                   |
| O_EXCL     | 仅与O_CREAT连用，如果文件已存在，则open失败                  |
| O_TRUNC    | 如果文件存在，将文件的长度截至0                              |
| O_APPEND   | 已追加的方式打开文件，每次调用write时，文件指针自动先移到文件尾 |
| O_NONBLOCK | 非阻塞方式打开，无论有无数据读取或等待，都会立即返回进程之中 |
| O_NODELAY  | 非阻塞方式打开                                               |
| O_SYNC     | 同步打开文件，只有在数据被真正写入物理设备后才返回           |

- 使用完文件以后，要记得使用close来关闭文件。一旦调用close，则该进程对文件所加的锁全都被释放，并且使文件的打开引用计数减1，只有文件的打开引用计数为0以后，文件才会被真正的关闭。用ulimit -a 命令可以查看单个进程能同时打开的文件的上限。




### 读写文件

- 使用read和write来读写文件，它们统称为不带缓冲的IO

```c
#include <unistd.h>
ssize_t read(int fd,void *buf,size_t count);//文件描述符 缓冲区 长度
ssize_t write(int fd,void *buf,size_t count);
```

- 对于read和write函数，出错返回-1 读取完了之后，返回0，其他情况返回读写的字节数

 



### 改变文件大小

- 使用ftruncate函数可以文件大小

```c
#include <unistd.h>
int ftruncate(int fd,off_t length);
```

- 函数ftruncate会将参数fd指定的文件大小改为参数length指定的大小。参数fd为已打开的文件描述符，而且必须是以**写入模式**打开的文件。如果原来的文件大小比参数length大，则超过的部分会被删去（实际上修改了文件的inode信息）。执行成功则返回0，失败返回-1。

```c
//ftruncate.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd = open(argv[1],O_WRONLY);
    ERROR_CHECK(fd,-1,"open");
    printf("fd=%d\n",fd);
    int ret=ftruncate(fd,8);
    ERROR_CHECK(ret,-1,"ftruncate");
    return 0;
}
```

- 使用mmap函数经常配合函数ftruncate来扩大文件大小，使用mmap文件映射的效率比read和write的效率要高上三倍，利用了DMA技术
- 零copy技术：read时，磁盘数据到内核缓冲区，内核缓冲区到我们的buf，数据经过了两次转移

![image-20230706083152751](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230706083152751.png)

这个是正常的read和write的时候，如果是mmap的情况

![image-20230706083341803](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230706083341803.png)

> 所谓零copy是指在内核态和用户态一次copy都没有发生

- mmap 执行时，cpu知道把磁盘什么地址，映射到内存什么地址，cpu通知DMA把磁盘某几个block搬到内存的某个位置，然后直接进行磁盘的读写，避免了频繁的内核态和用户态的切换，cpu就空闲下来了，可以更多的去做别的事。mmap有一个缺点，就是不能改变文件的大小，而且一开始必须有一个确定的文件大小，这个文件大小不能是0，如果文件大小是0用mmap是没有意义的，因为文件大小是0，DMA就不知道把磁盘的哪一块映射到用户诚的堆空间上，所以一般结合ftruncate使用，先扩展一个大小的的文件，全存0就行

```c
#include <sys/mman.h>
void *mmap(void *adr,size_t len,int prot,int flag,int fd,off_t off);
```

- addr 参数用于指定映射存储区的地址。这里设置为NULL，这样就由系统自动分配（通常是在堆空间里面寻找）。
- fd参数是一个文件描述符，使用时必须已经打开。
- prot 参数用来表示权限。PROT_READ,PROT_WRITE表示可读可写。
- flag 参数在目前是采用MAP_SHARED，后面讲进程通信的时候会介绍其他类型
- 为什么mmap需要和ftruncate联合使用？因为分配的缓冲区的大小和偏移量的大小是有限制的，它必须是虚拟内存页的大小的整数倍。如果文件大小比较小，那么超过文件大小返回的缓冲区操作将不会修改文件；如果文件大小为0，还会出现Bus error
- 那么到底为什么使用ftruncate来写一个很大的文件？因为用ftruncate写一个很大的文件比如512M 这个时间非常短，所以一般使用ftruncate和mmap结合使用。批量的block刷成0 是很快的



### 文件定位

- 函数lseek将文件指针设定到相对于whence，偏移值为offset的位置。它的返回值是读写点距离文件开始的距离。

```c
#include <sys/types.h>
#include <unistd.h>
//和fseek非常像
off_t lseek(int fd,off_t offset,int whence);//fd 文件描述符
//whence 可以是下面三个常量的一个
//SEEK_SET 从文件头开始计算
//SEEK_CUR 从当前指针开始计算
//SEEK_END 从文件尾开始计算

#include <func.h>
//使用lseek定位光标，这里实现一个功能，从文件开头开始算起，偏移5个字节，写入xiongda 7个字节
int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_RDWR);
    ERROR_CHECK(fd,-1,"open");
    off_t ret;
    ret = lseek(fd,5,SEEK_SET);//相对于开头偏移5个字节
    ERROR_CHECK(ret,-1,"lseek");
    write(fd,"xiongda",7);
    close(fd);
    return 0;
}
```



#### 文件空洞

文件只有1个字节， 我们使用lseek偏移到1024个字节处写东西，然后文件变成1025  其他字节全是0  这个也是用来和mmap结合来使用的，和ftruncate功能类似

```c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int fd;
    fd=open(argv[1],O_RDWR);
    ERROR_CHECK(fd,-1,"open");
    off_t ret;
    ret = lseek(fd,1024,SEEK_SET);//相对于开头偏移5个字节
    ERROR_CHECK(ret,-1,"lseek");
    write(fd,"1",1);
    close(fd);
    return 0;
}
```

通地ls -l 看到文件大小变成了1025

