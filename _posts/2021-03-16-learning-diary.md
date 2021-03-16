---
layout: post
title:  "2021 Mar 16th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 
Keep it simple: as simple as possible, but no simpler. – A.Einstein
{: .message }

## LeetCode
[59. 螺旋矩阵 II](https://leetcode-cn.com/problems/spiral-matrix-ii/)
和昨天题目差不多，递归，一层一层剥洋葱，只是这次是添加数字。注意基线条件，奇偶不同导致最后是1还是0，二者都需check。

## A Tour of C++ 1st
阅读ch13-14，完成本书阅读。ch13讲并发，ch14讲C++历史与compatibility兼容性。

### Concurrency
标准库提供<thread>实现线程，并发编程时通过锁和其他机制防止 data race(操作共享objects）。

用unique_lock<mutex>加锁
{% highlight cpp %}
#include <mutex>

std::mutex m; // controlling mutex
int sh; // shared data

void f() {
    std::unique_lock<std::mutex> lck {m}; // acquire mutex
    sh += 7;                              // manipulate shared data
}   // release mutex implicitly
{% endhighlight %}

预防dead lock，可以通过破环下面第二个条件：
- 互斥
- resource holding(没有一次性获取所有资源)
- 不可剥夺
- 循环等待

{% highlight cpp %}
#include <mutex>

void f() {
    // ...
    std::unique_lock<std::mutex> lck1 {m1, defer_lock}; // defer_lock: don’t yet try to acquire the mutex
    std::unique_lock<std::mutex> lck2 {m2, defer_lock};
    std::unique_lock<std::mutex> lck3 {m3, defer_lock};
    // ...
    lock(lck1, lck2, lck3);					 // acquire all three locks
    // ... manipulate shared data ...
}   // release mutex implicitly
{% endhighlight %}

标准库<condition_variable>通过wait与notify实现thread等待某些条件(event)，以支持通信。

#### Communicating Tasks

future and promise 这个模型比较有意思，两个任务传值避免了显式使用锁。

为了避免简单的promise，标准库特地包了一层，用packaged_task得到task type(函数类型)。

同时标准库提供async()实现异步任务，需要小心的是：
> There is an obvious limitation: Don’t even think of using async() for tasks that share resources needing locking – with async() you don’t even know how many threads will be used because that’s up to async() to decide based on what it knows about the system resources available at the time of a call.

避免使用async()操作需要加锁的共享资源，因为不清楚究竟有多少线程会被使用。


使用future的例子：
{% highlight cpp %}
#include <iostream>
#include <future>
#include <thread>
 
int main()
{
    // future from a packaged_task
    std::packaged_task<int()> task([]{ return 7; }); // wrap the function
    std::future<int> f1 = task.get_future();  // get a future
    std::thread t(std::move(task)); // launch on a thread
 
    // future from an async()
    std::future<int> f2 = std::async(std::launch::async, []{ return 8; });
 
    // future from a promise
    std::promise<int> p;
    std::future<int> f3 = p.get_future();
    std::thread( [&p]{ p.set_value_at_thread_exit(9); }).detach();
 
    std::cout << "Waiting..." << std::flush;
    f1.wait();
    f2.wait();
    f3.wait();
    std::cout << "Done!\nResults are: "
              << f1.get() << ' ' << f2.get() << ' ' << f3.get() << '\n';
    t.join();
}
{% endhighlight %}

### History And Compatibility

> The first ISO C++standard (ISO/IEC 14882-1998) [C++,1998] was ratified by a 22-0 national vote in 1998. A ‘‘bugfix release’’ of this standard was issued in 2003, so you sometimes hear people refer to C++03, but that is essentially the same language as C++98.

C++03本质是C++98的bugfix release版本。

关于各个版本的相同与差异，书里这个Venn diagram很不错。mark

然后关于不兼容C时需要做一些额外的工作：
- extern "C" (linkage problem)
- keyword difference
- void* implicitly conversion
- etc

### Conclusion
- C++11提供了一些并发编程的特性可以使用，以后需要深入学习
- 关于GCC的一些使用阅读
- C++11前历史回顾与特性兼容简介

## Reference
1. [c++ reference thread future](https://en.cppreference.com/w/cpp/thread/future)
2. [GNU gcc invoking g++](https://gcc.gnu.org/onlinedocs/gcc/Invoking-G_002b_002b.html)
3. [GNU gcc debugging options](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html#Debugging-Options)
4. [GNU gcc libstdc++ concurrency](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_concurrency.html)
5. [Difference between g++ and gcc](https://stackoverflow.com/questions/172587/what-is-the-difference-between-g-and-gcc)
