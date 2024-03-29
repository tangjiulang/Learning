# const

**`const`对象必须初始化**

使用**值传递初始化**时，被初始化的对象是否为`const`与初始化对象是否为`const`无关

####  **顶层`const`和底层`const`**

通常在指针/引用与`const`符同时使用时会用到这个概念。修饰指针本身的`const`称为顶层`const`，修饰指针所指向对象的`const`称为底层`const`。底层`const`与顶层`const`是两个互相独立的修饰符，互不影响。

引用自带底层`const`

可以将底层`const`的指针（或引用）指向（或绑定）到非`const`对象，但不允许非底层`const`的指针（或引用）指向（或绑定）到`const`对象。 （即：`const`对象不允许通过任何方式（指针/引用）被修改。）

#### const 与函数

```cpp
void fcn(const int i) { /* ... */ }
```

**形参`i`是否为`const`与传入的实参是否为`const`是完全无关的**

##### 值传递的`const`

因为值传递的`const`形参在调用上与非`const`形参没有区别（大概是指，无论形参是否为`const`，实参都不会被修改。），所以仅仅使用`const`无法区分参数类别，所以无法实现函数重载

##### **`const`指针/引用的形参**

由于底层`const`描述实参性质（不允许在调用函数内部被修改），可以在调用时区分`const`，所以使用底层`const`的指针/引用可以实现函数重载

#### **`const`与类**

1. 类的对象的`const`修饰表示该对象的成员变量不允许被修改。
2. 无论类的成员变量本身是否为`const`，只要对象声明为`const`，成员变量就不允许被修改。

#####  **`const`与类的成员函数**

当对象被声明为`const`时，该对象不能调用非`const`函数，因为非`const`函数可能修改成员变量。

1. 将成员函数声明为`const`函数，则可以被`const`对象调用，声明`const`函数的方法为在其参数列表后添加`const`关键字。
2. `const`成员函数中不允许修改成员变量。也即，并非所有成员函数都可以被声明为`const`函数，C++会在编译时检查被声明为`const`的函数是否修改了成员变量，若是，则报错，编译不通过。

与底层`const`形参一样，`const`成员函数也可以实现重载。同样，当非常量对象调用函数时，编译器会优先调用非常量版本的函数。

##### **总结**

- 当函数不修改成员变量时，尽可能将函数声明为`const`函数，因为`const`函数可以被非`const`对象和`const`对象调用，而非`const`函数只能被非`const`对象调用。
- `const`函数并不意味着数据安全，虽然不能通过`const`函数修改成员变量，但是这样的`const`仅为顶层`const`（即成员变量本身不能被修改），若成员变量包含非底层`const`的指针/引用，虽然成员变量本身不能被修改，但依然可以通过这些指针/引用修改其指向/绑定的对象。

#### **`const`成员函数实现机制**

```c++
void Number::set(const Number *const this, int num) { number = num; } 
// 仅作为参考，实际上，C++规定显式定义this指针为非法操作
```

# Static

**静态成员变量有以下特点：**

1. 静态成员变量是该类的所有对象所共有的。对于普通成员变量，每个类对象都有自己的一份拷贝。而静态成员变量一共就一份，无论这个类的对象被定义了多少个，静态成员变量只分配一次内存，由该类的所有对象共享访问。所以，静态数据成员的值对每个对象都是一样的，它的值可以更新；
2. 因为静态数据成员在全局数据区分配内存，由本类的所有对象共享，所以，它不属于特定的类对象，不占用对象的内存，而是在所有对象之外开辟内存，在没有产生类对象时其作用域就可见。因此，在没有类的实例存在时，静态成员变量就已经存在，我们就可以操作它；
3. 静态成员变量存储在全局数据区。`static` 成员变量的内存空间既不是在声明类时分配，也不是在创建对象时分配，而是在初始化时分配。静态成员变量必须初始化，而且只能在类体外进行。否则，编译能通过，链接不能通过。在`Example` 5中，语句`int Myclass::Sum=0;`是定义并初始化静态成员变量。初始化时可以赋初值，也可以不赋值。如果不赋值，那么会被默认初始化，一般是 0。静态数据区的变量都有默认的初始值，而动态数据区（堆区、栈区）的变量默认是垃圾值。
4. `static` 成员变量和普通 `static` 变量一样，编译时在静态数据区分配内存，到程序结束时才释放。这就意味着，`static` 成员变量不随对象的创建而分配内存，也不随对象的销毁而释放内存。而普通成员变量在对象创建时分配内存，在对象销毁时释放内存。
5. 静态数据成员初始化与一般数据成员初始化不同。初始化时可以不加 `static`，但必须要有数据类型。被 `private、protected、public` 修饰的 `static` 成员变量都可以用这种方式初始化。静态数据成员初始化的格式为：＜数据类型＞＜类名＞::＜静态数据成员名＞=＜值＞
6. 类的静态成员变量访问形式1：＜类对象名＞.＜静态数据成员名＞
7. 类的静态成员变量访问形式2：＜类类型名＞::＜静态数据成员名＞，也即，静态成员不需要通过对象就能访问。
8. 静态数据成员和普通数据成员一样遵从`public,protected,private`访问规则；
9. 如果静态数据成员的访问权限允许的话（即`public`的成员），可在程序中，按上述格式来引用静态数据成员 ；
10. `sizeof` 运算符不会计算 静态成员变量。

