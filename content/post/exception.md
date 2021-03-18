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

其中第一个参数是产生异常的对象，第二个参数是处理异常所需要的信息。**不要直接调用该函数，该函数应该交由编译器处理。**

这个函数真正意义上“抛出了一个异常”。如果该异常不被捕获，将会导致程序的崩溃；如果程序顺利执行完毕没有触发异常，一个`jmp`指令将会被用于跳过`catch`部分。

### Structured Exception Handling
如何在抛出异常以后正确捕获异常？Windows从很早以前的版本起就提供了结构化异常处理(Structured Exception Handling, SEH)来完成这项工作。


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