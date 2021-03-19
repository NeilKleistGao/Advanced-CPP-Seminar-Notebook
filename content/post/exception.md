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