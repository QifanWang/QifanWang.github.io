---
layout: post
title:  "CS:APP Attack Bomb"
categories: Labs
tags: [architecture, assembling]
toc: true
--- 
There is no good or bad in life, except what is good according to its own season.
{: .message }

关于Y86-64的实现，相关工具与编译需要注意。

## Part A
工作目录在 sim/misc 中。
将对应的 C 代码翻译成对应的Y86-64指令，除了共同的程序设置，如栈区设置与调用main函数设置以外，需要注意用 yas 编译汇编代码，用 yis 模拟运行。

### sum.ys: Iteratively sum linked list elements
简单的循环计算链表和，
{% highlight nasm %}
 # long sum_list(list_ptr ls)
 # ls in %rdi
sum_list:
	xorq %rax, %rax	# val = 0
loop:
	andq %rdi, %rdi
	je end			# loop condition
	mrmovq (%rdi), %rcx	# ls->val in rcx
	addq %rcx, %rax	# val += ls->val
	irmovq $8, %rbx
	addq %rbx, %rdi	# &(ls->next) in rdi
	mrmovq (%rdi), %rdi	# ls = ls->next
	jmp loop
end:
	ret
{% endhighlight %}

最终 %rax 的结果为0x0cba

### rsum.ys: Recursively sum linked list elements
递归版链表求和，
{% highlight nasm %}
 # long rsum_list(list_ptr ls)
 # ls in %rdi
rsum_list:
	xorq %rax, %rax	# val = 0
	andq %rdi, %rdi
	je end		# ls != 0
	mrmovq (%rdi), %rcx	# ls->val in rcx
	irmovq $8, %rbx
	addq %rbx, %rdi	# &(ls->next) in rdi
	mrmovq (%rdi), %rdi	# ls = ls->next
	pushq %rcx
	call rsum_list	
	popq %rcx
	addq %rcx, %rax	# rest + val
end:
	ret
{% endhighlight %}
需要注意，调用者保存寄存器 %rcx 被使用。

### copy.ys: Copy a source block to a destination block
复制 src 内存数据到 dest 位置，
{% highlight nasm %}
 # long copy_block(long *src, long *dest, long len)
 # src in %rdi, dest in %rsi and len in %rdx
copy_block:
	xorq %rax, %rax	# result = 0
loop:
	andq %rdx, %rdx
	jle end		# len <= 0 jump end
	
	mrmovq (%rdi), %rcx	# val = *src
	irmovq $8, %rbx
	addq %rbx, %rdi	# src ++
	rmmovq %rcx, (%rsi)	# *dest = val
	addq %rbx, %rsi	# dest ++
	xorq %rcx, %rax	# result ^= val
	irmovq $1, %rbx
	subq %rbx, %rdx	# len --
	jmp loop
end:
	ret
{% endhighlight %}

## Part B
工作目录在 sim/seq 中。
实现 iaddq 指令，需要修改 seq-full.hcl 文件。

根据书P266页的图4-18内容，iaddq 指令可参考 OPq 与 irmovq 指令。

```
icode:ifun <-- M1[PC] # 取指 
rA:rB <-- M1[PC+1] 
valC <-- M8[PC+2] 
valP <-- PC+10

valB <-- R[rB]        # 译码

valE <-- valB + valC  # 执行

R[rB] <-- valE        # 写回
PC <-- valP           # 更新 PC

```

改写 seq-full.hcl 部分控制信号，
{% highlight c %}
 // add valid instr
 bool instr_valid = icode in 
 	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
	       IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };

 // instr need register ID
 bool need_regids =
 	icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };

 // instr need constant
 bool need_valC =
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };
	
	
 // What register should be used as the B source?
 word srcB = [
	icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ } : rB;
 	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
 	1 : RNONE;  # Don't need register
 ];

 // What register should be used as the E destination?
 word dstE = [
 	icode in { IRRMOVQ } && Cnd : rB;
	icode in { IIRMOVQ, IOPQ, IIADDQ } : rB;
 	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
 	1 : RNONE;  # Don't write any register
 ];

 // Select input A to ALU
 word aluA = [
 	icode in { IRRMOVQ, IOPQ } : valA;
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
 	icode in { ICALL, IPUSHQ } : -8;
 	icode in { IRET, IPOPQ } : 8;
 	# Other instructions don't need ALU
 ];

 // Select input B to ALU
 word aluB = [
 	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
 	icode in { IRRMOVQ, IIRMOVQ } : 0;
 	# Other instructions don't need ALU
 ];
 
 ## Should the condition codes be updated?
 bool set_cc = icode in { IOPQ, IIADDQ };
{% endhighlight %}
重新 make 并根据[官方文档](http://csapp.cs.cmu.edu/3e/archlab32.pdf)进行测试。注意，需要 make 指定版本。

由于使用的是tcl8.6版本，编译时会提示
> error: ‘Tcl_Interp’ {aka ‘struct Tcl_Interp’} has no member named ‘result’

所以需要添加#define USE_INTERP_RESULT
> GUIMODE=-DHAS_GUI -DUSE_INTERP_RESULT

另外，由于不在SunOS实验，注释这两行，保证链接成功。
{% highlight c %}
/* Hack for SunOS */
// extern int matherr();
// int *tclDummyMathPtr = (int *) matherr;
{% endhighlight %}


## Part C

未完待续。

//todo
{% highlight nasm %}

{% endhighlight %}

## Conclusions


## Reference
1. [CS:APP Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)
2. [arch lab readme](http://csapp.cs.cmu.edu/3e/README-archlab32)
3. [arch lab writeup](http://csapp.cs.cmu.edu/3e/archlab32.pdf)

