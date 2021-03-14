---
layout: post
title:  "2021 Mar 14th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 
Nothing is true, everything is permitted. ―The Creed's maxim.
{: .message }

## LeetCode
[706. 设计哈希映射](https://leetcode-cn.com/problems/design-hashmap/)
用 vector<list<pair<int,int>>> 模拟HashMap，发生哈希冲撞直接添加入链表。和昨天题目类似。用 std::find_if 查找key值相同的pair。

## ISOC++ wiki
今天单纯读[wiki](https://isocpp.org/wiki/faq/virtual-functions),把每个FAQ总结一下。

### What is a “virtual member function”? 

virtual 成员函数是实现面向对象范式的核心，方便代码扩展(old code to call new code)。

virtual function 使基类成员函数的实现可以被派生类替换。如所谓的基类指针/引用访问派生类实例成员方法。

实现上，既可以完全替换(override)，也可以部分替换(argument)，后者是通过call基类方法实现的。

### Why are member functions not virtual by default?

不是所有类都需要成为基类。

并且有虚函数的类的对象额外需要一个机器字长的空间(实现虚函数调用机制的虚指针，其指向虚表)。这个开销可能是很大的，并且会妨碍与其他语言(如C和Fortran)的数据兼容。

### How can C++ achieve dynamic binding yet also static typing?

当一个指针指向一个对象，这个对象可能是一个指针应指类型的派生类(e.g., a Vehicle* pointing to a Car obejct)。指针类型是静态的，但被指类型是动态的(可以是其他派生类)。

> Static typing means that the legality of a member function invocation is checked at the earliest possible moment: by the compiler at compile time. The compiler uses the static type of the pointer to determine whether the member function invocation is legal. If the type of the pointer can handle the member function, certainly the pointed-to object can handle it as well. 

Static typing 意思是函数调用尽可能早检查，即编译期。编译器用指针的类型决定成员函数的调用是否合法。

> Dynamic binding means that the address of the code in a member function invocation is determined at the last possible moment: based on the dynamic type of the object at run time. It is called “dynamic binding” because the binding to the code that actually gets called is accomplished dynamically (at run time). Dynamic binding is a result of virtual functions.

Dynamic binding 意为成员函数调用的代码地址尽可能晚确定，即运行时。代码被绑定是动态完成的，这也是虚函数的结果。

### What is a pure virtual function?

> A pure virtual function is a function that must be overridden in a derived class and need not be defined. A virtual function is declared to be “pure” using the curious =0 syntax. 

这个派生类必须实现其实有歧义。因为派生类可以不实现虚函数，让自身也成为一个抽象类。

{% highlight cpp %}
    class Base {
    public:
        void f1();      // not virtual
        virtual void f2();  // virtual, not pure
        virtual void f3() = 0;  // pure virtual
    };
    Base b; // error: pure virtual f3 not overridden
{% endhighlight %}

Base是一个抽象类(因为有一个纯虚函数)，不能直接创建示例。

{% highlight cpp %}
    class Derived : public Base {
        // no f1: fine
        // no f2: fine, we inherit Base::f2
        void f3();
    };
    Derived d;  // ok: Derived::f3 overrides Base::f3
{% endhighlight %}

抽象类用于定义接口。一个只包含纯虚函数的类通常被称为接口。

我们也可以为纯虚函数提供定义：
{% highlight cpp %}
    Base::f3() { /* ... */ }
{% endhighlight %}

这有时十分有用，可以为派生类提供一些共同的实现。但Base::f3()在派生类中<strong>仍需overridden</strong>。否则派生类也是抽象类。

>  If you don’t override a pure virtual function in a derived class, that derived class becomes abstract.

{% highlight cpp %}
    class D2 : public Base {
        // no f1: fine
        // no f2: fine, we inherit Base::f2
        // no f3: fine, but D2 is therefore still abstract
    };
    D2 d;   // error: pure virtual Base::f3 not overridden
{% endhighlight %}

### Conclusion
纯虚函数是否定义与重写那块容易搞错，明日继续。

## Reference
1. [ISO cpp wiki FAQ Inheritance — virtual functions](https://isocpp.org/wiki/faq/virtual-functions)
