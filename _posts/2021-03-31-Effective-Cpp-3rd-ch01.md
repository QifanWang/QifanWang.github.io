---
layout: post
title:  "Effective C++ 3rd Chapter 1"
categories: c++
tags: [reading, effective]
toc: true
--- 
To realise the value of one millisecond, ask the athlete who has won a silver medal in the Olympics.
{: .message }

阅读Effective C++ 3rd(2005) 第一章。

## Item 1: View C++ as a federation of languages.

C++多范式，本质上包含四个子部分，
- C
- Object-Oriented C++
- Template C++
- The STL

当我们学习语言的一些规则时，需要记住其应用的相应的部分，并适时切换。

例如，C语言的pass-by-value在OOP中适时地被pass-by-reference-to-const代替，由于没有特化具体类型Template也是如此，而到了STL中，由于iterator模仿C指针，pass-by-value又重新被应用。

## Item 2: Prefer const, enums, and inlines to '#define's.

简而言之，在编译期而不是在预处理期，做更多事。

### Why
宏仅被预处理器展开，对编译器是不可见的，所以在编译出错时，我们不知道宏被定义在何处，也不知道具体常量的究竟是哪一个宏(因为宏不在symbol table中)。

当宏被constant代替时，编译器可以将其加入symbol table中。并且有时能编译得到更小的代码，因为宏展开是盲目的替换产生多个副本，应用constant只有一个副本。

### Two Special Cases
用constant代替宏时，注意两个特殊情况。

第一个是字符串常量（用常量常指针或C++ string），这个老生常谈。

第二个是 class-specific constants。这是为了限制一个constant仅作用于一个类中，为了保证只有一个副本，可以声明其为静态成员，
{% highlight cpp %}
class GamePlayer {
private:
	static const int NumTurns = 5;// constant declaration
	int scores[NumTurns];// use of constant
	//...
};
{% endhighlight %}

上面NumTurns是declaration不是definition。通常C++需要程序员为所有使用的事物提供一个定义，但static 且整型 的 class-specific constants是个例外。只要我们不使用它们的地址，我们可以仅提供declaration。但如果我们需要用这些常量的地址，或是没用地址时编译器坚持要求提供definition。我们可以提供一个separate definition，例如，
{% highlight cpp %}
const int GamePlayer::NumTurns;	// definition of NumTurns; see 
				// below for why no value is given
{% endhighlight %}

我们将上面这个definition放入实现文件而不是头文件中。因为初始值已经在declaration处提供，在definition处不允许提供初始值。

宏就无法提供类似class-specific这种限定作用范围的常量。因为一旦'#define'，除非有对应的'#undefed'，剩余编译过程都会应用对应宏展开。宏也因此无法提供任何像'private'的封装。

### Note
一些老编译器可能不接受上面的语法(static class-specific constants)，因为它认为在类静态成员声明时提供初始值是非法的。类中初始化仅被允许在整型常量上。为了防止这类情况，可以在定义时提供初始值。例如，
{% highlight cpp %}
class CostEstimate {
private:
	static const double FudgeFactor;	// declaration of static class
						// constant; goes in header file
};

const double					// definition of static class
 CostEstimate::FudgeFactor = 1.35;		// constant; goes in impl. file
{% endhighlight %}

### Enum Hack
通常了解这些已经足够(use class-specific constant)，还有一个例外场景是在编译整个类时应用类中的class constant，比如初始化一个成员数组。一些老编译器会拒绝(gcc 9.3.0没有)，认为这时不知道数组大小。于是，可以使用enum hack技巧，这是C语言中可将Enum视为int特性的一个应用。
{% highlight cpp %}
class GamePlayer {
private:
	enum { NumTurns = 5 };	// “the enum hack” — makes 
				// NumTurns a symbolic name for 5
	int scores[NumTurns];	// fine...
};
{% endhighlight %}
Enum hack这个技巧值得被学习。
首先，因为enum hack有些方面更像宏，而不是constant。例如，取const的地址是合法的，取宏的地址是非法，取enum地址也是非法的。如果我们不想整型常量被取地址(指针or引用)，就可以用enum。此外，虽然好的编译器不会为整型的const对象留出存储空间(除非创建指针或对象引用)，但邋遢的编译器可能会，而且你可能不愿意为此类对象留出内存。像'#define'一样，枚举永远不会导致那种不必要的内存分配。

