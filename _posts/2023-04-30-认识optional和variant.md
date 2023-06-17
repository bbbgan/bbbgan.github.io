---
layout: post
title: 认识optional、variant以及tuple
categories: [C++]
description: 
keywords: 
---

### 写在前面

`variant`是*C++17*推出的**类型安全的联合体**，`std::variant`的实例在任意时刻保有一个可选类型的值，但是如果为void，是不允许的，这个情况下可用 `std::variant<std::monostate>`代替。`variant`其推出的目的就是为了取代C中的`union`！

optional是为了替代 `std::pair<T,bool>`

## why not union

使用`union`有一个问题，我们并不知道当前的存储类型是什么，并且在union进行析构或者切换的时候也不会去调用析构函数，这在针对平凡析构函数的类型的时候是们有问题的，但当对象不拥有平凡析构函数，比如`string`或者`vector`，会因为无法调用析构函数出现内存泄漏的情况。

```c++
union S
{
    std::string str;
    std::vector<int> vec;
    ~S() { } // what to delete here?
};
```

比如这样一个联合体，union不会去调用析构函数，本质上只是分配出一块内存，然后去操作这块内存，把这块内存解释成不同的类型。因此上面这种情况是错误的，在最新的gcc中也无法成功编译。因为union需要成员具有平凡析构函数。

## What is variant

在上面的情况下，variant应运而生，其实只需要知道variant是类型安全的联合体就行了！variant的简单使用直接参考[ cppreference.com](https://en.cppreference.com/w/cpp/utility/variant)，这里主要谈一些特性：

+ 如果不使用值初始化变量，那么该变量将使用第一种类型初始化。在这种情况下，第一个替代类型必须具有默认构造函数。如果没有类型是具有默认构造函数的，则可以用`std:：monostate`当成第一个参数
+ 由于需要管理类型安全，variant对象比union对象会大一点，但是不会发生堆的分配。
+ variant类调用非平凡类型的析构函数和构造函数，在切换到新的变体之前，更改当前存储的类型，则会调用底层类型的析构函数。
+ variant的赋值可以通过
  + the assignment operator
  + `emplace`
  + `get` and then assign a new value for the currently active type
  + a visitor
+ 取值可以通过，大部分时候variant都是搭配`std::visit`一起使用的
  + `get` and then assign a new value for the currently active type
  + a visitor

**如果需要处理的对象都具有平凡析构函数，那么使用union其实也没有不妥当。**



## std::optional

类模板 `std::optional` 管理一个*可选*的容纳值，既可以存在也可以不存在的值。相当于替代 `std::pair<T,bool>` ，任何一个 `optional<T>` 的实例在给定时间点要么*含值*，要么*不含值*。如果有值，可以使用operator*() 和 operator->() 运算符访问，但是这并不意味着是动态分配，事实上，optional的值存在对象空间。

当一个 `optional<T>` 对象被按语境转换成 bool 时，若对象*含值*则转换返回 true ，若对象*不含值*则返回 false 。当您使用std:：optional时，您将增加内存占用。至少需要一个额外的字节。下面是伪代码：

```c++
template <typename T>
class optional
{
  bool _initialized;
  std::aligned_storage_t<sizeof(T), alignof(T)> _storage;

public:
   // operations
};
```

虽然bool类型通常只占用一个字节，但可选类型需要遵守对齐规则，因为真正的类型要大一点，具体取决于实现和包含的类型。

## std::tuple
std::tuple是一种异质容器， 取值赋值喝variant相似，可以简单的看做一个扩展的pair类型。其析构函数是否是平凡的取决于容器中的类型是否都是平凡的析构函数。

## Reference

[Everything You Need to Know About std::variant from C++17 - C++ Stories (cppstories.com)](https://www.cppstories.com/2018/06/variant/)

[std::variant Doesn’t Let Me Sleep](https://pabloariasal.github.io/2018/06/26/std-variant/)



