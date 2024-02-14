---
title: 王道-Linux第一周-第1节sprintf-chmod-getcwd-chidr
date: 2023-06-19 04:13:52
tags:
---



[TOC]

## 1.文件读写

### 格式化读写

```c
#include <stdio.h>
int printf(const char* format,...);
//相当于fprintf(stdout,format,...);
int scanf(const char *format,...);
int fprintf(FILE *stream,const char* format,...);
int fscanf(FILE *stream,const char* format,...);
int sprintf(char *str,const char *format,...);
//eg:sprintf(buf,"the string is;%S",str);
int sscanf(char *str,const char* format,...); //把字符串的转成各种类型的变量
```



- fprintf 将格式化后的字符串写入到文件流stream中
- sprintf将格式化后的字符串写入到字符串str中



```c
//sprintf.c
#include <stdio.h>

typedef struct{
    int num;
    float score;
    char name[20];
}stu;

int main()
{
    stu s = {1001,92,"lele"};
  	//char buf[]初始化，记住！
    char buf[1024] = {0};
    sprintf(buf,"%d %5.2f %s",s.num,s.score,s.name);
    puts(buf);
    return 0;
}
```



```c
//sscanf.c
#include <stdio.h>
typedef struct{
    int num;
    float score;
    char name[20];
}stu;

int main()
{
  	//对结构体进行初始化，记住！
    stu s={0};
    char buf[1024] = "1001 92.30 lele";
  	//sscanf和scanf会忽略空格，记住要使用地址
    sscanf(buf,"%d%f%s",&s.num,&s.score,s.name);
    printf("%d %5.2f %s\n",s.num,s.score,s.name);
    return 0;
}
```



### 单个字符读写

- 使用下列函数可以一次读写一个字符

  ```c
  #include <stdio.h>
  int fgetc(FILE* stream);
  int fputc(int c,FILE* stream);
  int getc(FILE* stream);//等同于 fgetc(FILE* stream)
  int putc(int c,FILE* stream);//等同于fputc(int c,FILE *stream);
  int getchar(void);//等同于fgetc(stdin);
  int putchar(int c);//等同于fputc(int c,stdout);
  ```

- getchar和putchar从标准输入输出流中读写数据，其他函数从文件流stream中读写数据

### 字符串读写

```c
char *fgets(char *s,int size,FILE *stream);
int fputs(const char *s,FILE *stream);
int puts(const char *s);//等同于fputs(const char *s,stdout);
char *gets(char *s);//等同于fgets(const char *s,int size,stdin);
```



- fgets和fputs从文件流stream中读写一行数据；
- puts和gets从标准输入输出流中读写一行数据；
- fgets可以指定目标缓冲区的大小，所以相对于gets安全，但是fgets调用时，如果文件中当前行的字符个数大于size，则下一次fget调用时，将继续读取该行剩下的的字符，fgets读取一行字符时，保留字符的换行符。
- fputs不会在行尾自动添加换行符，但是puts会在标准输出流中自动添加一换行符。



## 2.文件定位

- 文件定位指读取或设置文件当前读写点，所有的通过文件指针读写数据的函数，都是从文件的当前读写点读写数据的。常用的函数有：

```c
#include <stdio.h>
int feof(FILE *stream);
//判断是否读到文件结尾，通常的用法为while(!feof(fp)),没什么太多用处，不建议使用
int fseek(FILE *stream,long offset,int whence);
//设置当前读写点到偏移whence 长度为offset处
long ftell(FILE *stream);
//用来获得文件流当前的读写位置
void rewind(FILE *stream);
//把文件流的读写位置移至文件开头 相当于fseek(fp,0,SEEK_SF);
```

- fseek设置当前读写点到偏移whence 长度为offset处，whence可以是：SEEK_SET（文件开头）、SEEK_CUR(文件当前位置)、SEEK_END(文件末尾)
- ftell 获取当前的读写点
- rewind将文件当前读写点移动到文件头





我们来演示下feof

```c
//feof.c
#include <stdio.h>

int main()
{
    FILE *fp = fopen("file","rb+");
    char c;
    while(!feof(fp)){
        c = fgetc(fp);
        putchar(c);
    }
    return 0;
}
```

我们echo helloworld > file 写入一个file文件

gcc feof.c -o feof

./feof

执行后

![image-20230626082954550](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626082954550.png)

发现多打印了一个看不出来的字符

feof会通过报错来判断是否执行到文件结尾，但是如果已经到文件结尾的话，while循环会多读一次，不是提前告诉是结尾，然后不进入循环，是进入循环后才知道执行到了文件结尾

那怎么修改上面的程序呢，才能正确执行呢？

```c
//feof1.c
#include <stdio.h>

int main()
{
    FILE *fp = fopen("file","rb+");
    char c;
    while(!feof(fp)){
        c = fgetc(fp);
        if(-1==c){
            printf("%d\n",c);
        }else{
            putchar(c);
        }
    }
    return 0;
}
```

就是判断一下，不为-1的时候才打印，执行feof1

![image-20230626083638424](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626083638424.png)



## 3.文件的权限

```c
#include <sys/stat.h>
int chmod(const char* path,mode_t mode);
//mode 形如：0777 是一个八进制整型
//path 参数指定的文件被修改为具有mode参数给出的访问权限
```



