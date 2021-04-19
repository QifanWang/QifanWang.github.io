---
layout: post
title:  "Effective C++ 3rd Chapter 3"
categories: c++
tags: [reading, effective]
toc: true
--- 
The love that lasts the longest is the love that is never returned.
{: .message }

阅读Effective C++ 3rd(2005) 第三章。

## Item 13:Use objects to manage resources.
Resource Acquisition Is Initialization (RAII):

- Resources are acquired and immediately turned over to resource-managing objects.
- Resource-managing objects use their destructors to ensure that resources are released.

用对象管理资源，利用析构归还资源。

使用智能指针而不是裸指针操作对象，以防止忘记delete，提前的retrun与break，或者delete前的exception等情况。

auto_ptr(C++11后被unique_ptr代替)实现单独拥有资源，auto_ptr的复制语义不是复制资源而是移动资源。

实现 reference-counting smart pointer (RCSP) 的shared_ptr可以实现资源被多个所有者拥有。RCSP很像GC，但不同的是，其无法打破循环引用的情况。

## Item 14:Think carefully about copying behavior in resource-managing classes.
非堆上的资源不适合用智能指针管理。有时仍需要程序员使用自定义的资源管理类，对于RAII类的定义，需要考虑如何处理复制情形，

- Prohibit copying. 例如，锁的Mutex资源，禁止复制这类同步原语资源。实现上，可以继承UnCopy类实现。

- Reference-count the underlying resource. 可以用shared_ptr作为类的字段，并指定'deleter'代替shared_ptr默认的析构函数。

- Copy the underlying resource. 例如部分实现string是通过指针指堆上的字符串，这类可以实现复制。复制必须是deep copy才行。

- Transfer ownership of the underlying resource. 转移资源所有权在C++11后的移动语义中很常见，被弃置的auto_ptr也是如此实现。

一个引用计数的例子，指定unlock代替destructor，
{% highlight cpp %}
class Lock {
public:
	explicit Lock(Mutex *pm)	// init shared_ptr with the Mutex
	: mutexPtr(pm, unlock)		// to point to and the unlock func
	{				// as the deleter†
		lock(mutexPtr.get( ));	// see Item15 for info on “get” 
	}
private:
	std::tr1::shared_ptr<Mutex> mutexPtr;	// use shared_ptr
};						// instead of raw pointer
{% endhighlight %}
但如此有可能出现构造失败，调用析构函数的情况。这里也就是调用作为'deleter'的unlock函数，会造成本来没有上锁，但解锁的错误。[见此](https://www.aristeia.com/BookErrata/ec++3e-errata.html#p68LockCtorProb)。可以通过用先上锁，进行修改。
{% highlight cpp %}
explicit Lock(Mutex *pm)
{
	lock(pm);
	mutexPtr.reset(pm, unlock);
}
{% endhighlight %}

## Item 15:Provide access to raw resources in resource-managing classes.
有些库的设计，就是用裸指针管理资源，其API的参数也会裸指针类型。自定义的资源管理类不仅要管理这些资源，也需要提供裸露指针的接口。一般来说，有两种接口可以选择，显式或隐式。

{% highlight cpp %}
class Font {
public:
	...
	FontHandle get() const 
	{ return f;}// explicit conversion function

	operator FontHandle() const// implicit conversion function
	{ return f;}
};
{% endhighlight %}

显式有人嫌弃繁琐，隐式容易出错，例如赋值时发生隐式转换。使用哪种取决于设计要求。

## Item 16:Use the same form in corresponding uses of new and delete.
通常，使用new 表达式时，
1. 内存被分配。通过operator new 函数分配。
2. 一个或多个构造函数在内存上构造对象。

使用delete表达式时，
1. 一个或多个析构函数在内存上析构对象。
2. 内存被归还。

不正确地析构与归还内存都是UB行为，所以
> if you use [] in a new expression, you must use [] in the corresponding delete  expression. If you don’t use [] in a new expression, don’t use [] in the matching delete expression.

当有typedef时尤其需要注意，
{% highlight cpp %}
typedef std::string AddressLines[4];
std::string *pal = new AddressLines; 	// note that "new AddressLines" 
					// returns a string*, just like 
					// "new string[4]" would
delete [] pal;				// fine
{% endhighlight %}

为了防止类似情况，避免typedef数组类型。

## Item 17:Store newed objects in smart pointers in standalone statements.
考虑下面这个例子，一个函数展示处理时优先级，另一个函数根据优先级处理动态分配的Widget对象。

{% highlight cpp %}
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
{% endhighlight %}

如下的调用方式可以通过编译，

{% highlight cpp %}
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
{% endhighlight %}

但仍然可能泄漏资源。

### order of evaluation
编译器生成processWidget函数调用前，需要参数赋值。第一个参数赋值分为两部分，
- Execution of the expression “new Widget”.
- A call to the tr1::shared_ptr constructor.

为了底层实现优化，C++并没有像Java和C#那样指定[参数赋值顺序](https://en.cppreference.com/w/cpp/language/eval_order)。
参数赋值时执行顺序可能是，
1. Execute “new Widget”.
2. Call priority.
3. Call the tr1::shared_ptr constructor.

当调用priority函数出现异常时，表达式new Widget返回的指针可能"丢失"(因为并没有存入shared_ptr以防止资源泄漏)，造成资源泄漏。

所以用单独的语句构造智能指针，
{% highlight cpp %}
std::tr1::shared_ptr<Widget> pw(new Widget);	// store newed object
						// in a smart pointer in a
						// standalone statement
processWidget(pw, priority());			// this call won’t leak
{% endhighlight %}

## Conclusions
Item 13:
- To prevent resource leaks, use RAII objects that acquire resources in their constructors and release them in their destructors.
- Two commonly useful RAII classes are tr1::shared_ptr and auto_ptr.tr1::shared_ptr is usually the better choice, because its behavior when copied is intuitive. Copying an auto_ptr sets it to null.

Item 14:
- Copying an RAII object entails copying the resource it manages, so the copying behavior of the resource determines the copying behavior of the RAII object.
- Common RAII class copying behaviors are disallowing copying and performing reference counting, but other behaviors are possible.

Item 15:
- APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.
- Access may be via explicit conversion or implicit conversion. In general, explicit conversion is safer, but implicit conversion is more convenient for clients.

Item 16:
- If you use [] in a new expression, you must use [] in the corresponding delete expression. If you don’t use [] in a new expression, you mustn’t use [] in the corresponding delete expression.

Item 17:
- Store newed objects in smart pointers in standalone statements. Failure to do this can lead to subtle resource leaks when exceptions are thrown.

本章为OOP范式的经典的RAII资源管理思想。

## Reference
1. [init shared_ptr with unlock function problem](https://www.aristeia.com/BookErrata/ec++3e-errata.html#p68LockCtorProb)
2. [C++ reference: order of evaluation](https://en.cppreference.com/w/cpp/language/eval_order)