其次，出于实用主义，Enum Hack也是模板元编程的一个基础技巧，很多代码都会用到。

### Macro used as like function
在许多C语言代码中，程序员为了实现函数，但又不想调用函数的开销，往往会用宏代替函数。为了避免运算符优先级问题，往往在宏定义中用括号包住参数。但在宏展开时，仍然很容易引入错误。
{% highlight cpp %}
// call f with the maximum of a and b
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

int a = 5, b = 0;
CALL_WITH_MAX(++a, b);		// a is incremented twice
CALL_WITH_MAX(++a, b+10);	// a is incremented once
{% endhighlight %}

我们可以使用，inline template function代替这些'宏函数'，这样既没有额外开销，也保证行为易预测(没有参数多次赋值)且类型安全。而且这是一个真实的函数，它遵守作用域规则，可以被封装，像'宏函数'则完全不行。
{% highlight cpp %}
template<typename T>					// because we don’t
inline void callWithMax(const T& a, const T& b) 	// know what T is, we
{ 							// pass by reference-to-
	f(a > b ? a : b);				// const — see Item20
}
{% endhighlight %}

## Item 3: Use const wheneve possible.

尽可能多用const以告知编译器和其他程序员更多信息，即某些对象不应该被修改。

指针的左物右指，老生常谈。

'const T' 与 'T const' 没有区别。

### STL const iterator
STL 中iterator模仿指针。它们'常量指针'和左物右指的规则有区别，
{% highlight cpp %}
std::vector<int> vec

const std::vector<int>::iterator iter =		// iter acts like a T* const
 vec.begin( );
 *iter = 10;					// OK, changes what iter points to
 ++iter;					// error! iter is const
 
 std::vector<int>::const_iterator cIter =	// cIter acts like a const T*
  vec.begin( );
 *cIter = 10;					// error! *cIter is const
 ++cIter;					// fine, changes cIter
{% endhighlight %}

### const in function declaration
const最常用于函数声明。const可以指定函数返回，参数，对于成员函数甚至可以指定整个函数。

指定函数返回值是常量通常是不正确的，但如果返回是一个引用或是指针，这样做就有意义--保证安全与效率的前提下减少出错。比如，
{% highlight cpp %}
class Rational { ... };

const Rational operator*(const Rational& lhs, const Rational& rhs);
{% endhighlight %}
我们经常看见类似代码。为什么需要返回一个常指针？因为如果没有const指定返回，我们可以得到类似代码，
{% highlight cpp %}
Rational a, b, c;
// ...
(a* b) = c;	// invoke operator= on the 
		// result of a*b!
{% endhighlight %}
或者是，类似赋值的隐式转换，
{% highlight cpp %}
if (a * b = c) // oops, meant to do a comparison!
{% endhighlight %}

为了避免上述错误，我们可以指定返回位常指针。

关于指定参数const，也尽可能使用之。

### const Member Functions
指定整个函数const，既方便函数接口便于理解(是否修改对象)，又可说明函数操作const对象以提高性能(编译器对于const函数的优化)。

是否指定函数const，也是重载。
{% highlight cpp %}
class TextBlock {
public:
	const char& operator[](std::size_t position) const	// operator[] for
	{ return text[position]; }				// const objects
	
	char& operator[](std::size_t position)			// operator[] for
	{ return text[position]; }				// non-const objects
private:
	std::string text;
};

int main() {
	TextBlock tb("Hello");
	std::cout << tb[0];				// calls non-const TextBlock::operator[]	
	const TextBlock ctb("World");
	std::cout << ctb[0];				// calls const TextBlock::operator[]
}
{% endhighlight %}

通常有两个派别看常函数，bitwise constness/logical constness

