# auto 关键字

## 优先使用 auto 而不是显式声明

#### auto 的优点

避免未初始化的变量，变量声明引发的歧义，直接持有封装体的能力

避免“类型截断”问题

## 当 auto 推导出非预期类型的时候应使用显式的类型初始化

当我们使用代理类（一个类的存在是为了模拟和对外行为和另外一个类保持一致）的时候，auto 推导出的是代理类的对象，而不是我们所希望的类对象，显式定义的时候代理类会发生隐式转换成我们所希望的类，但是 auto 的时候不会，所以在这样的情况下，我们应该使用显式的类型初始化

例如：

```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

## 当创建对象时，{} 和 () 的区别

#### 使用 {} 的好处

##### c++ 指定初始化的三种方式，只有 {} 支持所有的

```cpp
class test{
    private:
    int x = 10;
    int y{10};
    int z(10);   // 错误
};
void solve() {
    int x = 10;
    int y{10};
    int z(10);
    
}
void solve1() {
    std::atomic<int> ai1{0};
    std::atomic<int> ai2(0);
    std::atomic<int> ai3 = 0; // 错误，在c++17之后支持
}
```



##### {} 初始化，阻止了隐式收缩转换，表达式不能保证被初始化对象表现出来，代码无法通过编译

```cpp
double x = 1.0, y = 2.0;
int z1{x + y};
```

会警告

```cpp
warning: narrowing conversion of '(x + y)' from 'double' to 'int' inside { } [-Wnarrowing]
int z1{x + y};
```

##### 能够帮助开发者调用默认构造函数

```cpp
Widget w1(10);	//使用参数10调用Widget的构造函数
Widget w2();	//最令人恼火的解析！声明一个
				//名字是w2返回Widget的函数
Widget w3{};	//不带参数调用Widget的默认构造函数
```

#### {} 的局限性

当auto声明的变量使用花括号初始化时，它的类型被推导为**std::initializer_list**, 而使用一样的初始化表达式，别的方式声明的变量能产生更符合实际的类型

如果不涉及 **std::initializer_list** () 和 {} 是一样的

如果涉及，{}会优先推导成 **std::initializer_list**

```cpp
class Widget {
public:
	Widget(int i, bool b);	
	Widget(int i, double d);
	Widget(std::initializer_list<long double> il);

	...
};
```

```cpp
Widget w1(10, true); 	//使用圆括号，和以前一样，调用
						//第一个构造函数

Widget w2{10, true}; 	//使用花括号，但是现在调用
						//std::initializer_list版本的
						//构造函数（10和true转换为long double）

Widget w3(10, 5.0);		//使用圆括号，和以前一样，调用
						//第二个构造函数

Widget w4{10, 5.0};		//使用花括号，但是现在调用
						//std::initializer_list版本的
						//构造函数（10和5.0转换为long double）
```

甚至普通的拷贝和移动构造函数也会被**std::initializer_list**构造函数所劫持：

```cpp
class Widget {
public:
	Widget(int i, bool b);	
	Widget(int i, double d);
	Widget(std::initializer_list<long double> il);
	operator float() const;		//转换到float
	...
};					
Widget w5(w4);				//使用圆括号，调用拷贝构造函数
Widget w6{w4};				//使用花括号，调用
							//std::initializer_list构造函数
							//（w4 转换到float，然后float
							//转换到long double）
Widget w7(std::move(w4));	//使用圆括号，调用移动构造函数
Widget w8{std::move(w4)};	//使用花括号，调用
							//std::initializer_list构造函数
							//（同w6一样的原因）
```

编译器使用带**std::initializer_list**参数的构造函数来匹配花括号初始化的决心如此之强，就算最符合的**std::initializer_list**构造函数不能调用，它还是能胜出

```cpp
class Widget {
public:
	Widget(int i, bool b);	
	Widget(int i, double d);
	Widget(std::initializer_list<bool> il);	//元素类型
											//现在是bool
	...
};	
Widget w{10, 5.0};	//错误！需要收缩转换（narrowing conversion）
```

在花括号初始化中，只有当这里没有办法转换参数的类型为**std::initializer_list**时，编译器才会回到正常的重载解析

假设你使用空的花括号来构造对象，这个支持默认构造函数并且支持**std::initializer_list**构造函数。那么你的空的花括号意味着什么呢？如果它们意味着“没有参数”，你得到默认构造函数，但是如果它们意味着“空的**std::initializer_list**”你得到不带元素的**std::initializer_list**构造函数。

规则是你会得到默认构造函数，空的花括号意味着没有参数，不是一个空的**std::initializer_list**

```cpp
class Widget {
public:
	Widget();		//默认构造函数
	//std::initializer_list构造函数
	Widget(std::initializer_list<int> il);	
	...
};	
Widget w1;		//调用默认构造函数
Widget w2{};	//也调用默认构造函数
Widget w3();	//最令人恼火的解析！声明一个函数！
Widget w4({});		//使用空的list调用
					//std::initializer_list构造函数
