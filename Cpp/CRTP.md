```cpp
 // The Curiously Recurring Template Pattern (CRTP)
template <class T>
class Base
{
    // methods within Base can use template to access members of Derived
};
class Derived : public Base<Derived>
{
    // ...
};
```

从基类对象的角度来看，派生类对象其实就是本身，这样的话只需要使用类型转换就可以把基类转化成派生类，从而实现基类对象对派生对象的访问。

具体的例子

```cpp
template <typename T>
class Base {
public:
    void interface() {
        static_cast<T*>(this)->imp();
    };
};

class Derived : public Base<Derived> {
public:
    void imp() {
        std::cout<< "in Derived::imp" << std::endl;  
    }
};

int main() {
  Base<Derived> b;
  d.interface();
  
  return 0;
}
```

使用`static_cast`进行类型转换，从而调用派生类的成员函数。可能会有人感到好奇，为什么不用`dynamic_cast`进行类型转换呢？主要是因为**dynamic_cast应用于运行时，而模板是在编译器就进行了实例化**。

在这个例子中，派生类Derived中定义了一个成员函数`imp()`，而该函数在基类Base中是没有声明的，所以，我们可以理解为**对于CRTP，在基类中调用派生类的成员函数，扩展了基类的功能**。而对于普通继承，则是**派生类中调用基类的成员函数，扩展了派生类的功能**，这就是我们所说的`颠倒继承`。

## 作用

### 静态多态

```cpp
#include <iostream>

#include <iostream>

template <typename T>
class Base{
 public:
  void interface(){
    static_cast<T*>(this)->imp();
  }
  void imp(){
    std::cout << "in Base::imp" << std::endl;
  }
};

class Derived1 : public Base<Derived1> {
 public:
  void imp(){
    std::cout << "in Derived1::imp" << std::endl;
  }
};

class Derived2 : public Base<Derived2> {
 public:
  void imp(){
    std::cout << "in Derived2::imp" << std::endl;
  }
};

class Derived3 : public Base<Derived3>{};

template <typename T>
void fun(T& base){
    base.interface();
}


int main(){
  Derived1 d1;
  Derived2 d2;
  Derived3 d3;

  fun(d1);
  fun(d2);
  fun(d3);

  return 0;
}
```

### 代码复用

```cpp
template<typename T>
class Base {
 public:
  void PrintType() {
    T &t = static_cast<T&>(*this);
    std::cout << typeid(t).name() << std::endl;
  }
};

class Derived : public Base<Derived> {};
class Derived1 : public Base<Derived1> {};

template<typename T>
void PrintType(T base) {
  base.PrintType();
}

int main() {
  Derived d;
  Derived1 d1;
  PrintType(d);
  PrintType(d1);
  return 0;
}
```

## 局限性

CRTP 的基类实际上是一个模板类，得到的例如 Base\<Drived> 和 Base\<Drived1> 实际上是两个变量，所以是不能放在同一个容器里面的