---
layout: post
title:  "Binder通信之mmap原理"
description: 学习内存映射技术
categories: [blog, Android]
---

# Binder 概览

Binder用于完成Android<b>进程</b>之间的<b>通信</b>（IPC）。  

> 进程间通信（IPC，InterProcess Communication）是指在不同进程之间传播或交换信息。


#Binder用途：
- 进程间通信(AIDL,Messenger)
- 与本地Service通信

<br><br><br>

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

案例:使用mmap完成文件拷贝。
```c

int main()
{
   struct stat s;
   const char * file_name = "~/Downloads/test.txt";

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

>binder正是借助用户进程之间的内存共享完成的跨进程通信。

案例：mmap实现跨进程内存共享案例

文件ipc_mmap_a.c

```c

int main()
{
    struct stat s;
    const char * file_name = "~/Downloads/test.txt";

    int fd = open(file_name,O_RDWR);
   
    //获取文件状态信息保存到s
    int status = fstat(fd,&s);
   
    //获取文件大小
    int size = s.st_size;

    //映射地址到进程的地址空间
    char *mapped = mmap(0,size,PROT_READ,MAP_SHARED,fd,0);

    close(fd);

    /* 每隔两秒查看存储映射区是否被修改 */  
    while (1) {  
        printf("%s\n", mapped);  
        sleep(2);  
    }  
  
   return 0;
}
```

terminal 命令编译&运行 >

```
 $ gcc -o ipc_mmap_a ipc_mmap_a.c
 $ ./ipc_mmap_a 
```

输出结果：

```
test content

test content
.. 
```

这里两秒输出一次test.text中的内容。仔细的同学注意到这里的map模式区别于之前的拷贝案例改成了MAP_SHARED，接下来我们在再开启一个terminal，相当于另外一个进程。编译执行ipc_mmap_b.c的内容。

文件：ipc_mmap_b.c

```c
int main()
{
    struct stat s;
    const char * file_name = "/Users/tangjie/Downloads/test.txt";

    int fd = open(file_name,O_RDWR);
   
    //获取文件状态信息保存到s
    int status = fstat(fd,&s);
   
    //获取文件大小
    int size = s.st_size;

    //映射地址到进程的地址空间
    char *mapped = mmap(0,size,PROT_READ | PROT_WRITE,MAP_SHARED,fd,0);

    close(fd);

    mapped[1] = '#'; 

    printf("%s\n", mapped);   
    return 0;
}
```
这里可以看到我们看到新的进程模块也读取了同一块区域的内容，并直接通过映射地址修改了内容，即'e'改成'#'。编译运行后。

查看先前打开的a terminal的打印输出。

```
test content

t#st content

t#st content
```

再B进程修改了共享的映射区域后，A进程读到的内容从test content也变成了t#st content。继而完成了<b>共享的通信</b>。

<b>Android Binder驱动部分解析</b>

同样对于Android Binder，在打开设备描述文件之后，Binder驱动会通过mmap在内核缓存开辟物理分页，通信内容保存在此区域，内核空间和用户空间映射到同存有内核缓存区域的映射地址，把用户空间的地址交给进程就能直接获取到内核缓存区域内容。

```c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{

	int ret;
	struct vm_struct *area;
	struct binder_proc *proc = filp->private_data;
	const char *failure_string;
	struct binder_buffer *buffer;

	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;

	if (binder_debug_mask & BINDER_DEBUG_OPEN_CLOSE)
		printk(KERN_INFO
			   "binder_mmap: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
			   proc->pid, vma->vm_start, vma->vm_end,
			   (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
			   (unsigned long)pgprot_val(vma->vm_page_prot));

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS)
	{
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

	if (proc->buffer)
	{
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}

	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
	if (area == NULL)
	{
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
	proc->buffer = area->addr;
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;

    ...

	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
	if (proc->pages == NULL)
	{
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}
	proc->buffer_size = vma->vm_end - vma->vm_start;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;

	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma))
	{
		ret = -ENOMEM;
		failure_string = "alloc small buf";
		goto err_alloc_small_buf_failed;
	}
	buffer = proc->buffer;
	INIT_LIST_HEAD(&proc->buffers);
	list_add(&buffer->entry, &proc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(proc, buffer);
	proc->free_async_space = proc->buffer_size / 2;
	barrier();
	proc->files = get_files_struct(current);
	proc->vma = vma;

	/*printk(KERN_INFO "binder_mmap: %d %lx-%lx maps %p\n", proc->pid, vma->vm_start, vma->vm_end, proc->buffer);*/
	return 0;
}
```


