---
layout: post
title:  "CS:APP Bomb Lab"
categories: Labs
tags: [reverse engineering, disassembling]
toc: true
--- 
Love is like a mirror: once broken, that ends it.
{: .message }

使用 GDB 与 objdump 工具，解读汇编代码，推理出答案字符串，即可完成此实验。[见此](https://github.com/QifanWang/learning-csapp/tree/master/handout/bomb)。

总体上，分为六个阶段的汇编代码解读。

## phase_1

phase_1 用到的两个函数，
string_length 其参数一(rdi)是待测字符串指针，返回(eax)为字符串长度。
strings_not_equal 其参数一(rdi)是 输入字符串指针， 参数二(rsi)是比较的字符串指针。返回(eax)为0，则说明二者相等。

汇编代码如下，
{% highlight nasm %}
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq 
{% endhighlight %}
答案为移入 rsi的地址(0x402400)代表的字符串。可用下面这个 GDB Command 输出结果。
>  print (char *) 0xbfff890 

> Examine a string stored at 0xbffff890

## phase_2

用到的函数，
read_six_numbers 其参数一(rdi)是一个字符串(字符指针)，参数二是一个长度为6的int数组(数组指针)，其调用sscanf将字符串作为输入为数组六个元素赋值，如果没有输入没有包含6个数字则会调用 explode_bomb 函数退出程序。

汇编代码以及添加的注释如下，
{% highlight nasm %}
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp	; allocate array space and
  400f02:	48 89 e6             	mov    %rsp,%rsi	; set first address as 2nd parameter
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)	; first element must be 1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax	; since 2nd, the element
  400f1a:	01 c0                	add    %eax,%eax	; must be twice as big as 
  400f1c:	39 03                	cmp    %eax,(%rbx)	; the prior one.
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx	; ++ iterator 
  400f29:	48 39 eb             	cmp    %rbp,%rbx	; loop condition: rbx != rbp
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx	; loop init, rbx is iterator
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp	; rbp is end position
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq
{% endhighlight %}

这段汇编会将字符串作为输入赋值给六个数字，并检查是否为从1开始且以2为比例的等比序列。
所以答案是 1 2 4 8 16 32 

## phase_3

汇编代码以及添加的注释如下，
{% highlight nasm %}
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	; the 4th parameter of sscanf 
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx	; the 3rd parameter of sscanf 
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi	; 2nd para is "%d %d"
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax	; return value of sscanf must be greater than 1
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)	; the 1st input num must not be above 7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax	; the 1st input num decides jump-position
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)	; jump table at 0x402470
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax		; when the 1st input num is 0
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax		; when the 1st input num is 2
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax		; when the 1st input num is 3
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax		; when the 1st input num is 4
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax		; when the 1st input num is 5
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax		; when the 1st input num is 6
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax		; when the 1st input num is 7
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax		; maybe dafault
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax		; when the 1st input num is 1
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax	; the 2nd input num must be eqaul to %eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	retq 
{% endhighlight %}

通过GDB命令，
`
print (char *) 0x4025cf
`
打印ssacnf第二个参数，可以得到"%d %d"，说明需要输入两个数字。
根据汇编代码知，第一个数字必须范围在0~7之间，
通过GDB命令，
`
x/16w 0x402470
`
打印0x402470处的跳转表的内容，可以得到两个数字必须满足的关系，

first num | second num
--- | ---
0 | 0xcf = 207
1 | 0x137 = 311
2 | 0x2c3 = 707
3 | 0x100 = 256
4 | 0x185 = 389
5 | 0xce = 206
6 | 0x2aa = 682
7 | 0x147 = 327

即是答案。

## phase_4

汇编代码以及添加的注释如下，
{% highlight nasm %}
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	; the 4th parameter of sscanf
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx	; the 3rd parameter of sscanf
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi	; the 2nd para of sscanf is "%d %d"
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax	; return value of sscanf must be equal to 2
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)	; the 1st input num must be below or equal to 0xe
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx	; the 3rd parameter of func4 is 0xe
  40103f:	be 00 00 00 00       	mov    $0x0,%esi	; the 2nd para of func4 is 0x0
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi	; the 1st para of func4 is the 1st input num
  401048:	e8 81 ff ff ff       	callq  400fce <func4>
  40104d:	85 c0                	test   %eax,%eax	; return value of func4 must be 0
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)	; the 2nd input num must equal to 0
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	retq 
{% endhighlight %}

通过GDB命令，
`
print (char *) 0x4025cf
`
打印ssacnf第二个参数，可以得到"%d %d"，说明需要输入两个数字。
根据汇编代码知，第一个数字必须范围小于等于14，第二个数字必须为0，且调用 func4 的结果的返回值必须为0。

