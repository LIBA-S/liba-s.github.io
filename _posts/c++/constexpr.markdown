---
layout: "post"
title: "constexpr"
date: "2021-05-26 14:15"
---


### constant expression
https://en.cppreference.com/w/cpp/language/constant_expression

常量表达式：可以在编译器求值。

使用场景：
1） non-type template arguments
2) array size
3) 任何其他需要constant expression 的地方



```C++
template<int N>
class fixed_size_list
{ /*...*/ };

fixed_size_list<X> mylist;  // X must be an integer constant expression

int numbers[X];  // X must be an integer constant expression
```

### const vs constexpr

1) 应用于对象objects
- const申明的为常量（constant），保证一旦初始化，其值不再改变，编译器可以根据其进行优化。
- constexpr是用来适配（constant expression）的。

2) 应用于函数functions
- const只能用于声明**非静态成员函数**，以保证其不修改任何非静态成员变量（mutable 成员除外）。
- constexpr可以用于声明**成员函数**和**普通函数**，同样，用于适配constant expression场景。
  - 函数必须是**非virtual**的，并且简单。通常**只允许一个返回值**。
  - c++14，使用更加宽松，允许**asm**，**goto**,标签语句**除了case和default**，**try block**
    **非字面量类型的变量定义**，**static 和 thread storage duation变量** 和 **未初始化的变量定义**。
  - **实参** 和 **返回值** 必须是 **literal type**。

**Literal types are the types of constexpr variables and they can be constructed, manipulated, and returned from constexpr functions.**

注意：即使对象未使用constexpr，也可以将其应用于constant expression的场景
```C++
  int main()
  {
    const int N = 3;
    int numbers[N] = {1, 2, 3};  // N is constant expression
  }
```

```C++
struct complex
{
    // constant-expression constructor
    constexpr complex(double r, double i) : re(r), im(i) { }  // OK: empty body
    // constant-expression functions
    constexpr double real() { return re; }
    constexpr double imag() { return im; }
private:
    double re;
    double im;
};
constexpr complex COMP(0.0, 1.0);         // creates a literal complex
double x = 1.0;
constexpr complex cx1(x, 0);              // error: x is not a constant expression
const complex cx2(x, 1);                  // OK: runtime initialization
constexpr double xx = COMP.real();        // OK: compile-time initialization
constexpr double imaglval = COMP.imag();  // OK: compile-time initialization
complex cx3(2, 4.6);                      // OK: runtime initialization
```

对象能应用于constant expression的条件：
1） **const**
2） 必须是**integral** 和 **enumeration type**
3） 申明时需要初始化，并且初始化表达式本身必须是**constant expression**。

函数能应用于constant expression的条件：
1）必须以**constexpr**声明
