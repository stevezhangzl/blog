---
title: DNS协议详解
date: 2023-05-07 19:35:15
tags:
---



是UDP协议

为什么使用UDP协议而不是TCP协议，因为UDP就两个包，一去一回，TCP比较麻烦

DNS的包结构如下：

![image-20230507193749571](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230507193749571.png)





## Header

### 1.Transaction ID

请求与响应配对

会话ID：我们说一去一回嘛，通过这个Transcation ID判断发的包是否匹配

### 2.Flags

![image-20230507194216137](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230507194216137.png)

- QR：通过这1位，就能知道这个UDP包是查询包还是响应包



### 3.请求数据

Questions

### 4.应签数据

- Answer RRS
- Authority RRS
- Additional RRS

![image-20230507194648973](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230507194648973.png)



## Body

### 1.Queries

- Name：域名
- Type：类型
- Class：查询类

![image-20230507194945706](/Users/zhanglong/Library/Application Support/typora-user-images/image-20230507194945706.png)