Widget w5{{}};		//同上
```

## 优先使用 $nullptr$ 而不是 $0$ 或者 $NULL$

$0$ 和 $NULL$ 都不是指针类型，$nullptr$ 优势在于，可以理解为一个指向任意类型的指针

同时，如果返回值是一个指针，$nullptr$ 能使得 $result$ 的类型更明显

相比较于 $0$ 和 $NULL$，优先使用 $nulltpr$

避免了函数类型和指针类型之间的重载

## 优先使用声明别名而不是 $typedef$

$typedef$ 不支持模板化，但是别名声明支持

模板别名避免了 $::type$ 后缀，模板中 typedef 还需要使用 $typename$ 前缀

声明别名使得函数指针的声明更加容易理解

```cpp
typedef void (*FP)(int, const std::string&);
using FP = void (*)(int, const std::string&);
```

$typedef$ 不支持模板化，但是别名声明支持

```cpp
template<typename T>
struct A {
  typedef std::list<T> type;
};

template<typename T>
using B = std::list<T>; // 模板别名，必须是一个类型

template<typename T>
class C{
  A<T>::type la; //error: need 'typename' before 'A<T>::type' because 'A<T>' is a dependent scope
    // 不知道是不是一个类型名
  typename A<T>::type lla;
  B<T> lb;
};
```

## 优先使用作用域限制的 $enums$ 而不是无作用域的 $enum$

$enums$ 有作用域，$enum$ 没有作用域，作用于全局，$enum$ 会造成命名空间的污染

```cpp
enum status1 {black, white, red};
auto white = false;
//error: 'auto white' redeclared as different kind of symbol
//note: previous declaration 'status1 white'
enum class status2 {black, white, blue};
auto blue = false;
```

无作用域的 enum 会将所有的元素都隐式转换成整型，或者浮点类型

有作用域的 enums 不会将所有的元素隐式转换到其他类型

有定义域的 enum 可以提前被声明的，可以不指定枚举元素而进行声明

没有定义域的 enum 只有指定潜在类型的时候才可以前置声明

```cpp
enum status1 : std::int16_t;
void continueProcessing(status1 s);

enum class status2;
void continueProcessing(status2 s);
```

## 优先使用delete关键字删除函数而不是priavte却又不实现的函数

用 =delete 标识拷贝复制函数和拷贝赋值函数为删除的函数 deleted function

```cpp
template <class charT, class traits>
class basic_ios : public ios_base {
  public:
  basic_ios(const basic_ios&) = delete;
  basic_ios& operator=(const basic_ios&) = delete;
};
```

delete 删除函数是在编译期的时候就被发现的，私有函数不声明只能在链接时才会诊断出

删除函数一般定义成公有的，如果定义成私有的，编译器可能只警报该函数为私有的，得到的报错信息不准确

删除函数的优势在于，任何函数都是可删除的，但是只有成员变量才可以是私有的

```cpp
bool islucky(int num);

bool islucky(bool) = delete;
bool islucky(char) = delete;
bool islucky(double) = delete;

islucky(3);
islucky(true); //无法引用 函数 "islucky(bool)" (已声明) -- 它是已删除的函数
islucky('a'); //无法引用 函数 "islucky(char)" (已声明) -- 它是已删除的函数
```

两种特殊的指针

一个是 void* 指针，没有办法对他们解引用，递增或递减的操作

一个是 char* 指针，标识指向 c 类型的字符串，而不是指向独立字符的指针

## 使用 override 关键字声明覆盖的函数

#### 使用覆盖的函数需要满足：

基类的函数是虚函数

函数必须完全一样（除了析构函数）

参数类型一样

常量特性一样

返回值和异常声明

函数的引用修饰符一样

#### 使用 override

有些编译器会将没有覆盖的代码接收而不发出警告

使用 override 可以将覆盖函数的所有的问题都揭露出来

```cpp
class Base {
  public:
  virtual void mf1() const;
  virtual void mf2(int x);
  virtual void mf3() &;
};

class Derived : public Base {
  public:
  virtual void mf1() override; //使用“override”声明的成员函数不能重写基类成员
  virtual void mf1() const override;
  virtual void mf2(double x) override; //使用“override”声明的成员函数不能重写基类成员
  virtual void mf2(int x) override;
  virtual void mf3() && override; //使用“override”声明的成员函数不能重写基类成员
  virtual void mf3() & override;
};
```

同时，他也可以帮助我们评估更改基类的虚函数的标识符可能会引起的后果

## 优先使用 const_iterator 而不是 iterator

const_iterator 在 STL 中等价于指向 const 的指针，指向的数值不能被修改

应该尽可能在没有必要修改指针指向的内容的时候使用 const_iterator