**静态成员函数的特点：**

1. 出现在类体外的函数定义不能指定关键字 `static`；
2. 静态成员之间可以相互访问，即静态成员函数（仅）可以访问静态成员变量、静态成员函数；
3. 静态成员函数不能访问非静态成员函数和非静态成员变量；
4. 非静态成员函数可以任意地访问静态成员函数和静态数据成员；
5. 由于没有 `this` 指针的额外开销，静态成员函数与类的全局函数相比速度上会稍快；
6. 调用静态成员函数，两种方式：

- 通过成员访问操作符(.)和(->)，也即通过类对象或指向类对象的指针调用静态成员函数。
- 直接通过类来调用静态成员函数。＜类名＞::＜静态成员函数名＞（＜参数表＞）。也即，静态成员不需要通过对象就能访问。

**静态全局变量有以下特点：**

1. 该变量在全局数据区分配内存；
2. 未经初始化的静态全局变量会被程序自动初始化为0（自动变量的自动初始化值是随机的）；
3. 静态全局变量在声明它的整个文件都是可见的，而在文件之外是不可见的； 　
4. 静态变量都在全局数据区分配内存，包括后面将要提到的静态局部变量。对于一个完整的程序，在内存中的分布情况如下：【代码区】【全局数据区】【堆区】【栈区】，一般程序的由 `new` 产生的动态数据存放在堆区，函数内部的自动变量存放在栈区，静态数据（即使是函数内部的静态局部变量）存放在全局数据区。自动变量一般会随着函数的退出而释放空间，而全局数据区的数据并不会因为函数的退出而释放空间。

**静态局部变量有以下特点：**

1. 静态局部变量在全局数据区分配内存；
2. 静态局部变量在程序执行到该对象的声明处时被首次初始化，即以后的函数调用不再进行初始化；
3. 静态局部变量一般在声明处初始化，如果没有显式初始化，会被程序自动初始化为0；
4. 静态局部变量始终驻留在全局数据区，直到程序运行结束。但其作用域为局部作用域，当定义它的函数或语句块结束时，其作用域随之结束；

**静态函数的好处：（类似于静态全局变量）**

1. 静态函数不能被其它文件所用；
2. 其它文件中可以定义相同名字的函数，不会发生冲突；

# this

1）一个对象的`this`指针并不是对象本身的一部分，不会影响`sizeof`(对象)的结果。

2）`this`作用域是在类内部，当在类的非静态成员函数中访问类的非静态成员的时候，编译器会自动将对象本身的地址作为一个隐含参数传递给函数。也就是说，即使你没有写上`this`指针，编译器在编译的时候也是加上`this`的，它作为非静态成员函数的隐含形参，对各成员的访问均通过`this`进行。

其次，`this`指针的使用：

（1）在类的非静态成员函数中返回类对象本身的时候，直接使用 `return *this`。

（2）当参数与成员变量名相同时，如`this->n = n` （不能写成`n = n`)。

`this` 本身是一个 `T* const` 指针，在成员函数的开始执行前构造，在成员的执行结束后清除

# sizeof

- 空类的大小为1字节
- 一个类中，虚函数本身、成员函数（包括静态与非静态）和静态数据成员都是不占用类对象的存储空间。
- 对于包含虚函数的类，不管有多少个虚函数，只有一个虚指针,`vptr`的大小。
- 普通继承，派生类继承了所有基类的函数与成员，要按照字节对齐来计算大小
- 虚函数继承，不管是单继承还是多继承，都是继承了基类的`vptr`。(32位操作系统4字节，64位操作系统 8字节)！
- 虚继承,继承基类的`vptr`。

# volatile

- `volatile` 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素（操作系统、硬件、其它线程等）更改。所以使用 `volatile` 告诉编译器不应对这样的对象进行优化。
- `volatile` 关键字声明的变量，每次访问时都必须从内存中取出值（没有被 `volatile` 修饰的变量，可能由于编译器的优化，从 CPU 寄存器中取值）
- `const` 可以是 `volatile` （如只读的状态寄存器）
- 指针可以是 `volatile`

# union

联合（`union`）是一种节省空间的特殊的类，一个 `union` 可以有多个数据成员，但是在任意时刻只有一个数据成员可以有值。当某个成员被赋值后其他成员变为未定义状态。联合有如下特点：

- 默认访问控制符为 `public`
- 可以含有构造函数、析构函数
- 不能含有引用类型的成员
- 不能继承自其他类，不能作为基类
- 不能含有虚函数
- 匿名 `union` 在定义所在作用域可直接访问 `union` 成员
- 匿名 `union` 不能包含 `protected` 成员或 `private` 成员
- 全局匿名联合必须是静态（`static`）的

# explicit

- explicit 修饰构造函数时，可以防止隐式转换和复制初始化
- explicit 修饰转换函数时，可以防止隐式转换，但按语境转换除外

