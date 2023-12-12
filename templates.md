# 模板元编程

## SFINAE

很多时候匹配失败并不意味着编译错误，所以在遇到 failure 的时候，还需要进行其他的尝试

SFINAE最主要的作用，是保证编译器在泛型函数、偏特化、及一般重载函数中遴选函数原型的候选列表时不被打断。除此之外，它还有一个很重要的元编程作用就是实现部分的编译期自省和反射。

## enable_if

```cpp
template<bool, typename _Tp = void>
struct enable_if
{ };

// Partial specialization for true.
template<typename _Tp>
struct enable_if<true, _Tp>
{ typedef _Tp type; };

```

## is_same

```cpp
template <typename T, typename U>
struct myis_same {
    static constexpr bool value = false;
};

template <typename T>
struct myis_same<T, T> {
    static constexpr bool value = true;
};
```

## enable_if重定义错误

```cpp
// template <typename T, typename T2 = typename std::enable_if<std::is_same<Aclass, T>::value>::type>
template <typename T, typename T2 = int>
unsigned int len(T const& t) {
    return 0;
}

// template <typename T, typename T2 = typename std::enable_if<std::is_same<Bclass, T>::value>::type>
template <typename T, typename T2 = double>
unsigned int len(T const& t) {
    return 1;
}
//error: redefinition of ‘template<class T, class T2> unsigned int len(const T&)
```

本质上是重定义了 `T2` 的默认参数，实际的模板依旧是

```cpp
template <typename T, typename T2>
```

将上面代码修改为

```cpp
// template <typename T, typename T2 = typename std::enable_if<std::is_same<Aclass, T>::value>::type>
template <typename T, typename std::enable_if<std::is_same<Aclass, T>::value>::type* = nullptr>
unsigned int len(T const& t) {
    return 0;
}

// template <typename T, typename T2 = typename std::enable_if<std::is_same<Bclass, T>::value>::type>
template <typename T, typename std::enable_if<std::is_same<Bclass, T>::value>::type* = nullptr>
unsigned int len(T const& t) {
    return 1;
}
```

通过 `type* -> void*` 来匹配任意类型的指针，同时使用 `nullptr` 来赋值给任意类型的指针

# 模板编程

## 函数模板

``` c++
template<typename T>
bool equivalent(const T& a, const T& b) {
    return !(a < b) && !(b < a);
}

template<>
bool equivalent(const std::string& a, const std::string& b) {
    return a + b < b + a;
}
```

注意：

```cpp
T& a, T& b 
```

由于 `T` 推导得到的类型是一个，所以 `a` 和 `b` 必须是同一种类型，所以

``` cpp
std::cout << equivalent(2, 3) << '\n';
std::cout << equivalent(3, 3.0) <<'\n';
std::cout << equivalent<double>(3, 3.0) << '\n';
std::cout << equivalent("123", "123") << '\n';
```

四句中，第二句是错误的，因为编译器推导的时候，3 是整数，而 3.0 是浮点数，是两种不同的类型，所以报错，对此，我们可以进行像第三句一样，指定推导类型。

注意：如果一个函数模板有一个以上的模板类型参数，则每个模板类型参数之前都必须加上关键字 `class` 或 `typename`

## 类模板

模板类函数类的声明和实现必须放在同一个 .h 文件内

```cpp
template<typename T = int> // 默认参数为 int
class bignumber {
    T _v;
public:
    bignumber(T a) : _v(a) { }
    inline bool operator<(const bignumber& b) const;
    inline T operator+(const bignumber& b) {
        return _v + b._v;
    } // 类模板内的成员函数
};

template<typename T>
bool bignumber<T>::operator<(const bignumber<T> &b) const {
    return _v < b._v;
} // 类模板外的成员函数
```

调用的时候，我们也可以使用默认参数，或者自己决定参数类型

```cpp
bignumber<> a(1), b(2); // 默认参数
bignumber<double> c(2.0), d(3.3); // 指定参数
std::cout << (a < b) << '\n';
std::cout << (a + b) << '\n';
std::cout << (c + d) << '\n';
```

## 模板实例化

### 隐式实例化

```cpp
bignumber<> a(1), b(2); // 默认参数
bignumber<double> c(2.0), d(3.3); // 指定参数
// std::cout << (a < b) << '\n';
std::cout << (a + b) << '\n';
std::cout << (c + d) << '\n';
```

