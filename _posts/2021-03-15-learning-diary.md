---
layout: post
title:  "2021 Mar 15th Learning Diary"
categories: journal
tags: [reading]
toc: true
--- 
The time you enjoy wasting is not wasted time. –Bertrand Russell
{: .message }

## LeetCode
[54. 螺旋矩阵](https://leetcode-cn.com/problems/spiral-matrix/)
递归完成，一层一层剥洋葱。注意基线条件，奇偶不同导致最后是1还是0，二者都需check。


## ISOC++ wiki
读完[virtual functions wiki faq](https://isocpp.org/wiki/faq/virtual-functions)。

### What’s the difference between how virtual and non-virtual member functions are called?

non-virtual member function在编译器静态检查函数位置，virtual member function在运行时通过隐藏的虚指针找到虚表(虚表拥有指向虚函数的指针)，从而查找相应的函数地址。

实现上空间与时间的开销是很小的。空间上，每个object一个虚指针，每个类一个虚表。时间上，two extra fetches，找虚表，查虚表。

> Note: the above discussion is simplified considerably, since it doesn’t account for extra structural things like multiple inheritance, virtual inheritance, RTTI, etc., nor does it account for space/speed issues such as page faults, calling a function via a pointer-to-function, etc. 

开销小的结论，是没有考虑结构上的问题如多继承，虚继承，RTTI等，也没考虑页错误与通过函数指针call函数的时空开销。

### What happens in the hardware when I call a virtual function? How many layers of indirection are there? How much overhead is there?

深入考虑实现虚函数的问题，其实每个编译器实现都不同(compiler-dependent)，但大体上是以下范式。

比如一个类有五个虚函数。
{% highlight cpp %}
// Your original C++ source code
class Base {
public:
  virtual arbitrary_return_type virt0( /*...arbitrary params...*/ );
  virtual arbitrary_return_type virt1( /*...arbitrary params...*/ );
  virtual arbitrary_return_type virt2( /*...arbitrary params...*/ );
  virtual arbitrary_return_type virt3( /*...arbitrary params...*/ );
  virtual arbitrary_return_type virt4( /*...arbitrary params...*/ );
  // ...
};
{% endhighlight %}

Step #1: 编译器会建立一个 static table 囊括五个函数指针，并将其放在static memory的某处。很多编译器会在编译定义这五个函数的cpp文件时定义static table，即虚表。如果在目标硬件平台上，一个函数指针是一个机器字长(实际上大部分都是如此)，那么虚表将用掉5个机器字长大小的内存。用伪代码表示：
`
// Pseudo-code (not C++, not C) for a static table defined within file Base.cpp
// Pretend FunctionPtr is a generic pointer to a generic member function
// (Remember: this is pseudo-code, not C++ code)
FunctionPtr Base::__vtable[5] = {
  &Base::virt0, &Base::virt1, &Base::virt2, &Base::virt3, &Base::virt4
};
`

Step #2: 编译器会在编译时添加一个隐藏指针到Base类的每一个object中，即虚指针。相当于增加一个隐藏的data member，就像编译器重写了这个类：
{% highlight cpp %}
// Your original C++ source code
// Your original C++ source code
class Base {
public:
  // ...
  FunctionPtr* __vptr;  // Supplied by the compiler, hidden from the programmer
  // ...
};
{% endhighlight %}

Step #3: 编译器通过constructor初始化虚指针的内容，即虚表的首地址。


当我们定义一个派生类Der，并重写了前三个虚函数。Der的虚表将被编译器定义成如下形式:
`
// Pseudo-code (not C++, not C) for a static table defined within file Der.cpp
// Pretend FunctionPtr is a generic pointer to a generic member function
// (Remember: this is pseudo-code, not C++ code)
FunctionPtr Der::__vtable[5] = {
  &Der::virt0, &Der::virt1, &Der::virt2, &Base::virt3, &Base::virt4
                                          ↑↑↑↑          ↑↑↑↑ // Inherited as-is
};
`

在通过Deri contructor初始化虚指针时就使用其对应的虚表。

当调用一个虚函数时，我们通常是通过基类指针/引用：
{% highlight cpp %}
// Your original C++ code
void mycode(Base* p)
{
  p->virt3();
}
{% endhighlight %}

但编译器并不知道到底哪一个虚函数被调用，它只是通过虚指针查找：
`
// Pseudo-code that the compiler generates from your C++
void mycode(Base* p)
{
  p->__vptr[3](p);
}
`

从硬件平台角度看，通常调用虚函数的过程就是：加载虚指针，加载函数指针，call函数。

Conclusions:
- Objects of classes with virtual functions have only a small space-overhead compared to those that don’t have virtual functions.
- Calling a virtual function is fast — almost as fast as calling a non-virtual function.
- You don’t get any additional per-call overhead no matter how deep the inheritance gets. You could have 10 levels of inheritance, but there is no “chaining” — it’s always the same — fetch, fetch, call.

> Caveat: I’ve intentionally ignored multiple inheritance, virtual inheritance and RTTI. Depending on the compiler, these can make things a little more complicated.
> Caveat: Everything in this FAQ is compiler-dependent. Your mileage may vary.

此FAQ仅为简单示意。

### How can a member function in my derived class call the same function from its base class?

Use Base::f();

> When you call a virtual function using its fully-qualified name (the class-name followed by “::”), the compiler does not use the virtual call mechanism, but instead uses the same mechanism as if you called a non-virtual function. Said another way, it calls the function by name rather than by slot-number.

编译器会通过函数名进行调用(name-mangling scheme)，就像调用普通函数那样在编译期确定函数。

{% highlight cpp %}
void Der::f()
{
  Base::f();  // Or, if you prefer, this->Base::f();
}
{% endhighlight %}

编译器会将函数调用转换成以下类似形式，我们用最简单的 name-mangling scheme 表示：

`
void __Der__f(Der* this)  // Pseudo-code only; not real
{
  __Base__f(this);        // Pseudo-code only; not real
}
`

### I have a heterogeneous list of objects, and my code needs to do class-specific things to the objects. Seems like this ought to use dynamic binding but can’t figure it out. What should I do?

提取共同行为，用虚函数执行，而不是else-if 判断类型。

### When should my destructor be virtual?

> When someone will delete a derived-class object via a base-class pointer.

因为通过基类指针析构时，需要绑定到正确的destructor，才能正确释放资源。假如destructor非虚函数，delete 基类指针时永远只会调用基类的destructor。

a simplified rule of thumb:
<strong>make your destructor virtual if your class has any virtual functions.</strong>

> Note: in a derived class, if your base class has a virtual destructor, your own destructor is automatically virtual. You might need an explicitly defined destructor for other reasons, but there’s no need to redeclare a destructor simply to make sure it is virtual. No matter whether you declare it with the virtual keyword, declare it without the virtual keyword, or don’t declare it at all, it’s still virtual.

只要基类有 virtual destructor，派生类无论是否声明destructor，无论声明时是否用 virtual 关键字，派生类的destructor都是virtual function。 

### Why are destructors not virtual by default?

不是所有类都需要参与继承结构，成为基类。

### What is a “virtual constructor”?

C++不允许virtual constructor，但我们可以用别的方法实现类似想法：
{% highlight cpp %}
class Shape {
public:
  virtual ~Shape() { }                 // A virtual destructor
  virtual void draw() = 0;             // A pure virtual function
  virtual void move() = 0;
  // ...
  virtual Shape* clone()  const = 0;   // Uses the copy constructor
  virtual Shape* create() const = 0;   // Uses the default constructor
};
class Circle : public Shape {
public:
  Circle* clone()  const;   // Covariant Return Types; see below
  Circle* create() const;   // Covariant Return Types; see below
  // ...
};
Circle* Circle::clone()  const { return new Circle(*this); }
Circle* Circle::create() const { return new Circle();      }
{% endhighlight %}

> Note: The return type of Circle’s clone() member function is intentionally different from the return type of Shape’s clone() member function. This is called Covariant Return Types, a feature that was not originally part of the language. If your compiler complains at the declaration of Circle* clone() const within class Circle (e.g., saying “The return type is different” or “The member function’s type differs from the base class virtual function by return type alone”), you have an old compiler and you’ll have to change the return type to Shape*.

我们注意到clone()函数的返回类型在基类与派生类中不同，这被称为 Covariant Return Types 。这并非语言原本的特性。有些老编译器要求我们改变return type。

### Why don’t we have virtual constructors?

> A virtual call is a mechanism to get work done given partial information. In particular, virtual allows us to call a function knowing only an interfaces and not the exact type of the object. To create an object you need complete information. In particular, you need to know the exact type of what you want to create. Consequently, a “call to a constructor” cannot be virtual.

虚函数机制是通过partial information 实现的，我们只用通过接口调用虚函数而不关心具体对象类型。而构造一个对象是通过 complete information 实现的，我们必须了解构造对象的类型。换个角度想，如果可以虚构造函数，我们只有在运行时通过动态绑定才能了解被调用的constructor是哪一个，对象类型是什么，那么编译时类型信息将不完整，打破了static typing的设计。

我们可以通过工厂模式间接进行虚构造，如下面user()函数并不关心生成的类：

{% highlight cpp %}
    struct F {  // interface to object creation functions
        virtual A* make_an_A() const = 0;
        virtual B* make_a_B() const = 0;
    };
    void user(const F& fac)
    {
        A* p = fac.make_an_A(); // make an A of the appropriate type
        B* q = fac.make_a_B();  // make a B of the appropriate type
        // ...
    }
    struct FX : F {
        A* make_an_A() const { return new AX(); } // AX is derived from A
        B* make_a_B() const { return new BX();  } // BX is derived from B
    };
    struct FY : F {
        A* make_an_A() const { return new AY(); } // AY is derived from A
        B* make_a_B() const { return new BY();  } // BY is derived from B
    };
    int main()
    {
        FX x;
        FY y;
        user(x);    // this user makes AXs and BXs
        user(y);    // this user makes AYs and BYs
        user(FX()); // this user makes AXs and BXs
        user(FY()); // this user makes AYs and BYs
        // ...
    }
{% endhighlight %}

### Why a pure virtual destructor must have a definition?

非FAQ内容但有价值，答案是:

> A pure virtual destructor must have a definition, since all base class destructors are always called when the derived class is destroyed

[见此](https://en.cppreference.com/w/cpp/language/destructor#Pure_virtual_destructors)


## A Tour of C++ 1st
阅读ch11-12，ch11讲标准库里Utilities，ch12将标准库里的数值计算。

### Resource Management
核心就是RAII(Resource Acquisition Is Initialization)

#### lock
标准库的lock 类
{% highlight cpp %}
#include <mutex>

std::mutex m; // used to protext access to shared data
// ...
void f() {
    std::unique_lock<std::mutex> lck {m};
}
{% endhighlight %}

> A thread will not proceed until lck’s constructor has acquired its mutex, m (§13.5). The correspond-ing destructor releases the resource. So, in this example, unique_lock’s destructor releases themutex when the thread of control leaves f() (through a return, by ‘‘falling off the end of the func-tion,’’ or through an exception throw).

在这段代码里，线程直到 lck's constructor 获取mutex之后才会继续进行。在线程离开f()之后(正常return或抛异常)，unique_lock的destructor释放mutex资源。

#### smart pointer
C++11 提供了make_shared，C++14才有make_unique，用法类似。
{% highlight cpp %}
shared_ptr<S> p1 {new S {1,"Ankh Morpork",4.65}};
auto p2 = make_shared<S>(2,"Oz",7.62);
{% endhighlight %}

> In particular, shared_ptrs do not in themselves provide any rules for which of their owners can read and/or write the shared object. Data races (§13.7) and other forms of confusion are not addressed simply by eliminating the resource management issues.

Stroustrup认为智能指针仍然是概念上的指针，能用容器就尽量用容器。智能指针使用场景:
-当需要共享对象且对象明显不止一个owener时，用shared_ptr
-处理多态时，基类指针用unique_ptr
-当多态对象需要共享时，shared_ptr

虽然unique_ptr可以用来返回资源集合，但容器类足够简单和高效。

### Specialized Containers
一些没算入容器类的容器:
- T[N]
- array<T, N>
- bitset<N>
- vector<bool> // a specialization of vector
- pari<T, U>
- tuple<T...>
- basic_string<C>
- valarray<T>

#### array
能用array<T, N>就尽量用，除非不得不用built-in array。

> An array, defined in <array>, is a fixed-size sequence of elements of a given type where the number of elements is specified at compile time. Thus, an array can be allocated with its elements on the stack, in an object, or in static storage. The elements are allocated in the scope where the array is defined.

> An array is best understood as a built-in array with its size firmly attached, without implicit, potentially surprising conversions to pointer types, and with a few convenience functions provided.  There is no overhead (time or space) involved in using an array compared to using a built-in array. An array does not follow the ‘‘handle to elements’’ model of STL containers. Instead, an array directly contains its elements.

array 内存分配灵活，比built-in数组提供了严格的数组size，避免了隐式的潜在类型转换(转换为指针类型)，并提供了一些方便的函数。
与built-in数组相比没有overhead。但不遵循STL容器的模型，array直接包含元素。

> When necessary, an array can be explicitly passed to a C-style function that expects a pointer.

通过首地址指针或成员函数data()，array可以兼容C风格函数，e.g.，
{% highlight cpp %}
void f(int∗p, int sz); //C-style interface

void g(){
	array<int,10> a;
	
	f(a,a.size());// error : no  conversion
	f(&a[0],a.size()); // C-style use
	f(a.data(),a.siz e()); // C-style use
	auto p = find(a.begin(),a.end(),777); //C++/STL-style use
	// ...
}
{% endhighlight %}

下面是一个应用array避免隐式转换的例子：
{% highlight cpp %}
void h(){
	Circle a1[10];
	array<Circle,10> a2;
	// ...
	Shape∗p1 = a1; //OK: disaster waiting to happen
	Shape∗p2 = a2; //error : no conversion of array<Circle,10> to Shape*
	p1[3].draw(); // disaster
}
{% endhighlight %}
在这里，如果 sizeof(Shape) < sizeof(Circle)，p1指针offset会出现问题(偏移量按照Shape大小计算)。

### Time
库<chrono>计算duration，

{% highlight cpp %}
using namespace std::chrono;  //see §3.3 

auto t0 = high_resolution_clock::now(); 
do_work(); 
auto t1 = high_resolution_clock::now(); 
cout << duration_cast<milliseconds>(t1−t0).count() << "msec\n";
{% endhighlight %}

### Function Adaptors
fuction adaptor 接受一个函数作为参数，返回一个函数(function object)。例如 bind()与mem_fn()实现柯里化(Currying)/partial evaluation。

{% highlight cpp %}
int pow(int,int);
double pow(double ,double); // pow() is overloaded

auto pow2 = bind(pow,_1,2);  // error : which  pow()?
auto pow2 = bind((double(∗)(double ,double))pow,_1,2); //OK (but ugly)
{% endhighlight %}

> The curious _1 argument to the binder is a placeholder telling bind() where arguments to the resulting function object should go.

> The placeholders are found in the (sub)namespace std::placeholders that is part of <functional>.

在绑定重载函数时，必须指定函数类型。

> The function adaptor mem_fn(mf) produces a function object that can be called as a nonmember function.

mem_fn()将一个成员函数转换为function object，e.g.，
{% highlight cpp %}
void user(Shape∗ p){
	p−>draw();
	auto draw = mem_fn(&Shape::draw);
	draw(p);
}
{% endhighlight %}

当需要指定函数类型时可以用function object。Function adaptor的应用很多都可以被lambda expression代替。

### Type Functions

> A type function is a function that is evaluated at compile-time given a type as its argument or returning a type. The standard library provides a variety of type functions to help library implementers and programmers in general to write code that take advantage of aspects of the language, the standard library, and code in general.

使用type functions 进行编译期计算类型，严格化类型检查，应用类型信息提高性能，即模板元编程。

tag dispatch 本质上是给template提供额外类型信息以重载。

库<type_traits>提供了一些type predicates。

### Random Numbers
c++ 随机数库
> the standard library in <random>. A random number generator consists of two parts: 
> [1] an engine that produces a sequence of random or pseudo-random values. 
> [2] a distribution that maps those values into a mathematical distribution in a range.

### Vector Arithmetic
vector容器设计时没有考虑数值计算方面的优化，我们使用valarray进行向量的数值计算。

> the standard library provides (in <valarray>)a vector-like template, called valarray, that is less general and more amenable to optimization for numerical computation

valarray提供了 stride access 以帮助实现多维计算。

> Consider accumulate(), inner_product(), partial_sum(), and adjacent_difference() before you write a loop to compute a value from a sequence; §12.3.

尽可能使用库进行数值计算。多使用<numeric>里的算法避免循环。

### Conclusion
- 虚函数实现本质就是虚指针与虚表。
- 考虑virtual destructor的特殊性，只要有virtual必须声明并定义。明确virtual constructor的设计上为什么不会存在。
- 尽量用标准库coding。
- 模板元编程本质就是编译期做更多的事。

### Confusion
> (Note: unless Circle is known to be final (AKA a leaf), you can reduce the chance of slicing by making its copy constructor protected.) 

关于 virtual function FAQ 谈到 slice 问题， 这个尚有疑问。什么算slicing，为什么protected标识后可以减少概率？

## Reference
1. [ISO cpp wiki FAQ Inheritance — virtual functions](https://isocpp.org/wiki/faq/virtual-functions)
2. [c++ reference pure virtual destructor](https://en.cppreference.com/w/cpp/language/destructor#Pure_virtual_destructors)
