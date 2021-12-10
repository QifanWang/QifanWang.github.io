---
layout: post
title:  "Beware of buffer after dup2 stdout"
categories: c
tags: [Linux system programming]
toc: true
--- 
Buffered IO and unbuffered IO are tit-for-tat.
{: .message }


## Issue

多进程使用管道通信时，

```c
#include <sys/types.h>
#include <unistd.h>
#include <signal.h>
#include <wait.h>
#include <stdio.h>
#include <time.h>

int main() {
    int child_write[2], ws, i;
    pid_t cpid;
    char buf[1024];
    
    if(pipe(child_write) < 0) {
        fprintf(stderr, "child pipe creation error.\n");
        return -1;
    }

    cpid = fork();
    if(cpid == 0) {
        // in child

        close(child_write[0]);
        dup2(child_write[1], STDOUT_FILENO);

        while(1) {
            puts("child is running!");
            sleep(1); // sleep is working normally
        }
    } else {
        // in parent

        close(child_write[1]);
        
        for(i = 0; i < 10; ++i) {
            printf("before read [%d]\n", i);
            int cnt = 0;
            if((cnt = read(child_write[0], buf, 4)) < 0) {
                printf("read error\n");
            }
            
            buf[cnt] = '\0';
            printf("read from pipe: %s\n", buf);
        }

        kill(cpid, SIGKILL);

        if(waitpid(cpid, &ws, 0) != cpid) {
            // error
            fprintf(stderr, "waitpid error.\n");
        }
        puts("close");
        close(child_write[0]);
    }

    return 0;
}

```

父进程调用 read 后一直阻塞，起初怀疑是子进程睡眠超过一秒，通过如下命令在 GDB 调试时进入子进程，子进程睡眠正常。
```
-exec set follow-fork-mode child
```

又怀疑是父进程system call 时被信号中断后重复 restart 但查阅文档并尝试屏蔽信号也没用。最后发现问题是 stdout 缓冲区引起的问题。[flushing stdout](https://stackoverflow.com/questions/13932932/why-does-stdout-need-explicit-flushing-when-redirected-to-file)。

## Pipe Buffer and Stream Buffer

Pipe system call 会在内核中创建一块缓冲区，为单向的文件读写提供缓冲，但需要区分其和 Stream buffer 的区别。Stream(File 结构体)为输入/输出/错误流提供缓冲区，而重定向后，Stream buffer 还是保留的。也就是说，子进程先写 Stream buffer，数据再从 Stream buffer 到内核的 Pipe buffer。[different buffer](http://www.pixelbeat.org/programming/stdio_buffering/?__cf_chl_rt_tk=Bnv9DErBLAaqpBKfwdPoDKO5y.I02WRPJtKZkSl2UqA-1639047806-0-gaNycGzNA2U)

Stream Buffer 的三种自动缓冲策略(无fflush等调用时)分别如下
```c
/* The possibilities for the third argument to `setvbuf'.  */
#define _IOFBF 0		/* Fully buffered.  */
#define _IOLBF 1		/* Line buffered.  */
#define _IONBF 2		/* No buffering.  */
```

dup2 调用虽然将文件描述符重新定位了，但是文件流(File结构体)中buffer没有改变，所以之前父进程长久的阻塞是由于 pipe buffer 无数据可读，数据全在子进程的流缓冲区中，直到流缓冲区满才写入 pipe buffer。于是，在 dup2 调用后通过 setvbuf 重新设置 stdout buffer 的缓冲策略，要么完全不缓冲，要么按行缓冲(需要缓冲区空间)。

```c
if(cpid == 0) {
        // in child

        close(child_write[0]);
        dup2(child_write[1], STDOUT_FILENO);

        setvbuf(stdout, NULL, _IONBF, 0); // _IOLBF also works

        while(1) {
            // write(child_write[1], "child is running!", 18); // Or use unbuffered IO function

            puts("child is running!");
            // fflush(stdout); //Or flush buffer when _IOFBF

            sleep(1); // sleep is working normally
        }

    } 
```

除此之外，还可以手动 fflush 清缓冲区，也可以用 unbuffered IO function 如 write 调用。

## Conclusion

CSAPP 中强调过 buffered IO 与 unbuffered IO 的不同，字符型 IO 与 binary IO 的不同，在混合使用时(最好别混合)都需要小心仔细，查看文档。额外需要注意缓冲区数据结构的问题，缓冲区由谁维护(kernel? 文件流?)，缓冲策略？切换文件描述符后，文件流究竟改变了什么？进程的文件流与文件描述符(Stream 与 file descriptor)均属于进程的"文件描述"，在 fork 的 exec 之后二者变化是不同的，descriptor 被保留而 stream became inaccessible (因为 file descriptor 才是最本质的描述，stream 在其上套了一层)。 [POSIX Standard IO Stream](https://pubs.opengroup.org/onlinepubs/009695399/functions/xsh_chap02_05.html#tag_02_05_01), [POSIX exec](https://pubs.opengroup.org/onlinepubs/009695399/functions/exec.html)

## Reference
1. [gdb forks](https://sourceware.org/gdb/onlinedocs/gdb/Forks.html)
2. [Does read/write blocked system call put the process in task-uninterruptible](https://stackoverflow.com/questions/56136082/does-read-write-blocked-system-call-put-the-process-in-task-uninterruptible-or-t)
3. [flushing stdout](https://stackoverflow.com/questions/13932932/why-does-stdout-need-explicit-flushing-when-redirected-to-file)
4. [different buffer](http://www.pixelbeat.org/programming/stdio_buffering/?__cf_chl_rt_tk=Bnv9DErBLAaqpBKfwdPoDKO5y.I02WRPJtKZkSl2UqA-1639047806-0-gaNycGzNA2U)
