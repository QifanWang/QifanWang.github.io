---
layout: post
title:  "Effective C++ 3rd Chapter 6"
categories: c++
tags: [reading, effective]
toc: true
--- 
Some components of a thriving friendship are honesty, naturalness, thoughtfulness, some common interests.
{: .message }

阅读Effective C++ 3rd(2005) 第六章。

## Item 32: Make sure public inheritance models “is-a.”
public 继承就是常用的继承关系，A is a subclass of B，基类的 public member 保持 public，protected member 保持 protected。
> public inheritance asserts that everything that applies to base class objects —everything!— also applies to derived class objects. 

## Item 33: Avoid hiding inherited names.
派生类的 name 可能会隐藏基类的 name，这会破坏 public 继承关于 is-a 的定义，
{% highlight cpp %}
class Base {
private:
	int x;
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
	...
};

class Derived: public Base {
public:
	virtual void mf1();
	void mf3();
	void mf4();
	...
};

Derived d;
int x;
...
d.mf1(); // fine, calls Derived::mf1
d.mf1(x); // error! Derived::mf1 hides Base::mf1

d.mf2(); // fine, calls Base::mf2
d.mf3(); // fine, calls Derived::mf3
d.mf3(x); // error! Derived::mf3 hides Base::mf3
{% endhighlight %}

C++这样设计，是为了避免程序员意外继承"祖先"类中的重载函数，但如果确实有重载函数需 public 继承(保证 is-a 的定义)，我们可以用 using declarations，

{% highlight cpp %}
class Base {
private:
	int x;
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
...
};

class Derived: public Base {
public:
	using Base::mf1; // make all things in Base named mf1 and mf3
	using Base::mf3; // visible (and public) in Derived’s scope

	virtual void mf1();
	void mf3();
	void mf4();
	...
}

Derived d;
int x;
...
d.mf1();	// still fine, still calls Derived::mf1
d.mf1(x);	// now okay, calls Base::mf1
d.mf2();	// still fine, still calls Base::mf2
d.mf3();	// fine, calls Derived::mf3
d.mf3(x);	// now okay, calls Base::mf3 (The int x is 
	// implicitly converted to a double so that
	// the call to Base::mf3 is valid.)
{% endhighlight %}

如果仅需要 private 继承部分 overloads，则可以使用 forwarding function 调用目标。
{% highlight cpp %}
class Base {
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	... // as before
};

class Derived: private Base {
public:
	virtual void mf1()	// forwarding function; implicitly
	{ Base::mf1(); } 	// inline — see Item 30. (For info
	... 				// on calling a pure virtual
};						// function, see Item 34.)

...
Derived d;
int x;
d.mf1(); // fine, calls Derived::mf1
d.mf1(x); // error! Base::mf1() is hidden
{% endhighlight %}

尽量避免重名隐藏。

## Item 34: Differentiate between inheritance of interface and inheritance of implementation.
纯虚函数，要求派生类继承接口，必须提供实现，
> The purpose of declaring a pure virtual function is to have derived classes inherit a function interface only.

简单虚函数，要求派生类继承接口与默认实现，可以重写实现，
> The purpose of declaring a simple virtual function is to have derived classes inherit a function interface as well as a default implementation.

简单虚函数有时会让派生类"无声地"使用基类的"默认实现"，比如派生类忘记重写时。可以改简单虚函数为纯虚函数声明，抽取出"默认实现"为一个 protected 成员函数或者定义这个纯虚函数，强制派生类"记得"实现。

非虚成员函数，要求派生类全部继承接口与实现。派生类不应该重定义它们， 见 Item 36。
> The purpose of declaring a non-virtual function is to have derived classes inherit a function interface as well as a mandatory implementation.

三种声明，提供不同的继承"精度"。

在继承关系中，不要仅声明 non-virtual member functions，析构函数显然是需要为 virtual function 的，同时，也不要畏惧在抽象类中声明 non-virtual member functions，有时需要让所有派生类都有相同的行为。

## Item 35: Consider alternatives to virtual functions.
有时虚函数是可以被其他设计代替的。

### The Template Method Pattern via the Non-Virtual Interface Idiom
有一派认为基类可以定义 public non-virtual function 作为派生类共同行为(也是 wrapper )，然后在其中调用virtual function 完成实际的工作。

这也是所谓 non-virtual interface (NVI) idiom 的定义，这些 virtual function 可以是 private 、 protected 或 public 。

### The Strategy Pattern via Function Pointers
通过函数指针完成 Strategy Pattern，具体操作可以在构造时传入一个函数指针作为成员。

这可以让行为与类型分离。同一个类的实例可以有不同的行为，并且在运行时更改行为(重新设置函数指针)。

但如果函数需要类内部数据用以计算，那么函数指针所指向的函数就需要被定义为 friend function ，这会破坏封装性。是否使用这个策略取决于设计。

### The Strategy Pattern via tr1::function
用 function object 代替函数指针，可以更灵活地使用函数，甚至可以通过 bind 实现函数柯里化。So Cool！！！

### The “Classic” Strategy Pattern
可以将 Strategy 也定义为一个类。之前的类，从成员为 函数指针与 function object 的设计，改为一个 Strategy 类的指针作为成员。大同小异，只是 Strategy 类也可以进行派生。

### Summary
- Use the non-virtual interface idiom (NVI idiom), a form of the Template Method design pattern that wraps public non-virtual member functions around less accessible virtual functions.
- Replace virtual functions with function pointer data members, a stripped-down manifestation of the Strategy design pattern.
- Replace virtual functions with tr1::function data members, thus allowing use of any callable entity with a signature compatible with what you need. This, too, is a form of the Strategy design pattern.
- Replace virtual functions in one hierarchy with virtual functions in another hierarchy. This is the conventional implementation of the Strategy design pattern.