#### bitwise constness
bitwise const 要求成员函数不修改对象中任何bit。这也是C++对于const member function的定义，即调用时不允许修改任何非静态数据成员。

但通过bitwise test的成员函数并不总是安全的，比如下面这个类中的operator[]，满足const member function的条件，但返回对象内部数据的引用。仍然对象内部数据有机会被修改，比如这个main函数中的'cctb'对象，
{% highlight cpp %}
class CTextBlock {
public:
	char& operator[](std::size_t position) const	// inappropriate (but bitwise
	{ return pText[position]; }			// const) declaration of
							// operator[]
private:
	char *pText;
};

int main() {
	const CTextBlock cctb("Hello");	// declare constant object
	char *pc = &cctb[0];		// call the const operator[] to get a
					// pointer to cctb’s data
	*pc = ’J’;			// cctb now has the value “Jello”
}
{% endhighlight %}

#### logical constness
这个派别认为，const member function可能修改对象bit，只要以用户无法察觉的方式。如下面这个只在用到时求长度的例子，
{% highlight cpp %}
class CTextBlock {
public:
	std::size_t length() const;
private:
	char *pText;
	std::size_t textLength;	// last calculated length of textblock
	bool lengthIsValid;	// whether length is currently valid
};

std::size_t CTextBlock::length() const
{
	if (!lengthIsValid) {
		textLength = std::strlen(pText);// error! can’t assign to textLength
		lengthIsValid = true;		// and lengthIsValid in a const
	}					// member function

	return textLength;
}
{% endhighlight %}
这显然不满足bitwise的要求。我们可以用 mutable 解放非静态数据成员。

{% highlight cpp %}
class CTextBlock {
public:
	std::size_t length() const;
private:
	char *pText;
	mutable std::size_t textLength;	// these data members may
	mutable bool lengthIsValid;	// always be modified, even in
};					// const member functions
std::size_t CTextBlock::length() const
{
	if (!lengthIsValid) {
		textLength = std::strlen(pText);// now fine
		lengthIsValid = true;		// also fine
	}
	return textLength;
}
{% endhighlight %}

### Avoiding Duplication in const and Non-const Member Functions
有时我们会为某个功能的member function 设置两个版本，一个 non-const 一个 const，如果它们的代码相同，可能会造成 Code Duplication。

通常，casting是一个bad idea，但为了避免Duplication，我们可以单独实现const版本，然后在non-const版调用之并去掉const修饰。
{% highlight cpp %}
class TextBlock {
public:
	...
	const char& operator[](std::size_t position) const// same as before
	{
		...
		...
		...
		return text[position];
	}
	char& operator[](std::size_t position)	// now just calls const op[]
	{
		return const_cast<char&>(			// cast away const on op[]’s return type;
			static_cast<const TextBlock&>(*this)	// add const to *this’s type;
			 [position]				// call const version of op[]
		);
	}
...
};
{% endhighlight %}
谨记，上面这样 non-const 版本 call const 版本是安全的，反之则可能破环bitwise const的规定。

## Item 4: Make sure that objects are initialized before they’re used.
永远先初始化对象再使用之。

不要依赖可能存在的零值初始化，它们并不可靠。

对于built-in type赋值，对于其他类型用 constructor。

### member initialization list
用constructor时，优先用member initialization list(before constructor body)。这样做不会像赋值初始化那样浪费地用default constructor再copy。同时，尽量不要遗漏成员。即使编译器会帮我们调用成员的default constructor，也最好手写列出，避免遗漏。

当数据成员是const或reference，它们必须被初始化。初始化列表就是最好的选择。

类有multiple constructors 或是基类时，在初始化列表时省略一些成员，是可取的。尤其是有些成员的值必须从文件或数据库中读取时，我们可以将这些初始化读取，封装成函数在constructor中使用。但在通常情况下，使用成员初始化列表更可取。

### initialization order
C++中关于初始化顺序是不变的，基类先于派生类初始化。在一个类中，数据成员的初始化顺序是按照它们的<strong>声明顺序</strong>，而不是初始化列表中的顺序(good style:满足二者顺序是一致)。

