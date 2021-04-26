---
layout: post
title:  "Effective C++ 3rd Chapter 5"
categories: c++
tags: [reading, effective]
toc: true
--- 
The mist is refracted by daylight to produce rainbows.
{: .message }

阅读Effective C++ 3rd(2005) 第五章。

## Item 26: Postpone variable definitions as long as possible.
尽可能延后变量定义，以减少可能的构造与析构开销。

尽可能延后到变量必须使用时，延后到构造参数都确定。

循环情况下，除非赋值开销小于构造析构开销且程序开销敏感，我们都应尽量在循环内部定义变量。

## Item 27: Minimize casting.
小心使用并尽量避免使用casting。

尽量使用C++ style casting:
- const_cast is typically used to cast away the constness of objects. It is the only C++-style cast that can do this.
- dynamic_cast is primarily used to perform “safe downcasting,” i.e., to determine whether an object is of a particular type in an inheritance hierarchy. It is the only cast that cannot be performed using the old-style syntax. It is also the only cast that may have a significant runtime cost.
- reinterpret_cast is intended for low-level casts that yield implementation-dependent (i.e., unportable) results, e.g., casting a pointer to an int. Such casts should be rare outside low-level code. 
- static_cast can be used to force implicit conversions (e.g., non-const object to const object (as in Item 3), int to double, etc.). It can also be used to perform the reverse of many such conversions (e.g., void* pointers to typed pointers, pointer-to-base to pointer-to-derived), though it cannot cast from const to non-const objects. (Only const_cast can do that.)

一个有趣的例子，
{% highlight cpp %}
class Base { ... };
class Derived: public Base { ... };
Derived d;
Base *pb = &d; // implicitly convert Derived* ⇒ Base*
{% endhighlight %}

在有些实现上，pb指针指向的地址可能不同于对象d的地址。
>  When that’s the case, an offset is applied at runtime to the Derived* pointer to get the correct Base* pointer value.

一个对象拥有多个地址，这在C++中是有可能的。事实上，在多继承关系中，这种现象总是发生。(单继承也可能发生。)

这说明，程序员不应该基于内存布局的假设进行casting。即使这在有些平台可以成功，但在其他平台会出现错误。

派生类成员函数中调用基类虚函数，直接用"Window::onResize();"的修饰方式调用，不要用static_cast试图转换类型，'static_cast<Base>(*this)'会创建一个临时的基类对象。

在四种casting中 dynamic_cast 尤其需要小心，尽量通过基类加虚函数或用容器包含派生类实例避免之。

尽量在单独语句中casting，
> Like most suspicious constructs, casts should be isolated as much as possible, typically hidden inside functions whose interfaces shield callers from the grubby work being done.
inside

## Item 28: Avoid returning “handles” to object internals.
公共成员函数接口如果返回对象内部数据成员的引用，会破坏封装性。类似指针逃逸。还有可能使const member function失去意义。
{% highlight cpp %}
class Rectangle {
public:
	...
	Point& upperLeft() const { return pData->ulhc; }
	Point& lowerRight() const { return pData->lrhc; }
	...
};

Point coord1(0, 0);
Point coord2(100, 100);
const Rectangle rec(coord1, coord2); // rec is a const rectangle from
									 // (0, 0) to (100, 100)
rec.upperLeft().setX(50); 			 // now rec goes from
									 // (50, 0) to (100, 100)!
{% endhighlight %}

这条Item的handle是指引用、指针或迭代器，而internal除了指数据成员，也可以指 private/protected 函数。

即使我们定义函数返回 const handle，返回 handle 还是会容易造成 dangling handles 问题，
> All that matters is that a handle is being returned, because once that’s being done, you run the risk that the handle will outlive the object it refers to.

有时返回 handle 不可避免，如定义 operator[] 返回容器内部数据时，这种函数就是例外。

## Item 29: Strive for exception-safe code.
异常可能会破坏设计函数时保持程序合法状态的期待。

When an exception is thrown, exception-safe functions:
- Leak no resources. 已经申请的资源都归还。
- Don’t allow data structures to become corrupted. 已经完成的操作(对象状态的修改)被还原或某种处理，总之，就是保证数据状态的(consistency)一致性(始终在合法状态)。

资源归还可以通过RAII完成。保持数据状态的一致性，则有三种保证:
1. basic guarantee 承诺函数抛出异常后，程序仍处于合法状态，即对象数据仍保持一致性。但究竟处于何种合法状态，可能无法预测或需要调用其他函数确定当前状态。
2. strong guarantee 承诺函数抛出异常后，程序状态不改变(如该函数调用前的状态)。这类函数拥有(atomic)原子性(函数要么操作状态要么不操作)。
3. no throw guarantee 承诺函数永远不抛出异常。可以添加 noexcept specifier 于函数声明处，C++11前仅有 'thow()' 标识。

大多数时候异常无法避免，只能在 basic guarantee 与 strong guarantee 中选择。

有一种 "copy and swap" 的技巧常常用于 strong guarantee ，具体做法就是复制一份待修改的对象数据，在副本上修改，然后交换。这个技巧通常也和 pimpl idiom 一同使用(因为交换开销更小)。

strong guarantee 有时开销很大不利于实践，且由于调用函数的 side effects 很难明确，调用函数本身是 basic guarantee 导致状态难以确认，所以有时可以不选择保证。

但无论如何也至少需要 basic guarantee。

