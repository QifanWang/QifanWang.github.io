---
layout: post
title:  "CS:APP Malloc Lab"
categories: Labs
tags: [system programming, memory]
toc: true
--- 
Don't you stop running and don't you ever look behind you.
{: .message }

阅读 csapp 3e 第九章。完成 Malloc 实验，编写 mm.c 代码，借助 mem_sbrk 实现 malloc, free, init and realloc 相关操作。

代码[见此](https://github.com/QifanWang/learning-csapp/tree/master/handout/malloclab-handout)。

终端 make 之后，通过下面命令调用评分程序，
```
mdriver -V -f short1-bal.rep
mdriver -V -f short2-bal.rep
```

## 宏与辅助函数
使用一些宏和辅助函数可以方便编程。但值得小心的是指针的算术计算，指针加减的最小单位是所指内容的大小(如char* 是一字节，但 void* 不是)。

{% highlight c %}
/* Basic constants and macros */
#define WSIZE 4 /* Word and header/footer size (bytes) */
#define DSIZE 8 /* Double word size (bytes) */
#define CHUNKSIZE (1<<12) /* Extend heap by this amount (bytes) */

#define MAX(x, y) ((x) > (y)? (x) : (y))

/* Pack a size and allocated bit into a word */
#define PACK(size, alloc) ((size) | (alloc))

/* Read and write a word at address p */
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val))

/* Read the size and allocated fields from address p */
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)

/* Given block ptr bp, compute address of its header and footer */
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)

/* Given block ptr bp, compute address of next and previous blocks */
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))


static char* heap_listp;

static void *coalesce(void *bp);
static void *extend_heap(size_t words);
static void *find_fit(size_t asize);
static void place(void*, size_t);
{% endhighlight %}

## Conclusions

本章知识十分有用，学习了系统底层是如何通过虚拟内存隔离进程，使用 mmap 映射磁盘设备到内存，private object 与 shared object 区别，在 fork 与 exec 系列函数中内存的变化，动态内存分配等。还是要多读文档和手册。

实验关于动态内存分配与回收的操作比较简单，主要按书上描述的结构模拟即可。如果要提升内存空间使用率(utilization)和吞吐效率(throughput)，还是需要实现书中所说的 GNU 实现使用的 Segregated Free Lists 结构。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [malloc lab readme](http://csapp.cs.cmu.edu/3e/README-malloclab)
3. [malloc lab writeup](http://csapp.cs.cmu.edu/3e/malloclab.pdf)
4. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/malloclab-handout)
