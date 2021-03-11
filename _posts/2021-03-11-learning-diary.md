---
layout: post
title:  "2021 Mar 11th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 

What you see is all you get. – Brian W. Kernighan
{: .message }

## LeetCode
[227. 基本计算器 II](https://leetcode-cn.com/problems/basic-calculator-ii/)
用双栈加优先级表解决，加入'$'符作为输入结尾便于统一处理，'$'符优先级最低(bigger)。

相比老式的atoi，对于c++的string类，我们可以用:
-	stoi
-	stoul
-	stol
-	stoll
-	stoull

string to int/unsigned long/long/long long/... 顾名思义

优先用static_cast而不是C style casting，以便在编译期检查类型。

## A Tour of C++
阅读ch6-7，ch6讲标准库总览，ch7讲string类。

### What's new

> These days, string is usually implemented using the short-string optimization. That is, short string values are kept in the string object itself and only longer strings are placed on free store.

短的放object内部，长的用object内的指针指heap里的字符串。

> The actual performance of strings can depend critically on the run-time environment.  In particular, in multi-threaded implementations, memory allocation can be relatively costly. Also, when lots of strings of differing lengths are used, memory fragmentation can result. These are the main reasons that the short-string optimization has become ubiquitous.

string类的性能很受run-time影响，所以用short-string optimization减少内存分配。

> string is really an alias for a general template basic_string with the character type char

用非character类型的字符串，关键在与如何用basic_string。

{% highlight cpp %}
template<typename Char>
class basic_string {
    // ... string of Char ...
};

using string = basic_string<char>
{% endhighlight %}

regular expression讲的大致和之前学的差不多，书里是ECMA标准里为ECMAScript所用规则。
> To  express the pattern, I use a raw string literal starting with R"( and terminated by )". This allows backslashes and quotes to be used directly in the string. Raw strings are particularly suitable for regular expressions because they tend to contain a lot of backslashes.

用 raw sring literal 表达比较方便，

{% highlight cpp %}
regex pat0 (R"(\w{2}\s∗\d{5}(−\d{4})?)"); // US postal code pattern: XXddddd-dddd and var iants
regex pat1 {"\\w{2}\\s∗\\d{5}(−\\d{4})?"}; //U.S. postal code pattern
{% endhighlight %}

> Assuming that we were not interested in the characters before the number (presumably separators), we could write: 
>   (?\s|:|,)∗(\d∗)     //spaces, colons, and/or commas followed by a number 
> This would save the regular expression engine from having to store the first characters: the (? variant has only one subpattern.

?符号取消subpattern

?符号取消greedy match，用lazy match，所以下面这个例子不会匹配两个标签（这里渲染问题，可以看原md文件）

Regular Expression: <(.∗?)>(.∗?)</\1>  //Three groups; the\1means ‘‘same as group 1’’

Text: "Always look for the <b>bright</b> side of <b>life</b>."

> Use at()rather than iterators or[]when you want range checking;
> Use iterators and []rather thanat()when you want to optimize speed;

at() 检查边界，超界抛异常；[] 不检查，超界则undefined behavior。

### Conclusion
都是比较细节的东西，一些偏琐碎的advice理解为什么这么设计就容易记住。



## Reference

1. [difference between static cast and c style casting](https://stackoverflow.com/questions/1609163/what-is-the-difference-between-static-cast-and-c-style-casting)
2. [reference of vector at()](http://www.cplusplus.com/reference/vector/vector/at/)
3. [reference of operator []](http://www.cplusplus.com/reference/vector/vector/operator[]/)


