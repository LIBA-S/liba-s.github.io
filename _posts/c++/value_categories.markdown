---
layout: "post"
title: "value categories"
date: "2020-09-10 14:15"
---

## 分类
1） glvalue： 能够决定对象的identity的表达式
2） prvalue： 表达式计算:

3)  xvalue: 一种glvalue，表明该对象资源可以被复用
4)  lvalue（历史原因：能放在赋值表达式左边）：glvalue，但不是xvalue
5)  rvalue（历史原因: 能放在赋值表达式右边）：prvalue 或者 xvalue


## 基础类型

1） lvalue
  - 变量的名字，不管变量是指向什么类型（即便是右值引用类型）
  - 函数返回值，**lvalue reference** 或者 **rvalue reference to function**
  - 字符串字面量，“hello world”

属性：
- 和glvalue一样
- 能够被取地址
- 能放在赋值运算符左边
- 能被用于初始化**lvalue reference**

2） prvalue
  - 字面量
  - 函数返回值，**non-references**
  - **this指针**
  - lambda表达式
  - enumerator

属性：
- 和rvalue一样
- 不能是**incompleted type**

3） xvalue
  - 函数返回值， **rvalue reference to object**， 比如：std::move(x)
