---
layout: post
title:  "const cast后可能的undefined behavior"
categories: c++
tags: [undefined behavior]
toc: true
--- 
Modifying the const object through a reference or pointer to non-const type results in undefined behavior. Any attempt to refer to a volatile object through a glvalue of non-volatile type (e.g. through a reference or pointer to non-volatile type) results in undefined behavior.
{: .message }

在使用C++11的const_const转换后，任何修改const对象与通过指向non-volatile类型对象的指针引用去索引volatile对象的行为，都是未定义行为(Undefined behavior, UB)。

## Issue

朋友发给我一段C++代码，

```cpp
#include <iostream>
#include <type_traits>
int main() {
    volatile constexpr int c=1;
    // volatile const int c=1;
    int &rc=const_cast<int&>(c);
    int *pc=const_cast<int*>(&c);
    rc++;
    (*pc)++;
    std::cout<<rc<<" "<<*pc<<std::endl;//3 3
    std::cout<<c<<std::endl;//if constexp=>1, if const=>3
}
```

他询问为什么代码中 rc 与 pc 索引的对象值都修改了，而原来的 c 却没有修改。在他将 constexpr 修改为 const 后，对象 c 的值真实被修改了。他疑惑是不是 constexpr 修改了 volatile 的修饰。事实上，上述代码导致了UB，任何实现的行为都是不确定的。

## cv (const and volatile) type qualifiers

根据[标准](https://en.cppreference.com/w/cpp/language/cv)任何除了函数类型与引用类型之外的类型T，可以被C++的类型系统额外定义三种类型：

1) const-qualified T

2) volatile-qualified T 

3) const-volatile-qualified T

当一个对象首次创建时，声明中的 cv qualifier 决定这个对象的常量性(constness)与易失性(volatility)。简单一点说，一个 const object 不能被修改，尝试直接修改会导致编译期错误，间接修改(如借助const_cast)会导致UB。一个 volatile object 的读写操作(实际上是 every access made through a glvalue expression of volatile-qualified type)都是针对优化的可见副作用。在一个单线程执行中，volatile 读写操作不会被优化去除，在指令重排时会与其他 volatile 读写保持相对顺序(具体见memory order相关话题)。任何通过 non-volatile glvalue 索引 volatile object 的行为也是UB。一个 const volatile object 具备二者的特点。

小小解释一下 [glvalue](https://en.cppreference.com/w/cpp/language/value_category#glvalue)，其为 generalized lvalue。每一个C++的表达式都有类型与值类别。值类别一个简单的分类就是左值和右值，但在C++历史中，这一概念随着标准修改发生变化，如下图为C++11后的分类。

![value category](/images/value_category.png)

C++11引入了移动语义，值类别根据是否有有同一性(identity)和是否可移动对表达式进行分类。同一性指判断表达式代表的实体与其他表达式实体是否相同是可能的。因此分类为，
1. have identity and cannot be moved from are called lvalue expressions;
2. have identity and can be moved from are called xvalue expressions;
3. do not have identity and can be moved from are called prvalue ("pure rvalue") expressions;
4. do not have identity and cannot be moved from are not used.

只要表达式有同一性，就是 glvalue。标准中有各种值类别的详细例子与解释，具体可阅读标准。

## Hit!

回到朋友的代码，如果我们通过 type trait 检验变量 c 的 cv 属性，例如插入如下代码，

```cpp
std::cout<<"c is volatile "<<std::is_volatile<decltype(c)>::value<<std::endl;
std::cout<<"c is const "<<std::is_const<decltype(c)>::value<<std::endl;
```

可以发现无论 volatile constexpr 还是 volatile const 修饰 c 的类型，其 cv 属性都是 const-volatile-qualified 没有区别。这与标准中关于 constexpr 的[解释](https://en.cppreference.com/w/cpp/language/constexpr)显然是一致的。p.s. 如果与标准不一致都可以报编译器bug了。
> A constexpr specifier used in an object declaration or non-static member function (until C++14) implies const.

而代码行为不同的原因是两种写法都有UB，
```cpp
int &rc=const_cast<int&>(c);
int *pc=const_cast<int*>(&c);
rc++;
(*pc)++;
```
变量 c 是const volatile对象，通过rc与pc修改一个 const 对象是UB。rc与pc显然为左值且指向类型为 non-volatile，通过二者索引 volatile 对象仍然是UB。因此无论实现有何种行为，做什么事都是不确定的。

## Conclusion

如果我们阅读标准中的 [const cast](https://en.cppreference.com/w/cpp/language/const_cast) ，从其中的Notes部分与代码示例部分可以看到关于UB的描述。的确，const cast 可以去除 cv 属性，但无论是绕过 const member function 的限制还是修改引用/指针的cv限制，需要注意的仍是对象的cv属性，以保证后续行为不是UB而是正确的。对于类型转换，时刻保持警惕。

## Reference
1. [const and volatile type qualifier](https://en.cppreference.com/w/cpp/language/cv)
2. [value category glvalue](https://en.cppreference.com/w/cpp/language/value_category#glvalue)
3. [constexpr specifier](https://en.cppreference.com/w/cpp/language/constexpr)
4. [const cast](https://en.cppreference.com/w/cpp/language/const_cast)