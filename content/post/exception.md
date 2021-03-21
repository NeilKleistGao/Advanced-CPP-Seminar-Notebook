---
title: "专题1：Exception"
date: 2021-03-13T11:56:19+08:00
draft: true
---

## 前言
---

## NeilKleistGao的部分（自行修改标题）
作者：[NeilKleistGao](https://github.com/NeilKleistGao)

---

请在此处编写。

---
## gerayking的部分（自行修改标题）
作者：[gerayking](https://github.com/gerayking)

---

请在此处编写。

---
## lannester666的部分（自行修改标题）
作者：[lannester666](https://github.com/lannester666)

---

请在此处编写。

---
## morningstarwang的部分（自行修改标题）
作者：[morningstarwang](https://github.com/morningstarwang)

---

请在此处编写。

---
## PtCu的部分（自行修改标题）
作者：[PtCu](https://github.com/PtCu)

---

请在此处编写。

---
## 使用规范
作者：[buggyminer](https://github.com/buggyminer)

---

异常的存在破坏了函数的封闭性。在有关内存操作的函数中产生的异常很容易产生内存问题。
比如，构造函数执行不完全时产生了异常，会导致对象只构造了一部分，且这部分占用的内存不会被析构函数释放。当然，在构造函数中抛出异常是推荐的做法，由异常信息代替了错误码——因为构造函数不能有返回值。否则，就不得不为所有可能构造失败的对象创建一个zombie状态，并编写一个特定的检活成员函数，并在所有可能的地方加入额外的判断条件——对象是否构造成功。
同样地，析构函数执行时产生了异常，可能导致内存泄漏问题。
对于构造函数，大致有以下解决思路：

1、使用智能指针

这是推荐的做法。
将需要动态申请的资源分配到智能指针上，那么可以保证当程序退出构造函数时，智能指针被销毁，其指向的内存得到释放。

2、用try-catch块单独包裹构造函数

不推荐。除非该对象是一个单例类，否则都会违背DRY原则。

3、在构造函数内部捕获异常并再次抛出

在内部的异常处理中，仅仅进行资源的释放。如果有必要，可以附加更多的异常信息。

4、在构造函数内部捕获异常并不抛出

对于有些类，即使创建失败，对象并没有占有任何资源时，其仍然可以视为一个自洽的系统。比如stack，我们需要不断地判断其是否为空。面对这种情况，在构造函数内部处理异常并使对象迁移到一个合法的“空”状态是完全可行的。

而对于析构函数，问题就简单的多：

> You can throw an exception in a destructor, but that exception must not leave the destructor; if a destructor exits by emitting an exception, all kinds of bad things are likely to happen because the basic rules of the standard library and the language itself will be violated. Don’t do it. *isocpp*

> Item 11: Prevent exceptions from leaving destructors *more effective C++*

析构函数的异常必须在析构函数中捕捉。
如果抛出异常，就说明内存泄漏已经发生，析构函数未能完成它的工作。
此外，如果析构函数是在处理异常的代码中被调用，那么析构函数产生的异常会直接触发terminate。而我们知道，析构函数作为资源释放的函数经常用于处理异常，因此它自身必须是严格异常安全的。为了提高性能，可以加上nonexcept；这样还有一个好处，就是能够尽早地让有问题的析构函数崩溃。


src：

[C++ Exception Optimizations. An experiment.](https://isocpp.org/files/papers/P1676R0.pdf)

[Exceptions and Error Handling](https://isocpp.org/wiki/faq/exceptions)

[C++17 standard](https://www.iso.org/standard/68564.html)

---

