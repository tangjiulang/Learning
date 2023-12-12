# C++类型推导

## template 类型推导

```cpp
template<typename T>
void f(ParamType param);
f(expr)
```

分三种情况

1.  $ParamType$ 是一个非通用的指针或者引用

   ```cpp
   template<typename T>
   void f(const T& param);
   void f1(T & param);
   void solve() {
   	int x = 27;
   	const int cx = x;
   	const int &rx = x;
   	f1(x);// T 是 int，param 类型是 int &
   	f(x); // T 是 int，param 类型是 const int &
   	f1(cx);// T 是 const int，param 类型是const int &
   	f(cx);// T 是 const int，param 类型是 const int &
   	f1(rx);//T 是 const int，param 类型是 const int & 
   	f(rx);// T 是 const int &，param 类型是 const int &
   }
   ```

   其中，const 属性是 T 类型推导的一部分

   引用特性会被类型推导忽略

2. $ParamType$ 是一个通用的引用

   ```cpp
   template<typename T>
   void f(T&& param);
   
   void solve() {
   	int x = 27;
   	const int cx = x;
   	const int &rx = x;
   
   	f(x);
   	f(cx);
   	f(rx);
   	
   	f(27); // 27是右值，所以 T 是 int，param 的类型是 int&& 
   }
   ```

   如果 expr 是左值的话，就和 & 的推导一样，如果 expr 是右值，就会变成一个右值引用

3. $ParamType$ 既不是指针也不是引用

   param 是一个新的对象，不会被 expr 的 const 和引用影响

4. 数组参数

   ```cpp
   template<typename T>
   void f(T param);
   
   void solve() {
   	char a[13];
   	f(a); // T 的类型是 char*，param 的类型也是 char*
   }
   ```

   在传递参数的时候，数组会退化成指针，但是有一个巧妙的方法是，声明参数为数组的引用

   ```cpp
   template<typename T>
   void f(T &param);
   
   void solve() {
   	char a[13];
   	f(a); // T 的类型是 char [13], param 的类型是 char (&)[13]
   }
   ```

5. 函数参数

   和数组类似

## auto 类型推导

auto 推导和 template 相似，但是 auto 假设花括号初始化代表的是 std::initializer_list 但是模板类型并不是

auto 在函数返回值或者 lambda 参数里面执行模板的类型推导，而不是通常意义下的 auto 类型推导

## 理解 decltype

给定一个变量名或者表达式，decltype 会告诉你这个变量名或者表达式的类型

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container &c, Index i) -> decltype(c[i]) {
    authenticateUser();
    return c[i];
}
```

其中 -> 指的是尾随类型（优势是：在定义返回值类型的时候使用函数参数），在这种情况下，返回值就是 [] 操作子的返回值

如果我们运行：

```cpp
std::deque<int> d;
...
authAndAccess(d, 5) = 10;
```

那么返回的就是 T& 是一个左值，可以运行

如果不加后面的，写作：

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container &c, Index i) {
    return c[i];
}
```

调用时

![image-20230325165039196](C:\Users\tyl\Desktop\modernc++学习\15445\img\image-20230325165039196.png)

因为我们知道 auto 在作为函数返回值的时候是采用的 template 的推导方法，而 template 会忽略初始表达式的引用，所以返回的是一个 int，临时的右值对象，给一个右值赋值 10 是不允许的，所以出错。

因为如此，在某种情况下我们又需要 decltype 的类型推导，所以在c++14 我们开始使用 decltype(auto)，其中 auto 指定需要推导的类型，decltype 表明在推导的过程中用 decltype 的 推导规则

```CPP
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container &c, Index i) {
    return c[i];
}	
```

但是这样只能引入一个左值引用，如果传递一个右值，右值不能和左值绑定，传不进去

我们可以修改为

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container &&c, Index i) {
    return c[i];
}
```

来接收右值，同时这样由于右值引用是临时容器，并不能成功返回，所以我们需要进一步修改为

```cpp
template<typename Container, typename Index>
auto authAndAccess(Container &&c, Index i) {
    return std::forward<Container>(c)[i];
}
```

