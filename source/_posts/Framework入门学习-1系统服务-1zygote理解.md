---
title: Framework入门学习-1系统服务-1zygote理解
date: 2023-08-05 06:46:42
tags:
---



[TOC]



- 启动SystemServer
- 孵化应用进程



常用类、JNI函数、主题资源、共享库





## 1.启动三段式

![image-20230805065840476](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230805065840476.png)



## 2.Zygote的启动流程



- 进程是怎么启动的？
- 进程启动之后做了什么？



### 2.1 Zygote的进程是怎么启动的

![image-20230805070053266](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230805070053266.png)

init.rc 配置文件里配置启动哪些进程，其中就有Zygote进程



### 2.2 Zygote进程启动之后做了什么？

- Zygote的Native世界
- Zygote的Java世界



#### 2.2.1 Zygote的native世界

![image-20230805071737872](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230805071737872.png)
