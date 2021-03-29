---
title: "专题1：Exception"
date: 2021-03-13T11:56:19+08:00
draft: true
---

## 前言
---

## C++ Exception实现机制
作者：[NeilKleistGao](https://github.com/NeilKleistGao)

---

### 环境申明
+ 操作系统: Windows10
+ IDE: Visual Studio 2019
+ 编译器：MSVC 19
+ C++标准：C++17
+ 优化选项：无优化

### std::exception实现
MSVC下`std::exception`的实现在头文件`vcruntime_exception.h`中：
```c++
class exception
{
public:
    exception() noexcept
        : _Data()
    {
    }

    explicit exception(char const* const _Message) noexcept
        : _Data()
    {
        __std_exception_data _InitData = { _Message, true };
        __std_exception_copy(&_InitData, &_Data);
    }

    exception(char const* const _Message, int) noexcept
        : _Data()
    {
        _Data._What = _Message;
    }

    exception(exception const& _Other) noexcept
        : _Data()
    {
        __std_exception_copy(&_Other._Data, &_Data);
    }

    exception& operator=(exception const& _Other) noexcept
    {
        if (this == &_Other)
        {
            return *this;
        }

        __std_exception_destroy(&_Data);
        __std_exception_copy(&_Other._Data, &_Data);
        return *this;
    }

    virtual ~exception() noexcept
    {
        __std_exception_destroy(&_Data);
    }

    _NODISCARD virtual char const* what() const
    {
        return _Data._What ? _Data._What : "Unknown exception";
    }

private:

    __std_exception_data _Data;
};
```

其中`__std_exception_data`为一个结构体，包含一个what字符串和一个清理标志。

Linux下的实现和该实现类似，但不包含有`char const* const message`参数的构造函数。

### __CxxThrowException
我们以一个简单的`try/catch`为例：
```c++
#include <iostream>
#include <exception>

int main() {
	try {
		throw std::exception("test");
	}
	catch (const std::exception& e) {
		std::cout << e.what() << std::endl;
	}

	return 0;
}
```

根据上一节的内容，我们在`std::exception`类内部找不到异常处理的内容：它只是一个存储异常信息数据的类，而不负责异常处理逻辑。

我们使用反汇编查看对应的汇编代码：
```assembly
mov         dword ptr [ebp-4],0
push        offset string "test"
lea         ecx,[ebp-0F8h]
call        std::exception::exception
push        offset __TI1?AVexception@std@@
lea         eax,[ebp-0F8h]
push        eax
call        __CxxThrowException@8
```

可以看到`try`部分内的逻辑，本质上是调用了一个`__CxxThrowException`函数来进行处理的，这个函数应该有两个参数（两次`push`）。

我们可以在微软官方的文档中找到这个函数的声明：
```c++
extern "C" void __stdcall _CxxThrowException(
   void* pExceptionObject,
   _ThrowInfo* pThrowInfo
);
```

其中第一个参数是产生异常的对象，第二个参数是处理异常所需要的信息。由于该函数使用`__srdcall`修饰，` __TI1?AVexception@std@@`对应`_ThrowInfo`类型指针。

**不要直接调用该函数，该函数应该交由编译器处理。**

这个函数真正意义上“抛出了一个异常”。该函数将会创建一个异常记录（exception record），并要求运行时环境处理该异常。如果程序顺利执行完毕没有触发异常，一个`jmp`指令将会被用于跳过`catch`部分。

由于`__CxxThrowException`无法查阅到其实现方式，只能看到反汇编的结果，难以阅读，我们尝试通过使用IDA来定位`catch`语句被引用的地方：

![](https://i.loli.net/2021/03/25/mFoSN5fU6g7Kiza.png)

我们可以看到编译器保存了函数的相关信息（FuncInfo）结构体：其中包含了两个回退（Unwind）结构体（下文中会再提到），以及一个HandlerType，该结构体中保存有catch对应的偏移地址以及`exception`的虚表指针（`stru_41C138`中，在`_ThrowInfo`也包含有指向该结构体的指针）。

FuncInfo的定义如下（已删除宏和注释，完整见`ehdata.h`）：
```c++
typedef const struct _s_FuncInfo
{
	unsigned int		magicNumber:29;		
	unsigned int		bbtFlags:3;			
	__ehstate_t			maxState;			
	UnwindMapEntry*		pUnwindMap; // 回退表
	unsigned int		nTryBlocks;
	TryBlockMapEntry*	pTryBlockMap; // try块表
	unsigned int		nIPMapEntries;
	void*				pIPtoStateMap;
	ESTypeList*			pESTypeList;
	int					EHFlags;
} FuncInfo;
```

我们跳过`TryBlockMapEntry`（同样在`ehdata.h`中），直接来看`HandlerType`：
```c++
typedef const struct _s_HandlerType {
	unsigned int	adjectives;
	TypeDescriptor*	pType; // 类型描述符，根据之前逆向的结果，其中包含有虚表指针
	ptrdiff_t		dispCatchObj;
	void *			addressOfHandler; // catch地址
} HandlerType;
```

`__CxxThrowException`首先会检索`FuncInfo`，找到对应的try entry，并根据try entry 结构后附带的catch entry逐一匹配，找到合适的处理代码。如果无法找到，则需要涉及到栈回退。

### Stack Unwind（栈回退）
如果`exception`无法被捕获，运行时环境需要跳转到更高一层的上下文（higher execution context）中寻找`catch`块；如果程序完全没有理会该`exception`，则程序将会调用`terminate`函数自动终止。

关键问题就在于：我们要如何完成“跳转”操作？考虑下面的这段代码：
```c++
#include <iostream>
#include <exception>

struct Foo
{
	~Foo() {
		std::cout << "rua" << std::endl;
	}
};

void foo() {
	Foo f;
	throw std::exception("test");
}

int main() {
	
	try {
		foo();
	}
	catch (std::exception e) {
		// ...
	}
	
	return 0;
}
```

我们在`foo`函数中抛出一个异常，我们该如何在`main`中将其捕获？

接着上一节的内容，我们来查看回退相关的结构体：
```c++
typedef const struct _s_UnwindMapEntry {
	__ehstate_t	toState;
	void		(__cdecl * action)(void);
} UnwindMapEntry;
```

该结构体中包含有一个`funclet`，当异常被抛出而找不到`catch`时，程序会自动调用该`funclet`，它将完成栈回退的计算，以及必要的析构操作等。如果`toState`值为`-1`，则代表当前函数处在最上层。

我们可以使用IDA查看`foo`函数的回退`funclet`：

![](https://i.loli.net/2021/03/25/krmiUqD4boXZjEK.png)

可以看到其的确调用了`Foo`结构体的析构函数。由于`foo`函数没有参数，所以不需要进行额外的调整。

---
## gerayking的部分（自行修改标题）
作者：[gerayking](https://github.com/gerayking)

---

### 	异常默认处理情况

当发生异常当未有异常捕捉进行处理时，在默认情况下，这将导致程序异常终止，这种情况成为**意外异常**，

这种情况下，程序将会调用terminate()，默认情况下，terminate()会调用abort()函数。

而在大多数情况下，项目如果因为出现异常而宕机是很危险的行为，所以可以直接修改terminate()函数进行意外异常的根本处理.

```
Function handling termination on exception
Calls the current terminate handler.

By default, the terminate handler calls abort. But this behavior can be redefined by calling set_terminate.

This function is automatically called when no catch handler can be found for a thrown exception, or for some other exceptional circumstance that makes impossible to continue the exception handling process.

This function is provided so that the terminate handler can be explicitly called by a program that needs to abnormally terminate, and works even if set_terminate has not been used to set a custom terminate handler (calling abort in this case).


```

 cite  [C++ reference](http://www.cplusplus.com/reference/exception/terminate/)



C++提供了`set_terminate`的方法交给程序员来自定义兜底的函数处理。

具体使用实例如下

```cpp
// terminate example
#include <iostream>       // std::cout, std::cerr
#include <exception>      // std::exception, std::terminate
using namespace std;
void MyQuit(){
    cout<<"Terminate due to uncaught exception !!!!\n";
    exit(5);
}
int main (void) {
    set_terminate(MyQuit);
    try {
        throw "as";
    }
    catch (exception &a) {
        std::cout<<"caught exception !!!\n";
    }
    return 0;
}
```



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
### 通过值抛出，通过引用捕获

—— Excerpt from 《 C++ Coding Standards: 101 Rules,  guidelines and best practices 》

**摘要**

​	学习正确捕获 (catch)：按值（而不是指针）抛出异常，并按引用（通常为const的引用）捕获异常，这是与异常语义配合最佳的组合。当抛出相同的异常时，应该优先使用throw;, 避免使用throw e; 

**讨论**

​	抛出异常时，要通过值抛出对象，避免抛出指针，因为如果抛出指针，则需要处理内存管理问题。不能抛出指向堆栈分配值的指针，因为在指针到达调用处之前栈还没有展开。虽然抛出指向动态分配内存的指针是可行的 [如果要报告的错误不是以 "out of memory" (内存不足）开始的], 但是这样就将释放内存的负担放在了捕获处。如果确实必须抛出一个指针，请考虑抛出诸如shared_ptr \<T> 之类的类似于值的智能指针，而不是普通的T *。
​	通过值抛出可以说是 “集人间宠爱于一身”，因为这时编译器本身将负责管理异常对象的内存这一负责过程。我们所需要操心的就是确保为异常类实现一个非抛出的副本构造函数。
​	除非要抛出的是智能指针（这已经增加了要保持多态的间接性），否则通过引用捕获异常就是唯一可行的好办法。按值捕获纯值会导致在捕获处引起切片问题，这会粗暴地去除通常至关重要的异常对象的多态性。通过引用捕获可以保留异常对象的多态性。
在重新抛出异常e时，应该只写成throw；而不是throw e;，因为第一种形式总是能够保持重新抛出对象的多态性。

```cpp
catch( MyException& e ) { //catch by reference to
    non-conste.AppendContext("Passed through here");   //modify
    throw;
    }
```



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

