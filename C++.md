#### std::rank

查看数组维度

```c++
std::cout << std::rank<int>::value << '\n';      // 0
std::cout << std::rank<int[]>::value << '\n';    // 1
std::cout << std::rank<int[][2]>::value << '\n'  // 2
```

#### std::numeric_limits

获得基础类型的极值和属性，包括最大值，最小值，是否是符号数等

```cpp
std::cout << std::numeric_limits<int>::min() << '\n';
std::cout << std::numeric_limits<int>::max() << '\n';
```

## 折叠表达式

1. 一元右折叠： (形参包 运算符 ...)

   展开为 ($E_1$ 运算符 ($E_2$ 运算符 ... ($E_{n-1}$ 运算符 $E_n$)))

   ```cpp
   template<typename... Args>
   bool all(Args... args) { return (... && args); }
    
   bool b = all(true, true, true, false);
   // 在 all() 中，一元左折叠展开成
   //  return ((true && true) && true) && false;
   // b 是 false
   ```

2. 一元左折叠：(... 运算符 形参包)

   展开为 (($E_1$ 运算符 $E_2$ 运算符)... $E_{n-1}$) 运算符 $E_n$

3. 二元右折叠：(形参包 运算符 ... 运算符 初值)

   ($E_1$ 运算符 (... 运算符 ($E_{N−1}$ 运算符 ($E_N$ 运算符 $I$))))

   ```cpp
   template<typename... Args>
   int sum(Args&&... args)
   {
   //  return (args + ... + 1 * 2);   // 错误：优先级低于转换的运算符
       return (args + ... + (1 * 2)); // OK
   }
   ```

4. 二元左折叠：(初值 运算符 ... 运算符 形参包)

   (((($I$ 运算符 $E_1$) 运算符 $E_2$) 运算符 ...)  运算符 $E_N$)

## 形参包 $template<typename ... Args>$

### 包展开  =（模式）...

模式：为含有形参包的完整表达式

包展开：后随 ... 的模式

在同一个模式下的多个形参包，形参个数必须相同

```cpp
template<typename...> struct Tuple {};
template<typename T1, typename T2> struct Pair {};

template<class... Args1>
struct zip {
	template<class... Args2>
	struct with {
		typedef Tuple<Pair<Args1, Args2>...> type;
		template<class ...Args3>
		struct txt {
			typedef Tuple<Tuple<Args1, Args2, Args3>...> type1;
		};
	};
};

typedef zip<short, int>::with<unsigned short, unsigned>::txt<double, float>::type1 t1;	
```

如果多个形参包是内嵌展开，会做会将两个形参包做笛卡尔积

```cpp
template<class... Args>
void g(Args... args)
{
    f(const_cast<const Args*>(&args)...); 
    // const_cast<const Args*>(&args) 是模式，它同时展开两个包（Args 与 args）
 
    f(h(args...) + args...); // 嵌套包展开：
    // 内层包展开是 “args...”，它首先展开
    // 外层包展开是 h(E1, E2, E3) + args 它其次被展开
    // （成为 h(E1, E2, E3) + E1, h(E1, E2, E3) + E2, h(E1, E2, E3) + E3）
}
```

### 展开场所

#### 参数实参列表

```cpp
f(args...) // f(E1, E2, E3)
```

#### 有括号的初始化器

```cpp
Class test(&args...) // 调用 Class::Class(&E1, &E2, &E3)
```

#### 花括号包括的初始化列表

```cpp
template<typename... Ts>
void func(Ts... args)
{
    const int size = sizeof...(args) + 2;
    int res[size] = {1, args..., 2};
 
    // 因为初始化器列表保证顺序，所以这可以用来对包的每个元素按顺序调用函数：
    int dummy[sizeof...(Ts)] = {(std::cout << args, 0)...};
}
```

#### 模板实参列表

模板实参列表的任何位置使用

```cpp
container<A, B, C...> t1; // 展开成 container<A, B, E1, E2, E3> 
container<C..., A, B> t2; // 展开成 container<E1, E2, E3, A, B> 
container<A, C..., B> t3; // 展开成 container<A, E1, E2, E3, B> 
```

#### 参数形参列表

```cpp
template<typename... Ts>
void f(Ts...) {}
 
f('a', 1); // Ts... 会展开成 void f(char, int)
f(0.1);    // Ts... 会展开成 void f(double)
```

#### 模板形参

```cpp
template<typename... T>
struct value_holder
{
    template<T... Values> // 会展开成非类型模板形参列表，
    struct apply {};      // 例如 <int, char, int(&)[5]>
};
```

#### 基类说明符与成员初始化器列表

```cpp
template<class... Mixins>
class X : public Mixins...
{
public:
    X(const Mixins&... mixins) : Mixins(mixins)... {}
};
```

#### lambda 捕获

```cpp
template<class... Args>
void f(Args... args)
{
    auto lm = [&, args...] { return g(args...); };
    lm();
}
```

#### 折叠表达式，using 声明

