---
layout: post
title:  "Effective C++ 3rd Chapter 7: Templates and Generic Programming"
categories: c++
tags: [reading, effective]
toc: true
--- 
Beauty lives with kindness. - William Shakespeare
{: .message }

阅读Effective C++ 3rd(2005) 第七章。

## Item 41: Understand implicit interfaces and compile-time polymorphism.
模板 Template 定义的是隐式接口，接口约束基于表达式，通过模板实例化与函数重载实现编译时多态。

## Item 42: Understand the two meanings of typename.
关键字 typename 在模板参数列表中可以和 class 一样声明类型参数，但它可以额外用于声明 dependent qualified name 为类型名[1]。

但需要注意 dependent qualified type(nested dependent type) 的情况，例如，声明一个 dependent qualified type 的变量 temp ,
{% highlight cpp %}
template<typename T>
class Derived: public Base<T>::Nested { // base class list: typename not allowed
public:
	explicit Derived(int x)
	: Base<T>::Nested(x) // base class identifier in mem init list: typename not allowed
	{
		// use of nested dependent type
		// name not in a base class list or
		// as a base class identifier in a
		// mem. init. list: typename required
		typename Base<T>::Nested temp;
		...
	}
	...
};
{% endhighlight %}

为了避免长长的类型名，可以用 typedef 定义一个短类型名，在C++11也可用 using 。
{% highlight cpp %}
template<typename IterT>
void workWithIterator(IterT iter)
{
	typedef typename std::iterator_traits<IterT>::value_type value_type;
	value_type temp(*iter);
	...
}
{% endhighlight %}

## Item 43: Know how to access names in templatized base classes.
当OOP的继承与模板产生交集，出现继承一个模板类的情况时，基模板类的 name (函数名，成员名等)无法直接获取(由于存在模板完全特化的写法，编译器不会在通用模板里查找基类 name)。
{% highlight cpp %}
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
...
void sendClearMsg(const MsgInfo& info)
{
	//write "before sending" info to the log;
	sendClear(info); // if Company == CompanyZ, this function doesn’t exist! 
	//write "after sending" info to the log;
}
...
};
{% endhighlight %}

如sendClear函数是基类的成员函数，基类模板的实例化需要Company有某个函数，但CompanyZ类可能没有该个函数。编译器会在 parsing 阶段就报告错误，而不是在实例化LoggingMsgSender时报错。

解决方法有通过 this 指针显式调用 sendClear，通过 using 声明 sendClear 函数或 qualified 显式调用 sendClear 函数，告知编译器需要继承 sendClear 函数。

这是一个生僻但有用的例子，告诉我们如何获取模板化基类中的 name 。

## Item 44: Factor parameter-independent code out of templates.
模板实例化，仅为真正使用的类型实例化。

模板实例化带来的 code replication 是隐式的，如果需要注意生成的代码大小，则需要注意模板中独立于模板参数的代码(这部分代码每一个实例化都会生成)。

模板非类型参数带来的 code bloat ，可通过替换参数为函数参数或数据成员来减轻。但有时引入模板基类也会带来减少编译器优化和关于指针的心智负担，需要针对具体平台具体分析。

模板类型参数带来的 code bloat，可以借助所有指针为相同的二进制表示，进行指针类型的模板实例化。这也需要针对具体平台具体分析。

## Item 45: Use member function templates to accept “all compatible types.”
Middle 与 Top 拥有继承关系，但实例化后，
{% highlight cpp %}
SmartPtr<Middle> 

SmartPtr<Top>
{% endhighlight %}
则没有继承关系。

但有时我们希望实现一些类似"继承"的操作，如基类指针指向派生类或者隐式转换，于是可以为这些函数设计函数模板接受特定类型的参数，
{% highlight cpp %}
template<typename T>
class SmartPtr {
public:
	template<typename U>
	SmartPtr(const SmartPtr<U>& other) // initialize this held ptr with other’s held ptr
	: heldPtr(other.get()) { ... }
	T* get() const { return heldPtr; }
	...
private:
	T *heldPtr; // built-in pointer held by the SmartPtr
};
{% endhighlight %}

这类复制构造函数可称为 generalized copy constructor ，不属于 Item 5 所说的编译器自动生成的函数，如果需要"抑制"编译器自动生成行为，还需要自定义这些函数。如，
{% highlight cpp %}
template<class T> class shared_ptr {
public:
	shared_ptr(shared_ptr const& r); // copy constructor
	
	template<class Y>
	shared_ptr(shared_ptr<Y> const& r); // generalized copy constructor
	
	shared_ptr& operator=(shared_ptr const& r); // copy assignment
	
	template<class Y>
	shared_ptr& operator=(shared_ptr<Y> const& r); // generalized copy assignment
	...
};
{% endhighlight %}

