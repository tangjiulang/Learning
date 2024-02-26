## 四个cast

#### dynamic_cast

*dynamic_cast* 只能够用在指向类的指针或者引用上(或者void*)。这种转换的目的是确保目标指针类型所指向的是一个有效且完整的对象。

同隐式转换一样，这种转换允许upcast(从派生类向基类的转换)。

但*dynamic_cast* 也能downcast(从基类向派生类的转换)当且仅当转过去的指针所指向的目标对象有效且完整

* dynamic_cast是运行时处理的，运行时会进行类型检查（这点和static_cast差异较大）
* dynamic_cast不能用于内置基本数据类型的强制转换，并且dynamic_cast只能对指针或引用进行强制转换
* dynamic_cast如果转换成功的话返回的是指向类的指针或引用，转换失败的话则会返回nullptr
* 使用dynamic_cast进行上行转换时，与static_cast的效果是完全一样的
* 使用dynamic_cast进行下行转换时，dynamic_cast具有类型检查的功能，比static_cast更安全。并且这种情况下dynamic_cast会要求进行转换的类必须具有多态性（即具有虚表，直白来说就是有虚函数或虚继承的类），否则编译不通过
* 需要有虚表的原因：类中存在虚表，就说明它有想要让基类指针或引用指向派生类对象的情况，dynamic_cast认为此时转换才有意义（事实也确实如此）。而且dynamic_cast运行时的类型检查需要有运行时类型信息，这个信息是存储在类的虚表中的
* 在C++中，编译期的类型转换有可能会在运行时出现错误，特别是涉及到类对象的指针或引用操作时，更容易产生错误。dynamic_cast则可以在运行期对可能产生问题的类型转换进行测试

#### static_cast

*static_cast*能够完成指向相关类的指针上的转换。upcast和downcast都能够支持，但不同的是，并不会有运行时的检查来确保转换到目标类型上的指针所指向的对象有效且完整。因此，这就完全依赖程序员来确保转换的安全性。但反过来说，这也不会带来额外检查的开销。

用于基本内置数据类型之间的转换

用于指针之间的转换

`static_cast`可以用于指针之间的转换，这种转换类型检查非常严格，不同类型的指针是直接不给转的，除非使用`void*`作为中间参数，我们知道隐式转换下`void*`类型是无法直接转换为其它类型指针的，这时候就需要借助static_cast来转换了。

```cpp
int type_int = 10;
float* float_ptr1 = &type_int; // int* -> float* 隐式转换无效
float* float_ptr2 = static_cast<float*>(&type_int); // int* -> float* 使用static_cast转换无效
char* char_ptr1 = &type_int; // int* -> char* 隐式转换无效
char* char_ptr2 = static_cast<char*>(&type_int); // int* -> char* 使用static_cast转换无效

void* void_ptr = &type_int; // 任何指针都可以隐式转换为void*
float* float_ptr3 = void_ptr; // void* -> float* 隐式转换无效
float* float_ptr4 = static_cast<float*>(void_ptr); // void* -> float* 使用static_cast转换成功
char* char_ptr3 = void_ptr; // void* -> char* 隐式转换无效
char* char_ptr4 = static_cast<char*>(void_ptr); // void* -> char* 使用static_cast转换成功
```

不能转换掉expression的const或volitale属性

```cpp
int temp = 10;

const int* a_const_ptr = &temp;
int* b_const_ptr = static_cast<int*>(a_const_ptr); // const int* -> int* 无效

const int a_const_ref = 10;
int& b_const_ref = static_cast<int&>(a_const_ref); // const int& -> int& 无效

volatile int* a_vol_ptr = &temp;
int* b_vol_ptr = static_cast<int*>(a_vol_ptr); // volatile int* -> int* 无效

volatile int a_vol_ref = 10;
int& b_vol_ref = static_cast<int&>(a_vol_ref); // volatile int& -> int& 无效
```

`static_cast`也是有明显缺点的，那就是无法消除`const`和`volatile`属性、无法直接对两个不同类型的指针或引用进行转换和下行转换无类型安全检查等

#### reinterpret_cast

