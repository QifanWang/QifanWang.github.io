---
layout: post
title:  "Effective C++ 3rd Chapter 8: Customizing new and delete"
categories: c++
tags: [reading, effective]
toc: true
--- 
What sculpture is to a block of marble, education is to the soul. - Joseph Addison
{: .message }

阅读Effective C++ 3rd(2005) 第八章。

## Item 49: Understand the behavior of the new-handler.
如果 operator new 在申请内存分配时失败，在旧的实现上会返回一个空指针，程序员也可以自定义错误处理的函数 new-handler 。

设置 new-handler 函数可以通过标准库中的 set_new_handler 函数，
{% highlight cpp %}
namespace std {
	typedef void (*new_handler)();
	new_handler set_new_handler(new_handler p) throw();
}
{% endhighlight %}

当 operator new 无法满足内存分配请求时，它将重复调用 new-handler 直到找到足够的内存。一个 well-designed new-handler 需要满足下列任意一个要求，
- Make more memory available. 使下一次内存申请能够成功。一个实现方法是在程序初始化时申请一大块内存，然后在 new-handler 中使用。
- Install a different new-handler. 当前 new-handler 无法额外得到足够内存，但其他 new-handler 可以完成，通过 set_new_handler 进行设置。
- Deinstall the new-handler. 向 set_new_handler 传一个空指针。当没有 new-handler 可以调用，operator new 将在内存分配失败时抛出异常。
- Not return. 调用 abort 或 exit 结束程序。

实现 class-specific new-handler 可以通过 RAII 记录原先的 new-handler 然后还原。更进一步，让每一个类都拥有 class-specific new-handler 可以通过 curiously recurring template pattern (CRTP) 的技巧，但这可能引入多继承。可以看书上的例子，十分具有技巧性。

### nothrow new 
为了仍然有 failure-yields-null behavior，可以通过 “nothrow” 形式的 new 分配内存。
{% highlight cpp %}
Widget *pw2 = new (std::nothrow) Widget;
{% endhighlight %}

需要注意的是，当 nothrow new 分配内存失败，不抛出异常而是返回空指针。但分配内存成功，在构造阶段(这里是call Widget constructor)也有可能抛出异常。

## Item 50: Understand when it makes sense to replace new and delete.
程序员想自定义 operator new 与 operator delete 往往有三个原因，
1. To detect usage errors. 
2. To improve effieciency.
    - To increase the speed of allocation and deallocation.
    - To reduce the space overhead of default memory management.
    - To compensate for suboptimal alignment in the default allocator.
    - To cluster related objects near one another.
    - To obtain unconventional behavior.
3. To collect usage statistics.

自定义 new 需要考虑 alignment 的问题。

定制内存管理 new 十分容易，但要写一个通用的、运行良好的通常很难。编译器在内存管理函数中通常提供了 debugging 与 logging 的功能，也有一些内存池的开源实现可以用于提高内存分配回收效率。总之，定制前衡量设计的必要性，保证定制的通用性。

## Item 51: Adhere to convention when writing new and delete.
当重写 operator new 与 operator delete 时，需要满足一些惯例。

如处理 0-byte 请求，分配失败后重复调用 new-handler 直到成功分配内存，退出程序或者抛出异常。
{% highlight cpp %}
void* operator new(std::size_t size) throw(std::bad_alloc)
{				// your operator new might
	using namespace std;	// take additional params
				
	if (size == 0) {	// handle 0-byte requests
		size = 1;	// by treating them as	
	}			// 1-byte requests
	
	while (true) {
		attempt to allocate size bytes;Customizing new and delete
		if ( the allocation was successful )
			return ( a pointer to the memory );
		// allocation was unsuccessful; find out what the
		// current new-handling function is (see below)
		new_handler globalHandler = set_new_handler(0);
		set_new_handler(globalHandler);
		if (globalHandler) (*globalHandler)();
		else throw std::bad_alloc();
	}
}
{% endhighlight %}

可以自定义 member operator new 专门为特定类型分配内存，一般定义在基类中，通过继承关系复用代码。
{% highlight cpp %}
class Base {
public:
	static void* operator new(std::size_t size) throw(std::bad_alloc);
	...
};

class Derived: public Base	// Derived doesn’t declare
{ ... };			// operator new
Derived *p = new Derived;	// calls Base::operator new!
{% endhighlight %}

定义的成员函数版本，需要处理派生类(派生类 size 不等于 基类 size )调用情况，如这里直接调用标准版本 operator 分配内存。
{% highlight cpp %}
void* Base::operator new(std::size_t size) throw(std::bad_alloc)
{
	if (size != sizeof(Base))		// if size is “wrong,”
		return ::operator new(size);	// have standard operator
						// new handle the request
	...	// otherwise handle
		// the request here
}
{% endhighlight %}

定义 operator new[] 也是类似原理，只是需要记住我们只能分配一大块内存，并不知道内存上对象的数量，因为，
- 派生类 size 可能大于 基类 size
- 参数 size 包括数组大小记录的内存

定义 operator delete 需要注意处理空指针的情况，还有派生类大小不同的情况，
{% highlight cpp %}
class Base {	// same as before, but now operator delete is declared
public:
	static void* operator new(std::size_t size) throw(std::bad_alloc);
	static void operator delete(void *rawMemory, std::size_t size) throw();
	...
};
void Base::operator delete(void *rawMemory, std::size_t size) throw()
{
	if (rawMemory == 0) return;	// check for null pointer

	if (size != sizeof(Base)) {		// if size is “wrong,”
		::operator delete(rawMemory);	// have standard operator
		return;				// delete handle the request
	}
	deallocate the memory pointed to by rawMemory;
	return;
}
{% endhighlight %}

