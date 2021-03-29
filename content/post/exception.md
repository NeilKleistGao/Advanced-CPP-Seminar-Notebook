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

请在此处编写。

---