```cpp
#include <iostream>

using namespace std;

struct A {
  A(int) {}
  operator bool() const { return true; }
};

struct B {
  explicit B(int) {}
  explicit operator bool() const { return true; }
};

void doA(A a) {}

void doB(B b) {}

int main() {
  A a1(1);     // OK：直接初始化
  B b1(1);     // OK：直接初始化
  
  A a2 = 1;    // OK：复制初始化
//B b2 = 1;    // 错误：被 explicit 修饰构造函数的对象不可以复制初始化
  
  A a3{1};     // OK：直接列表初始化
  B b3{1};     // OK：直接列表初始化
  
  A a4 = {1};  // OK：复制列表初始化
//B b4 = {1};  // 错误：被 explicit 修饰构造函数的对象不可以复制列表初始化
  
  A a5 = (A)1; // OK：允许 static_cast 的显式转换
  B b5 = (B)1; // OK：允许 static_cast 的显式转换
  
  doA(1);      // OK：允许从 int 到 A 的隐式转换
//doB(1);      // 错误：被 explicit 修饰构造函数的对象不可以从 int 到 B 的隐式转换
  
  if (a1)
    ; // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
  if (b1)
    ; // OK：被 explicit 修饰转换函数 B::operator bool() 的对象可以从 B 到 bool 的按语境转换
  
  bool a6(a1); // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
  bool b6(b1); // OK：被 explicit 修饰转换函数 B::operator bool() 的对象可以从 B 到 bool 的按语境转换
  
  bool a7 = a1; // OK：使用转换函数 A::operator bool() 的从 A 到 bool 的隐式转换
//bool b7 = b1; // 错误：被 explicit 修饰转换函数 B::operator bool() 的对象不可以隐式转换
  
  bool a8 = static_cast<bool>(a1); // OK：static_cast 进行直接初始化
  bool b8 = static_cast<bool>(b1); // OK：static_cast 进行直接初始化
  return 0;
}
```

# friend

友元提供了一种 普通函数或者类成员函数 访问另一个类中的私有或保护成员 的机制。也就是说有两种形式的友元：

（1）友元函数：普通函数对一个访问某个类中的私有或保护成员。

（2）友元类：类A中的成员函数访问类B中的私有或保护成员

优点：提高了程序的运行效率。

缺点：破坏了类的封装性和数据的透明性。

总结：

- 能访问私有成员
- 破坏封装性
- 友元关系不可传递
- 友元关系的单向性
- 友元声明的形式及数量不受限制

注意，友元函数只是一个普通函数，并不是该类的类成员函数，它可以在任何地方调用，友元函数中通过对象名来访问该类的私有或保护成员。

友元类的声明在该类的声明中，而实现在该类外。

- 友元关系没有继承性 假如类B是类A的友元，类C继承于类A，那么友元类B是没办法直接访问类C的私有或保护成员。
- 友元关系没有传递性 假如类B是类A的友元，类C是类B的友元，那么友元类C是没办法直接访问类A的私有或保护成员，也就是不存在“友元的友元”这种关系。

# decltype

`decltype` 的作用是“查询表达式的类型”

对于`decltype(e)`而言，其判别结果受以下条件的影响：

如果`e`是一个没有带括号的标记符表达式或者类成员访问表达式，那么的`decltype(e)`就是`e`所命名的实体的类型。此外，如果`e`是一个被重载的函数，则会导致编译错误。 否则 ，假设`e`的类型是`T`，如果`e`是一个将亡值，那么`decltype(e)`为`T&&` 否则，假设`e`的类型是`T`，如果`e`是一个左值，那么`decltype(e)`为`T&`。 否则，假设`e`的类型是`T`，则`decltype(e)`为`T`。

标记符指的是除去关键字、字面量等编译器需要使用的标记之外的程序员自己定义的标记，而单个标记符对应的表达式即为标记符表达式。

```cpp
int i = 4;
int arr[5] = { 0 };
int *ptr = arr;
struct S{ double d; }s ;
void Overloaded(int);
void Overloaded(char);//重载的函数
int && RvalRef();
const bool Func(int);

//规则一：推导为其类型
decltype (arr) var1; //int[] 标记符表达式
decltype (ptr) var2;//int *  标记符表达式
decltype(s.d) var3;//doubel 成员访问表达式
//decltype(Overloaded) var4;//重载函数。编译错误。

//规则二：将亡值。推导为类型的右值引用。
decltype (RvalRef()) var5 = 1;

//规则三：左值，推导为类型的引用。
decltype ((i))var6 = i;     //int&
decltype (true ? i : i) var7 = i; //int&  条件表达式返回左值。
decltype (++i) var8 = i; //int&  ++i返回i的左值。
decltype(arr[5]) var9 = i;//int&. []操作返回左值
decltype(*ptr)var10 = i;//int& *操作返回左值
decltype("hello")var11 = "hello"; //const char(&)[9]  字符串字面常量为左值，且为const左值。


//规则四：以上都不是，则推导为本类型
decltype(1) var12;//const int
decltype(Func(1)) var13=true;//const bool
decltype(i++) var14 = i;//int i++返回右值
```

