---
layout: post
title:  "CS:APP Performance Lab"
categories: Labs
tags: [optimization]
toc: true
--- 
Make the most of today. Seize the day!
{: .message }

阅读 csapp 3e 第五章，完成关于优化的实验。实验内容主要是修改 kernels.c 中两个有关图像处理的函数，降低 CPE(Cycle Per Element) 以提高效率。

`make driver` 编译。

`driver` 测试。

## rotate
该函数实现将一个正方形图像像素点逆时针旋转 90 度，原先代码为，
{% highlight c %}
void naive_rotate(int dim, pixel *src, pixel *dst) 
{
    int i, j;

    for (i = 0; i < dim; i++)
	for (j = 0; j < dim; j++)
	    dst[RIDX(dim-1-j, i, dim)] = src[RIDX(i, j, dim)];
}
{% endhighlight %}

通过宏 #define RIDX(i,j,n) ((i)*(n)+(j)) 将二维索引定位到一维的连续存储，内存访问为程序瓶颈(缓存与换页)。

由于测试数据中 dim 必然为 64 的倍数，可以利用分块进行加速，
{% highlight c %}
char rotate_descr[] = "rotate: Current working version, 8 * 8 block";
void rotate(int dim, pixel *src, pixel *dst) 
{
    int dim_1 = dim - 1;
    int seg = 8;
    int i, j, A, B;
    
    for(A = 0; A < dim; A += seg)
    	for(B = 0; B < dim; B += seg)
    	    for(i = A; i < A + seg; ++i)
    	    	for(j = B; j < B + seg; ++j) {
    	    	    dst[RIDX(dim_1-j, i, dim)] = src[RIDX(i, j, dim)];
    	    	}
    	    	   
}
{% endhighlight %}

经过实验比较 4 * 4，8 * 8，16 * 16，32 * 32 与 64 * 64 分块的运行效率，选择 8 是更好的解。较之原解法有近乎两倍的提升。
```
Rotate: 11.8 (naive_rotate: Naive baseline implementation)
Rotate: 20.6 (rotate: Current working version, 8 * 8 block)
```

尝试对"内存两层循环"进行循环展开，但 CPE 提升不明显(从 8.9 到 18.1)
{% highlight c %}
18.1 / 8.9 = 2.033707865168539
20.6 / 11.8 = 1.7457627118644068
{% endhighlight %}

循环展开代码过长，就不放代码。

## smooth
该函数实现将一个正方形图像像素点“模糊化”的操作，具体来说就是一个像素点的新值为其紧邻的像素点(最多八个)与其旧值的平均值。

优化重点在于减少函数调用与分支预测(判断周围紧邻的像素点有几个)。

可以先计算特殊四个角与四边的像素点新值，再用循环处理常见的像素点(周围有八个像素点)。

代码过长，可见[4](https://github.com/QifanWang/learning-csapp/tree/master/handout/perflab-handout)。

## Conclusions
最终结果，rotate 拥有近乎两倍的性能提升，而 smooth 拥有超过四倍的性能提升。
```
Summary of Your Best Scores:
  Rotate: 20.4 (rotate: Current working version, 8 * 8 block)
  Smooth: 68.6 (smooth: Current working version, loop unrolling.)
```

本章学习的优化不同于数据结构与算法设计上的优化，主要内容在于：

基本编码原则
- 消除连续的函数调用。尽可能将计算移出循环外。
- 消除不必要的内存引用。引入临时变量保存中间结果。

低级优化
- 展开循环。
- 通过多个累积变量与重结合等手段，提高指令级并行。
- 重写条件操作，使得编译器采用条件数据传送。

优化是一个需要谨慎对待的问题，不要过早优化，也不要在优化时引入错误。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [perf lab readme](http://csapp.cs.cmu.edu/3e/README-perflab)
3. [perf lab writeup](http://csapp.cs.cmu.edu/3e/perflab.pdf)
4. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/perflab-handout)
