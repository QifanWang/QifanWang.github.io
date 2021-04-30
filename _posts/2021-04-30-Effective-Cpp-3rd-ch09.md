---
layout: post
title:  "Effective C++ 3rd Chapter 9: Miscellany"
categories: c++
tags: [reading, effective]
toc: true
--- 
To realise the value of one minute, ask the traveler who has just missed his train.
{: .message }

阅读Effective C++ 3rd(2005) 第九章。完成阅读本书。

## Item 53: Pay attention to compiler warnings.
重视编译器开最高 Warn level 后的所有 warnings，理解 warning message 的具体意思。

不要写依赖编译器 warning 的代码，因为这本质上是 dependent on implementation , 尽量让代码 Portable 且在最高警告级别下 warning-free 。

## Item 54: Familiarize yourself with the standard library, including TR1.
熟悉标准库与TR1内容，其实 TR1 已经包含在 C++11 的标准内容中。

C++98 标准库主要内容
- The Standard Template Library (STL), including containers (vector, string, map, etc.); iterators; algorithms (find, sort, transform, etc.); function objects (less, greater, etc.); and various container and function object adapters (stack, priority_queue, mem_fun, not1, etc.).
- Iostreams, including support for user-defined buffering, internationalized IO, and the predefined objects cin, cout, cerr, and clog.
- Support for internationalization, including the ability to have multiple active locales. Types like wchar_t (usually 16 bits/char) and wstring (strings of wchar_ts) facilitate working with Unicode.
- Support for numeric processing, including templates for complex numbers (complex) and arrays of pure values (valarray).
- An exception hierarchy, including the base class exception, its derived classes logic_error and runtime_error, and various classes that inherit from those.
- C89’s standard library. Everything in the 1989 C standard library is also in C++.

之前书中已经介绍过的TR1特性有，
- smart pointers
- tr1::function
- tr1::bind

TR1提供的比较独立的特性有，
- Hash tables used to implement sets, multisets, maps, and multimaps.
- Regular expressions, including the ability to do regular expression-based search and replace operations on strings, to iterate
through strings from match to match, etc.
- Tuples, a nifty generalization of the pair template that’s already in the standard library.
- tr1::array, essentially an “STLified” array, i.e., an array supporting member functions like begin and end. 
- tr1::mem_fn, a syntactically uniform way of adapting member function pointers.
- tr1::reference_wrapper, a facility to make references act a bit more like objects.
- Random number generation facilities that are vastly superior to the rand function that C++ inherited from C’s standard library.
- Mathematical special functions, including Laguerre polynomials, Bessel functions, complete elliptic integrals, and many more.
- C99 compatibility extensions, a collection of functions and templates designed to bring many new C99 library features to C++.

TR1也提供了一些模板元编程的特性，
- Type traits.
- tr1::result_of

使用 boost 提供的 TR1 实现需要注意 name space 问题。

## Item 55: Familiarize yourself with Boost.
介绍 Boost 库，以及向 Boost 贡献代码的相关知识。

## Conclusions
Item 53:
- Take compiler warnings seriously, and strive to compile warning-free at the maximum warning level supported by your compilers.
- Don’t become dependent on compiler warnings, because different compilers warn about different things. Porting to a new compilermay eliminate warning messages you’ve come to rely on.

Item 54:
- The primary standard C++ library functionality consists of the STL, iostreams, and locales. The C89 standard library is also included.
- TR1 adds support for smart pointers (e.g., tr1::shared_ptr), generalized function pointers (tr1::function), hash-based containers, regular expressions, and 10 other components.
- TR1 itself is only a specification. To take advantage of TR1, you need an implementation. One source for implementations of TR1 components is Boost.

Item 55:
- Boost is a community and web site for the development of free, open source, peer-reviewed C++ libraries. Boost plays an influential role in C++ standardization.
- Boost offers implementations of many TR1 components, but it also offers many other libraries, too.

 
本章讲定制一些杂项，比较简单，至此完成 Effective C++ 3rd(2005) 的全部内容。

## References
1. [Boost](https://www.boost.org/)
