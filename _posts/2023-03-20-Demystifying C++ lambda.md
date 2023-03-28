---
layout: post
title: Demystifying C++ lambda
categories: [cpp]
description: 
keywords: cpp
---


> *lambda*表达式也叫做匿名表达式，是纯右值表达式，它的类型是独有的无名非联合非聚合类类型，被称为*闭包类型*（*closure type*），因此*lambda*表达式的类型并不是`function`，常用的格式：**[ captures ] ( params ) specs requires ﻿(optional) { body }**，更多用法参考 [cppreference.com](https://en.cppreference.com/w/cpp/language/lambda)，

# how it works

#### **一个简单的例子，利用lamda表达式遍历数组**

```c++
#include <stdio.h>

template<int N, typename Func>
void print(int (&nums)[N], Func&& f) {
  for (int i = 0; i < N; ++i) {
    f(nums[i]);
  }
}

int main() {
  int arr[] = {1, 2, 3, 4, 5};
  auto visited = [](int x) { printf("%d ", x); };
  print(arr, visited);
}
```

通过[C++ Insights (cppinsights.io)](https://cppinsights.io/)可以得到展开结果。显而易见

+ *lambda*函数被定义成了一个类，通过重载`operator()`实现函数功能，

+ 闭包类型为 `__lambda_12_18`，并不是我们认为的 `std::function`，只有编译期知道这个类型，只能通过`auto`或者模板参数推导出来。

+ `using retType_12_18 = void (*)(int);`让该类型可以传递给 `void (*)(int)`指针

  > 无捕获lambda可以转换为函数指针（尽管它是否适合C调用约定取决于实现），但这并不意味着它是一个函数指针。

```C++
#include <stdio.h>

template<>
void print<5, __lambda_12_18 &>(int (&nums)[5], __lambda_12_18 & f)
{
  for(int i = 0; i < 5; ++i) {
    f.operator()(nums[i]);
  }
}

int main()
{
  int arr[5] = {1, 2, 3, 4, 5};
    
  class __lambda_12_18
  {
    public: 
    inline /*constexpr */ void operator()(int x) const
    {
      printf("%d ", x);
    }
    
    using retType_12_18 = void (*)(int);
    inline constexpr operator retType_12_18 () const noexcept
    {
      return __invoke;
    };
    
    private: 
    static inline /*constexpr */ void __invoke(int x)
    {
      __lambda_12_18{}.operator()(x);
    }
    public:
    // /*constexpr */ __lambda_12_18() = default;
  };
  
  __lambda_12_18 visited = __lambda_12_18{};
  print(arr, visited);
  return 0;
}
```



#### **一个可以捕获值的例子，捕获了值a和b的引用。**

```c++
int main()
{
  int a = 10;
  int b = 20;
  auto f1 = [a, &b](int x) { 
  	return a + b + x;
  };
  f1(7);
  return 0;
}
```

展开结果如下，我们可以知道捕获的值成为了这个类的私有成员变量，并且在包含了闭包之后，该类型就不能自动转换成函数指针的类型了。

```c++
int main()
{
  int a = 10;
  int b = 20;
    
  class __lambda_5_13  // 只有编译期可以推导出来
  {
    public: 
    inline /*constexpr */ int operator()(int x) const
    {
      return (a + b) + x;
    }
    private: 
    int a;
    int & b;
    
    public:
    __lambda_5_13(int & _a, int & _b)
    : a{_a}
    , b{_b}
    {}
  };
  
  __lambda_5_13 f1 = __lambda_5_13{a, b};
  return 0;
}
```

#### Attention

当捕获指针的时候，需要小心，比如当我们捕获this指针的时候，我们按值捕获或者按引用捕获的结果都是一样的糟糕，即生命周期并不会被延长，因为这是一个指针。需要捕获`*this`，或者将该类继承自`enable_shared_from_this`，使用 `shared_from_this`就可以增加引用计数，延长生命周期。

还有一点，我们按值捕获的时候，是不能更改里面的值的，比如下面这个，如果一定需要更改，则在参数列表后面加上*mutable*

```
int main()
{
  int a = 10;
  int b = 20;
  auto f1 = [ a, &b](int x) {  // error cannot assign to a variable captured by copy in a non-mutable lambda
  	a = b;
  };
  auto f1 = [ a, &b](int x) mutable {  // ok
  	a = b;
  };  
  f1(7);
  return 0;
}
```

# 总结

+ *lambda*是一个栈对象。它有一个构造函数和析构函数。它将遵循所有的C++规则。lambda的类型将包含捕获的值/引用；它们将是该对象的成员，就像任何其他类型的任何其他对象成员一样。
+ *lambda*函数被定义成了一个类，通过重载`operator()`实现函数功能，
+ 无捕获*lambda*可以转换为函数指针（尽管它是否适合C调用约定取决于实现），但这并不意味着它是一个函数指针。


# Reference

[c++ - C++11 lambda implementation and memory model - Stack Overflow](https://stackoverflow.com/questions/12202656/c11-lambda-implementation-and-memory-model)

[C++ Lambda Under the Hood](https://medium.com/software-design/c-lambda-under-the-hood-9b5cd06e550a)