```c
//chmod.c
#include <stdio.h>
#include <sys/stat.h>
int main(int argc,char *argv[])
{
    if(argc!=2){
        printf("error args\n");
        return -1;
    }

    int ret;
    //传一个8进制的mode 这样能看的懂
    ret = chmod(argv[1],0666);
    if(-1==ret){
        perror("chmod");
        return -1;
    }
    return 0;
}
```

> 这里使用了perror  那怎么判断哪些函数可以使用perror进行打印呢？
>
>  我们man 2 chmod    //**这里注意：1 是命令  2 是系统调用  3是库函数**
>
> /RETURN 看到返回值的注释
>
> 这种带errno is set appropriately的就是可以使用perror进行打印错误的函数。

![image-20230626092132496](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626092132496.png)

> 文件权限 三组  从左到右分别是 拥有者 - 用户组 - 其他人
>
> -rwx-rw-rw   按  421权限来判断，是8进制数

touch file

生成的file的权限是-rw-rw-r--  664   我们执行a.out 把文件权限变成666

执行./a.out file后

file权限变成

-rw-rw-rw

> 我们执行完成后./a.out file 怎么知道程序的返回值是什么，有没有错误码？
>
> echo $?

![image-20230626131458033](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626131458033.png)

> 如果我们只用./a.out 而不加文件呢？

![image-20230626131601824](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626131601824.png)

发现错误码是255



## 4.目录操作

### 4.1 获取、改变当前目录

```c
#include <unistd.h> //头文件
char *getcwd(char *buf,size_t size);//获取当前目录，相当于pwd命令
int chdir(const char *path);//修改当前目录，即切换目录，相当于cd命令
```

```c
//getcwd.c
#include <stdio.h>
#include <unistd.h>
int main()
{
    char buf[512] = {0};
    //获取当前路径，填入buf
    getcwd(buf,sizeof(buf));
    puts(buf);
    return 0;
}
```

执行./a.out

显示

![image-20230626132617924](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626132617924.png)

子进程的路径是由父进程cp过来的，这个a.out执行的父进程是谁呢？

父进程是bash 执行子进程./a.out 继承了父进程的路径bash的路径



```c
//chdir.c
#include <stdio.h>
#include <unistd.h>
int main()
{
    char buf[512] = {0};
    //获取当前路径，填入buf
    getcwd(buf,sizeof(buf));
    puts(buf);
    chdir("../");//改变路径到上级目录
    puts(getcwd(NULL,0));//这里不用申请空间，系统给你空间
    return 0;
}
```

注意这里getcwd第一个参数没有传入buf，而是传入NULL，系统分配的空间，这个后面再说

> 这里，你猜会不会改变当前命令行的目录？
>
> 答案是不会，历为改变的是子进程的当前目录，不是父进程的当前的目录，改变子进程的当前目录对父进程不会有影响。
>
> 那么为什么要在代码里把这个目录给改变呢？
>
> 比如在某个路径下进行日志的读写，如果不去这个路径下，每次都要打开文件都要传一个绝对路径才能打开文件，否则就打不开，这时改变路径会方便很多。

![image-20230626135231984](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626135231984.png)

- getcwd()函数将当前的工作目录绝对路径复制到参数buf所指的内存空间，参数size为buf的空间大小。因此在调用此函数时，buf所指的内存空间要足够大，若工作目录绝对路径的字符串长度超过参数size的大小，则返回值为NULL，errno的值则为**ERANGE**

- 倘若参数buf为NULL，getcwd()会依参数size的大小自动配置内辈子（使用malloc()），如果参数size也为0，则getcwd()会依工作目录绝对路径的字符串程度来决定所配置的内存大小，进程可以在使用完此字符串后自动利用free()来释放此空间。所以常用的形式：

  ```c
  getcwd(NULL,0);
  ```
  
- chdir()函数：用来将当前的工作目录改变成以参数path所指的目录：

```c
#include <stdio.h>
int main(){
  chdir("/tmp"); //不旦可以改变相对路径 ../ 也可以改变成绝对路径 一般都是相对路径
  printf("current working directory:%s\n",getcwd(NULL,0));
  return 0;
}
```

### 4.2 创建和删除目录

```c
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
int mkdir(const char *pathname,mode_t mode);//创建目录，mode是目录权限
int rmdir(const char *pathname); //删除目录
```



 

```c
//mkdir.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int ret;
    ret=mkdir(argv[1],0777);
    ERROR_CHECK(ret,"mkdir");
    return 0;
}
```

然后我们执行./mkdir dir1 创建的目录的权限 是775

![image-20230626185713894](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626185713894.png)



而不是777 这是因为系统调用 还要受到掩码的影响所以是775 而不是777  因为它的上一级目录dir_op的目录权限是775

![image-20230626190128647](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230626190128647.png)

```c
//rmdir.c
#include <func.h>

int main(int argc,char *argv[])
{
    ARGS_CHECK(argc,2);
    int ret;
    ret=rmdir(argv[1]);
    ERROR_CHECK(ret,"rmdir");
    return 0;
}
```







- 如果需要查看库函数mkdir而不是shell命令mkdir，使用man命令的需要指定系统函数帮助手册（mkdir(2)）
- 如果函数的（指针）参数使用const来进行修饰，这个参数就称为传入参数。这意味着在函数内部是不能通过指针来修改传入参数的内容的

```shell
man 2 mkdir
```