调用函数 func4 第一个参数(%edi)为输入的第一个数字，第二个参数(%esi)为0，第三个参数(%edx)为14。

在函数func4 中会递归调用，
{% highlight nasm %}
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax
  400fd4:	29 f0                	sub    %esi,%eax
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx
  400fdb:	01 c8                	add    %ecx,%eax
  400fdd:	d1 f8                	sar    %eax
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	retq 
{% endhighlight %}
这个过程相当于根据返回值和其他两个参数，推测在0~14范围内的第一个参数，可以推出第一个参数仅可能为0，1，3，7。

## phase_5

函数phase_5有设置 canary 值进行栈破坏检查，可以忽略。

调用 string_length 计算输入字符串长度。
{% highlight nasm %}
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb>
{% endhighlight %}
输入串长度必须为6。

input 字符串的字符经过“操作”，放入 %rsp+0x10 处开始的数组，作为操作后的字符串。
“操作”后的字符串需要与 0x40245e 地址开始的字符串内容相同，通过命令
`
print (char *) 0x40245e
`
可知，于 0x40245e 地址开始的字符串是"flyers"。

关于“操作”的部分代码及添加注释，
{% highlight nasm %}
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx	; %rbx is address of input string
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx		; use every input char element's 4 lowest bit
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx	; to index of string at 0x4024b0 
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)	; use the new char to construct the 1st para of string_not_equal
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax		; loop condition
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)		; end of char array
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi		; 2rd para of string_not_equal, content is "flyers"
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi		; 1st para of string_not_equal
  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax	; use %eax as iterator
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
{% endhighlight %}



所谓“操作”，是将 input 的每个字符取出，将其低四位作为下标，取 0x4024b0 处开始的字符串中的字符，以构建操作后的字符串(%rsp+0x10 处开始)。
通过命令
`
print (char *) 0x4024b0
`
可知，于 0x4024b0 处开始的字符串为 "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"，需要从这个字符串中取字符以构建“flyers”字符串。

object | index | input char
--- | --- | ---
f | 9  | 0x49 I
l | 15 | 0x4f O
y | 14 | 0x4e N
e | 5 | 0x45 E
r | 6 | 0x46 F
s | 7 | 0x47 G

这里答案不唯一，因为高四位有多种可能。

## phase_6

函数 phase_6 有些长，这里分段说明，

### read six numbers
调用 read_six_numbers 从 input 字符串中读取六个数字，数字存储在以 %rsp 开始的数组中。
{% highlight nasm %}
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi	; array address at %rsp
  401106:	e8 51 03 00 00       	callq  40145c <read_six_numbers>
{% endhighlight %}


### loop check

代码添加注释如下，
{% highlight nasm %}
  40110b:	49 89 e6             	mov    %rsp,%r14	; %r14 stores stack pointer
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d	; %r12d is iterator i in loop, init as 0
  401114:	4c 89 ed             	mov    %r13,%rbp	; %r13 stores the i th element address, so does %rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax	; get a number from 6 numbers
  40111b:	83 e8 01             	sub    $0x1,%eax	
  40111e:	83 f8 05             	cmp    $0x5,%eax	; the num must be below or equal to 6
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d	; %r12d i iterator ++
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d	; loop condition: i in [0, 6)
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx	; %ebx is a new iterator j in inner loop, init as i + 1
  401135:	48 63 c3             	movslq %ebx,%rax	; %ebx index a input number,
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)	; the j th number must not be equal to i th num
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx	; %ebx iterator j ++
  401148:	83 fb 05             	cmp    $0x5,%ebx	; loop condition: j in [i+1, 5]
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
{% endhighlight %}
用一个双重循环，检查 input numbers 是否符合条件，
1. 每个数字小于等于6
2. 每个数字互不相同

### change every input number
通过一个循环将 input number 的值改为 7 - num，由于 input number 在0~6范围内，改变后在1~7范围内，
代码添加注释如下，
{% highlight nasm %}
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi	; array end address 
  401158:	4c 89 f0             	mov    %r14,%rax	; %rax get stack pointer address
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx	; %ecx is 7
  401160:	89 ca                	mov    %ecx,%edx	; %edx is 7
  401162:	2b 10                	sub    (%rax),%edx	; %edx is 7 - num 
  401164:	89 10                	mov    %edx,(%rax)	; input: num -> 7 - num
  401166:	48 83 c0 04          	add    $0x4,%rax	; %rax offset to next input num
  40116a:	48 39 f0             	cmp    %rsi,%rax	; loop condition: to end of array
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
{% endhighlight %}

### initiate a linked list

于 %rsp+0x20 处初始化了一个指针数组，每个指针按顺序对应 input number，且指针指向一个链表上元素，链表是有序的。

