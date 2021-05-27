---
layout: post
title:  "CS:APP Performance Lab"
categories: Labs
tags: [optimization]
toc: true
--- 
Make the most of today. Seize the day!
{: .message }

关于优化的实验，主要是修改 kernels.c 中两个有关图像处理的函数，降低 CPE(Cycle Per Element) 以提高效率。

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

待续


## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [perf lab readme](http://csapp.cs.cmu.edu/3e/README-perflab)
3. [perf lab writeup](http://csapp.cs.cmu.edu/3e/perflab.pdf)
