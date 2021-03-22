---
layout: post
title:  "Avoid unsigned integer in C++"
categories: c++
tags: [code smell]
toc: true
--- 
Avoid using unsigned numbers, except in specific cases or when unavoidable.
{: .message }

## Modulo wrapping
无符号整数在C/C++中计算遵循取模原则，即使是溢出也是取模，也不同于有符号整数的溢出(undefined behavior)。

例如，无符号整数相减（补码加法），永远不会得到负数。
{% highlight cpp %}
#include <iostream>
 
int main()
{
	unsigned int x{ 3 };
	unsigned int y{ 5 };
 
	std::cout << x - y << '\n';
	// if unsigned int is 4 bytes, output: 4294967294 
	return 0;
}
{% endhighlight %}

## Remembering the terms signed and unsigned 
混合计算有符号整数与无符号整数往往会带来意外。C++为兼容C的语法，在计算表达式时会将有符号整数隐式转换为无符号数，e.g.，
{% highlight cpp %}
/* buggy code in C style*/
float sum_elements(float a[], unsigned length) {
	int i;
	float result = 0;
	
	for(i = 0; i <= lengh - 1; ++i) 
		result += a[i];
	return result;
}
{% endhighlight %}
这是来自CSAPP 3rd第二章练习2.25的例子，由于取模，当legnth传实参为0时，循环将不会结束（除非int i 可以大与等与 unsigned类型最大值），并且访问数组之外的内存。

## -2147483647-1
一个有趣的事实，在C/C++中表达32位int如果用'-2147483648'，相当于先parse数字再处理符号，这里有一个隐式转换，有些编译器会将'2147483648'处理为long类型。但根据旧标准，数字常量无法用int表示时，也会用unsigned long 类型。虽然这里直接赋值时确实是最小数（long -> int truncate），但面对隐式转换，仍需要谨慎。
我的机器上，在stdint.h中，有
{% highlight c %}
# define INT32_MIN		(-2147483647-1)
{% endhighlight %}

工程上不用深入这些繁琐细节，C++ Style 表示类型范围可以用头文件limits里的numeric_limits模板类，更便捷。

## Conclusion
- 无符号整数取模溢出，且常常潜藏类型转换
- 绝不要在C/C++中用无符号类型去保证一个非负输入（变量/函数参数等），use assertion or exception
- 除非我们需要bit pattern and bit manipulation(位表示和位操作)，或是需要取模算术，或是嵌入式系统编程(资源紧缺)，否则，应当谨慎使用并最好不用无符号类型。

## Reference
1. [Google C++ Style: Integer Types](https://google.github.io/styleguide/cppguide.html#Integer_Types)
2. [C++ Reference: Implicit Conversion](https://en.cppreference.com/w/cpp/language/implicit_conversion)
3. [Learn C++: Unsigned Integer](https://www.learncpp.com/cpp-tutorial/unsigned-integers-and-why-to-avoid-them/)