即 input number 对应了一个指针，其所指位置在链表上的顺序，代表该 num 在 6 个数字中的顺序。相当于排序。

{% highlight nasm %}
  40116f:	be 00 00 00 00       	mov    $0x0,%esi	; si is loop iterator i, init as 0
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx	; rdx is its next element, an address
  40117a:	83 c0 01             	add    $0x1,%eax	
  40117d:	39 c8                	cmp    %ecx,%eax	; eax increments to i th num
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx		; if num <= 1, stores 0x6032d0 as first address 
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)	; store an address at a new array after input num
  40118d:	48 83 c6 04          	add    $0x4,%rsi		; i++
  401191:	48 83 fe 18          	cmp    $0x18,%rsi		; loop condition: to end
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx	; get i th number
  40119a:	83 f9 01             	cmp    $0x1,%ecx		; if num[i] <= 1
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax		; %eax gets 1
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx		; dx gets first address
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
{% endhighlight %}

于 0x6032d0 开始存储的六个地址为恰好为链表的六个节点，其排列本身就按照顺序，
通过
`
x/24w 0x6032d0
`
命令可以得到，
{% highlight nasm %}
0x6032d0 <node1>:       0x0000014c      0x00000001      0x006032e0      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x00603300      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x00603310      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
{% endhighlight %}
倒数第二列即是链表节点的 next 字段，可以看出其排列本身就按照顺序，

input number (改变后) 排第 i (1-based)位小，对应指针数组中的指针即指向 nodei 位置。

### change the linked list

通过一个循环，将链表重新按指针数组中元素顺序排序，即指针数组第i个元素的 next 为 i+1 元素(指针)，
{% highlight nasm %}
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx	; bx gets 1st pointer
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax	; ax is loop iterator, init as 2nd address of pointer array
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi	; si gets end address of pointer array
  4011ba:	48 89 d9             	mov    %rbx,%rcx	; cx gets 1st pointer
  4011bd:	48 8b 10             	mov    (%rax),%rdx	; dx gets i th pointer
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)	; i-1 th pointer-to linked element's next is i th pointer
  4011c4:	48 83 c0 08          	add    $0x8,%rax	; ax iterator ++
  4011c8:	48 39 f0             	cmp    %rsi,%rax	; loop condition: to end of pointer array
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx	; cx gets next pointer
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)	; linked list tail element's next is NULL
{% endhighlight %}

通过命令，
`
x/12w $rsp+0x20
`
可以查看指针数组的内容。

指针数组重新代表了链表。

### check the linked list
链表节点的 32 bit 值必须大于下一个节点的 32 bit 值。
{% highlight nasm %}
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp	; bp is loop iterator, init as 5
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax	; ax get linked list bx pointer element's next
  4011e3:	8b 00                	mov    (%rax),%eax	; ax gets pointer-to 32 bit value
  4011e5:	39 03                	cmp    %eax,(%rbx)	; bx pointer-to value must be greater or equal to ax(next element)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx	; bx get next element
  4011f2:	83 ed 01             	sub    $0x1,%ebp	; bp --
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
{% endhighlight %}


### inference

由于必须满足链表重新排序后的大小关系，

linked node 32-bit value | node name | input number after change(7 - x) | original input number
--- | --- | --- | ---
0x0000039c | 0x6032f0 node3 | 3 | 4
0x000002b3 | 0x603300 node4 | 4 | 3
0x000001dd | 0x603310 node5 | 5 | 2
0x000001bb | 0x603320 node6 | 6 | 1
0x0000014c | 0x6032d0 node1 | 1 | 6
0x000000a8 | 0x6032e0 node2 | 2 | 5

故可以推得答案。

## Conclusions
通过这个实验熟悉x86-64汇编代码，以及用gdb进行逆向工程分析代码。

使用 objdump -d xxx 得到汇编代码，进行分析。

实验时需要仔细看 instruction code 的意思，一个指令搞错意思后面推理可能会出现连锁的错误。阅读汇编代码比较好的方法是，分段(根据大致意思以及jump指令分code blocks)并添加注释。

GDB 有些命令十分有用，尤其是一些打印内存数据的命令，这个[Cheat Sheet](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)与tui结合使用可以事半功倍。

## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [My bomb lab](https://github.com/QifanWang/learning-csapp/tree/master/handout/bomb)
3. [bomblab writeup](http://csapp.cs.cmu.edu/3e/bomblab.pdf)
4. [CS:APP Student Site](http://csapp.cs.cmu.edu/3e/students.html)
3. [Two-page x86-64 GDB cheat sheet](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)
4. [Beej's Quick Guide to GDB](http://beej.us/guide/bggdb/)
