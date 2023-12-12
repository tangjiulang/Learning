#### std::mem_fn

我们可以使用std::mem_fn生成指向类成员函数的指针的包装对象，该对象可以存储，复制和调用指向类成员函数的指针。而我们实际使用的是std::mem_fn的返回值std::_Mem_fn这个类，而我们在调用std::_Mem_fn中重载的()方法时，可以使用类对象、派生类对象、对象引用（包括std::reference_wrapper）、对象的右值引用、指向对象的指针（包括智能指针）来作为第一个参数传递进去。

```cpp
#include <functional>
#include <iostream>
#include <memory>
#include <utility>
#include <string>

class Age {
private:
    int age;
public:
    Age(int age) : age(age) {}
    Age(const Age& a) : age(a.age) {}
    Age(Age &&A) : age(std::move(A.age)) {} 
    virtual bool compare(const Age &A) const { 
        std::cout << age << ' ' << A.age << '\n';
        return age < A.age;
    }
};

class Child : public Age {
private:
    std::string name;
public:
    Child(int a, std::string name) : Age(a), name(name) {}
    Child(const Age &a, const std::string &name) : Age(a), name(name) {}
    Child(const Child &a) : Age(static_cast<Age>(a)), name(a.name) {}
    Child(Child &&a) : Age(std::move(static_cast<Age>(a))), name(std::move(a.name)) {}
    virtual bool compare(const Child &A) const {
        return static_cast<Age>(*this).compare(static_cast<Age>(A));
    }
};


int main() {
    Age a(8);
    Age b(9);
    std::shared_ptr<Age> c(new Age(10));
    auto cp = std::mem_fn(&Age::compare);
    std::cout << cp(a, b) << '\n';
    std::cout << cp(c, 9) << '\n';
    std::cout << cp(std::move(a), b) << '\n';
    Child ch(11, "abc");
    std::cout << cp(ch, a) << '\n';
}
```

#### std::ref

这主要是考虑函数式编程（如std::bind）在使用时，是对参数直接拷贝，而不是引用

`std::ref`和`std::cref`事实上是模板函数，返回值是一个`std::reference_wrapper`对象，而`std::reference_wrapper`虽然是一个对象，可是他却能展现出和普通引用类似的效果

当我们在函数式编程（如std::bind）中需要对参数进行引用传递时，只需要使用用`std::ref`或`std::cref`修饰该引用即可。

```cpp
void fun(int &a, int &b, const int &c) {
    std::cout << a << ' ' << b << ' ' << c << '\n';
    a ++;
    b ++;
    std::cout << a << ' ' << b << ' ' << c << '\n';
}

int main() {
    int a = 1, b = 1, c = 1;
    std::function<void()> f = std::bind(fun, a, std::ref(b), std::cref(c));
    std::cout << "before: ";
    std::cout << a << ' ' << b << ' ' << c << '\n';    
    f();
    std::cout << "after: ";
    std::cout << a << ' ' << b << ' ' << c << '\n';
}
```

#### std::function