---
layout: post
title:  "Effective C++ 3rd Chapter 2"
categories: c++
tags: [reading, effective]
toc: true
--- 
She walks in beauty, like the night of cloudless climes and starry skies.
{: .message }

阅读Effective C++ 3rd(2005) 第二章。

## Item 5: Know what functions C++ silently writes and calls.
关于OOP范式，如果没有显式声明，C++会隐式生成default constructor, copy constructor, destructor and copy assignment operator(C++11后还有关于move的两个函数)。Note: 仅在被需要时生成。

当有继承关系时，默认构造函数和析构函数主要为编译器提供了一个放置幕后代码的地方，比如调用基类和非静态数据成员的构造函数和析构函数，即隐式call基类constructor and destructor。Note: 如果基类没有定义destructor为virtual function，派生类将不会继承基类destructor的虚函数属性。

隐式生成的copy constructor and copy assignment operator只能简单复制数据成员(当数据成员可复制或拥有copy constructor and copy assignment)。

### When compiler refuse to generate copy constructor and copy assignment operator?
当类的数据成员为引用时或者被指定为 const 时，编译器无法替程序员做决定，只能拒绝生成有关复制的函数。

Why?
复制时，如果引用重新绑定，这违反C++的语法(引用不可rebiding)；如果通过引用修改其引用对象为被复制内容，会影响同样引用这个对象的其他引用或者指针。对待const，语法定义其不能修改(除非强制转换)，所以也无法复制。

另外，当基类声明 copy assignment operator 为 private 时，编译器也无法成功地为派生类隐式生成 copy assignment operator，因为派生类需要覆盖基类数据成员(call copy assignment operator)，但又不具备权限(private)。

## Item 6: Explicitly disallow the use of compiler-generated functions you do not want.
编译器生成的构造函数与析构函数十分容易避免，程序员自定义即可。但如果想禁止一个类被复制，就需要一些难度(post-C++11)。C++11之后的处理方式见后。

我们可以声明 copy constructor and copy assignment operator 为 private，但需要注意的是类中成员函数与友元函数仍可以调用它们。所以更进一步，我们仅仅声明，而故意不定义。如果某处进行了调用，链接阶段将会报错。

### Uncopyable trick
如果需要在编译阶段提前报错(通常这是一个好习惯)，可以定义一个Uncopyable基类，
{% highlight cpp %}
class Uncopyable {
protected:
	Uncopyable() {}		// allow construction
	~Uncopyable() {}	// and destruction of
				// derived objects...
private:
	Uncopyable(const Uncopyable&);	// ...but prevent copying
	Uncopyable& operator=(const Uncopyable&);
};
{% endhighlight %}

对于禁止复制的类，我们可以使之继承Uncopyable，且不声明 copy ctor and copy assign operator
{% highlight cpp %}
class HomeForSale: private Uncopyable { // class no longer
	... 				// declares copy ctor or
}					// copy assign. operator
{% endhighlight %}

如此，一旦HomeForSale在某处被复制，编译器会为之生成隐式copy ctor and copy assign operator(仅在需要时生成，Item 5)，这个隐式生成版本会call基类的 copy ctor and copy assign operator，但由于基类中是private没有权限而最终导致失败(Item 5里rejection内容)。

这个技巧有些与众不同，比如继承方式是private，比如作为基类destructor是非虚函数(这点很特殊，因为一般基类destructor is virtual)。而且，虽然Uncopyable是空类，但可能出现多继承，无法进行空基类优化。当然我们可以忽略这些直接使用之，比如按照上述方式使用Boost库中的noncopyable类(命名略怪)。

Rust相关处理很像这个技巧。实现Concept才是正途。

### C++11 mark delete 
C++11之后直接标记为delete即可，例如unique_ptr类，
{% highlight cpp %}
// Disable copy from lvalue.
unique_ptr(const unique_ptr&) = delete;
unique_ptr& operator=(const unique_ptr&) = delete;
{% endhighlight %}

## Item 7: Declare destructors virtual in polymorphic base classes.
老生常谈。

<strong>declare  a  virtual  destructor  in  aclass if and only if that class contains at least one virtual function. </strong>

只要可成为多态的基类(用到virtual)，就必须声明virtual destructor。

继承STL中的类(vector, map 等均无virtual destructor)十分危险。

<strong>pure destructor must have a definition</strong>

