---
layout: post
title:  "2021 Mar 13th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 
Do not multiply entities beyond necessity.
				–William Occam
{: .message }

## LeetCode
[705. 设计哈希集合](https://leetcode-cn.com/problems/design-hashset/)
用 vector<list<int>> 模拟哈希表，发生哈希冲撞直接添加入链表。按照一些标准HashMap实现,链表增长到一定长度会构造为一个红黑树，例如Java的HashMap。

### cpp technique
有一个名为[Erase-remove idiom]((https://en.wikipedia.org/wiki/Erase%E2%80%93remove_idiom))的技巧可用于删除C++容器内的元素。当然，我发现c++20以后一些容器内部也陆续添加了erase与erase_if成员方法。在c++20以前这个技巧还十分有用。

[Algorithm remove](https://en.cppreference.com/w/cpp/algorithm/remove)
> Removing is done by shifting (by means of move assignment) the elements in the range in such a way that the elements that are not to be removed appear in the beginning of the range. Relative order of the elements that remain is preserved and the physical size of the container is unchanged. Iterators pointing to an element between the new logical end and the physical end of the range are still dereferenceable, but the elements themselves have unspecified values (as per MoveAssignable post-condition). A call to remove is typically followed by a call to a container's erase method, which erases the unspecified values and reduces the physical size of the container to match its new logical size.

这个技巧本质上是因为 move 元素会导致序列尾部出现一段 unspecified value，通过 erase 去除之并将容器的size缩小到逻辑大小。

删除字符串里的空格或空白符，e.g.
{% highlight cpp %}
#include <algorithm>
#include <string>
#include <iostream>
#include <cctype>
 
int main()
{
    std::string str1 = "Text with some   spaces";
    str1.erase(std::remove(str1.begin(), str1.end(), ' '),
               str1.end());
    std::cout << str1 << '\n';
 
    std::string str2 = "Text\n with\tsome \t  whitespaces\n\n";
    str2.erase(std::remove_if(str2.begin(), 
                              str2.end(),
                              [](unsigned char x){return std::isspace(x);}),
               str2.end());
    std::cout << str2 << '\n';
}
{% endhighlight %}

## A Tour of C++ 1st
阅读ch10，讲标准库里algorithm

### What's new

> std::back_insert_iterator is a LegacyOutputIterator that appends to a container for which it was constructed. The container's push_back() member function is called whenever the iterator (whether dereferenced or not) is assigned to. Incrementing the std::back_insert_iterator is a no-op.

back_insert()帮助构造back_insert_iterator，这个迭代器每次被赋值时，对应容器的push_back()函数被调用。

{% highlight cpp %}
#include <iostream>
#include <vector>
#include <algorithm>
#include <iterator>
 
int main()
{
  std::vector<int> v;
  std::generate_n(
    std::back_insert_iterator<std::vector<int>>(v), // C++17: std::back_insert_iterator(v)
    10, [n=0]() mutable { return ++n; }             // or use std::back_inserter helper
  );
 
  for (int n : v)
    std::cout << n << ' ';
  std::cout << '\n';
  // Output: 1 2 3 4 5 6 7 8 9 10
}

void foo2()
{
    std::vector<int> v{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::fill_n(std::back_inserter(v), 3, -1);
    for (int n : v)
        std::cout << n << ' ';
    // Output: 1 2 3 4 5 6 7 8 9 10 -1 -1 -1
}
{% endhighlight %}

Iterator是一个很好的设计，用于分离算法与容器，即算法通过特定迭代器(Iterator类别有不同)操作容器。既通用了接口(Iterator)，又特定了接口(Iterator类别)。具体可看[c++ reference iterator library](https://en.cppreference.com/w/cpp/iterator)

Iterator is a concept(c++20).

c++20之前我们用[named requirements](https://en.cppreference.com/w/cpp/named_req)描述类似concept的想法，我称之为具名要求。

比较简单的概念，predicate类似用于判定的functor，和JAVA里相似(从函数式里抄的，应该都一样)。

### Conclusion
配着标准看书，algorithm许多东西都可多用多写以熟悉。

## Reference
1. [wiki erase-remove idiom](https://en.wikipedia.org/wiki/Erase%E2%80%93remove_idiom)
2. [c++ reference algorithm remove](https://en.cppreference.com/w/cpp/algorithm/remove)
3. [c++ reference iterator back_inserter](https://en.cppreference.com/w/cpp/iterator/back_inserter)
4. [c++ reference iterator back_insert_iterator](https://en.cppreference.com/w/cpp/iterator/back_insert_iterator)
5. [c++ reference iterator library](https://en.cppreference.com/w/cpp/iterator)
6. [c++ reference named requirements](https://en.cppreference.com/w/cpp/named_req)