如果无法实现任何 exception safety guarantee 则需要在文档中说明原因(调用其他合法代码导致没有选择)，让用户和代码维护者知晓。

## Item 30: Understand the ins and outs of inlining.
inline有利有弊
pros:
- better than macro and no overhead of a function call.
- enable compilers to perform context-specific optimizations on the body of the funtion.

cnos:
- may increase the size of object code.
- Even with virtual memory, inline-induced code bloat can lead to additional paging, a reduced instruction cache hit rate, and the performance penalties that accompany these things.
- 有些 build environment 还无法 debug 内联函数。

但如果 inline 函数的函数体很短，则函数体代码生成可能小于调用函数的代码生成。

inline 有时会隐式生成，如在类中定义成员函数。

inline function 通常必须定义在头文件，大部分 build environment 在编译期内联。(有些在链接期内联，极少数如 .NET Common Language Infrastructure (CLI) 在运行时内联。)

模板实例化通常也在编译期(有些实现在链接期)，与函数内联是相互独立的。不要随意定义函数模板为 inline，正如前文所说，内联是有代价的。

编译器有时会拒绝 inline，如认为内联函数太复杂(有循环或递归)。虚函数绝对不会内联，因为
- virtual means “wait until runtime to figure out which function to call,” 
- and inline means “before execution, replace the call site with the called function.”

总之函数是否内联是取决 build environment 尤其是编译器。

如果函数通过函数指针调用，则编译器一般会拒绝内联(like virtual function call)，
{% highlight cpp %}
inline void f() {...} // assume compilers are willing to inline calls to f
void (*pf )() = f; // pf points to f
...
f(); // this call will be inlined, because it’s a “normal” call

pf(); // this call probably won’t be, because it’s through
	  // a function pointer
{% endhighlight %}

这些不会内联的"内联函数"有时也来自构造与析构函数。编译器针对有继承关系或数据成员需要构造的类，会为其构造与析构函数生成一些函数调用，所以这时也无法内联。

库的设计者尤其要考虑是否让用户可见内联函数。因为一旦修改，用户需要重新编译。不如提供一个函数接口，让用户可以仅重新链接。

谨慎使用 inline，指定真正值得内联的函数(短且常用的)。

## Item 31: Minimize compilation dependencies between files.
C++在分离接口与实现上做的不是很好。导致经常可能重编译，耗时久。

一个编译依赖(头文件)修改可能需要重编译很多文件。

仍然是使用 pimpl idiom 分离实现，
{% highlight cpp %}
#include <string> 	// standard library components
					// shouldn’t be forward-declared
#include <memory> 	// for tr1::shared_ptr; see below
class PersonImpl; 	// forward decl of Person impl. class
class Date; 		// forward decls of classes used in
class Address; 		// Person interface
class Person {
public:
	Person(const std::string& name, const Date& birthday,
			const Address& addr);
	std::string name() const;
	std::string birthDate() const;
	std::string address() const;
	...
private: 									// ptr to implementation;
	std::tr1::shared_ptr<PersonImpl> pImpl; // see Item 13 for info on
}; 											// std::tr1::shared_ptr
{% endhighlight %}

关键在于，
<strong> make your header files self-sufficient whenever it’s practical, and when it’s not, depend on declarations in other files, not definitions. </strong>

具体操作，
- Avoid using objects when object references and pointers will do. 定义引用和指针可以确定size，但定义实例则必须依赖 definition 才能确定size。
- Depend on class declarations instead of class definitions whenever you can. 尽量依赖 declaration 而不是 definition，减少用户不必要的依赖。
- Provide separate header files for declarations and definitions. 分离声明与实现。

通过 pimpl idiom 实现分离也称为Handle classes，通过抽象类定义工厂模式生产派生类，再进行实现分离称为Interface classes。

二者都可能带来运行时开销(access indirection、动态内存分配、虚表)，二者也无法使用 inline function。但使用这两个技巧仍然可以最小化实现变化时对用户的影响。

## Conclusions
Item 26:
- Postpone variable definitions as long as possible. It increases program clarity and improves program efficiency.

Item 27:
- Avoid casts whenever practical, especially dynamic_casts in performance-sensitive code. If a design requires casting, try to develop a cast-free alternative.
- When casting is necessary, try to hide it inside a function. Clients can then call the function instead of putting casts in their own code.
- Prefer C++-style casts to old-style casts. They are easier to see, and they are more specific about what they do.

Item 28:
- Avoid returning handles (references, pointers, or iterators) to object internals. Not returning handles increases encapsulation, helps const member functions act const, and minimizes the creation of dangling handles.

Item 29:
- Exception-safe functions leak no resources and allow no data structures to become corrupted, even when exceptions are thrown. Such functions offer the basic, strong, or nothrow guarantees.
- The strong guarantee can often be implemented via copy-and-swap, but the strong guarantee is not practical for all functions.
- A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls.

Item 30:
- Limit most inlining to small, frequently called functions. This facilitates debugging and binary upgradability, minimizes potential code bloat, and maximizes the chances of greater program speed.
- Don’t declare function templates inline just because they appear in header files

Item 31:
- The general idea behind minimizing compilation dependencies is to depend on declarations instead of definitions. Two approaches based on this idea are Handle classes and Interface classes.
- Library header files should exist in full and declaration-only forms. This applies regardless of whether templates are involved.

本章讲实现(Implementation)中的一些需要注意的细节，其中 exception-safe functions、 inline 与 编译依赖部分 很值得学习。