派生类析构时，先调用本身析构函数，然后依次调用基类析构函数(编译器生成调用)。所以析构函数必须要有定义，否则会出现链接错误。

更多讨论以前写过，[见此](https://qifanwang.github.io/journal/2021/03/15/learning-diary/#isoc-wiki)

## Item 8: Prevent exceptions from leaving destructors.
C++ 没有禁止destructor 产生异常。在destructor中抛出异常，程序会离开destructor, 可能会造成UB。

为了解决这列情况，通常有两种思路，
第一种是Terminate the program:
{% highlight cpp %}
DBConn::~DBConn()
{
	try { db.close(); }
	catch (...) {
		make log entry that the call to close failed;
		std::abort();
	}
}
{% endhighlight %}
直接退出程序，这样是最保险的做法，防止UB的产生。

第二种是Swallow the exception:
{% highlight cpp %}
DBConn::~DBConn()
{
	try { db.close(); }
	catch (...) {
		make log entry that the call to close failed;
	}
}
{% endhighlight %}
有时我们需要程序继续运行，这样会冒UB的风险，程序员必须保证程序继续执行是可靠无错误的。

如果设计接口时，用户需要针对异常作出反应，我们可以将destructor中将抛出异常的部分，单独抽出作为一个函数，
{% highlight cpp %}
class DBConn {
public:
	...
	void close()					// new function for
	{						// client use
		db.close();
		closed = true;
	}
	~DBConn()
	{
		if (!closed) {
			try {				// close the connection
				db.close();		// if the client didn’t
			}
			catch (...) {						// if closing fails,
				make log entry that call to close failed;	// note that and
				...						// terminate or swallow
			}
		}
	}
private:
	DBConnection db;
	bool closed;
};
{% endhighlight %}
如此，destructor里的调用就成为了备份操作。这样设计库接口，可以保证用户有机会处理异常(用户不处理就交给destructor)。用户无法抱怨是库swallow the exception OR terminate the program，因为他们可以处理这些异常。

## Item 9: Never call virtual functions during construction or destruction.
当在有继承关系的类的constructor OR destructor中调用虚函数时，被调用的虚函数通常不是我们希望的派生类的虚函数(surprise)。

派生类构造时，先构造基类，如果基类构造函数调用虚函数，这时调用的虚函数是基类版本(如果非纯虚函数)。从运行时类型角度看，这也是可以理解的，此时派生类部分成员没有被构造，对象当前类型只能被当作基类。直到派生类的构造函数开始执行时，对象才被当作派生类。

析构函数同理，当派生类析构函数开始执行时，派生类成员被C++假定为失效，对象被RTTI(Runtime Type Information)当作下一层次的基类。

值得注意的是，有时类的构造函数会调用一些成员函数，被调用的成员函数调用了虚函数，这也会造成Surprise。只是更不容易被察觉(间接)。

如果程序员希望实现不同类在构造或是析构时的一些多态操作，又希望减少代码冗余。我们可以将这种‘多态’，通过参数化差异或时数据成员差异实现。比如，构造每一个类的对象时进行打印信息，可以通过private static 函数生成基类构造函数需要的信息，
{% highlight cpp %}
class Transaction {
public:
	explicit Transaction(const std::string& logInfo);
	void logTransaction(const std::string& logInfo) const; // now a non-virtual func
	...
};
Transaction::Transaction(const std::string& logInfo)
{
	...
	logTransaction(logInfo);				// now a non-virtual func
}
class BuyTransaction: public Transaction {
public:
	BuyTransaction( parameters )
	: Transaction(createLogString( parameters ))		// pass log info
	{ ... }							// to base class
	...							// constructor
private:
	static std::string createLogString( parameters );
};
{% endhighlight %}

## Item 10: Have assignment operators return a reference to *this.
Just a convention.
类的赋值操作符，均返回解this指针的引用。即使是'+='与'-='等操作符，或是参数类型非 const T &，也最好遵循这个约定(STL与built-in types均遵循)。
{% highlight cpp %}
class Widget {
public:
	Widget& operator=(const Widget& rhs)
	{
		...
		return *this;
	}
	Widget& operator+=(const Widget& rhs)
	{
		...
		return *this;
	}
	Widget& operator=(int rhs)
	{
		...
		return *this;
	}
	...
};
{% endhighlight %}

## Item 11: Handle assignment to self in operator=.
由于C++ aliasing的设计，一个对象被多个引用索引或是指针指向的情况时有出现。即使类型不同，也有可能出现这种情况，比如在继承结构中，
{% highlight cpp %}
class Base { ... };
class Derived: public Base { ... };
void doSomething(const Base& rb, 	// rb and *pd might actually be
			Derived* pd);	// the same object
{% endhighlight %}
程序员必须保证当函数参数为多个引用或指针时，即使操作同一个对象，结果也是正确的。

### Self assignment examples
考虑类设计如下，
{% highlight cpp %}
class Bitmap { ... };
class Widget {
	...
private:
	Bitmap *pb;	// ptr to a heap-allocated object
};
{% endhighlight %}

这是一个operator=的unsafe(self-assignment unsafe AND exception unsafe)实现，
{% highlight cpp %}
Widget&
Widget::operator=(const Widget& rhs)
{
	delete pb;			// stop using current bitmap
	pb = new Bitmap(*rhs.pb);	// start using a copy of rhs’s bitmap
	return *this;
}
{% endhighlight %}

当出现自赋值时，this与rhs引用同一个对象，将出现解引用deleted指针的UB情况。一个比较简单的解决方法是identify test，如果同一个对象则直接返回，
{% highlight cpp %}
Widget&
Widget::operator=(const Widget& rhs)
{
	if (this == &rhs) return *this;// identity test: if a self-assignment,

	delete pb;
	pb = new Bitmap(*rhs.pb);
	return *this;
}
{% endhighlight %}

这样只解决了self-assignment unsafe，但如果在new操作或是Bitmap复制构造时出现异常，this索引对象的内容已经被delete成空。我们可以利用顺序，通过覆盖消除抛出异常可能带来的影响，
{% highlight cpp %}
Widget& Widget::operator=(const Widget& rhs)
{
	Bitmap *pOrig = pb;		// remember original pb
	pb = new Bitmap(*rhs.pb);	// point pb to a copy of rhs’s bitmap
	delete pOrig;			// delete the original pb
	return *this;
}
{% endhighlight %}
这个例子也可以在函数开始处添加 identify test，但需要注意的是引入if语句也会带来运行时开销。

还有一个copy and swap的例子，本质就是先复制一份rhs，然后与之交换内容，这里就不赘述了。

## Item 12: Copy all parts of an object.
自定义复制相关函数时，如果增加数据成员，勿忘更新复制相关函数(更不用说constructor)。

在继承关系中也是如此，因为我们放弃了编译器提供的隐式copy constructor AND copy assignment operator，所以隐式调用基类相关复制函数也被放弃了，自定义时需要显式地调用。

所以Item 12的关键是，
- copy  all  local data members
- invoke the appropriate copying function in all base classes, too.

### Avoid code duplication
实践中，两个复制函数可能会有相当一部分逻辑重合，我们应当抽取出，作为第三个函数被它们调用。而不是两个函数调用对方(无论是两种调用的任何一个)，这样容易出错(corrupt objects)。

## Conclusions
Item 5:
- Compilers may implicitly generate a class’s default constructor, copy constructor, copy assignment operator, and destructor.

Item 6:
- To disallow functionality automatically provided by compilers, declare the corresponding member functions private and give no implementations. Using a base class like Uncopyable is one way to do this.
- Since C++11, we could just mark them as 'delete'.

Item 7:
- Polymorphic base classes should declare virtual destructors. If a class has any virtual functions, it should have a virtual destructor.
- Classes not designed to be base classes or not designed to be used polymorphically should not declare virtual destructors.

Item 8:
- Destructors should never emit exceptions. If functions called in a destructor may throw, the destructor should catch any exceptions, then swallow them or terminate the program.
- If class clients need to be able to react to exceptions thrown during an operation, the class should provide a regular (i.e., non-destructor) function that performs the operation.

Item 9:
- Don’t call virtual functions during construction or destruction, because such calls will never go to a more derived class than that of the currently executing constructor or destructor.

Item 10:
- Have assignment operators return a reference to *this .

Item 11:
- Make sure operator= is well-behaved when an object is assigned to itself. Techniques include comparing addresses of source and target objects, careful statement ordering, and copy-and-swap.
- Make sure that any function operating on more than one object behaves correctly if two or more of the objects are the same.

Item 12:
- Copying functions should be sure to copy all of an object’s data members and all of its base class parts.
- Don’t try to implement one of the copying functions in terms of the other. Instead, put common functionality in a third function that both call.

本章主要为OOP范式构造、析构与复制的注意事项。
