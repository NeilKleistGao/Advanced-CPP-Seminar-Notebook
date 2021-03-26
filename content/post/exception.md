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