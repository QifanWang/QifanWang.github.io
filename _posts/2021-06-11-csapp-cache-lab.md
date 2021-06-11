---
layout: post
title:  "CS:APP Performance Lab"
categories: Labs
tags: [optimization]
toc: true
--- 
So every defect of the mind may have a special reciept.
{: .message }

阅读 csapp 3e 第六章，完成关于存储器缓存的实验。实验内容分为两部分，(1)写一个缓存模拟器(2)与改写二维数组的转秩函数,减少缓存 miss 次数。

`make` 编译。

`driver.py` 测试。

## Part A
编写 csim.c 代码，完成一个可以读取 valgrind 的 memory trace ，模拟缓存命中、丢失与替换，并输出三种情况统计次数的缓存模拟器。

由于需要读取命令行参数以模拟不同大小的缓存，使用 [getopt](https://en.wikipedia.org/wiki/Getopt) 处理命令行参数。

定义结构体 CacheLine 模拟每一行缓存，由于不需要真实读取数据，所以保留了 valid bit 与 tag 但省略了 block 部分。
因为采用 LRU 替换策略，为简单起见，给每一行缓存增加一个时间戳，命中与加入缓存组时更新，替换时遍历组内所有行找到最‘旧’的行替换之。
当然可以通过 哈希表 + 双向链表 完成更高效率的LRU实现，但这里仅是模拟不追求效率。
{% highlight c %}
typedef struct {
    int validBit;
    unsigned tag;
    // no data block, only simulating
    int timeStamp; // for LRU replacement policy
} CacheLine;
{% endhighlight %}

通过 malloc 动态地完成“缓存”的建立后，程序读取 trace file 进行模拟。
将数据地址进行位操作提取出缓存信息(tag, S, offset)后，由于只用关心数据读写，所以程序只用模拟三种情况，

类型 | 访存次数 | 解释
--- | --- | ---
DATA_LOAD | 1 | read
DATA_STORE | 1 | write
DATA_MODIFY | 2 | read, then write

读与写，可以模拟作同样的访存行为(hitCache)，当没有命中时，先尝试填充缓存(fillCache)，若失败则执行替换(evict)。
故 DATA_LOAD 与 DATA_STORE 可以进行相同的模拟，即尝试访存。而 DATA_MODIFY 相当于访存两次，第二次必中故只需增加命中次数即可。
以下为三种读写"缓存"行为的模拟函数，
{% highlight c %}
// try to hit cache, if succeed return 1
int hitCache(CacheLine* cacheSpace, unsigned setInd, int cntLines, unsigned tag, int timeStamp) {
    for(int i = 0; i < cntLines; ++i) {
        unsigned cacheInd = getCacheInd(setInd, cntLines, i);
        if(cacheSpace[cacheInd].tag == tag && cacheSpace[cacheInd].validBit == 1) {
            cacheSpace[cacheInd].timeStamp = timeStamp;
            return 1;
        }
    }
    return 0;
}

// try to fill a record into a set, if succeed return 1
int fillCache(CacheLine* cacheSpace, unsigned setInd, int cntLines, unsigned fillTag, int timeStamp) {
    for(int i = 0; i < cntLines; ++i) {
        unsigned cacheInd = getCacheInd(setInd, cntLines, i);
        if(cacheSpace[cacheInd].validBit == 0) {
            cacheSpace[cacheInd].validBit = 1;
            cacheSpace[cacheInd].tag = fillTag;
            cacheSpace[cacheInd].timeStamp = timeStamp;
            return 1;
        }
    }
    return 0;
}

// evict a record in a set with LRU replcement policy
void evict(CacheLine* cacheSpace, unsigned setInd, int cntLines, unsigned fillTag, int timeStamp) {
    int pivotInd = 0, pivotVal = cacheSpace[getCacheInd(setInd, cntLines, 0)].timeStamp;
    for(int i = 1; i < cntLines; ++i) {
        unsigned cacheInd = getCacheInd(setInd, cntLines, i);
        if(cacheSpace[cacheInd].timeStamp < pivotVal) {
            pivotVal = cacheSpace[cacheInd].timeStamp;
            pivotInd = i;
        }
    }
    
    int evictInd = getCacheInd(setInd, cntLines, pivotInd);
    cacheSpace[evictInd].validBit = 1;
    cacheSpace[evictInd].tag = fillTag;
    cacheSpace[evictInd].timeStamp = timeStamp;
}
{% endhighlight %}

小心编写一些常用的 helper function，这一部分实验开始时因为 hex2dec 逻辑错误仅通过部分测试。

代码[见此](https://github.com/QifanWang/learning-csapp/blob/master/handout/cachelab-handout/csim.c)。

## Part B
编写 trans.c 中的 transpose_submit 函数，完成二维数组的转秩，同时尽量减少缓存丢失次数。

可以通过分块(blocking)处理，减少 miss 次数。
测试中的缓存大小为 ( s = 5,E = 1,b = 5)，共有 32 组，每组一行，一行 32 字节(8个int元素)。一次缓存8个元素，可以分块为 8*8 提高效率。
针对测试中的 32 * 32 与 61 * 67 的测试，使用 8 * 8 与 16 * 16 分块就可以满分通过。但 64 * 64 的情况要求更严格，由于冲突不命中(conflict miss)存在，命中率难以提高。

参考这篇[文章](https://zhuanlan.zhihu.com/p/28585726)。在 8 * 8 分块中再进行分块处理，假设原数组为，
```
A,B
C,D
```

先将前四行的两个块转秩后，放在结果数组前四行，
```
A',B'
X ,X
```

再读结果数组中右上小块与原数组 8*8 的左下小块(转秩)，进行交换，
```
A',C'
B',X
```

最后读取原数组右下小块(转秩)填充结果数组右下，
```
A',C'
B',D'
```

代码[见此](https://github.com/QifanWang/learning-csapp/blob/master/handout/cachelab-handout/trans.c)。

## Conclusions
本章学习了各种存储技术、存储器层次结构与高速缓存组织结构，为写出缓存友好的代码，关键在于：

- 专注内循环，大部分计算与访存取决于内循环。
- 利用空间局部性，按数据存储在内存中的顺序、以步长为1进行读写。
- 利用时间局部性，读取一个数据对象后就立刻地尽可能地使用它。
- 善用分块进行优化。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [cache lab readme](http://csapp.cs.cmu.edu/3e/README-cachelab)
3. [cache lab writeup](http://csapp.cs.cmu.edu/3e/cachelab.pdf)
4. [C library getopt](https://en.wikipedia.org/wiki/Getopt)
5. [马天猫的CS学习之旅 CS:APP配套实验4：Cache Lab笔记](https://zhuanlan.zhihu.com/p/28585726)
6. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/cachelab-handout)
