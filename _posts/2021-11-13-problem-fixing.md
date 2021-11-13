---
layout: post
title:  "Problem fixing notes"
categories: c++
tags: [miscellaneous , overload resolution , template meta programming ]
toc: true
--- 
Faith is the bird that feels the ligh when the dawn is still dark.
{: .message }

好友做 LeetCode 时出现的问题：在调用标准库的 transform 函数时，模板实例化出错(LC的编译器只给出"推导出错导致模板candidates无一可用"的信息)，但使用 `::toupper` 传入函数却没有问题。
问题简化代码如下， `foo` 函数将一个字符串所有小写字母转成大写，
```cpp
std::string foo(std::string s) {
    // std::transform(s.begin(), s.end(), s.begin(), toupper); // compilation error 
    // std::transform(s.begin(), s.end(), s.begin(), std::toupper); // compilation error 
    std::transform(s.begin(), s.end(), s.begin(), ::toupper); // correct
    return s;
}
```

## Safe usage

查看标准，[toupper function notes](https://en.cppreference.com/w/cpp/string/byte/toupper#Notes)记载如果 `toupper` 的实参的值不是 `unsigned char` 表示范围或 `EOF` 的话，会出现 UB , 安全的使用形式是将参数转成 `unsigned char` 类型，如

```cpp
char my_toupper(char ch)
{
    return static_cast<char>(std::toupper(static_cast<unsigned char>(ch)));
}

std::string str_toupper(std::string s) {
    std::transform(s.begin(), s.end(), s.begin(), 
                // static_cast<int(*)(int)>(std::toupper)         // wrong
                // [](int c){ return std::toupper(c); }           // wrong
                // [](char c){ return std::toupper(c); }          // wrong
                   [](unsigned char c){ return std::toupper(c); } // correct
                  );
    return s;
}
```

但实际上， notes 示例中 wrong 的使用方式是不会导致编译错误的，只是有 UB 的风险(官方的示例"貌似"没有UB风险，但如果考虑不是 `basic_string<char>` 的其他 string 类型就需要小心了)。

## Name lookup

暂时先忽略 Safe usage 的考虑，回到编译错误上，由于 Leetcode 允许代码中无 scope resolution operator `::`，如没有 `std::` 就可直接调用 std 命名空间里的函数，我猜测是做了类似算法竞赛中"万能"头文件的处理，

```cpp
#include <bits/stdc++.h>

using namespace std;
```

所以有可能是重复名字导致 `transform` 函数模板实例化失败。于是我在 LC 上进行如下实验，

```cpp
  auto f = function<int(int)>(std::toupper);
// LC error msg
// Line 4: Char 37: error: address of overloaded function 'toupper' does not match required type 'std::function<int (int)>'
//         auto f = function<int(int)>(std::toupper);
//                                     ^~~~~~~~~~~~
// /usr/include/ctype.h:125:12: note: candidate function
// extern int toupper (int __c) __THROW;
//            ^
// /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/bits/locale_facets.h:2643:5: note: candidate template ignored: could not match '_CharT (_CharT, const std::locale &)' against 'std::function<int (int)>'
//     toupper(_CharT __c, const locale& __loc)
//     ^
// 1 error generated.
```
根据以上的报错信息可以看出， 命名空间 std 里 `toupper` 的重载会导致模板实例化出错。更为准确的[说法](https://en.cppreference.com/w/cpp/language/function_template)是，

> Template argument deduction takes place after the function template name lookup (which may involve argument-dependent lookup) and before overload resolution.

在 name lookup 后找到两个 candidates , 然后模板参数推导时发生错误。

```cpp
  auto h = function<int(int)>(toupper);
// LC error msg
// Line 6: Char 37: error: address of overloaded function 'toupper' does not match required type 'std::function<int (int)>'
//         auto h = function<int(int)>(toupper);
//                                     ^~~~~~~
// /usr/include/ctype.h:125:12: note: candidate function
// extern int toupper (int __c) __THROW;
//            ^
// /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/bits/locale_facets.h:2643:5: note: candidate template ignored: could not match '_CharT (_CharT, const std::locale &)' against 'std::function<int (int)>'
//     toupper(_CharT __c, const locale& __loc)
//     ^
// 1 error generated.
```
使用 Unqualified name lookup 也会导致相同的问题，

```cpp
  auto g = function<int(int)>(::toupper); // compilation success
```
但使用这种形式 Qualified name lookup 能够正确寻找。根据[标准](https://en.cppreference.com/w/cpp/language/qualified_lookup)中所述，左边为空的 `::` 是查找 global namespace scope(aka file/global scope) 中的名字。

C库的 `toupper` 既由于C的头文件 `ctype.h` 被 include 所以被加入 global space ，又因为 C++ 标准库 `cctype` 被添加入了 std namespace 中。[标准](https://en.cppreference.com/w/cpp/header/cctype) 中描述了，也可查看本地机器上的 `cctype` 文件。

所以我的猜测是， LC 对提交代码在 Solution 类中使用了 using directives ，所以所有 std namespace 均可见。不管是 `std::toupper` 还是 unqualified name lookup 都会查找 std namespace 中的名字，并由于重载导致编译出错。但使用 `::toupper` 可以查找到本位于 std 命名空间外的 `toupper` 函数。

## Conclusion

熟悉C++ Name lookup 规则， 尤其是关于命名递归式查找，一旦找到停止递归的规则。[规则描述](https://en.cppreference.com/w/cpp/language/qualified_lookup#Namespace_members)

## Reference
1. [toupper function](https://en.cppreference.com/w/cpp/string/byte/toupper#Notes)
2. [cctyp 标准概要，最好看本地头文件](https://en.cppreference.com/w/cpp/header/cctype)
3. [scope](https://en.cppreference.com/w/cpp/language/scope)
4. [unqualified name lookup](https://en.cppreference.com/w/cpp/language/unqualified_lookup)
5. [qualified name lookup](https://en.cppreference.com/w/cpp/language/qualified_lookup#Namespace_members)
