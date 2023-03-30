---
layout: post
title:  "ISOCPP FAQ: Exceptions and Error Handling (First Part)"
categories: ISOCPP FAQ
tags: [c++, exception, translation]
toc: true
--- 
If you want to live a happy life, tie it to a goal, not to people or things.
{: .message }

这篇笔记是 ISOCPP 中 [C++ 异常与错误处理的 FAQ](https://isocpp.org/wiki/faq/exceptions)的翻译与阅读心得(我写的心得以括号形式组织)。


## Why use exceptions?

使用异常能给我们带来什么好处？简单的回答是"使用异常进行错误处理可以使您的代码更简单、更干净，并且不太可能遗漏错误"。但是老式的 `errno` 和 if 语句有什么错呢？使用他们，错误处理和正常代码紧密交织在一起。代码就会变得混乱，并且很难确保您已经处理了所有错误(意大利面条代码或老鼠窝般的测试，说的就是你 `golang err`)。

首先，有些事情是没有异常就无法做到的。考虑在构造函数中检测到的错误；如何报告构造错误？(因为构造函数没有"返回值"，无法返回错误码。当然，二段式构造并设置类的错误状态字段是可以的，但很臃肿)C++可以抛出一个异常指示构造错误。这就是 `RAII`(资源获取即初始化)的基础，`RAII` 也是一些最有效的现代c++设计技术的基础：构造函数的工作是为类建立 `invariant` (保证成员函数运行的环境)，这通常需要获取资源，如内存、锁、文件、套接字等。

### code example

想象一下，没有异常，我们将如何处理构造函数中检测到的错误？请记住，通常调用构造函数以在变量中初始化/构造对象：
```cpp
    vector<double> v(100000);   // needs to allocate memory
    ofstream os("myfile");      // needs to open a file
```

`vector` 或 `ofstream` (输出文件流)构造函数可以将变量设置为坏状态(`ifstream` 默认下就是如此)，从而使后续的每个操作都失败。这并不理想。例如，在 `ofstream` 的情况下，如果忘记检查 `open` 操作是否成功，输出就会消失。对于大部分类来说结果更糟。我们需要写：
```cpp
    vector<double> v(100000);   // needs to allocate memory
    if (v.bad()) { /* handle error */ } // vector doesn't actually have a bad(); it relies on exceptions
    ofstream os("myfile");      // needs to open a file
    if (os.bad())  { /* handle error */ }
```

这是对每个对象一个额外的测试(必须写/记住或忽略，就像C/golang)。对于由多个对象组成的类来说，这非常混乱，特别是当这些子对象相互依赖时。可以看 [The C++ Programming Language](http://stroustrup.com/3rd.html) 的第 14 章 8.3 节，附录[Appendix E: Standard-Library Exception Safety](http://stroustrup.com/3rd_safe0.html)与论文 [Exception safety: Concepts and techniques](http://stroustrup.com/except.pdf)以了解更多。

因此，如果没有异常，构造函数的编写可能会很棘手，那么普通的函数呢？我们可以返回错误代码，也可以设置一个非局部变量(如代表上一个错误的序号的 `errno`)。设置并读取一个全局变量很需要技巧，除非您立即测试它，否则其他一些函数可能已经重新设置了它(想想自定义 `signal handler` 时需要小心地记录 `errno` 并还原)。如果可能有多个线程访问全局变量，请不要考虑这种技术(还得控制并发)。而返回值的问题在于，选择错误返回值需要很聪明，而且有时是不可能的：
```cpp
    double d = my_sqrt(-1);     // return -1 in case of error
    if (d == -1) { /* handle error */ }
    int x = my_negate(INT_MIN); // Duh?
```

这个 `negate()` 没有可能的返回值用以表达错误：每一个可能的int都是某些int的正确答案，并且针对补码表示中最小的负数是没有正确答案的。在这种情况下，我们将需要一对返回值对，并且像往常一样记得测试(想想 golang 的返回值设计 `ret, err = foo()`)。请参阅 [Stroustrup’s Beginning programming book](http://stroustrup.com/programming.html)以获得更多示例和解释。(还要考虑不同的系统对于不同错误码的定义不一样，如 UNIX 错误码与C标准库错误码，还有自定义的错误码，这更麻烦了)

### Common objections to the use of exceptions

- 异常实现的开销大。
- 在硬实时系统禁止异常。
- 从 `new` 调用的构造函数抛出异常会导致内存泄漏

针对第一条，现代c++实现将使用异常的开销减少到百分之几(据说是 `3%`，不知道这个数字代表哪方面的开销)，这是与没有错误处理相比的。编写带有错误返回码和测试代码也不是免费的。根据经验，当不抛出异常时，异常处理的成本非常低。它在某些实现上没有任何成本。所有成本都是在抛出异常时产生的。也就是说，普通代码比使用错误返回代码和测试的代码更快。只有在出现错误时才会产生成本。

关于第二条，硬实时和安全关键应用程序(比如飞行控制软件)的确要禁止异常，如在 `JSF++` 中 Stroustrup 自己完全禁止异常。如果计算时间过长，就会有人死亡。出于这个原因，我们必须保证响应时间，而在当前的工具支持下，我们无法为异常做到这一点。在这种设计下，甚至堆分配都是被禁止的。实际上，对于错误处理， `JSF++` 建议模拟了异常的使用，以预期有一天我们有了正确的工具，即使用异常。(这也是为什么一些项目禁用异常的由来，追求响应时间，但此时标准库与非 `std:nothrow` 的 `new` 也不能用了，因为他们会有异常)

最后一条是一个由一个编译器的错误引起的迷思，这个错误在十多年前就被立即修复了。

## How do I use exceptions?

可以查询 [The C++ Programming Language](http://stroustrup.com/3rd.html) 的第 14 章 8.3 节与附录[Appendix E: Standard-Library Exception Safety](http://stroustrup.com/3rd_safe0.html)。附录侧重于在高要求的应用程序中编写异常安全代码，不是为新手编写的。

在 C++ 中，异常用于表示无法在本地处理的错误(这个本地指当前栈帧)，例如在构造函数中获取资源的失败，
```cpp
    class VectorInSpecialMemory {
        int sz;
        int* elem;
    public:
        VectorInSpecialMemory(int s) 
            : sz(s) 
            , elem(AllocateInSpecialMemory(s))
        { 
            if (elem == nullptr)
                throw std::bad_alloc();
        }
        ...
    };
```

但是，不要将异常简单地用作从函数返回值的另一种方式(再怎么也是有开销的)。大多数用户应认为，语言定义鼓励他们**将异常处理代码视为错误处理代码**，并且语言实现被优化以体现这个假设。

一个关键技术是 `RAII`，它使用带有析构函数的类来对资源管理施加顺序。例如，

```cpp
    void fct(string s)
    {
        File_handle f(s,"r");   // File_handle's constructor opens the file called "s"
        // use f
    } // here File_handle's destructor closes the file  
```

如果 `fct()` 的 `use f` 部分抛出异常，析构函数仍将被调用，文件将被正确关闭。这与常见的不安全用法形成对比，

```cpp
    void old_fct(const char* s)
    {
        FILE* f = fopen(s,"r"); // open the file named "s"
        // use f
        fclose(f);  // close the file
    }
```

如果 `old fct` 的 `use f` 部分抛出异常或简单地返回，则文件不会正确关闭(文件资源泄漏了！)。在C程序中，**longjmp()** 是一个额外的危险。(具体可以看 [C 标准 longjmp](https://en.cppreference.com/w/c/program/longjmp) 如何使用)

## What shouldn’t I use exceptions for? 

既然异常被设计用来支持错误处理，那么何时不应该使用？

具体使用场景，

- 使用 `throw` 只表示一个错误(这特别意味着函数不能完成它所声称的功能并建立它的后置条件 `post-condition` )。
- 只有当您知道可以处理错误时，才使用 `catch` 来指定错误处理操作(可能是将其转换为另一种类型并重新抛出该类型的异常，例如 `catch bad_alloc` 并重新 `throw no_space_for_file_buffers`。这一过程最好不要缩减异常信息，否则增大了 debug 难度)。

禁止使用的场景，

- 不要使用 `throw` 来表示函数使用中的编码错误。使用 `assert` 或其他机制，要么将进程发送到调试器，或使进程崩溃并收集崩溃转储以供调试(gdb 读 core dump 文件)。
- 如果发现某组件的意外违反了某个 `invariant`，请不要使用 `throw`，还是使用 `assert` 或其他机制终止程序。抛出异常不会治愈内存损坏，而且可能导致重要用户数据的进一步损坏(除了进一步损坏，还会将真正违反 `invariant` 的位置隐藏起来。例如，拿到函数返回的一个空指针，直到很久之后/其他线程才访问它，试问最初的错误场景如何找到，这很费时间 debug )。
- 不要用异常操作控制流。

在其他语言中，异常还有其他常见的用法，但在 c++ 中却不一定是惯用用法。而且这些用法 c++ 的实现故意没有很好地支持(这些实现是基于异常用于错误处理的假设进行优化的，所以不支持其他非错误处理的用法是一种减少开销的思想)。

尤其是不要对控制流使用异常。不要简单地将 `throw` 看作函数返回值的另一种方式。这样做会很慢，而且会让大多数c++程序员感到困惑，因为他们已经习惯了只将异常用于错误处理。同样，`throw` 也不是退出循环的好方法。

## What are some ways try/catch/throw can improve software quality?

消除 if 语句的使用。(又想起 golang 了)

常用的 `try/catch/throw` 替代方法是返回一个返回码(有时称为错误代码)，该返回码通过某些条件语句明确测试。例如，`printf() scanf() malloc()` 以这种方式工作：调用者应该测试返回值以查看函数是否成功。

虽然返回码技术有时是最合适的错误处理技术，但添加不必要的 if 语句会产生一些严重的副作用，

- 质量降级：众所周知，条件语句包含错误的可能性比其他类型语句大约要高出十倍。因此，在其他条件不变时，如果您可以从代码中消除条件子句/条件语句，则可能写出鲁棒的代码。
- 降低软件发布速度：由于条件语句是分支点，与白盒测试所需的测试用例数量有关，因此不必要的条件语句增加了用于测试的时间。基本上，如果您不遍历每个分支，则在测试条件下将有部分代码永远不会执行，直到您的用户/客户看到它们。(本质就是提高了程序员写单元测试增加分支/语句覆盖的难度)。
- 增加开发成本：发现 bug，bug 修复和测试都因不必要的控制流复杂度而增加。

因此，与 `if + return err code` 相比，使用 `try/catch/throw` 可以写出较少错误的代码，开发成本更低，并且发布速度更快。当然，如果您的组织没有任何 `try/catch/throw` 的经验知识，那首先要在玩具项目上使用它，只是为了确保开发者知道自己在做什么 - 我们应该于上前线之前在靶场熟悉我们的武器。

## I’m still not convinced: a 4-line code snippet shows that return-codes aren’t any worse than exceptions; why should I therefore use exceptions on an application that is orders of magnitude larger?

因为异常比返回码更容易扩展。4行示例代码的问题在于它只有4行。写4000行，你就能看到区别了。下面是一个经典的四行示例，首先带有异常：
```cpp
try {
  f();
  // ...
} catch (std::exception& e) {
  // ...code that handles the error...
}
```

下面是同样的例子，这次使用返回码(rc代表返回码)：
```cpp
int rc = f();
if (rc == 0) {
  // ...
} else {
  // ...code that handles the error...
}
```

人们指着那些玩具样例说：“异常并不能提高编码、测试或维护成本；为什么我要在实际项目中使用它们呢?”

原因是，异常可以帮助您处理现实世界的应用程序。您可能不会从一个玩具示例中看到太多好处。

在现实世界中，检测问题的代码必须将错误信息传播回处理该问题的某个函数。这种“错误传播”通常需要经过数十个函数，如 `f1() calls f2() calls f3(), etc.`，并且在 `f10() 或 f100()` 中发现了问题。有关该问题的信息需要一直传播回 `f1()`，因为只有 `f1()` 有足够的上下文知道应该如何处理。在交互式应用程序中，`f1()` 通常在主事件循环上(可能离循环所在栈帧很近)，但是无论如何，检测到问题的代码通常与处理问题的代码不同，并且错误信息需要在两者之间的栈帧传播。

异常使这种错误传播变得容易：
```cpp
void f1()
{
  try {
    // ...
    f2();
    // ...
  } catch (some_exception& e) {
    // ...code that handles the error...
  }
}
void f2() { ...; f3(); ...; }
void f3() { ...; f4(); ...; }
void f4() { ...; f5(); ...; }
void f5() { ...; f6(); ...; }
void f6() { ...; f7(); ...; }
void f7() { ...; f8(); ...; }
void f8() { ...; f9(); ...; }
void f9() { ...; f10(); ...; }
void f10()
{
  // ...
  if ( /*...some error condition...*/ )
    throw some_exception();
  // ...
}
```

只有检测错误的代码 `f10()` 和处理错误的代码 `f1()` 有些杂乱。

然而，使用返回码会强迫错误传播的杂乱代码进入这两者之间的所有函数。下面是使用返回码的等效代码：
```cpp
int f1()
{
  // ...
  int rc = f2();
  if (rc == 0) {
    // ...
  } else {
    // ...code that handles the error...
  }
}
int f2()
{
  // ...
  int rc = f3();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f3()
{
  // ...
  int rc = f4();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f4()
{
  // ...
  int rc = f5();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f5()
{
  // ...
  int rc = f6();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f6()
{
  // ...
  int rc = f7();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f7()
{
  // ...
  int rc = f8();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f8()
{
  // ...
  int rc = f9();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f9()
{
  // ...
  int rc = f10();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f10()
{
  // ...
  if (...some error condition...)
    return some_nonzero_error_code;
  // ...
  return 0;
}
```

返回码解决方案展开了错误逻辑。函数 `f2()` 到 `f9()` 具有显式手工代码以将错误条件传播回 `f1()`。这是不好的：

- 它使函数 `f2() through f9()` 与额外的决策逻辑相混淆。(最常见的 bug 原因，中间的传播决策逻辑只要写错一个，错误处理就断了)
- 它增加了代码量。
- 它将函数 `f2() through f9()` 编程逻辑的简单性掩盖住了。(提高开发者的脑力成本)
- 它要求返回值表现两个不同的职责，函数 `f2() through f10()` 将需要处理`函数成功，结果是xxx`和`函数失败，错误信息是yyy`。当 `xxx` 和`yyy` 的类型不同时，有时需要额外的引用参数来将成功和不成功的情况传播给调用者。(下一个FAQ也会提到)

如果非常局限地看上面示例中的 `f1()` 和 `f10()`，异常不会给您带来太大的改进。但如果你睁开眼睛，放眼全局，你会发现两者之间的所有函数都有很大的不同。

结论是，异常处理的一个优点是以一种更干净简单的方法，将错误信息传播回调用者，让调用者可以处理错误。另一个优点是，函数不需要额外的机制来传播“成功”和“失败”的情况。玩具示例通常既不会强调错误传播，也不会强调对两种返回类型的处理，因此它们不代表现实世界代码。

## How do exceptions simplify my function return type and parameter types?

使用返回码时，通常需要两个或多个不同的返回值：一个表示函数成功并给出计算结果，另一个将错误信息传播回调用者。例如，如果函数有5种失败的方式，那么您可能需要多达6个不同的返回值：成功的计算返回值，以及5种错误情况下的不同的返回值。

让我们把它简化为两种情况：

- “I succeeded and the result is xxx.”
- “I failed and the error information is yyy.”

让我们来看一个简单的例子：我们想要创建一个Number类，它支持四种算术运算:加、减、乘和除。这显然可以重载操作符，因此我们定义它们：

```cpp
class Number {
public:
  friend Number operator+ (const Number& x, const Number& y);
  friend Number operator- (const Number& x, const Number& y);
  friend Number operator* (const Number& x, const Number& y);
  friend Number operator/ (const Number& x, const Number& y);
  // ...
};
```

使用也很简单，
```cpp
void f(Number x, Number y)
{
  // ...
  Number sum  = x + y;
  Number diff = x - y;
  Number prod = x * y;
  Number quot = x / y;
  // ...
}
```

但是我们有一个问题：处理错误。加法可能导致溢出，除法可能导致除零或下溢等。如何既报告 `I succeeded，结果为xxx`，又报告 `I failed，错误信息为yyy`。

如果我们使用异常，这很容易。把 **异常看作是一个单独的返回类型，只在需要时使用**。因此，我们只需定义所有异常，并在需要时抛出它们：

```cpp
void f(Number x, Number y)
{
  try {
    // ...
    Number sum  = x + y;
    Number diff = x - y;
    Number prod = x * y;
    Number quot = x / y;
    // ...
  }
  catch (Number::Overflow& exception) {
    // ...code that handles overflow...
  }
  catch (Number::Underflow& exception) {
    // ...code that handles underflow...
  }
  catch (Number::DivideByZero& exception) {
    // ...code that handles divide-by-zero...
  }
}
```

但是如果我们使用返回码而不是异常，生活会变得困难和混乱。当您不能同时将正确的数字和错误信息(包括错误的详细信息)塞进number对象中时(可以塞进去，那也会增大对象尺寸)，您可能最终会使用额外的引用参数来处理两种情况中的一种：成功或失败，或者两者都是。在不损失通用性的情况下，我将通过一个正常的返回值处理计算结果，而通过一个引用参数处理错误情况，但是您也可以轻松地做相反的事情。结果如下：
```cpp
class Number {
public:
  enum ReturnCode {
    Success,
    Overflow,
    Underflow,
    DivideByZero
  };
  Number add(const Number& y, ReturnCode& rc) const;
  Number sub(const Number& y, ReturnCode& rc) const;
  Number mul(const Number& y, ReturnCode& rc) const;
  Number div(const Number& y, ReturnCode& rc) const;
  // ...
};
```

现在这段代码是如何使用它，
```cpp
int f(Number x, Number y)
{
  // ...
  Number::ReturnCode rc;
  Number sum = x.add(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number diff = x.sub(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number prod = x.mul(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number quot = x.div(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  // ...
}
```

这样做的关键在于，您通常必须搞乱使用返回码的函数接口，特别是如果有更多的错误信息要传播回调用者。例如，如果有5个错误条件，并且错误信息需要不同的数据结构，那么您可能会得到一个相当混乱的函数接口。

所有这些混乱都不会出现异常。异常可以被认为是一个单独的返回值，就好像函数根据可抛出的内容，自动增长了新的返回类型和返回值一样。

注意：请不要给我(这条大概是 Stroustrup 写的)写信，说您建议使用返回码并将错误信息存储在命名空间范围，全局或静态变量，例如 `number::lasterror()`。那不是线程安全的。即使您今天没有多线程，您也应该很少想永久阻止任何人在多线程下使用这个类。当然，如果真的阻止，您应该写很多丑陋注释，警告未来的程序员这段代码不是线程安全的，并且如果没有实质性的重写，这段代码可能无法成为线程安全的。

## What does it mean that exceptions separate the “good path” (or “happy path”) from the “bad path”?

这是异常相对于返回码的另一个优点。

好路径("good path" "happy paht" 多种说法)是指当程序执行都很顺利，没有问题时的控制流路径。

坏路径( "bad path" "error path")是控制流在出现问题时所采取的路径。

如果处理得当，异常可以将二者分开。

下面是一个简单的例子：函数 `f()` 依次调用函数 `g()`， `h()`， `i()`和`j()`。如果其中任何一个失败并出现 `foo` 或 `bar` 错误，`f()`将立即处理错误，然后成功返回。如果发生任何其他错误，`f()`将把错误信息传播回调用方。

```cpp
void f()  // Using exceptions
{
  try {
    GResult gg = g();
    HResult hh = h();
    IResult ii = i();
    JResult jj = j();
    // ...
  }
  catch (FooError& e) {
    // ...code that handles "foo" errors...
  }
  catch (BarError& e) {
    // ...code that handles "bar" errors...
  }
}
```

好路径和坏路径被清晰地分开。好路径是 `try` 块的主体，您可以线性地阅读代码，如果没有错误，控制流将以最简单的路径通过这些行。坏路径是`catch`块的主体和调用方中匹配的 `catch` 块的主体。(调用方如果没有匹配的 `catch` 块，则一路 unwind 结束)。

使用返回码而不是异常会使这种情况变得混乱，以至于很难看到相对简单的算法(错误处理的逻辑混杂进算法的控制流中)。好路径和坏路径不可避免地混杂在一起：

```cpp
int f()  // Using return-codes
{
  int rc;  // "rc" stands for "return code"
  GResult gg = g(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  HResult hh = h(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  IResult ii = i(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  JResult jj = j(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  // ...
  return Success;
}
```

将好路径与坏路径混合，很难看到代码应该做什么。与使用异常的版本(几乎是自我记录错误处理的代码)对比，异常版本下的算法功能非常明显。

## Conclusion

这篇笔记仅记录 [C++ 异常与错误处理的 FAQ](https://isocpp.org/wiki/faq/exceptions)的第一部分，其他部分见之后笔记。基本总计总结如下。

异常的设计目的：
- 报告构造函数的错误(最主要的)
- 不用设置 errno 或设计返回码表达错误。

针对反对异常的理由与解释，
反对理由 | 解释
--- | ---
开销大。 | 开销成本在出现错误时才产生，无错误时比返回码更好，因为减少分支。
硬实时系统不可用 | 确实，在硬实时系统需要避免异常、标准库，甚至是堆分配。
从 `new` 调用的构造函数抛出异常会导致内存泄漏。| fake! 这是当年的编译器bug ，已修复。
	
使用异常的场景：
- 用 `throw` 仅表达错误(函数无法完成功能并建立 post-condition)
- 仅 `catch` 当前上下文可以处理的错误，否则继续 throw (throw过程也可以转换异常类型，但不要缩减信息)

禁止使用异常的场景：
- 不用 `throw` 表达使用函数的编码错误。(用 `assert` 或发射进程到gdb或使其崩溃收集 core dump 进行 gbd 调试)
- 发现某组件的意外违反了某 `invariant`，不要使用 `throw`，还是使用 `assert` 或其他机制终止程序(和上一点相同，本质是不要隐藏致命错误)
- 不要用异常操作控制流
	
从软件工程角度看异常的优点：
- 减少返回码判断的条件语句，提高代码鲁棒性(因条件语句含bug的可能性更高)
- 减少分支，更易测试(因为更容易分支覆盖了)
- 减少控制流复杂度，以减少发现 bug 与修复的开发成本。
- 代码更干净简单(将错误信息传播回调用者更方便)
- 不需要额外参数与返回类型区分成功与错误
- 分离代码的好路径与坏路径

## Reference
1. [C++ 异常与错误处理的 FAQ](https://isocpp.org/wiki/faq/exceptions)
2. [The C++ Programming Language](http://stroustrup.com/3rd.html)
3. [Appendix E: Standard-Library Exception Safety](http://stroustrup.com/3rd_safe0.html)
4. [Exception safety: Concepts and techniques](http://stroustrup.com/except.pdf)
5. [Stroustrup’s Beginning programming book](http://stroustrup.com/programming.html)
6. [C 标准 longjmp](https://en.cppreference.com/w/c/program/longjmp) 