## Item 46: Define non-member functions inside templates when type conversions are desired.
函数模板不能支持参数的隐式转换，
> implicit type conversion functions are never considered during template argument deduction. Never. Such conversions are used during function calls,

如果需要实现参数的隐式转换，可以在类模板中声明且定义友元函数(因为需要支持所有参数都可隐式转换)以实现。

需要注意的是，这个友元函数需要定义在类中，否则会有 link error 。

### have the friend call a helper 
另外，为了实现 inline 不被"拒绝"，如果这个友元函数太长，可以将语句代替为调用一个模板函数，模板函数作为 helper 提供实现。

## Item 47: Use traits classes for information about types.
traits 提供编译时类型信息。
> traits let you do: they allow you to get information about a type during compilation.

traits 是实现思想，这个类型信息是类型之外(external)的信息，实现上可以通过一个模板类完成，如 STL 中的 iterator_traits，
{% highlight cpp %}
template<typename IterT> // template for information about
struct iterator_traits;  // iterator types
{% endhighlight %}

design and implement a traits class:
- Identify some information about types you’d like to make available (e.g., for iterators, their iterator category).
- Choose a name to identify that information (e.g., iterator_category ).
- Provide a template and set of specializations (e.g., iterator_traits ) that contain the information for the types you want to support.

如何定义与使用 traits 并用 tag 作为参数得到编译时 traits 信息。
- Create a set of overloaded “worker” functions or function templates (e.g., doAdvance ) that differ in a traits parameter. Implement each function in accord with the traits information passed.
- Create a “master” function or function template (e.g., advance ) that calls the workers, passing information provided by a traits class.

上述具体实现可以看书和 STL 的 iterator_traits ，这段实在经典！

## Item 48: Be aware of template metaprogramming.
模板元编程 Template metaprogramming (TMP) 本质是编译期计算/执行，其结果为模板实例化的生成代码。
> a template metaprogram is a program written in C++ that executes inside the C++ compiler. When a TMP program finishes running, its output — pieces of C++ source code instantiated from templates — is then compiled as usual.

递归对于模板元编程十分重要，
> TMP loops don’t involve recursive function calls, they involve recursive template instantiations.

TMP 应用的三个场景，
- Ensuring dimensional unit correctness. 编译时检查类型间的限定关系，保证正确性。
- Optimizing matrix operations. 通过 expression templates 可以优化临时变量与合并循环，如连乘，使最终程序使用更少内存且运行效率更高。
- Generating custom design pattern implementations. 一些设计模式如 Strategy, Observer, Visitor等，可以通过 policy-based design 技巧，创造模板代表独立的 design choices(policies)，这也被称作是 generative programming 。

TMP 功能强大，适合库的开发者。

## Conclusions
Item 41:
- Both classes and templates support interfaces and polymorphism.
- For classes, interfaces are explicit and centered on function signatures. Polymorphism occurs at runtime through virtual functions.
- For template parameters, interfaces are implicit and based on valid expressions. Polymorphism occurs during compilation through template instantiation and function overloading resolution.

Item 42:
- When declaring template parameters, class and typename are interchangeable.
- Use typename to identify nested dependent type names, except in base class lists or as a base class identifier in a member initialization list.

Item 43:
- In derived class templates, refer to names in base class templates via a “ this-> ” prefix, via using declarations, or via an explicit base class qualification.

Item 44:
- Templates generate multiple classes and multiple functions, so any template code not dependent on a template parameter causes bloat.
- Bloat due to non-type template parameters can often be eliminated by replacing template parameters with function parameters or class data members.
- Bloat due to type parameters can be reduced by sharing implementations for instantiation types with identical binary representations.

Item 45:
- Use member function templates to generate functions that accept all compatible types.
- If you declare member templates for generalized copy construction or generalized assignment, you’ll still need to declare the normal copy constructor and copy assignment operator, too.

Item 46:
- When writing a class template that offers functions related to the template that support implicit type conversions on all parameters, define those functions as friends inside the class template.

Item 47:
- Traits classes make information about types available during compilation. They’re implemented using templates and template specializations.
- In conjunction with overloading, traits classes make it possible to perform compile-time if...else tests on types.

Item 48:
- Template metaprogramming can shift work from runtime to compile-time, thus enabling earlier error detection and higher runtime performance.
- TMP can be used to generate custom code based on combinations of policy choices, and it can also be used to avoid generating code inappropriate for particular types.

本章讲模板和泛型编程。比较新的点是，typename 声明 dependent qualified type，继承模板基类的 name 问题，智能指针的复制构造，traits的定义与使用技巧。

## References
1. [C++ references keywords: typename](https://en.cppreference.com/w/cpp/keyword/typename)

