---
layout: "post"
title: "value categories"
date: "2020-09-10 14:15"
---

##  CRTP (奇异递归模板编程)

```c++
template<class T>
class Base {
  ...
};

class Derived : public Base<Derived> {
};
```

例子：enabled_shared_from_this<A>

```c++
template <typename T>
struct Base {
  void foo() {
    (static_cast<T*>(this))->foo();
  }
};

struct Derived : public Base<Derived> {
  void foo() {
    cout << "derived foo" << endl;
  }
};

struct AnotherDerived : public Base<AnotherDerived> {
  void foo() {
    cout << "AnotherDerived foo" << endl;
  }
};

template<typename T>
void ProcessFoo(Base<T>* b) {
  b->foo();
}

int main()
{
    Derived d1;
    AnotherDerived d2;
    ProcessFoo(&d1);
    ProcessFoo(&d2);
    return 0;
}
```
