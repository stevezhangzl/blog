---
title: 王道-Linux第一周-第3节硬链接原理解析-seekdir和rewinddir使用
date: 2023-06-28 13:13:24
tags:
---



[toc]

## 1.硬连接和软连接



先看一下现象

1.我们echo hello > file里，创建一个file

2.创建file的**软连接** file1  ` ln -s file file1`

![image-20230628135921321](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628135921321.png)

引用计数没有变，还是1，这个就相当于我们使用的快捷方式，file1中存储的是file的路径，我们把file删除掉再cat file1

![image-20230628140045598](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628140045598.png)



3.然后我们创建硬连接file1 `ln file file1`

![image-20230628140137834](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628140137834.png)

发现引用计数变成了2 每有一个硬连接，引用计数就会加1

我们使用ls -il这个命令看下

![image-20230628140230909](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230628140230909.png)

这两个文件的d_ino都是一个，也就是这个目录的两个dirent结构体，但他们的d_ino指向的是同一个文件内容，所以修改哪个都会改变最后文件的内容，只是两个dirent结构，一个叫file一个叫file1



这也引出了另一个问题，就是为什么我们删除文件很快，但copy文件很慢，因为删除文件的时候，只是删除了这个dirent结构体，但dirent结构体指向的d_ino的文件并没有去删除，所以很快



那什么时候，这个block所在的区域可以去再利用回收写新的内容呢？ 就是引用计数变成0后，这个block就会被回收，这个区域会被覆盖写新的内容，并把新的d_ino指给dirent 。 使用的是lazy模式，这样会对磁盘有好处。也更快。



1. 磁盘上每一份内容都有唯一的inode编号，不管是文件还是文件夹
2. 硬连接数是引用计数的一种，c和java内存管理的不同，c用的是引用计数ARC java用的是垃圾回收GC
3. 如何判断能不能去写某个inode值对应的磁盘，这个inode的引用计数为零



ls -ali

![image-20230629131556828](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230629131556828.png)

> 这里的. 和.. 硬连接数为什么是2和8？
>
> 这里的. 为什么是2 因为有上级目录的day18是一个  有当前目录一个

![image-20230629131928988](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230629131928988.png)

> d_ino都是2000310
>
> 而..为什么是8
>
> 因为 自己的. 是一个   里面每个目录的..都是一个 然后最外层的的目录是一个    一个是8个  也就是说直接可以猜出来wangdao目录下有6个文件夹



硬连接相当于一个入口，会找到d_ino 和d_ino的一个关系 dirent

copy以后 d_ino就不一样了 因为是一份新的block 分配新的block了

![image-20230630103945708](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230630103945708.png)



> 这个偏移d_off是什么？
>
> 这个偏移是磁盘地址，这个地址是没有虚拟地址一说的，都是物理地址，内核去操作，我们不用关注，一般不用用户去操作



- seekdir()函数用来设置目录流目前的读取位置，再调用readdir()函数时，便可以从此新位置开始读取。参数offset代表距离目录文件开头的偏移量

- 使用readdir()时，如果已经读取到目录末尾，又想重新开始读，则可以使用rewindir函数将文件重新定位到目录文件的起始位置。

- telldir()函数用来返回目录流当前的读取位置 

  

## 2.文件状态信息获取

dirent信息  文件内容

文件属性信息	描述inode信息  通过stat接口可以获取

- 使用stat结构体（又称为inode信息）可以获取文件信息

```c
//结构体
struct stat {
   dev_t     st_dev;         /* 如果是设备，返回设备描述符，否则为0 */
   ino_t     st_ino;         /* i节点号 这个就是inode */
   mode_t    st_mode;        /* 文件类型 */
   nlink_t   st_nlink;       /* 链接数 */
   uid_t     st_uid;         /* 属性ID */
   gid_t     st_gid;         /* 组ID */
   dev_t     st_rdev;        /* 设备类型 */
   off_t     st_size;        /* 文件大小，字节表示 */
   blksize_t st_blksize;     /* 块大小 默认大小512 */
   blkcnt_t  st_blocks;      /* 块数 */

   /* Since Linux 2.6, the kernel supports nanosecond
      precision for the following timestamp fields.
      For the details before Linux 2.6, see NOTES. */

   struct timespec st_atim;  /* 最后访问时间 */
   struct timespec st_mtim;  /* 最后修改时间 */
   struct timespec st_ctim;  /* 最后权限修改时间 */

#define st_atime st_atim.tv_sec      /* Backward compatibility */
#define st_mtime st_mtim.tv_sec
#define st_ctime st_ctim.tv_sec
};
```

注意ctime使用

```c
char *ctime(const time_t *timep);
```