## Item 36: Never redefine an inherited non-virtual function.
virtual function 是动态绑定的， non-virtual function 是静态绑定的。如果在派生类中重定义 non-virtual function ，在调用时不会有运行时多态行为，更违反 Item 32 & 34 的思想。

需要重定义和运行时多态，用 virtual function 即可。

## Item 37: Never redefine a function’s inherited default parameter value.
如Item 36 所说， non-virtual function 不能被重定义，故本 Item 讨论的是拥有默认参数的 virtual function 。

虚函数是动态绑定，而默认参数是静态绑定。当继承的虚函数有默认参数时，不要重定义默认参数。即使重定义了，动态绑定时也会用指针/引用类型(一般是基类)的默认参数。

派生类中不修改虚函数的默认参数，也会导致 code duplication ，并且未来一旦修改基类中该虚函数的默认参数，所有派生类都需要修改。

最好的方法是，虚函数不要声明默认参数。用 NVI idiom 进行重构，在接口上进行默认参数的声明。

## Item 38: Model “has-a” or “is-implemented-in-terms-of” through composition.
类与成员的关系，在应用领域可理解为拥有 has-a ， 在实现领域可理解为被xx实现 is-implemented-in-terms-of 。

## Item 39: Use private inheritance judiciously.
如果派生类 private 继承 基类，则基类所有成员在派生类中都成为 private , 且编译器在函数调用时不会将派生类对象视作基类对象。

私有继承意味着派生类被基类"实现"，“is-implemented-in-terms-of” ，派生类没有继承接口而是继承实现。
>  Private inheritance means nothing during software design, only during software implementation.

同样意味着"实现"，在可以用 compound 时用之，在不得不用 private inheritance 时用之。

不得不用私有继承的情况是，
- 需要基类 protected members 的权限 或 重定义 基类虚函数实现。其实这也可以通过定义一个新类 public 继承后，再 compound 封装，实现之。
- 继承 empty class 的情况，会比 compound 节约空间(减少一个字节带来的 byte padding)。这也被称作 empty base optimization (EBO)，编译器大部分都会实现这个优化。这个优化仅适用于单继承。

所谓的 empty class 只是没有非静态数据成员，它可以包含 
- typedefs
- enums
- static data members
- non-virtual functions

在STL中有很多有用的 empty classes ，如 unary_function 与 binary_function 可供用户自定义的 function object 继承。这时就可以用私有继承得到EBO[1]。

在C++中对于 class 不声明 access-specifier 则默认为 private inheritance[2]，
> If access-specifier is omitted, it defaults to public for classes declared with class-key struct and to private for classes declared with class-key class.

## Item 40: Use multiple inheritance judiciously.
C++ 调用函数时 name lookup ，是先匹配名字再确认权限 (accessibility)。当多继承的派生类出现函数名二义性时，需要显式指定类名，
{% highlight cpp %}
mp.BorrowableItem::checkOut(); // ah, that checkOut...
{% endhighlight %}

对于菱形继承问题，如果不加任何限制，C++默认重复继承(基类多副本)，如果需要去重，C++通过 virtual 继承解决。

虚继承会带来对象大小和获取数据成员速度上的开销。而且需要额外的心智考虑类的构造，
1. classes derived from virtual bases that require initialization must be aware of their virtual bases, no matter how far distant the bases are, and 
2. when a new derived class is added to the hierarchy, it must assume initialization responsibilities for its virtual bases (both direct and indirect).

尽量不用 virtual base classes，即使用也尽量不要声明数据成员。

多继承应用场景为，当需要结合 public 和 private 继承时。

## Conclusions
Item 32:
- Public inheritance means “is-a.” Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object.

Item 33:
- Names in derived classes hide names in base classes. Under public inheritance, this is never desirable. 
- To make hidden names visible again, employ using declarations or forwarding functions.

Item 34:
- Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes always inherit base class interfaces.
- Pure virtual functions specify inheritance of interface only. 
- Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation. 
- Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

Item 35:
- Alternatives to virtual functions include the NVI idiom and various forms of the Strategy design pattern. The NVI idiom is itself an example of the Template Method design pattern.
- A disadvantage of moving functionality from a member function to a function outside the class is that the non-member function lacks access to the class’s non-public members.
- tr1::function objects act like generalized function pointers. Such objects support all callable entities compatible with a given target signature.

Item 36:
- Never redefine an inherited non-virtual function.

Item 37:
- Never redefine an inherited default parameter value, because default parameter values are statically bound, while virtual functions — the only functions you should be redefining — are dynamically bound.

Item 38:
- Composition has meanings completely different from that of public inheritance. 
- In the application domain, composition means has-a. In the implementation domain, it means is-implemented-in-terms-of.

Item 39:
- Private inheritance means is-implemented-in-terms of. It’s usually inferior to composition, but it makes sense when a derived class needs access to protected base class members or needs to redefine inherited virtual functions.
- Unlike composition, private inheritance can enable the empty base optimization. This can be important for library developers who strive to minimize object sizes.

Item 40:
- Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance. 
- Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It’s most practical when virtual base classes have no data.
- Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.

这章讲OOP继承相关的注意事项。比较新的点有，继承关系中的命名隐藏需要小心，虚函数的默认参数不能被重定义，各种继承的设计意义，多继承的基类重复问题。

## References
1. [C++ references empty base optimization](https://en.cppreference.com/w/cpp/language/ebo)
2. [C++ references derived class](https://en.cppreference.com/w/cpp/language/derived_class)
3. [Wikipedia multiple inheritance, the diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)
