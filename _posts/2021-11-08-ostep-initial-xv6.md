---
layout: post
title:  "OSTEP: initial-xv6 lab"
categories: Labs
tags: [system programming, operating system lab]
toc: true
--- 
What sculpture is to a block of marble, education is to the soul.
{: .message }

开始 OSTEP 实验系列，通过向 xv6 添加一个系统调用
```c
int getreadcount(void)
```
其返回计数器记录所有进程系统调用 **read()** 的次数。

通过下面命令调用编译与评分程序，
```sh
./test-getreadcounts.sh -s
```

在 Ubuntu 20.04 上实验，需要安装 `expect` 和 `gawk` 才能顺利运行评分脚本。见 [Issue](https://github.com/remzi-arpacidusseau/ostep-projects/issues/28)

## System Call Background

该实验主要是理解系统调用的细节，在 CSAPP 中统一将所有中断、陷阱、错误与中止称为异常(Exception)，

| Class      | Cause                         | Async/sync | Return behavior                     |
| ---------- |:-----------------------------:|:----------:| -----------------------------------:|
| Interrupt  | Signal from I/O device        | Async      | Always returns to next instruction  |
| Trap       | Intentional exception         | Sync       | Always returns to next instruction  |
| Fault      | Potentially recoverable error | Sync       | Might return to current instruction |
| Abort      | Nonrecoverable error          | Sync       | Never returns                       |

中断例如键盘中断、时钟中断等，陷阱是主动的系统调用，错误有例如 devide error / protection fault / page fault ，中止将会退出程序。其实可将以上都视作 Trap , 因为都是从用户态陷入内核态，但实际上实现时用的词汇可能并没有那么准确，比如实际上通过 **int** 指令(short for interrupt)接 exception number 进入内核态，xv6 在 **traps.h** 中定义 exception number 为宏等等。 CSAPP 是根据原因与行为来分类。以下为 x86-64 中的异常号与对应异常。

| Exception number | Description              | Exception class   |
| ---------------- | ------------------------ | ------------------|
| 0                | Divide error             | Fault             |
| 13               | General protection fault | Fault             |
| 14               | Page fault               | Fault             |
| 18               | Machine check            | Abort             |
| 32–255           | OS-defined exceptions    | Interrupt or trap |

xv6 定义了宏 SYSCALL(name) 进行系统调用，
```c
#define SYSCALL(name) \
  .globl name; \
  name: \
    movl $SYS_ ## name, %eax; \
    int $T_SYSCALL; \
    ret
```
File: **usys.S**

其中 **T_SYSCALL** 为 traps.h 中的宏，即对应系统调用的 Exception number, 而 **%eax** 寄存器保存 system call number 区分不同的系统调用。我们首先在 **usys.S** 和 **syscall.h** 中增加 getreadcount 的 syscall number 和对应的汇编代码。

## After Call int instruction

了解 [Kernel Side](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/initial-xv6/background.md#kernel-side-trap-tables) 如何完成 IDT (就是Trap table 向量表，就是 Exception no 与其对应的代码)的初始化后，我们要知道当 application 使用 **int** 指令后, 系统与 OS 做了什么。

在 **int** 指令后，硬件接管改变当前优先级，并存部分上下文(即PC, regs, eflags, stack pointer等，定义在 **trapframe** 中，硬件与OS各存部分，见 **x86.h** 的 trapframe 注释)，然后将控制交给 OS , 即根据 IDT 寻找到对应代码位置。在 xv6 中 IDT 指定的代码地址在 **vectors.S** （其由 **vectors.pl** 生成）。

```
.globl vector64
vector64:
  pushl $64
  jmp alltraps
```
File: **vector.S**

例如，System Call 是 64 号，对应的代码先压入 trapno (其实就是 Exception no) 进入 trapframe 然后 **jmp alltraps** ，而 alltraps 就是完成压入一些其他 segment registers / general registers 完成 trap frame 的建立，设置数据段(改变 segment selector) 然后调用 C trap handler 

## C trap handler


```c
void
trap(struct trapframe *tf)
{
  if(tf->trapno == T_SYSCALL){
    if(myproc()->killed)
      exit();
    myproc()->tf = tf;
    syscall();
    if(myproc()->killed)
      exit();
    return;
  }
  ... // processing other trap nos
} 
```
File: **trap.c**

trap 函数即为 trap handler (这又是一个用词问题，其实这里处理的是所有 exception no 对应的异常)，其本质上是一个 gateway 函数，根据 trapframe 中的 **trapno** 完成不同的操作。当 trapno 为 T_SYSCALL(64 号)时， trap handler 检查进程是否存活(离开 trap handler 时也要检查)，为当前进程关闭中断(防止当从CPU读进程状态时，进程被重新调度)并改变当前进程的 trapframe (为了系统调用后恢复上下文)，然后调用 syscall() 函数。

## Which System Call ?

syscall 函数将已经保存的 trapframe 中取出 **%eax** (在 usys.S 保存的系统调用号) 并检查合法性，然后在 system call table **syscalls[]** 根据调用号寻找对应的函数代码。 syscalls 数组是一个函数指针数组，其函数指针指向 int(void) 函数，于是我们修改表的内容并增加 sys getreadcount 函数。 


新增的代码逻辑就是新增一个 static 计数器变量并在 sys_read 函数中自增，在 sys getreadcount 函数中返回即可。单个CPU多进程是安全的，因为内核态进程不会被 reschedule 故无数据竞争；但考虑到多CPU时多进程，多个CPU上的进程同时进入内核态，这会带来数据竞争问题（OS内核也是程序），所以需要给这个计数器加 spinlock（其他多CPU共享的数据结构也有考虑数据竞争的问题，如 file table 等）并考虑初始化与带锁的写。代码[见此](https://github.com/QifanWang/xv6-public/commit/f7062291ea54a91d9467a7b0d016ebe1c2c9530c)

## Conclusion

这个实验本质不难，主要是通过系统调用熟悉 xv6 内核代码，了解 exception control flow 的细节，并了解如何增加系统调用。

## Reference
1. [Basic guidance](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/initial-xv6/README.md)
2. [Some issue](https://github.com/remzi-arpacidusseau/ostep-projects/issues/28)
3. [Kernal background](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/initial-xv6/background.md)
4. [My solution code](https://github.com/QifanWang/xv6-public/commit/f7062291ea54a91d9467a7b0d016ebe1c2c9530c)

