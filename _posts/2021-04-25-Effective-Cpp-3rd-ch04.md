---
layout: post
title:  "Effective C++ 3rd Chapter 4"
categories: c++
tags: [reading, effective]
toc: true
--- 
Nature and human life are as various as our several constitutions.
{: .message }

阅读Effective C++ 3rd(2005) 第四章。


## Item 18: Make interfaces easy to use correctly and hard to use incorrectly.
利用类型系统
> the type system is your primary ally in preventing undesirable code from compiling.
 
保持接口的consistency，例如 STL 中的容器均有 size()。保持接口和 built-in type 兼容。

用智能指针管理资源，使用shared_ptr 并定义 deleter 以解决 cross DLL problem.
> This problem crops up when an object is created using new in one dynamically linked library (DLL) but is deleted in a different DLL. On many platforms, such cross-DLL new/delete pairs lead to runtime errors.

在申请资源时就想好如何回收。

## Item 19: Treat class design as type design.
将类的设计当作类型设计，并考虑如下问题，
- How should objects of your new type be created and destroyed?
- How should object initialization differ from object assignment?
- What does it mean for objects of your new type to be passed by value?
- What are the restrictions on legal values for your new type?
- Does your new type fit into an inheritance graph?
- What kind of type conversions are allowed for your new type?
- What operators and functions make sense for the new type?
- What standard functions should be disallowed?
- Who should have access to the members of your new type?
- What is the “undeclared interface” of your new type?
- How general is your new type?
- Is a new type really what you need?

## Item 20: Prefer pass-by-reference-to-const to pass-by-value.
没有move语义时，默认函数传值与返回，都是copy。所以引用作为参数。

用基类指针或引用解决 传值时的slicing problem(派生类实参而基类形参时，传值仅构造基类)。

传值更适用于 built-in type 还有 STL 中的iterator 与 function object，这些类型为传值而设计，复制高效且不会出现slicing problem

类小并不代表，复制时开销就小。

## Item 21: Don’t try to return a reference when you must return an object.
函数返回引用可能会造成返回了一个栈上对象的引用。
> The fact is, any function returning a reference to a local object is broken. (The same is true for any function returning a pointer to a local object.)

返回堆上对象的引用，又会带来何处delete的问题，造成资源泄漏。

至于返回局部静态对象的引用，不仅会带来线程安全问题，如果程序中需要多个该对象进行使用，还会带来比较时的问题。更是错误。

C++11 之后有移动语义就方便许多。

## Item 22: Declare data members private.
成员函数可以提供对于数据成员更精准的权限。

通过成员函数保持invariants,更直观和简单。

成员函数通过封装可以根据程序场景使用不同的实现。

不用protected理由除了保持consistency，还有数据成员一旦修改(删除)所有用到的代码都需要修改。protected数据成员在继承类可以使用，这也是很大的代码量。不如通过成员函数读写。

## Item 23: Prefer non-member non-friend functions to member functions.
使用 non-member non-friend function 是为了保证封装性。

可以将 non-member non-friend function 放入特定的 namespace 中。而这个 namespace 又可以根据用途类别分成不同的头文件。未来扩展功能，就可以不断在新文件中扩展namespace的函数。

## Item 24: Declare non-member functions when type conversions should apply to all parameters.
如定义 Rational 类想支持整型与浮点型在计算时的隐式转换，需要用 non-member function进行传参时的隐式转换。

## Item 25: Consider support for a non-throwing swap.
对于复制开销大的类, 可以用一个类的private字段指向它(pimpl idiom, pointer to implementation)，swap时用pimpl类进行交换指针。实现时必须保证swap不会抛出异常。

用成员函数，定义pimpl类的交换，
{% highlight cpp %}
class Widget {
public:
	...
	void swap(Widget& other)
	{
		using std::swap;	// avoid qualifying swap function
		swap(pImpl, other.pImpl);
	}
	...
};
{% endhighlight %}

如果不模板化Widget，则可以于std命名空间完全特化swap函数模板,
{% highlight cpp %}
class Widget {
public:
	...
	void swap(Widget& other)
	{
		using std::swap;
		swap(pImpl, other.pImpl);
	}
	...
};
namespace std {
	template<>
	void swap<Widget>(Widget& a, Widget& b)
	{
		a.swap(b);
	}
}
{% endhighlight %}

但当需要模板化Widget时，因为标准规定，可以在std name space里完全特化模板'templace<>'，但不能加新模板。为此需要自定义命名空间，
{% highlight cpp %}
namespace WidgetStuff {
	...
	template<typename T>
	class Widget { 
		void swap(Widget& other)
		{
			using std::swap;
			swap(pImpl, other.pImpl);
		}
	... 
	};
	...
	template<typename T>
	void swap(Widget<T>& a, Widget<T>& b)
	{
		a.swap(b);
	}
}
{% endhighlight %}

需要注意的是，在其他地方进行模板并使用swap时，为了Widget也能泛化且有可能调用特化的swap函数，使用如下形式
{% highlight cpp %}
template<typename T>
void doSomething(T& obj1, T& obj2)
{
	using std::swap; // make std::swap available in this function
	...
	swap(obj1, obj2); // call the best swap for objects of type T
	...
}
{% endhighlight %}
这得益于C++的命名搜索策略，
> C++’s name lookup rules ensure that this will find any T-specific swap at global scope or in the same namespace as the type T . (For example, if T is Widget in the namespace WidgetStuff, compilers will use argument-dependent lookup to find swap in WidgetStuff.) If no T-specific swap exists, compilers will use swap in std, thanks to the using declaration that makes std::swap visible in this function.

## Conclusions
Item 18:
- Good interfaces are easy to use correctly and hard to use incorrectly. You should strive for these characteristics in all your interfaces.
- Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
- Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.
- tr1::shared_ptr supports custom deleters. This prevents the cross-DLL problem, can be used to automatically unlock mutexes (see Item 14), etc.

Item 19:
- Class design is type design. Before defining a new type, be sure to consider all the issues discussed in this Item.

Item 20:
- Prefer pass-by-reference-to-const over pass-by-value. It’s typically more efficient and it avoids the slicing problem.
- The rule doesn’t apply to built-in types and STL iterator and function object types. For them, pass-by-value is usually appropriate.

Item 21:
- Never return a pointer or reference to a local stack object, a reference to a heap-allocated object, or a pointer or reference to a local static object if there is a chance that more than one such object will be needed. (Item 4 provides an example of a design where returning a reference to a local static is reasonable, at least in single-threaded environments.)

Item 22:
- Declare data members private. It gives clients syntactically uniform access to data, affords fine-grained access control, allows invariants to be enforced, and offers class authors implementation flexibility.
- protected is no more encapsulated than public.

Item 23:
- Prefer non-member non-friend functions to member functions. Doing so increases encapsulation, packaging flexibility, and functional extensibility.

Item 24:
- If you need type conversions on all parameters to a function (including the one that would otherwise be pointed to by the this pointer), the function must be a non-member.

Item 25:
- Provide a swap member function when std::swap would be inefficient for your type. Make sure your swap doesn’t throw exceptions.
- If you offer a member swap, also offer a non-member swap that calls the member. For classes (not templates), specialize std::swap, too.
- When calling swap, employ a using declaration for std::swap, then call swap without namespace qualification.
- It’s fine to totally specialize std templates for user-defined types, but never try to add something completely new to std.

本章主要讲类和函数接口设计，略难的是自定义swap需要特化swap函数模板或新增函数模板。