当继承关系中类的析构函数没有定义为 virtual 时，派生类 delete 时传入 operator delete 的参数 size 可能会出错。这也是为什么 Item 7 要求 virtual destructor 的原因。

## Item 52: Write placement delete if you write placement new.
狭义的 placement new 往往指额外接受一个指针作为参数的 operator new 函数，广义上的 placement new 就是额外多一个参数的 operator new 函数。这取决于上下文。

一个完整的 new expression 是先调用 operator new 分配内存再调用构造函数，如果在 call constructor 的过程中抛出异常，之前分配的内存交由 runtime 进行回收(用户代码无法回收因为没有得到指针)。但 runtime 针对 operator new 需要调用相应的 operator delete 函数，如果是 placement new 则需要相同参数的 operator delete 。
> if an operator new with extra parameters isn’t matched by an operator delete with the same extra parameters, no operator delete will be called if a memory allocation by the new needs to be undone.

故定义了 placement new 后需要定义所谓的 placement delete 以处理微小的内存泄漏，但用户代码 delete expression 使用的还是 “ normal operator delete ” 回收内存，这就需要定义两个 operator delete ,
{% highlight cpp %}
class Widget {
public:
	static void* operator new(std::size_t size, std::ostream& logStream)
	 throw(std::bad_alloc);
	static void operator delete(void *pMemory) throw();
	static void operator delete(void *pMemory, std::ostream& logStream)
	 throw();
	...
}

Widget *pw = new (std::cerr) Widget; // as before, but no leak this time
delete pw; // invokes the normal operator delete
{% endhighlight %}

当在类中自定义 placement new 时需要注意对 normal new 的隐藏效应，在继承关系由于是 operator new 的重载也会有 name hiding 现象，见Item 33。
只要类中定义一个 operator new 其他scope(global/base)都会被隐藏，array 形式与operator delete 同理，
{% highlight cpp %}
class Base {
public:
	...
	// this new hides the normal global forms
	static void* operator new(std::size_t size,
				  std::ostream& logStream)
	 throw(std::bad_alloc);
	...
};
Base *pb = new Base;			// error! the normal form of operator new is hidden
Base *pb = new (std::cerr) Base;	// fine, calls Base’s placement new

class Derived: public Base {	// inherits from Base above
public:
	...
	// redeclares the normal form of new
	static void* operator new(std::size_t size)
	 throw(std::bad_alloc);
	...
};
Derived *pd = new (std::clog) Derived;	// error! Base’s placement new is hidden
Derived *pd = new Derived;		// fine, calls Derived’s operator new
{% endhighlight %}

除非是为了禁止这些类通过某些形式 new/delete，否则为了避免命名隐藏，应该声明所有形式的 new/delete，
{% highlight cpp %}
class StandardNewDeleteForms {
public:
	// normal new/delete
	static void* operator new(std::size_t size) throw(std::bad_alloc)
	{ return ::operator new(size); }
	static void operator delete(void *pMemory) throw()
	{ ::operator delete(pMemory); }

	// placement new/delete
	static void* operator new(std::size_t size, void *ptr) throw()
	{ return ::operator new(size, ptr); }
	static void operator delete(void *pMemory, void *ptr) throw()
	{ return ::operator delete(pMemory, ptr); }

	// nothrow new/delete
	static void* operator new(std::size_t size, const std::nothrow_t& nt) throw()
	{ return ::operator new(size, nt); }
	static void operator delete(void *pMemory, const std::nothrow_t&) throw()
	{ ::operator delete(pMemory); }
};

class Widget: public StandardNewDeleteForms {			// inherit std forms
public:
	using StandardNewDeleteForms::operator new;		// make those forms visible
	using StandardNewDeleteForms::operator delete;

	// add a custom placement new
	static void* operator new(std::size_t size,
				  std::ostream& logStream)
	 throw(std::bad_alloc);
	// add the corresponding placement delete
	static void operator delete(void *pMemory,
				    std::ostream& logStream)
	 throw();
	...
};
{% endhighlight %}

这个代码示例没有列出 array 形式，而且C++17开始有 new/delete 的新形式。


## Conclusions
Item 49:
- set_new_handler allows you to specify a function to be called when memory allocation requests cannot be satisfied.
- Nothrow new is of limited utility, because it applies only to memory allocation; associated constructor calls may still throw exceptions.

Item 50:
- There are many valid reasons for writing custom versions of new and delete, including improving performance, debugging heap usage errors, and collecting heap usage information.

Item 51:
- operator new should contain an infinite loop trying to allocate memory, should call the new-handler if it can’t satisfy a memory request, and should handle requests for zero bytes. Class-specific versions should handle requests for larger blocks than expected.
- operator delete should do nothing if passed a pointer that is null. Class-specific versions should handle blocks that are larger than expected.

Item 52:
- When you write a placement version of operator new, be sure to write the corresponding placement version of operator delete. If you don’t, your program may experience subtle, intermittent memory leaks.
- When you declare placement versions of new and delete , be sure not to unintentionally hide the normal versions of those functions.

本章讲定制 new/delete 的一些要点，定义 new-handler 的规范，是否需要定制 new/delete，几种形式的 new/delete 以及规范，定义 placement new 的一些易错点。

## References
1. [C++ references operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new)