####  initialization of non-local static objects defined in different translation units
<strong>static object</strong> 从构造后一直存在到程序结束(所有静态对象在main完成执行后被析构)。Stack and heap-based objects显然不属于这类。它包含，
> global objects, objects defined at namespace scope, objects declared static  inside classes, objects declared static inside functions, and objects declared static at file scope.

定义在函数内部的静态对象也被称为 local static objects(local to a function)，其他的静态对象都是non-local static objects。

<strong>translation unit</strong>是将成为一个个目标文件的源代码。
> A translation unit is the source code giving rise to a single object file. It’s basically a single source file, plus all of its #include files.

问题，现在考虑两个编译单元，至少一个non-local static object，一个编译单元初始化这个静态对象，另一个使用之，那么很可能会出现初始化前被使用，因为
> the relative order of initialization of non-local static objects defined in different translation units is undefined.

举个例子，某个库有一个头文件fileSystem.h内容如下，
{% highlight cpp %}
class FileSystem {	// from your library’s header file
public:
	...
	std::size_t numDisks() const;	// one of many member functions
	...
};
extern FileSystem tfs;	// declare object for clients to use
			// (“tfs” = “the file system” ); definition
			// is in some .cpp file in your library
{% endhighlight %}

库用户定义了一个类，
{% highlight cpp %}
class Directory {				// created by library client
public:
	Directory( params );
	...
};
Directory::Directory( params )
{
	...
	std::size_t disks = tfs.numDisks();	// use the tfs object
	...
}
{% endhighlight %}

然后，这个类被构造时，
{% highlight cpp %}
Directory tempDir( params );	// directory for temporary files
{% endhighlight %}
如果'tfs'没有被提前构造，'tempDir'就会在未初始化前使用它。

这些代码可能是不同的人在不同时间写入不同的文件(translation units)，保证初始化顺序几乎不可能且繁琐(尤其是static objects由隐式模板实例化生成时)。

为了解决这个问题，我们改变设计，用local static objects 代替 static objects。

C++保证local static objects在对应函数第一次调用时执行到其定义处被初始化。所以我们用函数调用(返回对象引用)代替直接访问静态对象。这样有个额外的好处，如果函数没有被调用，local static objects将不会构造更不会析构。
{% highlight cpp %}
class FileSystem {...}; 	//as before

FileSystem& tfs()		//this replaces the tfs object; it could be
{				//static in the FileSystem class
	static FileSystem fs;	//define and initialize a local static object
	return fs;		//return a reference to it
}

class Directory {...};	// as before

Directory::Directory(params)	//as before, except references to tfs are 
{				//now to tfs()
	...
	std::size_t disks = tfs().numDisks();
	...
}

Directory& tempDir()			//this replaces the tempDir object; it
{					//could be static in the Directory class
	static Directory td(params);	//define/initialize local static object
	return td;			// return reference to it
}
{% endhighlight %}

谨记，用local static objects代替non-local static objects之后，在多线程编程中会成为问题(只要静态对象都一样)。一个解决之道是在程序单线程启动部分，就手动调用函数完成静态对象初始化，消除初始化相关的数据竞争。

如果初始化顺序出现循环依赖情况，必须解决循环依赖的设计问题。

## Conclusions
Item 1:
- Rules for effective C++ programming vary, depending on the part of C++ you are using.

Item 2:
- For simple constants, prefer const objects or enums to #defines.
- For function-like macros, prefer inline functions to #defines.

Item 3:
- Declaring something const helps compilers detect usage errors. const can be applied to objects at any scope, to function parameters and return types, and to member functions as a whole.
- Compilers enforce bitwise constness, but you should program using logical constness.
- When const and non-const member functions have essentially identical implementations, code duplication can be avoided by having the non-const version call the const version.

Item 4:
- Manually initialize objects of built-in type, because C++ only sometimes initializes them itself.
- In a constructor, prefer use of the member initialization list to assignment inside the body of the constructor. List data members in the initialization list in the same order they’re declared in the class.
- Avoid initialization order problems across translation units by replacing non-local static objects with local static objects.

-
