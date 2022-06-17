---
layout: post
title:  "虚函数调用与分支预测"
categories: c++
tags: [architecture]
toc: true
--- 
Beware of indirect branch prediction.
{: .message }

C++代码调用一个虚函数时，由于部分处理器的间接分支预测（Indirect branch prediction）机制，可能会带来运行时的预测错误惩罚。本文简单说明了一下相关问题。

## Issue

C++的虚函数机制大部分实现都是通过类中插入虚指针，为虚函数建立虚表完成的。朋友转发了一篇[zhihu文章](https://zhuanlan.zhihu.com/p/347996802)，其中提到虚函数调用除了通过虚指针跳转带来的时间开销与虚指针和虚表带来的空间开销外，还有分支预测失败带来的流水刷新惩罚(penalty):
>总结来说，虚函数产生性能差距的原因主要是分支预测失败导致的流水线冲刷开销。对于一些性能敏感的程序，在虚函数可用可不用的时候，可以考虑不使用虚函数以提高性能。

回想本科时的计组课程，当时我们上课提到的分支预测都是简单的true/false预测，其中实验还需要我们实现一个记忆上次预测结果的状态机，以实现流水线的分支预测。因此我产生疑惑，虚函数机制为什么与分支预测有关。

## Indirect branch predictor

原文章语焉不详，但在 Stack Overflow 上找到[相关的问答](https://stackoverflow.com/questions/7241922/how-has-cpu-architecture-evolution-affected-virtual-function-call-performance)。该问题提到的间接分支(indirect branch)是一种特殊的跳转指令，其跳转的目的地址值在寄存器中。[wiki](https://en.wikipedia.org/wiki/Indirect_branch)还提到这类型指令可以用于构建多路分支，比如一个jump table里每行存储地址值，将条件计算结果存入寄存器以确定跳转地址。一些子程序调用指令也是间接的，函数指针也是一种间接调用的机制。

以下为一些指令集的间接分支示例，

ISA | instruction
---|---
MSP430 | br r15
SPARC | jmpl %o7
MIPS | jr $ra
X86 (AT&T Syntax) | jmp *%eax
X86 (Intel Syntax) | jmp eax
ARM | mov pc, r2
Itanium (x86 family) | br.ret.sptk.few rp
6502 | jmp ($0DEA)
65C816 | jsr ($0DEA,X)
6809 | jmp [$0DEA], jmp B,X, jmp [B,X]
6800 | jmp 0,X
Z80 | jp (hl)
Intel 8080 | pchl
IBM System z | bcr cond,r1[2]
RISC-V | jalr x0, 0(x1)

处理器会实现[Indirect branch predictor](https://en.wikipedia.org/wiki/Branch_predictor#Indirect_branch_predictor)，有些是简单预测为上一次间接跳转的地址，有些使用两级自适应预测器与历史缓存，还有指令集支持“预加载对应指令分支预测结果”。因此，间接分支预测失败也会带来刷新流水线的惩罚。

## Conclusion

虚函数实现机制(大部分是)通过虚指针指向虚表，在运行时计算偏移得到具体函数的地址。这类似计算一个函数指针的值，再调用其指向的函数。从上述讨论中可以发现，只要有间接分支指令与相应的预测机制，都有可能引入间接分支预测的失败，若函数指针或switch语句编译后(在csapp某个lab的汇编代码见过)也引入了间接分支指令，那么也可能导致刷新流水线。问题本质在于是否有间接分支指令与处理器的预测。

## Reference
1. [C++ 虚函数详解](https://zhuanlan.zhihu.com/p/347996802)
2. [How has cpu architecture evolution affected virtual function call performance?](https://stackoverflow.com/questions/7241922/how-has-cpu-architecture-evolution-affected-virtual-function-call-performance)
3. [Indirect branch](https://en.wikipedia.org/wiki/Indirect_branch)
4. [Indirect branch predictor](https://en.wikipedia.org/wiki/Branch_predictor#Indirect_branch_predictor)
