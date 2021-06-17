---
layout: post
title:  "CS:APP Shell Lab"
categories: Labs
tags: [system programming, multi-process]
toc: true
--- 
Love starts as an attraction to another.
{: .message }

阅读 csapp 3e 第七、八章。完成 Shell 实验，编写 tsh.c 代码，实现一个拥有任务控制的简单版本 shell 程序。

代码[见此](https://github.com/QifanWang/learning-csapp/tree/master/handout/shlab-handout)。

写了一个脚本 tsh_test.sh 输出测试结果到 tsh.out 中，比较 tsh.out 内容与 tshref.out (通过 sdriver.pl -g 生成)的内容，检查结果是否正确。

## Signal Handler
自定义信号处理函数时，需要遵守一些“保守”的原则，

0. Keep handlers as simple as possible. 
1. Call only async-signal-safe functions in your handlers.
2. Save and restore errno.
3. Protect accesses to shared global data structures by blocking all signals.
4. Declare global variables with volatile.
5. Declare flags with sig_atomic_t.

尤其注意是信号的屏蔽(block)与还原，一个信号处理函数是可以被其他信号中断的，在读写全局数据时尤其要小心。 
我们的 shell 程序处理 SIGINT 与 SIGTSTP 仅仅是将信号“传递”给前台进程组中的进程，只用简单系统调用 kill 即可。
程序处理 SIGCHLD 稍稍复杂，调用 waitpid 回收 zombie process 时需要注意“信号无排队”现象，根据 Lab 要求 waitpid 不能阻塞，终止与停止的进程都需要处理(修改job表)，系统调用后的 errno 注意 ECHLD (多次调用 waitpid 可能使最后一次无子进程)。
阅读 man 手册，查看相关系统调用与宏十分有帮助。
{% highlight c %}
void sigchld_handler(int sig) 
{
    // puts("in sigchld_handler");
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;
    int status;

    Sigfillset(&mask_all);
    Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); //
    while((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) { // no bloking 
        
        // puts("after waitpid");
        if(WIFEXITED(status)) { // job terminates 
            // puts("delete job");
            deletejob(jobs, pid); // deleting jobs from global job table
        }
        if (WIFSTOPPED(status)) { // job stops by SIGSTOP or SIGTSTP
            struct job_t *job = getjobpid(jobs, pid);
            int jid = pid2jid(pid);
            printf("Job [%d] (%d) stopped by signal %d\n",jid, pid, WSTOPSIG(status));
            job->state = ST;
        }
        if (WIFSIGNALED(status)){
            // puts("signaled");
            int jid = pid2jid(pid);
            printf("Job [%d] (%d) terminated by signal %d\n",jid, pid, WTERMSIG(status));
            deletejob(jobs, pid); // deleting jobs from global job table
            // unix_error("wait child pid for a unknown reason.");
        }
    }
    if(errno && errno != ECHILD) {
        unix_error("waitpid error");
    }
    Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    errno = olderrno;
    return;
}

void sigint_handler(int sig) 
{
    // puts("in sigint_handler");
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t fg;
    
    Sigfillset(&mask_all);
    Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
    if((fg = fgpid(jobs)))
    	kill(-fg, sig); // send sig to every process in the foreground process group.
    Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    errno = olderrno;
    return;
}

void sigtstp_handler(int sig) 
{
    // puts("in sigtstp_handler");
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t fg;

    Sigfillset(&mask_all);
    Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
    if((fg = fgpid(jobs)))
    	kill(-fg, sig); // send sig to every process in the foreground process group.
    Sigprocmask(SIG_SETMASK, &prev_all, NULL);

    errno = olderrno;
    return;
}
{% endhighlight %}

## Wait Foreground Process

Shell 程序运行程序分为前台与后台，对于前台程序(job)需要等待其完成。
文档中建议在 waitfg 中使用 sleep 函数，但我选择使用书中 8.5.7 节建议的 sigsuspend 函数作为更好的“原子化”操作。
{% highlight c %}
/* 
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    sigset_t mask_all, prev;
    Sigfillset(&mask_all);
    while(1) {
        Sigprocmask(SIG_BLOCK, &mask_all, &prev); //Blocking all signals
        pid_t curfg = fgpid(jobs);
        // printf("in waitfg loop, pid %d and curfg %d\n", pid, curfg);
        if(pid != curfg) {
            Sigprocmask(SIG_SETMASK, &prev, NULL);
            break;
        } 
        else {
            // puts("suspend now");
            sigsuspend(&prev);
            if(errno == EINTR) {
                errno = 0;
            } else {
                unix_error("after suspend");
            }
            Sigprocmask(SIG_SETMASK, &prev, NULL);
        }
    }
    return;
}
{% endhighlight %}

## Evaluate the Command Line

对比书中的 eval 函数，我们需要额外做的就是信号量的屏蔽与还原以及区分前后台 job 处理。
父进程在 Fork 前屏蔽信号，在 addjob 后还原。这是为了防止竞争情形中子进程先终止，导致父进程进行SIGCHLD处理。子进程则需要在 execve 前还原信号。这是因为子进程会继承原来的屏蔽信号。前台 job 需要调用 waitfg 函数进行等待。
{% highlight c %}
void eval(char *cmdline) 
{
    char *argv[MAXARGS]; /* Argument list execve() */
    char buf[MAXLINE];   /* Holds modified command line */
    int background_flag; /* Should the job run in bg or fg? */
    pid_t pid;           /* Process id */
    sigset_t mask, prev_mask;
    Sigemptyset(&mask);
    Sigaddset(&mask, SIGCHLD);

    strcpy(buf, cmdline);
    background_flag = parseline(buf, argv);
    if(argv[0] == NULL)
        return; /* Ignore empty lines */
    
    if(!builtin_cmd(argv)) {
        Sigprocmask(SIG_BLOCK, &mask, &prev_mask);
        if((pid = Fork()) == 0) { /* Child runs user job*/
            if(setpgid(0, 0) < 0) {
                fprintf(stdout, "%s: setpgid error.\n", argv[0]);
                exit(EXIT_SUCCESS);                
            }
            Sigprocmask(SIG_SETMASK, &prev_mask, NULL); // child process inherits blocked vector, unblocking
            if(execve(argv[0], argv, environ) < 0) {
                fprintf(stdout, "%s: Command not found\n", argv[0]);
                exit(EXIT_SUCCESS);
            }
        }

        // printf("addjob\n");
        if(!background_flag) {
            addjob(jobs, pid, FG, cmdline);
            Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
            waitfg(pid);
        }
        else {
            addjob(jobs, pid, BG, cmdline);
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
            // printf("Background running pid %d: %s", pid, cmdline);
            Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
        }
        
    }
    return;
}
{% endhighlight %}

其他函数如处理内置命令的 builtin_cmd 函数与改变 job 前后台状态的 do_bgfg 函数相对简单，这里不再赘述。同时，我也学习书中技巧，封装了一些系统调用函数，并写了一个 getJobByArg 函数用于处理参数中 jobid 与 pid 的不同表示。

## Conclusions

这两章主要是链接、加载与库的知识，系统编程的基本概念，如异常控制流、进程、系统调用、信号处理函数与非本地跳转。

进行实验时，采取书中所述策略编写 Safe、Correct and Portable 信号处理函数是十分有用的。并发时的同步问题需要引起重视，我们需要善用信号屏蔽与还原以及其他手段。信号屏蔽一定需要还原，特别是在一些控制流结构(if与while)和调用函数前后，尤其小心。阅读 man 手册是了解系统接口使用细节的极好途径。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [shell lab readme](http://csapp.cs.cmu.edu/3e/README-shlab)
3. [shell lab writeup](http://csapp.cs.cmu.edu/3e/shlab.pdf)
4. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/shlab-handout)
