---
layout: post
title:  "Binder从底层到应用完全解析"
description: Binder是什么，在Android中角色
categories: [blog, Android基础]
---

# Binder Overview

Binder用于完成Android<b>进程</b>之间的<b>通信</b>（IPC）。  

> 进程间通信（IPC，InterProcess Communication）是指在不同进程之间传播或交换信息。


#Binder用途：
- 进程间通信(AIDL,Messenger)
- 与本地Service通信

<br>
# Binder实现IPC基本原理


<b>Linux mmap</b>

> mmap将一个文件或者其它对象映射进内存。

如果通过mmap读取一个文件那么它的工作流程如下：

1、 映射文件对象F到进程内存地址。

2、 进程A发起对映射地址区间发起读写操作,通过查询页表发现数据并没有在物理页上（内存分页）。因为目前只建立了地址映射，真正的硬盘数据还没有拷贝到内存中，引发缺页异常。

3、 缺页异常引发内核去装载磁盘数据到进程主存中。拷贝完成。

![index page](/assets/images/mmap_work.png)

可以看到mmap之后进程内存中存会有被映射区的指针。

C函数：

```c
void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize)
```

案例:

使用mmap完成文件拷贝

```c
#include <stdio.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>


int main()
{
   struct stat s;
   const char * file_name = "/Users/tangjie/Downloads/test.txt";

   int fd = open(file_name,O_RDONLY);
   
   //获取文件状态信息保存到s
   int status = fstat(fd,&s);
   
   //获取文件大小
   int size = s.st_size;

   //映射地址到进程的地址空间
   char *mapped = mmap(0,size,PROT_READ,MAP_PRIVATE,fd,0);

   printf("result = %s",mapped);
   close(fd);

   munmap(0,size);

   return 0;
}

```

mmap较之于传统IO有两个特点：

<ul>
    <li>比传统IO少一次内存拷贝，访问数据时发生缺页异常后从磁盘加载数据到用户进程完成拷贝。传统IO需要需要从磁盘->内核空间->用户空间。主要原因是操作系统为了权限和安全性考虑，把<b>部分指令封装到内核且内核空间只能供内核指令访问</b>。用户空间需要从内核空间访问数据。</li>
    <li><b>可以实现父子进程/用户进程之间的内存共享</b></li>
</ul>




