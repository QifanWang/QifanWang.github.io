---
layout: post
title:  "Avoid implicit conversion in C++"
categories: c++
tags: [code smell]
toc: true
--- 

使用C++编程时尽可能避免自定义的隐式类型转换
{: .message }

## Why explicit
在使用C++编程时, 如果调用一个函数使用错误类型参数(与函数参数类型不一致), 程序员当然希望编译时能报错.
但当expected parameter类型是一个拥有单参数构造函数的类时, 编译器可能将隐式地调用这个构造函数以传参(隐式转换); 或当错误类型有一个类型转换函数(转换类型是expected parameter类型), 编译器也可能会进行隐式类型转换.

隐式转换会使代码难以理解, 读者可能不会注意到隐式转换, 如果注意到也将产生新的疑问:

-	source type -> destination type
-	destination type -> source type
-	both?
-	which method is called by the compiler?

隐式转化弊大与利. 在 C++11 之后, 程序员可以通过 explicit 标记 single-argument constructor and conversion operator 以预防隐式类型转换.

## Note

类的 single-argument constructor and conversion operator <strong>通常</strong>需要标记 explicit, 类的 copy and move constructors 不应该被标记为 explicit.

如果 signle-constructors 参数类型是 std::initializer_list, 为了支持 copy-initialization (e.g., MyType m = {1, 2};), 我们应该避免标记 explicit.


## Bad Case
{% highlight cpp %}
struct S {
  int x;
  operator bool() const { return true; }
};

bool f() {
  S a{1};
  S b{2};
  return a == b;
}
{% endhighlight %}

由于比较时进行隐式转换, 函数f()返回 true.

## Reference


1. ["explicit" should be used on single-parameter constructors and conversion operators](https://rules.sonarsource.com/cpp/RSPEC-1709)
2. [cpp core guidelines c46 by default declare single-argument-constructors explicit](https://github.com/isocpp/CppCoreGuidelines/blob/036324/CppCoreGuidelines.md#c46-by-default-declare-single-argument-constructors-explicit)
3. [cpp core guidelines c164 avoid implicit conversion operators](https://github.com/isocpp/CppCoreGuidelines/blob/036324/CppCoreGuidelines.md#c164-avoid-implicit-conversion-operators)
4. [clang-tidy about explicit constructor](https://clang.llvm.org/extra/clang-tidy/checks/google-explicit-constructor.html)
5. [google style guide about implicit conversions](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions)
6. [cpp reference cast operator](https://en.cppreference.com/w/cpp/language/cast_operator)


