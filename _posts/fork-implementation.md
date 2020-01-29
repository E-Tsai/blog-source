---
title: Linux中的fork()是怎么实现的
date: 2019-09-22 02:02:07
tags: [technics, systems, CN]
categories: CS
---
![](https://img.shields.io/badge/Lan-CN-orange)

Fork和它的变体是Unix系统创建进程 (process) 的方法，在这篇文章里我们会探索它是如何实现的。
<!--more-->

## Fork()简介

在多任务实时操作系统内，我们需要一个由已经存在的进程创建子进程的方法。Unix系统使用系统调用来创建子进程，这是fork以及它的变体的工作。

fork会为子进程创建一个单独的[地址空间](https://en.wikipedia.org/wiki/Address_space)，以[写时复制](https://en.wikipedia.org/wiki/Copy-on-write)的方法分配物理内存。[src](https://en.wikipedia.org/wiki/Fork_(system_call)).

当我们的系统从BIOS启动时，在main函数中只有一个进程：0号进程。和所有其他进程一样，它有一系列“状态”，存储在它的栈和寄存器内，这些都会存储在它的`task_struct`里面。`task_struct`是系统实现进程地址空间隔离的方法。

```c
struct task_struct {
	/* these are hardcoded - don't touch */
	long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	long counter;
	long priority;
	long signal;
	struct sigaction sigaction[32];	/* info about how to handle every possible signal*/
	long blocked;	/* bitmap of masked signals */
	/* various fields */
	int exit_code;
	unsigned long start_code,end_code,end_data,brk,start_stack;
	long pid,father,pgrp,session,leader;
	unsigned short uid,euid,suid;
	unsigned short gid,egid,sgid;
	long alarm;
	long utime,stime,cutime,cstime,start_time;
	unsigned short used_math;
	/* file system info */
	int tty;		/* -1 if no tty, so it must be signed */
	unsigned short umask;
	struct m_inode * pwd;
	struct m_inode * root;
	struct m_inode * executable;
	unsigned long close_on_exec;
	struct file * filp[NR_OPEN];
	/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
	struct desc_struct ldt[3];
	/* tss for this task */
	struct tss_struct tss;
};
```



当整个系统的初始化结束后，进程0会启动一个内核线程 (`init()`)，然后进入idle模式。当操作系统中没有其他进程时，调度器 (scheduler) 会运行这个idle进程。



事实上，这个idle进程 (进程0)是唯一一个不是动态创建的进程。在编译内核的时候，我们就已经静态定义了它的栈空间 (它的名字是`init_task`). 



## Fork()实现

创建子进程需要内核权限，当fork()被调用时，一个新的`task_struct`被创建。父进程的信息被复制进子进程。

Linux允许两个进程共享资源，包括文件，signal handler，和虚拟内存。相应资源的`count`会增加，这样系统就不会错误地释放资源（比如关闭正在被读取的文件），创造危险的空指针。比如当父进程和子进程同时拥有指向某个`mm_struct`的指针，那么`mm_struct`中的`count`就会增加，操作系统会在释放它之前检查`count`域。



克隆进程的虚拟内存要麻烦一些。一组新的`vm_area_struct`，它们的`mm_struct`以及子进程的页表需要被操作系统生成。在此刻，父进程的虚拟内存还没有被复制。父进程的虚拟内存可能在物理内存中，可能在父进程正在执行的可执行映象中，或是在某个swap文件中 (因为它正被运行)。由此可见，复制它的开销并不小。



所以Linux使用写时复制策略，这意味着仅当两个进程之一尝试向其写入时才复制虚拟内存，它才会为子进程复制。只读的页面是共享的，为了使“写时复制”起作用，可写区域的页表条目被标记为只读，而描述它们的` vm_area_struct`数据结构被标记为`copy_on_write`，当进程之一尝试写入此虚拟内存时，将发生page fault，此时，Linux将复制内存并修复两个进程的页表和虚拟内存数据结构。


{% asset_img fork.jpg %}