如果我们将上面注释的那句话去掉，那么函数

```cpp
template<typename T>
bool bignumber<T>::operator<(const bignumber<T> &b) const {
    return _v < b._v;
} // 类模板外的成员函数
```

将不会实例化

### 显示实例化

显式实例化其实是特例化的一种，直接声明模板实例化

```CPP
template<int N>
class aTMP {
public :
    enum { ret = N * aTMP<N - 1>::ret };
};

template<> class aTMP<0> {
public:
    enum { ret = 1 };
};
```

### 关于 `template，typename，this` 关键字的使用

如果解析器在一个模板中遇到一个嵌套依赖名字，它假定那个名字不是一个类型，除非显式用 `typename` 关键字前置修饰该名词

```cpp
typename T::reType m
```

指定 `reType` 是一种类型

`template` 用于指明嵌套类型或函数为模板

`this` 用于指定查找基类的成员

```cpp
#include<iostream>

template<typename T>
class aTMP {
public:
    typedef const T reType;
};

void f() { std::cout << "globla f(n)\n"; }

template<typename T>
class Base {
public:
    template<int N = 99>
    void f() { std::cout << "member f(): " << N << '\n'; }
};

template<typename T>
class Derived:public Base<T> {
public:
    typename T::reType m; // 不可省略 typename
    Derived(typename T::reType a) : m(a) {}
    void df1() { f(); } // 调用全局 f（）
    void df2() { this->template f(); } // 调用基类的 f（）
    void df3() { Base<T>::template f<22>(); } // 强制基类的 f（）
    void df4() { ::f(); } // 强制全局 f（）
};

int main() {
    Derived<aTMP<int>> a(10);
    a.df1(), a.df2(), a.df3(), a.df4();
    return 0;
}
```

## `move`

将值转化为右值引用，运用了引用折叠技术

```cpp
template<typename T>
std::remove_reference_t<T>&& move(T&& value) {
    return static_cast<std::remove_reference_t<T>&&>(value);
}
```



## `forward`

```cpp
template <typename T>
T&& foward(std::remove_reference_t<T>& value) {
    return static_cast<T&&>(value);
}

template <typename T>
T&& foward(std::remove_reference_t<T>&& value) {
    return static_cast<T&&>(value);
}
```

## `traits`技术

在接口和定义中间的一层间接层，为同一类数据（包括自定义数据类型和内置数据类型）提供统一的操作函数

```cpp
enum Type{type_1, type_2, type_3};
class foo {
public:
    static const Type type = type_1;
};
class bar {
public:
    static const Type type = type_2;
};
template <typename T>
struct traits_type {
    static const Type type = T::type;
};

template <>
struct traits_type<int> {
    static const Type type = Type::type_3;
};


template <typename T>
void print(const T& a) {
    if (traits_type<T>::type == Type::type_1) {
        std::cout << "type_1" << '\n';
    }
    if (traits_type<T>::type == Type::type_2) {
        std::cout << "type_2" << '\n';
    }
    if (traits_type<T>::type == Type::type_3) {
        std::cout << "type_3" << '\n';
    }
}
```

#### 元函数

##### 定义

一个元函数可以是一个类模板，它的所有参数都是类型

或者一个类，带有一个名为 `type` 的可公开访问的嵌套类型结果

#### 元数据

##### 定义

编译器系统操纵的值，元数据不可变，不能有副作用

## `policy`

`Policy`（策略）是一种设计模式，用于将算法或行为与其所依赖的类分离，以便在运行时动态选择或切换不同的策略。`Policy` 通常通过模板参数、虚函数和接口等机制来实现。与 `Traits` 不同，`Policy` 更关注于在运行时选择不同的策略，提供了动态多态性的方式。

```cpp
template <typename Strategy>
class Context {
public:
    void execute() {
        Strategy strategy;
        strategy.execute();
    }
};
class StrategyA {
public:
    void execute() {
        // 策略A的执行逻辑
    }
};
class StrategyB {
public:
    void execute() {
        // 策略B的执行逻辑
    }
};
int main() {
    Context<StrategyA> contextA;
    contextA.execute();  // 使用策略A执行
    Context<StrategyB> contextB;
    contextB.execute();  // 使用策略B执行
    return 0;
}
```

## `trait` 和 `policy` 的区别

`trait` 注重特性

`policy` 注重行为

​	
