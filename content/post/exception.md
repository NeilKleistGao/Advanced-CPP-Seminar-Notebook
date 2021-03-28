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
## 内存管理
作者：[buggyminer](https://github.com/buggyminer)

---

异常的存在破坏了函数的封闭性。在有关内存操作的函数中产生的异常很容易产生内存问题。
比如，构造函数执行不完全时产生了异常，会导致对象只构造了一部分，且这部分占用的内存不会被析构函数释放。当然，在构造函数中抛出异常是推荐的做法，由异常信息代替了错误码——因为构造函数不能有返回值。否则，就不得不为所有可能构造失败的对象创建一个zombie状态，并编写一个特定的检活成员函数，还需要在所有可能的地方加入额外的判断条件——对象是否为zombie。
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

5、使用两段构造函数

破除了构造函数没有返回值的窘境，在语义上将构造拆分成了更小的单元，是完全可行的。虽然有些人认为没有必要或不够优雅，我可不觉得。

而对于析构函数，问题就简单的多：

> You can throw an exception in a destructor, but that exception must not leave the destructor; if a destructor exits by emitting an exception, all kinds of bad things are likely to happen because the basic rules of the standard library and the language itself will be violated. Don’t do it. *isocpp*

> Item 11: Prevent exceptions from leaving destructors *more effective C++*

析构函数的异常必须在析构函数中捕捉。
如果抛出异常，就说明内存泄漏已经发生，析构函数未能完成它的工作。
此外，如果析构函数是在处理异常的代码中被调用，那么析构函数产生的异常会直接触发terminate。而我们知道，析构函数作为资源释放的函数经常用于处理异常，因此它自身必须是严格异常安全的。为了提高性能，可以加上nonexcept；这样还有一个好处，就是能够尽早地让有问题的析构函数崩溃，从而暴露问题。


src：

[C++ Exception Optimizations. An experiment.](https://isocpp.org/files/papers/P1676R0.pdf)

[Exceptions and Error Handling](https://isocpp.org/wiki/faq/exceptions)

[C++17 standard](https://www.iso.org/standard/68564.html)

---

## catch by value & determinism



作者：[buggyminer](https://github.com/buggyminer)

---

主要来自Herb Sutter的两篇论文。他主张使用静态的异常类型检查以及按值捕获，希望通过这一方式在保证C++语言的统一性的同时消弭两种错误处理方式的分歧——异常和错误码。

> This paper aims to extend C++’s exception model to let functions declare that they throw a statically specified  type by value. This lets the exception handling implementation be exactly as efficient and deterministic as a local return by value, with zero dynamic or non-local overheads.

> Importantly, “zero overhead” is not claiming zero cost — of course using something always incurs  some cost. Rather, C++’s zero-overhead principle has always meant that (a) “you don’t pay for what  you don’t use” and (b) “when you do use it you can’t [reasonably] write it more efficiently by hand.”

exception的使用会对C++程序产生诸多性能上的影响，如二进制图像膨胀、运行时开销等问题，还会使得程序失去其时间和空间开销的确定性——后者对实时性高的系统而言是不能接收的。这些现象的根本原因是异常的存在违背了C++的两个基本原则，即零开销原则和确定性原则。

> The **zero-overhead principle** is a C++ design principle that states:
>
> 1. You don't pay for what you don't use.
> 2. What you do use is just as efficient as what you could reasonably write by hand.

> **Determinism**, in philosophy, theory that all events, including moral choices, are completely determined by previously existing causes. Determinism is usually understood to preclude free will because it entails that humans cannot act otherwise than they do. 

异常需要根据其动态类型进行处理，这就引出了一个经典的问题：dynamic_cast的复杂度是多少？

总之，exception在C++中的实现导致了其运行期的不确定性，并在一些情况下大幅提高了程序的开销，并需要RTTI的配合。具体解释请阅读原论文。

程序员喜欢折中。以下是作者给出的折中解决方案。

![image-20210328155735397](https://i.loli.net/2021/03/28/5KC6dkSyV3JBMO7.png)

![image-20210328161655752](https://i.loli.net/2021/03/28/3TWglRbFHEL2rOa.png)

异常是动态的。但是我们可以使用静态的异常，从而在编译期就得以确认程序的开销和性能。为了节省空间和保证一致性，往往通过引用捕捉异常——需要不断地在栈中获取异常的地址。但是在使用静态异常的前提下，我们就可以按值捕捉异常，避免了寻址的开销并保证了其唯一性。

即，使用异常的处理方式来传递错误码。

此外，他在新的论文中提出了一种完全消除全新的层次异常处理方式，可以完全取出任何按引用捕获异常的必要性。当然，现在还不能用。

```c++
try
{
	f(); // Calls open_file internally
}
catch( file_error, open_error, file_name fn )
{
	// Handle a "file error" that is also an "open error" and has an associated file name.
	std::cerr << "Failed to open file " << fn.value << std::endl;
}
catch( file_error, file_name fn )
{
	// Handle any "file error" that has an associated file name.
	std::cerr << "Failed to to access file " << fn.value << std::endl;
}
catch( file_error )
{
	// Handle any "file error".
	std::cerr << "Failed to access file" << std::endl;
}
```

src：

[Zero-Overhead Deterministic Exceptions: Catching Values]([Zero-Overhead Deterministic Exceptions: Catching Values (open-std.org)](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2021/p2232r0.html))

[Zero-overhead deterministic exceptions: Throwing values]([p0709r0.pdf (open-std.org)](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2018/p0709r0.pdf))

---

