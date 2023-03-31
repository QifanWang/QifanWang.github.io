---
layout: post
title:  "ISOCPP FAQ: Exceptions and Error Handling (Second Part)"
categories: ISOCPP FAQ
tags: [c++, exception, translation]
toc: true
--- 
The cure of boredom is curiosity. There is no cure for curiosity.
{: .message }

这篇笔记是 ISOCPP 中 [C++ 异常与错误处理的 FAQ](https://isocpp.org/wiki/faq/exceptions)的翻译与阅读心得(我写的心得以括号形式组织)。

## I’m interpreting the previous FAQs as saying exception handling is easy and simple; did I get it right?

No! Wrong! Stop! Go back! Do not collect $200.(这是大富翁游戏的规则，俚语)

之前 FAQ 传达的信息并不是异常处理简单易行。而是，异常处理是值得的。利大于弊。(no free lunch)

简单说，有以下代价：

- 异常处理不是免费的午餐。它需要 `discipline and rigor` 。要了解这些 `disciplines`，应该阅读剩下的 FAQ 和/或一本有关主题的好书。
- 异常处理不是万能的(not a panacea)。如果您与一个草率且无纪律的团队一起工作，那么无论他们使用异常还是返回代码，您的团队都可能会出现问题。不称职的木匠即使用了好锤子也干不好活。
- 异常处理不是一刀切(not one-size-fits-all)。即使您决定使用异常而不是返回代码，这并不意味着您可以将它们用于所有事情。这是 `discipline` 的一部分：您需要知道什么时候应该使用返回码，什么时候使用异常。
- 异常处理是一个方便的替罪羊(a convenient whipping boy)。如果你和那些指责自己的工具的人一起工作，不要提出使用异常(或者其他任何新的建议)。如果一个人的自我非常脆弱，以至于他们需要把自己搞砸的事情归咎于别人或其他事情，那么无论使用什么新技术，他们都会责怪。当然，理想情况下，你会和那些在情感上有能力学习成长的人一起工作：和他们在一起，你可以提出各种各样的建议，因为这些人会找到一种方法让它起作用，而你会在这个过程中获得乐趣。

幸运的是，在正确使用异常方面有很多智慧和见解。异常处理并不新鲜。整个行业已经经历了数百万行代码和许多人百年(person-centuries 类似人月神话说的 person-month)使用异常的努力。陪审团已经做出了裁决：异常可以被正确地使用，并且当它们被正确地使用时，它们可以改进代码。

Learn now.

## Exception handling seems to make my life more difficult; that must mean exception handling itself is bad; clearly I’m not the problem, right?? 

人可能就是问题所在。

c++ 异常处理机制可以是强大而有用的，但如果你有错误的心态，结果可能是一团糟。它是一个工具；正确使用它，它会帮助你；但如果使用不当，不要责怪工具。

如果您得到了不好的结果，例如，如果您的代码看起来不必要或过于混乱地使用 `try` 块，那么您可能患有错误的心态。这个 FAQ 给列出了一些错误的思维方式。

警告：不要简单化这些错误的心态。它们是指导方针和思维方式，而不是硬性规定(hard and fast rules)。有时你会做与它们建议完全相反的事情，不要写信告诉我"这是一个或多个例外"(没有双关语的意思，笑死 exception 既是异常也是例外)。我保证有例外。那不是重点。

以下是一些错误的异常处理思维，没有先后顺序：
- 返回码的思维方式：这会使程序员用 `Try Block` 块将其代码变混乱。基本上，他们将 `throw` 视作美化的返回码，而 `tra/catch` 是一种美化的“如果返回码表示错误”测试，并且将这些 `try blocks` 放在几乎所有可能 `throw` 的函数上。
- Java心态：在Java中，通过显式 `try/finally` 块回收非内存资源。当在 C++ 中使用此心态时，它会导致大量不必要的 `try` 块，与 RAII 相比，它会使代码杂乱并使逻辑更难追踪。从本质上讲，代码在好路径和坏路径之间来回交换(后者意味着在异常过程中采取的路径)。使用RAII，该代码大多是乐观的——这是所有的好路径，清理代码在资源持有对象的 destructor 中。这也有助于降低代码审查和单元测试的成本，因为可以孤立地验证这些资源持有对象(试想，如果用显式的 `try/catch` 块，每个副本必须单元测试并单独检查；它们不能作为一组处理。而有 RAII 则可以检查 destructor，当成一组)。
- 围绕 `pyhsical thrower` 组织 `exception class`，而不是 `throw` 的逻辑原因：例如，在银行应用程序中，假设五个子系统中的任何一个都可能会在客户资金不足时抛出异常。正确的方法是抛出代表 `throw`原因的异常，例如，“资金不足的异常”；错误的心态是每个子系统要抛出特定于子系统的异常。例如，Foo子系统可能会抛出`class FooException`的对象，Bar子系统可能会抛出`class BarException`的对象，等等。这通常会导致额外的`try/catch`块，例如，捕获`FooException`，将其重新包装到`BarException`中，然后将后者抛出。通常，异常类应该代表问题，而不是注意到问题的代码块。
- 在一个异常对象中使用比特位数据来区分不同的错误类别：假设银行应用程序中的`Foo`子系统会为不良帐号，试图清算非流动资产以及资金不足情况抛出异常。当这三种逻辑上不同的错误以相同的异常类表示时，`catcher` 需要用 `if` 弄清楚确切问题是什么。如果您的代码只想处理不良帐号，则需要捕获`master exception class`，然后使用`if`确定它是否是您真正要处理的问题，如果不是，则将其 `rethrow`。通常，首选方法应该是针对错误条件的逻辑类别，以编码为异常的不同类型，而不是异常对象的数据。
- 以子系统基础在子系统上设计异常类：在过去，任何给定返回码的特定含义都是局限于给定函数或API的(想想 Unix API 与 C 标准库)。这是因为一个函数使用`3`作为返回码来表示“成功”，而另一个函数使用 `3` 作为返回码的含义可以不同，这是可接受的。例如，还可以表示“failed due to out of memory”等。返回码的一致性当然是首选，但通常没有如此设计，因为它没必要。采取这种心态的人通常会以相同的方式处理 c++ 异常：他们假设异常类局限于子系统。这不会结束问题，例如，许多额外的 `try` 块需要 `catch`，然后重新包装，并抛出代表同一个问题的异常的变体(a repackaged variant of the same exception 简单说就是做了没必要的转换)。所以在大型系统中，必须以全系统范围的思维方式设计异常层次结构。使得异常类可以跨子系统边界，就好像它们是将建筑结构融合在一起的智力胶的一部分。(异常类的层次不用像返回码那样不一致设计，反而在一个系统里应该设计得一致)
- 使用RAW指针(而非智能指针)：这实际上只是非RAII的一种特殊情况，但我之所以说出来，是因为它很普遍。如上所述，使用RAW指针的结果是许多额外的 `try/catch` 块，其唯一的目的是 `delete object` 然后 `rethrow the exception`。
- 将逻辑错误与运行时情况相混淆：例如，假设您有一个函数 `f(Foo* p)`，绝对不能用 `nullptr` 调用。但是，您发现某处代码有时会传`nullptr`。有两种可能性：代码要么从外部用户那里获得了不良数据(例如，用户忘记填写字段，最终导致了`nullptr`)，所以传`nullptr`；要么只是代码中犯了一个错误。在前一种情况下，您应该抛出异常，因为它是一个运行时的情况(运行时信息无法通过代码审查检测到，这不是错误)。在后一种情况下，您应该在调用者代码中修复错误。如果再次发生，您仍然可以添加代码向日志文件中写入消息。如果再次发生，您甚至可以抛出异常，但是您不仅必须在 `f(Foo* p)` 中更改代码；您一定，一定，一定在`f(Foo* p)`的调用方中修复代码。(错误可能发生在 caller/callee 方，运行时错误可以用异常捕获，但如果有 caller 代码错误，也一定要修改才是。一定要弄清楚错误的根源)

还有其他错误的异常处理心态，但希望这些能帮助您解决问题。记住：不要把这些当做硬性规定。它们是指导方针，每一项都有例外。

## I have too many try blocks; what can I do about it? 

即使您使用的是 `try/catch/throw` 的语法，您也可能具有返回码的心态。例如，您几乎可以在每个调用上放一个 `try` 块：
```cpp
void myCode()
{
  try {
    foo();
  }
  catch (FooException& e) {
    // ...
  }
  try {
    bar();
  }
  catch (BarException& e) {
    // ...
  }
  try {
    baz();
  }
  catch (BazException& e) {
    // ...
  }
}
```

尽管这使用了 `try/catch/throw` 语法，但总体结构与返回码的操作方式非常相似，因此软件开发/测试/维护成本基本上与返回代码相同。换句话说，这种方法不会比使用返回码好多少。通常，这是不好的形式。

一种方法是对每个try块都问自己这个问题：为什么我在这里使用try块？有几个可能的答案：
- 你的回答可能是，这样我就可以处理异常了。我的 `catch` 子句处理错误并继续执行，而不抛出任何其他异常。我的调用者永远不知道发生了异常。我的 `catch` 子句不抛出任何异常，也不返回任何错误代码。在这种情况下，您保留 `try` 块，这可能是好的。
- 你的回答可能是，所以我可以有一个`catch`子句来做 `ABC` 事，然后我将重新抛出异常。在这种情况下，可以考虑将`try`块更改为一个对象，该对象的析构函数执行`ABC`。例如，如果你有一个`try`块，它的`catch`子句关闭一个文件，然后重新抛出异常，考虑用file对象析构函数关闭文件,替换整个块。这通常被称为RAII。
- 您的答案可能是：“所以我可以重新包装异常：`catch Xyzexception`，提取细节，然后扔一个 `Pqrexception`。”发生这种情况时，请考虑不需要此 `catch/repackage/rethrow` 的异常对象层次结构。这通常涉及扩大 `Xyzexception` 的含义，但显然您不应该扩大太多。(这还是异常类型的设计问题)
- 当然也有其他的答案，但以上是我看到的一些常见的答案。

重点是问为什么？如果你发现了你这样做的原因，你可能会发现有更好的方法来实现你的目标。

说了这么多，不幸的是，有些人的“返回码”思维已经深深地烙进了他们的心灵，他们似乎看不到任何其他选择。如果你就是这样，那还是有希望的：找一位导师。如果你做得对，你可能会得到它。风格有时是习得的，而不仅仅是教出来的。

## Can I throw an exception from a constructor? From a destructor? 

- 对于构造函数，可以：当不能正确地初始化(构造)对象时，应该从构造函数抛出异常。没有真正令人满意的替代方法可以通过抛出退出构造函数。更多细节，在后面的 FAQ 中。
- 对于析构函数，并非如此：你可以在析构函数中抛出异常，但该异常不能离开析构函数；如果析构函数通过发出异常退出，就可能发生各种糟糕的事情，因为标准库和语言本身的基本规则将被违反。不要这样做。更多细节，在后面的 FAQ 中。

具体示例和解释请参见 [Appendix E of TC++PL3e](http://stroustrup.com/3rd_safe0.html)

有一个警告：异常不能用于一些硬实时项目。例如，请参阅JSF [air vehicle C++ coding standards](http://stroustrup.com/JSF-AV-rules.pdf)。

## How can I handle a constructor that fails? 

抛出异常。

构造函数没有返回类型，因此不可能使用返回码。因此，发出构造函数故障的最佳方法是抛出异常。如果您没有使用异常的选项，那么“最不糟糕的”方法是通过设置内部状态位将对象置为“zombie”状态(用一个字段记录)，以便对象表现成 `dead`，即使从技术上讲仍然活着。(但如何回收 zombie 态对象也需要小心考虑)

实际上，实现 `zombie` 代码是相当丑陋的。您应该更喜欢异常而不是 `zombie` 对象，但是如果您无法使用异常，`zombie` 对象可能是最不坏的选择。可见相关 FAQ [Will I sometimes use any so-called “evil” constructs?](https://isocpp.org/wiki/faq/big-picture#use-evil-things-sometimes)

注意：如果构造函数以抛出异常结束，则与对象本身关联的内存将被清理，不存在内存泄漏。例如：
```cpp
void f()
{
  X x;             // If X::X() throws, the memory for x itself will not leak
  Y* p = new Y();  // If Y::Y() throws, the memory for *p itself will not leak
}
```

关于这个话题有一些细则，所以你需要继续读下去。具体来说，您需要知道如果构造函数本身分配内存，如何防止内存泄漏(这个在本主题的 FAQ How should I handle resources if my constructors may throw exceptions?)，并且还需要知道如果使用 [“placement” new](https://isocpp.org/wiki/faq/dtors#placement-new) 而不是上面示例代码中使用的普通new会发生什么。

## How can I handle a destructor that fails?

将消息写入日志文件。终止该进程。或者打电话问人。但是不要抛出异常！

这就是为什么(这段解释得很详细)：

c++的规则是，在一个异常的 `stack unwinding` 过程中调用的析构函数永远不能抛出另一个异常。例如，如果有 `throw Foo()`，将有 `stack unwinding`，所以
```cpp
throw Foo()
```
与
```cpp
 }
  catch (Foo e)
 {
```
中间所有的栈帧都被弹出。这被称为 `stack unwinding`。

想象一下，在 `stack unwinding` 期间，所有这些栈帧中的局部对象都将被销毁。如果其中一个析构函数抛出一个异常(比如抛出一个 `Bar` 对象)，那么c++运行时系统就会陷入两难境地：它是否应该忽略`Bar`并最终进入
```cpp
 }
  catch (Foo e)
 {
```
，这个最初的目的地？还是应该忽略`Foo`而寻找一个
```cpp
  }
  catch (Bar e)
  {
```
处理？者没有好的答案，任何选择都会丢失信息。(如果都找呢？试想有更多 destructor 抛出更多异常，运行时系统究竟要花多少开销记录它们？too expensive)

因此，c++语言保证它遭遇这种情况时将调用 `terminate()`，而`terminate()`将终止该进程。砰，你死了。

为了防止这种情况，简单方法就是永远不在 `destructor` 中抛出异常。但是，如果您真的想变得聪明，可以说在处理一个异常时，永远不要从 `destructor` 中抛出另一个异常。但是在第二种情况很困难：`destructor` 本身需要代码来处理抛出异常和做“其他事情”(比如归还资源)，而调用者无法保证当 `destuructor` 检测到错误时会发生什么(到底抛出一个异常，还是因为已经 `stack unwinding` 所以不抛出，还有“其他事情”是否完成)。因此，整个解决方案很难编写。因此，简单方案是总要完成“其他事情”。也就是说，永远不在 `destructor` 中抛出异常(**never throw an exception from a destructor**)

当然，“never”这个词应该加引号，因为在某些情况下，这个规则不成立。但至少99%的情况下，这是一个很好的经验法则。

## How should I handle resources if my constructors may throw exceptions?

对象中的每个数据成员都应该清理自己的资源。

**如果构造函数抛出异常，则不会运行对象的析构函数**。如果您的对象已经做了一些需要撤消的事情(例如分配内存、打开文件或锁定信号量这些必须归还资源)，这些需要撤消的事情必须由对象内部的数据成员记住。

例如，与其将内存分配到 raw 指针数据成员中，不如将分配内存到智能指针成员对象中，当智能指针失效时，该智能指针的析构函数将 `delete` 已分配的对象。模板 `std::unique_ptr`就是一个智能指针的例子。您也可以编写自己的引用计数智能指针，见[write your own reference counting smart pointer](https://isocpp.org/wiki/faq/freestore-mgmt#ref-count-simple)。还可以使用智能指针指向其他机器上的磁盘记录或对象，见[use smart pointers to “point” to disk records or objects on other machines](https://isocpp.org/wiki/faq/operator-overloading#op-ov-examples)。

顺便说一下，如果你的Fred类会被分配到一个智能指针中，对你的用户好一点，在你的Fred类中创建一个类型定义，
```cpp
#include <memory>
class Fred {
public:
  typedef std::unique_ptr<Fred> Ptr;
  // ...
};
```

这个类型定义简化了所有使用该对象的代码语法：你的用户可以用`Fred::Ptr`而不是`std::unique_ptr<Fred>`：
```cpp
#include "Fred.h"
void f(std::unique_ptr<Fred> p);  // explicit but verbose
void f(Fred::Ptr             p);  // simpler
void g()
{
  std::unique_ptr<Fred> p1( new Fred() );  // explicit but verbose
  Fred::Ptr             p2( new Fred() );  // simpler
  // ...
}
```

(以下为我对这条 FAQ 的总结)

其实这一节的关键是，让涉及资源的数据成员类型为可以在 `destructor` 中回收资源的类。这种类可以是智能指针，也可以是自己实现的 handler 类。因为构造函数抛出异常，对应的析构并不会被调用，所以我们只能依赖数据成员的RAII。这里需要注意，[C++ Lifetime](https://en.cppreference.com/w/cpp/language/lifetime)提到，

**Lifetimes of non-static data members and base subobjects begin and end following class initialization order.**

而根据[C++ Constructor initialization order](https://en.cppreference.com/w/cpp/language/constructor#Initialization_order)所示，数据成员的 initialization order 是

1. If the constructor is for the most-derived class, virtual bases are initialized in the order in which they appear in depth-first left-to-right traversal of the base class declarations (left-to-right refers to the appearance in base-specifier lists).
2. Then, direct bases are initialized in left-to-right order as they appear in this class's base-specifier list.
3. Then, non-static data member are initialized in order of declaration in the class definition.
4. Finally, the body of the constructor is executed.

ps. destructor 无论用户定义还是隐式定义，过程和上面刚好相反。

所以异常发生在初始化基类/数据成员，还是 body of the constructor，有时是值得区分的。但无论发生在哪，利用数据成员的RAII是一个保证不资源泄漏的好方案。

如果需要捕获成员初始化列表中的异常，可以用 [Function-try-block](https://en.cppreference.com/w/cpp/language/function-try-block)，具体可看标准，大致用法如下，

```cpp
#include <iostream>
#include <string>
 
struct S
{
    std::string m;
 
    S(const std::string& str, int idx)
    try : m(str, idx)
    {
        std::cout << "S(" << str << ", " << idx << ") constructed, m = " << m << '\n';
    }
    catch(const std::exception& e)
    {
        std::cout << "S(" << str << ", " << idx << ") failed: " << e.what() << '\n';
    } // implicit "throw;" here for constructor
};
```

## How do I change the string-length of an array of char to prevent memory leaks even if/when someone throws an exception?

如果您真正想做的是使用字符串，那么首先就不要使用char数组，因为 [arrays are evil](https://isocpp.org/wiki/faq/containers#arrays-are-evil)。相反，应该使用一些类似字符串类的对象。

例如，假设您想获得一个字符串的副本，修改副本，然后将另一个字符串附加到修改后的副本的末尾。使用 `array-of-char` 的方案看上去如此，
```cpp
void userCode(const char* s1, const char* s2)
{
  char* copy = new char[strlen(s1) + 1];    // make a copy
  strcpy(copy, s1);                         //   of s1...
  // use a try block to prevent memory leaks if we get an exception
  // note: we need the try block because we used a "dumb" char* above
  try {
    // ...code that fiddles with copy...
    char* copy2 = new char[strlen(copy) + strlen(s2) + 1];  // append s2
    strcpy(copy2, copy);                                    //   onto the
    strcpy(copy2 + strlen(copy), s2);                       //     end of
    delete[] copy;                                          //       copy...
    copy = copy2;
    // ...code that fiddles with copy again...
  }
  catch (...) {
    delete[] copy;   // we got an exception; prevent a memory leak
    throw;           // re-throw the current exception
  }
  delete[] copy;     // we did not get an exception; prevent a memory leak
}
```

像这样使用`char*`是乏味且容易出错的。为什么不直接使用`string`的对象呢？你的编译器可能会提供一个 `string-like class`，它可能和你自己编写的`char*`代码一样快，当然也更简单、更安全。例如，如果您正在使用来自标准化委员会的`std::string`，您的代码可能看起来像这样，

```cpp
#include <string>           // Let the compiler see std::string
void userCode(const std::string& s1, const std::string& s2)
{
  std::string copy = s1;    // make a copy of s1
  // ...code that fiddles with copy...
  copy += s2;               // append s2 onto the end of copy
  // ...code that fiddles with copy again...
}
```

`char*`版本需要编写的代码大约是`std::string`版本的三倍。大部分节省代码来自`std::string`的自动内存管理：在`std::string`版本中，我们不需要编写任何代码去：

- to reallocate memory when we grow the string. 
- to delete[] anything at the end of the function. 
- to catch and re-throw any exceptions.

## What should I throw?

c++不同于几乎所有其他支持异常的语言，当涉及到您可能抛出的事物时，它是非常包容的。事实上，你可以抛出任何你喜欢的东西。这就引出了一个问题，你应该抛出什么？

一般来说，最好抛出对象，而不是`built-ins`(指 int double 这类原生类型)。如果可能，应该抛出`std::exception`派生类的实例。通过让你自定义的异常类继承标准异常基类(直接或间接均可，最终继承了就可)，你可以让你的用户过得更轻松(因为用户可以选择通过`std::exception`捕获大多数异常)，而且你可能为他们提供了更多的信息(比如你的特定异常可能是`std::runtime_error`或其他的细化)。

最常见的做法是抛出一个临时对象，
```cpp
#include <stdexcept>
class MyException : public std::runtime_error {
public:
  MyException() : std::runtime_error("MyException") { }
};
void f()
{
   // ...
   throw MyException();
}
```

在这里，创建并抛出类型为`MyException`的临时对象。类`MyException`继承自类`std::runtime_error`，后者(最终)继承自类`std::exception`。

## What should I catch?

为了保持c++的传统，有不止一种方法来做到这一点(c++喜欢给程序员选择和权衡，这样他们就可以决定在具体情况下什么是最好的)，c++为捕获提供了各种各样的选择。

- You can catch by value.
- You can catch by reference.
- You can catch by pointer.

事实上，在声明函数形参时，您拥有全部的灵活性。而特定异常是否匹配(即是否会被捕获)特定catch子句的规则，与调用函数时的参数匹配规则几乎完全相同。

考虑到这些灵活性，你如何决定捕获什么呢？很简单：除非有很好的理由不这么做，否则就引用来捕获。避免按值捕获，因为这会导致生成副本，并且副本可能具有不同于抛出的行为，见[the copy can have different behavior](https://isocpp.org/wiki/faq/virtual-functions#virtual-ctors)(简单说就是 copy 可能多态)。只有在非常特殊的情况下，你才应该用指针来捕捉。(后续FAQ有使用指针与引用捕获的例子)

## But MFC seems to encourage the use of catch-by-pointer; should I do the same?

视情况而定。如果你正在使用MFC框架并捕获他们的一个异常，无论如何，按照他们的方式去做。任何框架都是如此：入乡随俗。不要试图强迫一个框架进入你的思维方式，即使你的思维方式更好。如果您决定使用一个框架，请接受它的思维方式，使用它的作者期望您使用的习惯用法。

但是，如果您要创建自己的框架和/或一个不直接依赖MFC框架的系统，那么请不要仅仅因为MFC这样做而捕获指针。当您不在罗马时，您不一定像罗马人一样。在这种情况下，您不应该。像MFC这样的库早于C++语言中异常处理的标准化，其中一些库使用向后兼容的异常处理形式，所以需要(或至少鼓励)您可以通过指针进行捕获。

通过指针捕获的问题是不清楚谁(如果有人)负责删除指向的对象。例如，考虑以下情况，
```cpp
MyException x;
void f()
{
  MyException y;
  try {
    switch ((rand() >> 8) % 3) {  // the ">> 8" (typically) improves the period of the lowest 2 bits
      case 0: throw new MyException;
      case 1: throw &x;
      case 2: throw &y;
    }
  }
  catch (MyException* p) {
    // should we delete p here or not???!?
  }
}
```

这里有三个基本问题：
1. 可能很难决定是否在catch子句 `delete p`。例如，如果对象x对于catch子句的作用域是不可访问的，比如当它隐藏在某个类的私有成员中或在其他编译单元中是静态的时(继承自 C 语言的语法，编译单元的 static 是编译单元私有的)，可能很难弄清楚该做什么。
2. 如果您通过在`throw`中始终使用`new`来解决第一个问题(因此在`catch`中需要始终使用`delete`)，那么异常总是使用堆，这会在抛出异常时导致问题，因为系统内存不足。
3. 如果在`throw`中始终不使用`new`(因此在`catch`中始终不使用`delete`)来解决第一个问题，那么您可能无法将异常对象分配为局部变量(如果是局部变量，它们可能会过早地被销毁)，在这种情况下，您将不得不担心线程安全、锁、信号量等问题(因为无法用局部变量了，只能用全局对象本质上不是线程安全的)。

这并不是说不可能解决这些问题。重点很简单：如果通过引用而不是指针进行捕获，那么生活就会更容易。没有必要让生活变得艰难。

总之，避免抛出指针表达式，并避免通过指针进行捕获，除非您使用的库希望您这样做。

## What does throw; (without an exception object after the throw keyword) mean? Where would I use it?

您可能会看到类似这样的代码：
```cpp
class MyException {
public:
  // ...
  void addInfo(const std::string& info);
  // ...
};
void f()
{
  try {
    // ...
  }
  catch (MyException& e) {
    e.addInfo("f() failed");
    throw;
  }
}
```

在这个例子中，语句`throw;`意味着 `re-throw` 当前异常。在这里，函数捕获异常(通过非const引用)，修改异常(通过向其添加信息)，然后重新抛出异常。通过在程序的重要函数中添加适当的catch子句，可以实现简单形式的栈跟踪(这算是一个 idiom 用法)。

另一个 `re-throwing idiom` 是 `exception dispatcher`：
```cpp
void handleException()
{
  try {
    throw;
  }
  catch (MyException& e) {
    // ...code to handle MyException...
  }
  catch (YourException& e) {
    // ...code to handle YourException...
  }
}
void f()
{
  try {
    // ...something that might throw...
  }
  catch (...) {
    handleException();
  }
}
```

这种 `idiom` 允许重用单个函数`handleException()`来处理很多其他函数中的异常。( [C++ try-block](https://en.cppreference.com/w/cpp/language/try_catch)有提到这个通配的捕获。)

## How do I throw polymorphically?

有时候人们如此写代码：
```cpp
class MyExceptionBase { };
class MyExceptionDerived : public MyExceptionBase { };
void f(MyExceptionBase& e)
{
  // ...
  throw e;
}
void g()
{
  MyExceptionDerived e;
  try {
    f(e);
  }
  catch (MyExceptionDerived& e) {
    // ...code to handle MyExceptionDerived...
  }
  catch (...) {
    // ...code to handle other exceptions...
  }
}
```

如果您尝试这样做，在运行时代码会进入`catch(...)`子句而不是`catch (MyExceptionDerived&)`子句。您可能会感到惊讶。这是因为您没有多态地抛出。在函数`f()`中，语句`throw e;`抛出与表达式e的**静态类型**相同类型的对象。换句话说，它抛出`MyExceptionBase`的一个实例(而不是`MyExceptionDerived`的实例，会发生 slicing 问题)。throw语句的行为就像抛出对象被复制一样，而不是创建虚拟副本(关于 [`making a “virtual copy”`](https://isocpp.org/wiki/faq/virtual-functions#virtual-ctors)，其本质感觉就是一个继承带来的`Covariant Return Type` 与 `object slicing` 问题，下面这个代码例子来自 [C++ throw notes](https://en.cppreference.com/w/cpp/language/throw#Notes)。标准提到，When rethrowing exceptions, the second form must be used to avoid object slicing in the (typical) case where exception objects use inheritance)。

```cpp
try
{
    std::string("abc").substr(10); // throws std::length_error
}
catch (const std::exception& e)
{
    std::cout << e.what() << '\n';
//  throw e; // copy-initializes a new exception object of type std::exception
    throw;   // rethrows the exception object of type std::length_error
}
```

幸运的是，纠正起来相对容易(这个方案是万一不知道 `throw` 的具体派生类时，用一个虚函数实现多态)，

```cpp
class MyExceptionBase {
public:
  virtual void raise();
};
void MyExceptionBase::raise()
{ throw *this; }
class MyExceptionDerived : public MyExceptionBase {
public:
  virtual void raise();
};
void MyExceptionDerived::raise()
{ throw *this; }
void f(MyExceptionBase& e)
{
  // ...
  e.raise();
}
void g()
{
  MyExceptionDerived e;
  try {
    f(e);
  }
  catch (MyExceptionDerived& e) {
    // ...code to handle MyExceptionDerived...
  }
  catch (...) {
    // ...code to handle other exceptions...
  }
}
```

注意，`throw`语句已被移动到一个虚函数中。`e.raise()`语句将表现出多态行为，因为`raise()`被声明为虚函数，而`e`是通过引用传递的。与前面一样，抛出的对象将是抛出实参的静态类型，但在`MyExceptionDerived::raise()`中，该静态类型是`MyExceptionDerived`，而不是`MyExceptionBase`(如此规避了 `object slicing`)。

## When I throw this object, how many times will it be copied?

视情况而定。可能是零。

抛出的对象必须具有公共可访问的复制构造函数。编译器可以生成代码，复制抛出对象任意次数，包括零次。然而，即使编译器从未实际地复制抛出的对象(比如通过捕获引用/指针)，它也必须确保这个异常类的复制构造函数存在并可访问。

(其实根据标准[C++ throw the exception object](https://en.cppreference.com/w/cpp/language/throw#The_exception_object)，除了 copy/move constructors，连 destructor 也要是 accessible 的。另外这里也会存在非强制的 copy/move elsion，不一定是捕获引用/指针的原因)

## Why doesn’t C++ provide a “finally” construct?

因为c++支持一种几乎总是更好的替代方法，RAII。基本思想是用局部对象表示资源，这样局部对象的析构函数就会释放资源。这样，程序员就不会忘记释放资源。例如，

```cpp
    // wrap a raw C file handle and put the resource acquisition and release
    // in the C++ type's constructor and destructor, respectively
    class File_handle {
        FILE* p;
    public:
        File_handle(const char* n, const char* a)
            { p = fopen(n,a); if (p==0) throw Open_error(errno); }
        File_handle(FILE* pp)
            { p = pp; if (p==0) throw Open_error(errno); }
        ~File_handle() { fclose(p); }
        operator FILE*() { return p; }   // if desired
        // ...
    };
    // use File_handle: uses vastly outnumber the above code
    void f(const char* fn)
    {
        File_handle f(fn,"rw"); // open fn for reading and writing
        // use file through f
    } // automatically destroy f here, calls fclose automatically with no extra effort
      // (even if there's an exception, so this is exception-safe by construction)
```

在系统中，在最坏的情况下，每个资源都需要一个资源句柄类(就是 resource handler)。但是，我们没有必要在每次获取资源时都使用`finally`子句。在现实的系统中，资源获取的数量远远大于资源的种类，因此RAII的代码比使用`finally`的代码要少(有一种 code reuse 的思想)。

另外，可以参考[Appendix E of TC++PL3e](http://stroustrup.com/3rd_safe0.html)中的资源管理示例。

## Why can’t I resume after catching an exception?

换句话说，为什么c++不提供返回到异常抛出点并从那里继续执行的原语呢？

基本上，从异常处理程序恢复的人永远不能确定，抛出点之后的代码是不是用来处理，使得似乎什么都没有发生，然后继续执行。而异常处理程序不知道在恢复之前要正确处理多少的上下文(想想写操作系统内核，处理异常陷入内核后，需要很多上下文判断，这是一个何种的系统调用/中断/fault/abort)。为了正确编写这样的代码，抛出异常代码的程序员和捕获异常代码的程序员需要熟悉彼此的代码和上下文。这产生了一种复杂的相互依赖关系，无论在哪里被允许都将导致严重的维护问题。

Stroustrup在设计c++异常处理机制时认真考虑了允许恢复的可能性，并且在标准化过程中对这个问题进行了相当详细的讨论。参见[The Design and Evolution of C++](http://stroustrup.com/dne.html)异常处理章节。

如果希望在抛出异常之前检查是否可以修复这个问题，请调用一个函数进行检查，然后仅当问题无法在本地处理时才抛出。`new_handler` 就是一个例子。(这里提到的`new_handler`可以通过[C++ memory new set_new_handler](https://en.cppreference.com/w/cpp/memory/new/set_new_handler)设置，本质上是一个自定义的异常处理函数指针，只不过专门针对 `new` 而已)

## Conclusion

这篇笔记与 [ISOCPP FAQ: Exceptions and Error Handling (First Part)](https://qifanwang.github.io/isocpp/faq/2023/03/27/isocpp-faq-exceptions-1st/) 一同完成了Exception主题 FAQ 的翻译与记录。本篇的总计如下。

异常是有代价的，不要简单地将异常机制看作
- 免费午餐(有使用上的开销与编码时心智负担)
- 万能解决方案(需要有好习惯的团队)
- 一刀切的使用(有时还是得用返回码)
- 替罪羊(团队技术氛围不行往往会指责异常机制)

使用异常，有些思维定式要修改(不绝对，只是 guidelines)
- 不要有返回码思维(不要将 `throw` 当作美化的返回码，不要对每个可能抛出异常的函数调用都`try-catch`)
- 不要有Java思维(C++没有`finally`，用RAII回收资源)
- 不要以异常抛出者分类组织异常，应该以异常的逻辑原因分类组织异常
- 不要用异常类的 bit/data 字段区分不同的异常，应该用异常的类型(否则又是`if`判断，陷入和返回码一样的窘境)
- 不要以局部系统来设计自己的异常类型，应该以系统整体的角度来设计(否则，子系统之间又有各种异常类型的转换，冗余且拉低性能)
- 不要用 raw 指针，而是用智能指针(这只是RAII的特例，本质就是更好地回收资源)
- 不要将编码的逻辑错误与运行时错误混淆(异常揭示运行时问题，但fix bug还是要靠人去排查真实的逻辑错误位置)

为避免太多 try blocks，在使用前思考为什么这里要用 try block，一些常见情形是，
- catch 处理错误并继续执行，不抛出异常，使得调用者完全不知道有发生了异常。这是可取的。
- catch 需要完成一些工作并继续抛出异常，比如关闭文件。这时需要仔细考虑“工作”是否可以通过 RAII 完成，从而去掉当前这个 try block。
- catch 可能需要重新包装异常，比如将 `Xyzexception` 包装成 `Pqrexception`。这时需要考虑你的异常对象层次结构，是否可以有更好的设计。

在OOP对于特殊的成员函数是否可以抛出异常，
- 构造函数可以抛异常(这是异常设计的主要目的)
- 构造函数抛异常后不会运行析构函数。如果有字段涉及资源，请保证它们被智能指针或 resource handler 类管理，以便正确析构并归还资源(如果要捕获成员初始化列表的异常，可以用 `function-try-block`)
- 析构函数不可抛异常(绝大部分情形是的，因为同时两个异常会打破标准库和语言规则，语言保证调用 `terminate`结束)
- p.s. 移动构造函数不抛异常，可以用在容器扩容(比如 vector)时，保证强类型安全，具体可看 `std::move_if_noexcept` 介绍。

关于具体使用 throw 与 catch 的一些注意：
- 对于 `throw` 最好抛出一个继承自 `std::exception` 的异常类的临时对象。
- 对于 `catch` 最好按引用捕获(避免按值捕获的复制与可能的`object slicing`，避免指针捕获以致不知道指针索引对象的所有者，由谁负责`delete`)
- 如果代码直接使用的框架按某种方式捕获异常，比如MFC按指针捕获，入乡随俗。因为它们可能要向后兼容。
- 如果使用后面无 exception 对象的 `throw`，这代表 `re-throw` 当前异常，可以与`catch`引用配合实现`捕获/修改/重抛`，也可以与`catch(..) + exception dispatcher` 配合实现异常分配。
- 如果要实现多态的 `throw` 需要小心，因为 `throw e;` 抛出的异常对象类型是表达式 `e` 的**静态类型**的对象。所以很容易发生 `object slicing`现象。解决方案是使用 `re-throw` 对象引用，或用虚函数实现 `throw` 派生类对象。具体可看 `FAQ How do I throw polymorphically?` 的代码示例。

关于语言标准的一些注意：
- 异常对象可能因为按引用捕获与 `copy/move elision` 在抛出到捕获过程中没有被复制过。但是标准规定，其 copy/move constructors 与 destructor 仍是要可访问的。
- C++不支持 `finally` 但通过 RAII 来管理资源。这样通过资源的种类小于资源的数量，可以很好复用代码。
- C++不支持捕获异常后返回到异常点继续执行，因为这产生从抛出处到捕获处的相互依赖，带来维护问题。可以调用一个函数检查问题是否可以在本地处理，无法处理才抛出。例如 `new_handler`。

## Reference
1. [C++ 异常与错误处理的 FAQ](https://isocpp.org/wiki/faq/exceptions)
2. [Appendix E of TC++PL3e](http://stroustrup.com/3rd_safe0.html)
3. [air vehicle C++ coding standards](http://stroustrup.com/JSF-AV-rules.pdf)
4. [Will I sometimes use any so-called “evil” constructs?](https://isocpp.org/wiki/faq/big-picture#use-evil-things-sometimes)
5. [“placement” new](https://isocpp.org/wiki/faq/dtors#placement-new)
6. [write your own reference counting smart pointer](https://isocpp.org/wiki/faq/freestore-mgmt#ref-count-simple)
7. [use smart pointers to “point” to disk records or objects on other machines](https://isocpp.org/wiki/faq/operator-overloading#op-ov-examples)
8. [C++ Lifetime](https://en.cppreference.com/w/cpp/language/lifetime)
9. [C++ Constructor initialization order](https://en.cppreference.com/w/cpp/language/constructor#Initialization_order)
10. [Function-try-block](https://en.cppreference.com/w/cpp/language/function-try-block)
11. [arrays are evil](https://isocpp.org/wiki/faq/containers#arrays-are-evil)
12. [the copy can have different behavior](https://isocpp.org/wiki/faq/virtual-functions#virtual-ctors)
13. [C++ try-block](https://en.cppreference.com/w/cpp/language/try_catch)
14. [C++ throw notes](https://en.cppreference.com/w/cpp/language/throw#Notes)
15. [C++ throw the exception object](https://en.cppreference.com/w/cpp/language/throw#The_exception_object)
16. [The Design and Evolution of C++](http://stroustrup.com/dne.html)
17. [C++ memory new set_new_handler](https://en.cppreference.com/w/cpp/memory/new/set_new_handler)
18. [ISOCPP FAQ: Exceptions and Error Handling (First Part)](https://qifanwang.github.io/isocpp/faq/2023/03/27/isocpp-faq-exceptions-1st/)