*reinterpret_cast*能够完成任意指针类型向任意指针类型的转换，即使它们毫无关联。该转换的操作结果是出现一份完全相同的二进制复制品，既不会有指向内容的检查，也不会有指针本身类型的检查。

基本上*reinterpret_cast*能做但*static_cast*不能做的转换大多都是一些基于重新解释二进制的底层操作，因此会导致代码限定于特定的平台进而导致差移植性。

* type-id和expression中必须有一个是指针或引用类型（可以两个都是指针或引用，指针引用在一定场景下可以混用，但是建议不要这样做，编译器也会给出相应的警告）。
* reinterpret_cast的第一种用途是改变指针或引用的类型
* reinterpret_cast的第二种用途是将指针或引用转换为一个整型，这个整型必须与当前系统指针占的字节数一致
* reinterpret_cast的第三种用途是将一个整型转换为指针或引用类型
* 可以先使用reinterpret_cast把一个指针转换成一个整数，再把该整数转换成原类型的指针，还可以得到原先的指针值（由于这个过程中type-i
* 和expression始终有一个参数是整形，所以另一个必须是指针或引用，并且整型所占字节数必须与当前系统环境下指针占的字节数一致）
* 使用reinterpret_cast强制转换过程仅仅只是比特位的拷贝，和C风格极其相似（但是reinterpret_cast不是全能转换，详见第1点），实际上
* reinterpret_cast的出现就是为了让编译器强制接受static_cast不允许的类型转换，因此使用的时候要谨而慎之
* reinterpret_cast同样也不能转换掉expression的const或volitale属性。

#### const_cast

`const_cast`的作用是去除掉`const`或`volitale`属性，前面介绍`static_cast`的时候我们知道`static_cast`是不具备这种功能的。使用格式如下：

```cpp
const_cast<type_id>(expression);
```

注意，在移除 `const` 之后假如真的向目标进行写操作将导致UB。

如果一个变量本来就不具备`const`属性，但是在传递过程中被附加了`const`属性，这时候使用`const_cast`就能完美清除掉后面附加的那个`const`属性了。

#### typeid

typeid用来检查表达式的类型。

这个操作会返回一个定义在<typeinfo>里面的const对象，这个对象可以同其他用typeid获取的const对象进行==或者!=操作，也可以用过.name()来获取一个表示类型名或者是类名的以空字符结尾的字符串。

当作用在类上时，要用到RTTI来维护动态对象的类型信息。假如参数表达式的类型是多态类，结果将会是派生得最完整的类

注意，当作用在指针上时，仅会考虑其本身类型。当作用在类上时，才会产生出动态类型，即结果是派生得最完整的类。倘若传入的是解引用的空指针，则会抛出bad_typeid异常。

此外，name()的实现依赖于编译器及其使用的库，也许并不一定是单纯的类型名。

## RTTI

RTTI是运行阶段类型识别（Runtime Type Identification）

#### 用途

假设有一个类层次结构，其中的类都是从一个基类派生而来的，则可以让基类指针指向其中任何一个类的对象。

有时候我们会想要知道指针具体指向的是哪个类的对象。因为：

- 可能希望调用类方法的正确版本，而有时候派生对象可能包含不是继承而来的方法，此时，只有某些类的对象可以使用这种方法。
- 也可能是出于调试目的，想跟踪生成的对象的类型。

#### typeid

1. 当typeid中的操作数是以下任意一种时，typeid得出的是静态类型，即编译时就确定的类型：

   * 一个任意的类型名

   * 一个基本内置类型的变量，或指向基本内置类型的指针或引用

   * 一个任意类型的指针（指针就是指针，本身不体现多态，多指针解引用才有可能会体现多态）

   * 一个具体的对象实例，无论对应的类有没有多态都可以直接在编译器确定

   * 一个指向没有多态的类对象的指针的解引用

   * 一个指向没有多态的类对象的引用

2. 当`typeid`中的操作数是以下任意一种时，`typeid`需要在程序运行时推算类型，因为其操作数的类型在编译时期是不能被确定的：

   - 一个指向含有多态的类对象的指针的解引用

   - 一个指向含有多态的类对象的引用

​	虚继承时，虚基类需表现出多态性，而不只是虚继承