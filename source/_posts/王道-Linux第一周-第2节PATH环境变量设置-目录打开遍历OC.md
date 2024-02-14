---
title: 王道-Linux第一周-第2节PATH环境变量设置-目录打开遍历OC
date: 2023-06-26 13:59:27
tags:
---



[toc]



## 1.如何修改ubuntu的PATH环境变量

在~/.bashrc中增加如下内容

PATH=$PATH:~/.local/bin



- 查看和修改环境变量

```
查看环境变量
echo $PATH
修改环境（系统路径）变量（只对本次生效）
export PATH=$PATH:新目录
```

## 2.目录

### 2.1 目录的存储原理



- 对于任何文件，为了快速定位文件在磁盘当中的位置 ，文件系统在设计的时候就需要利用专门的索引结构来管理所有的文件。索引结构的基本单位是索引结点，每个索引结点都具有固定的大小，里面存放了单个文件的位置、文件类型、权限、修改时间等等信息。文件系统将所有索引结点用数组的形式组织起来，并利用一个辅助位图来实现高效管理文件信息。
- 在Linux当中，目录是一种特别的文件，它的总体大小固定。它的数据块当中把很多文件的文件名和索引结点存放在一起。因为文件的文件名大小不一，为了避免磁盘碎片和支持频繁增加修改，所以目录采用链式结构来存储来组织各种文件的信息。链式结构的结点就是direct结点，它的定义如下。可以看出，要访问下个dirent结点，实际是依赖于本结点中的d_off属性。

```c
struct dirent{
	ino_t d_ino; //该文件的inode
  off_t d_off; //到下一个dirent的偏移
  unsigned short d_reclen; //文件名的长度
  unsigned char d_type; //所指的文件类型
  char d_name[256]; //文件名
};
```



### 2.2 目录相关操作

```c
#include <sys/types/h>
#include <dirent.h>

DIR *opendir(const char *name); //打开一个目录
struct dirent *readdir(DIR *dir);//读取目录的一项信息，并返回该项信息的结构体指针
void rewinddir(DIR *dir);//重新定位到目录文件的头部
void seekdir(DIR *dir,off_t offset);//用来设置目录流目前的读取位置
off_t telldir(DIR *dir);//返回目录流当前的读取位置
int closedir(DIR *dir);//关闭目录文件

#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
int stat(const char *pathname,struct stat *buf);//获取文件状态
```

- 读取目录信息的步骤为：

1. 用opendir函数打开目录，获得DIR指针。DIR称为目录流，类似于标准输入输出，每次使用readdir以后，它会将位置移动到下一个文件

2. 使用readdir函数迭代读取目录的内容

3. 用closedir函数关闭目录
   1. dirent当中采用了类似于变长数组的形式来存放文件名，但是会提供一些冗余的空间，这样当调整文件名的时候，如果新文件名的长度不超过原来分配空间，则不需要调整分配的空间。
   2. 类似于地址和内存的关系，inode(索引结点)描述了文件在磁盘上的具体位置信息。在ls命令中加上参数-i可查看文件inode信息。那么所谓的硬连接，就是指inode相同的文件。一个inode的结点上的硬连接的个数就称为引用计数。
   
   ```shell
   ls -ial
   查看所有文件的inode信息
   ln 当前文件 目标
   建立名为“目标”的硬连接
   ```
   
   1. 这里可以检查. 文件引用计数，发现是2，这是因为一个目录可以通过本目录或者是父目录来访问它本身，若它还有子目录，它的引用计数也会增加
   2. 删除磁盘上文件的时候，只有引用计数为0的时候，才会将磁盘内容移除文件系统（断开和目录的链接）
   3. 当然，为了避免引用死锁，一般用户是不能使用ln命令来为目录建立硬连接。
   


```c
//opendir.c
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
    while((p=readdir(dir))){
        printf("ino=%ld,len=%d,type=%d,name=%s\n",p->d_ino,p->d_reclen,p->d_type,p->d_name);
    }
    closedir(dir);
    return 0;
}
```

打印出dirent的ino 文件名的长度 文件的类型 文件名

![image-20230628075518383](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628075518383.png)

发现多了. 和.. 因为.和..也是文件  然后type 8 是文件  4 是文件夹  len是固定大小，不动态使用malloc realloc调整，空间换时间，冗余设计。

然后我们怎么对比ls出来的  要加上-i    ls -ali

![image-20230628075647345](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628075647345.png)





目录的深度优先遍历，使递归的方式

```c
#include <func.h>
//width代表空格数
int print_dir(char *path,int width){ //加这个width就为了打印目录中的东西的时候加一些空格
    DIR *dir;
    dir = opendir(path);
    ERROR_CHECK(dir,NULL,"opendir")
    struct dirent *p;
    char buf[1024];
    while((p=readdir(dir))){
        if(!strcmp(".",p->d_name)||!strcmp("..",p->d_name)){
            //如果是.和..什么都不做
        }else{
            //代表有  %*s 代表有width个 “”空格
            printf("%*s%s\n",width,"",p->d_name); 
            //判断他是目录，继续把它传给print_dir
            if(p->d_type==4){
                memset(buf,0,sizeof(buf));
                sprintf(buf,"%s%s%s",path,"/",p->d_name);
                print_dir(p->d_name,width+4);
            }

        }
    }
}

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2)
    print_dir(argv[1],0);
    return 0;
}
```

> printf("%*s%s\n",width,"",p->d_name);  注意这里，%\*s  可以打印  withd个数的 “”字符



但是上面程序有bug，在dir2里创建dir3再创建file2 执行./finddir . 就会报错

![image-20230628130859910](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628130859910.png)

分析程序是在打印了dir3后报错的，为什么会报错，每次传入printf_dir的是当前的文件名，在当前./目录下，去打开file2 肯定是找不到这个文件的，所以需要把path和文件名拼在一起，一起传入printf_dir 函数

```c
//判断他是目录，继续把它传给print_dir
if(p->d_type==4){
    memset(buf,0,sizeof(buf));
    sprintf(buf,"%s%s%s",path,"/",p->d_name);
    print_dir(buf,width+4); //这里上面把 path / d_name拼在一起后，给到buf 然后，我们把buf传入递归函数
}
```













