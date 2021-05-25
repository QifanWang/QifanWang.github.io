---
layout: post
title:  "CS:APP Data Lab"
categories: Labs
tags: [reading, classic labs]
toc: true
--- 
When a man speaks or acts with good intention, then happiness follows him like his shadow that never leaves him. -Sakyamuni
{: .message }

## Data Lab
熟悉C primitive type(主要是整数与浮点数) 底层表示，以及位运算与补码，即可完成此实验。

实现代码[见此](https://github.com/QifanWang/learning-csapp/blob/master/handout/datalab-handout/bits.c)
有些问题很难，这里简单记录一下。

### isTmax
判断是否最大值。
{% highlight cpp %}
//2
/*
 * isTmax - returns 1 if x is the maximum, two's complement number,
 *     and 0 otherwise 
 *   Legal ops: ! ~ & ^ | +
 *   Max ops: 10
 *   Rating: 1
 */
int isTmax(int x) {
  return !(~((x + !(x+1)) ^ (x+1)));
}
{% endhighlight %}

这里主要困难在于消除 -1 的影响，其同样进位。
这里利用一个一个定式，判断32位全1。
> !(~(x ^ y))

### isLessOrEqual
判断两个数字大小关系。
{% highlight cpp %}
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
int isLessOrEqual(int x, int y) {
  int sigX = x >> 31; // sigX/sigY == 0  <=> X/Y >= 0
  int sigY = y >> 31; // sigX/sigY == -1 <=> X/Y <  0

  int diffSigFlag = sigX ^ sigY; // same sign -> diffSigFlag = 0
                                 // diff sign -> diffSigFlag = -1

  // consider overflow, we get ret when x and y have different sign bit
  int diffRet = !(~sigX&sigY); // different sign return

  // when same sign, calculation will not overfolw
  int sig = 1 << 31; 
  int sub = y + (~x + 1); // y - x
  int sameRet = !(sub & sig); // sub >= 0 <=> return 1

  return (~diffSigFlag & sameRet) | (diffSigFlag & diffRet);
}
{% endhighlight %}

开始我想当然的用相减（补码加），判断符号。但是会有溢出情况，异符号直接比较，这里也用了分支的一个定式，

`
(flag & y) | ((~flag) & z); // flag all one -> y, all zero -> z
`

### logicalNeg
实现逻辑非(bang)符号。
{% highlight cpp %}
//4
/* 
 * logicalNeg - implement the ! operator, using all of 
 *              the legal operators except !
 *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
int logicalNeg(int x) {
  x = x>>16|x;
  x = x>>8|x;
  x = x>>4|x;
  x = x>>2|x;
  x = x>>1|x;
  //overlap: x == 0 -> x == 0; x != 0 -> x zero digit is 1
  return ~x&1;
}
{% endhighlight %}

这里将所有位"折叠"到最低位上，检查是否存在非零位。很有意思。

### howManyBits
最小表示bit位数。
{% highlight cpp %}
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12) = 5
 *            howManyBits(298) = 10
 *            howManyBits(-5) = 4
 *            howManyBits(0)  = 1
 *            howManyBits(-1) = 1
 *            howManyBits(0x80000000) = 32
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 90
 *  Rating: 4
 */
// so =hard
int howManyBits(int x) {
  int sig, bit0, bit1, bit2, bit4, bit8, bit16;

  sig = x >> 31;

  // -1 -> 0
  // Tmin -> Tmax
  x = (sig & ~x) | (~sig & x);

  bit16 = (!!(x >> 16)) << 4; // 16
  x = x >> bit16;

  bit8 = (!!(x >> 8)) << 3; 
  x = x >> bit8;

  bit4 = (!!(x >> 4)) << 2;
  x = x >> bit4;

  bit2 = (!!(x >> 2)) << 1;
  x = x >> bit2;

  bit1 = (!!(x >> 1)) << 0;
  x = x >> bit1;

  bit0 = !!x;

  return bit16 + bit8 + bit4 + bit2 + bit1 + bit0 + 1;
}
{% endhighlight %}

整个lab应该是最难的一道题了。用二分不断减位，值得注意的是非负数需要加一个符号位，负数可以映射成非负数（一一对应）。

浮点数部分比较简单。只要熟悉IEEE 754标准即可。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [My data lab handout](https://github.com/QifanWang/learning-csapp/blob/master/handout/datalab-handout/bits.c)

