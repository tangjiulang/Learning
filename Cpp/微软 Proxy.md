# 微软 Proxy

## add_qualifier_t

将 T 的类型固定为 qualifier_type 中的类型，例如 `add_qualifier_t<int, qualifier_type::const_lv>` 等同于 `const int &`

```cpp
enum class qualifier_type { lv, const_lv, rv, const_rv };
template <class T, qualifier_type Q> struct add_qualifier_traits;
template <class T>
struct add_qualifier_traits<T, qualifier_type::lv> : std::type_identity<T&> {};
template <class T>
struct add_qualifier_traits<T, qualifier_type::const_lv>
    : std::type_identity<const T&> {};
template <class T>
struct add_qualifier_traits<T, qualifier_type::rv> : std::type_identity<T&&> {};
template <class T>
struct add_qualifier_traits<T, qualifier_type::const_rv>
    : std::type_identity<const T&&> {};
template <class T, qualifier_type Q>
using add_qualifier_t = typename add_qualifier_traits<T, Q>::type;
template <class T, qualifier_type Q>
using add_qualifier_ptr_t = std::remove_reference_t<add_qualifier_t<T, Q>>*;
```

## recursive_reduction

通过 R 将所有类型进行归约，R 必须是返回一个类型的 `::type` 或者 `using`

```
template <template <class, class> class R, class O, class... Is>
struct recursive_reduction : std::type_identity<O> {};
template <template <class, class> class R, class O, class I, class... Is>
struct recursive_reduction<R, O, I, Is...>
    : recursive_reduction<R, R<O, I>, Is...> {};
template <template <class, class> class R, class O, class... Is>
using recursive_reduction_t = typename recursive_reduction<R, O, Is...>::type;
```

例如使用提取公共类型的  `std::common_type_t<A, B>`

```cpp
template <class A, class B>
using MyReduction = std::common_type_t<A, B>;

recursive_reduction_t<MyReduction, int, short, char, long, long long> xxx;
```

其中 xxx 对应的类型为公共类型 `long long`

递归过程为

1. MyReduction, int, short, char, long, long long，将 int 和 short 对于 MyReduction 进行规约，调用 recursive_reduction_t<MyReduction, MyReduction<int, shrot>, char, long, long long>
2. 上一步得到 recursive_reduction_t<MyReduction, int, char, long, long long>，再将 int 和 char 进行规约，recursive_reduction_t<MyReduction, MyReduction<int, char>, long, long long>
3. 上一步得到 recursive_reduction_t<MyReduction, int, long, long long>，再将 int 和 long 进行规约，
4. 上一步得到 recursive_reduction_t<MyReduction, long, long long>，再将 long 和 long long 进行规约，recursive_reduction_t<MyReduction, MyReduction<long, long long>>
5. 得到 recursive_reduction_t<MyReduction, long long>，此时匹配 template <template <class, class> class R, class O, class... Is>，并且 Is 为空结束递归，返回 O 也就是 long long 的类型

## is_consteval

判断 Expr 是否有默认构造函数，并且重载了 ()，并且重载的 () 是 consteval

```cpp
template <class Expr>
consteval bool is_consteval(Expr)
    { return requires { typename std::bool_constant<(Expr{}(), false)>; }; }
```

## is_tuple_like_well_formed

检查 T 是不是 tuple like 类型

```cpp
template <class T, std::size_t I>
concept has_tuple_element = requires { typename std::tuple_element_t<I, T>; };
template <class T>
consteval bool is_tuple_like_well_formed() {
  if constexpr (requires { { std::tuple_size<T>::value } ->
      std::same_as<const std::size_t&>; }) {
    if constexpr (is_consteval([] { return std::tuple_size<T>::value; })) {
      return []<std::size_t... I>(std::index_sequence<I...>) {
        return (has_tuple_element<T, I> && ...);
      }(std::make_index_sequence<std::tuple_size_v<T>>{});
    }
  }
  return false;
}
```

has_tuple_element 判断 T 类型是否有第 I 个元素

```cpp
requires { { std::tuple_size<T>::value } -> std::same_as<const std::size_t&>; }
```

判断 { std::tuple_size<T>::value } 是否是 const std::size_t& 类型， `{ expression } -> std::same_as<Type>;` 用于检查 expression 的返回类型是否精确匹配Type

`[]<std::size_t... I>((std::index_sequence<I...>) { ... }` 等价于

```cpp
template <std::size_t... I>
auto lambda(std::index_sequence<I...>) { ... }
```

`(lambda)(std::make_index_sequence<std::tuple_size_v<T>>{})` 将 `std::make_index_sequence<std::tuple_size_v<T>` 得到的序列作为参数来调用 lambda

## instantiated_t

利用 TL 和 Args... 来实例化 T

```cpp
template <template <class...> class T, class TL, class Is, class... Args>
struct instantiated_traits;
template <template <class...> class T, class TL, std::size_t... Is,
    class... Args>
struct instantiated_traits<T, TL, std::index_sequence<Is...>, Args...>
    : std::type_identity<T<Args..., std::tuple_element_t<Is, TL>...>> {};
template <template <class...> class T, class TL, class... Args>
using instantiated_t = typename instantiated_traits<
    T, TL, std::make_index_sequence<std::tuple_size_v<TL>>, Args...>::type;
```

## 检查类型属性

```cpp
template <class T>
consteval bool has_copyability(constraint_level level) {
  switch (level) {
    case constraint_level::none: return true;
    case constraint_level::nontrivial: return std::is_copy_constructible_v<T>;
    case constraint_level::nothrow:
      return std::is_nothrow_copy_constructible_v<T>;
    case constraint_level::trivial:
      return std::is_trivially_copy_constructible_v<T> &&
          std::is_trivially_destructible_v<T>;
    default: return false;
  }
}
template <class T>
consteval bool has_relocatability(constraint_level level) {
  switch (level) {
    case constraint_level::none: return true;
    case constraint_level::nontrivial:
      return std::is_move_constructible_v<T> && std::is_destructible_v<T>;
    case constraint_level::nothrow:
      return std::is_nothrow_move_constructible_v<T> &&
          std::is_nothrow_destructible_v<T>;
    case constraint_level::trivial:
      return std::is_trivially_move_constructible_v<T> &&
          std::is_trivially_destructible_v<T>;
    default: return false;
  }
}
template <class T>
consteval bool has_destructibility(constraint_level level) {
  switch (level) {
    case constraint_level::none: return true;
    case constraint_level::nontrivial: return std::is_destructible_v<T>;
    case constraint_level::nothrow: return std::is_nothrow_destructible_v<T>;
    case constraint_level::trivial: return std::is_trivially_destructible_v<T>;
    default: return false;
  }
}
```

has_copyability 检查类型 `T` 是否满足指定的拷贝性要求

has_relocatability 检查 `T` 是否可移动

has_destructibility 检查 `T` 是否可销毁

## destruction_guard

使用 RAII 管理成员对象的生命周期

```cpp
template <class T>
class destruction_guard {
 public:
  explicit destruction_guard(T* p) noexcept : p_(p) {}
  destruction_guard(const destruction_guard&) = delete;
  ~destruction_guard() noexcept(std::is_nothrow_destructible_v<T>)
      { std::destroy_at(p_); }

 private:
  T* p_;
};
```

删除拷贝构造函数

构造函数是拒绝隐式转换并且不会抛出异常

析构函数依据类型 T 的析构是否抛出异常决定，调用 destroy_at(p)，只调用析构函数，但是不释放内存

destruction_guard 适用于 placement new 的对象

## ptr_traits

提取类型 P 在适配 Q 之后的类型是否是指针类型

```cpp
template <class P, qualifier_type Q, bool NE>
struct ptr_traits : inapplicable_traits {};
template <class P, qualifier_type Q, bool NE>
    requires(requires { *std::declval<add_qualifier_t<P, Q>>(); } &&
        (!NE || noexcept(*std::declval<add_qualifier_t<P, Q>>())))
struct ptr_traits<P, Q, NE> : applicable_traits
    { using target_type = decltype(*std::declval<add_qualifier_t<P, Q>>()); };
```

`std::declval<add_qualifier_t<P, Q>>()` 构造了一个假想的对象来判断该类型是否可以解引用，如果可以解引用就提取去掉指针后的原类型

## invocable_dispatch

```cpp
template <class D, bool NE, class R, class... Args>
concept invocable_dispatch = (NE && std::is_nothrow_invocable_r_v<
    R, D, Args...>) || (!NE && std::is_invocable_r_v<R, D, Args...>);
```

检查 D(Args...) 的返回值是否可以转为 R，并且根据传入的 NE 要求 D 是否是 noexcept

- `is_invocable_r_v<R, Fn, ArgTypes...>` 检查是否可以使用参数 `ArgTypes...` 调用 `Fn`，并且返回值是否可以转换为 `R`。
- 如果 `Fn` 可以使用 `ArgTypes...` 调用，并且返回值可以转换为 `R`，则 `is_invocable_r_v` 为 `true`，否则为 `false`。

## func_ptr_t

```cpp
template <bool NE, class R, class... Args>
using func_ptr_t = std::conditional_t<
    NE, R (*)(Args...) noexcept, R (*)(Args...)>;
```

使用 std::condition_t<bool, T, F>，如果 bool 为 True，则为 T，否则为 F

## invoke_dispatch

```cpp
template <class D, class R, class... Args>
R invoke_dispatch(Args&&... args) {
  if constexpr (std::is_void_v<R>) {
    D{}(std::forward<Args>(args)...);
  } else {
    return D{}(std::forward<Args>(args)...);
  }
}
```

利用 c++17 的编译期判断，根据传入类型 R(return) 是否为 void 来判断是否有返回值，然后构造一个默认的 D，调用其重载的 () 运算符

## 调度器

### 转换调度器

```cpp
template <class D, class P, qualifier_type Q, class R, class... Args>
R indirect_conv_dispatcher(add_qualifier_t<std::byte, Q> self, Args... args)
    noexcept(invocable_dispatch_ptr_indirect<D, P, Q, true, R, Args...>) {
  return invoke_dispatch<D, R>(*std::forward<add_qualifier_t<P, Q>>(
      *std::launder(reinterpret_cast<add_qualifier_ptr_t<P, Q>>(&self))),
      std::forward<Args>(args)...);
}
template <class D, class P, qualifier_type Q, class R, class... Args>
R direct_conv_dispatcher(add_qualifier_t<std::byte, Q> self, Args... args)
    noexcept(invocable_dispatch_ptr_direct<D, P, Q, true, R, Args...>) {
  auto& qp = *std::launder(
      reinterpret_cast<add_qualifier_ptr_t<P, Q>>(&self));
  if constexpr (Q == qualifier_type::rv) {
    destruction_guard guard{&qp};
    return invoke_dispatch<D, R>(
        std::forward<add_qualifier_t<P, Q>>(qp), std::forward<Args>(args)...);
  } else {
    return invoke_dispatch<D, R>(
        std::forward<add_qualifier_t<P, Q>>(qp), std::forward<Args>(args)...);
  }
}
template <class D, qualifier_type Q, class R, class... Args>
R default_conv_dispatcher(add_qualifier_t<std::byte, Q>, Args... args)
    noexcept(invocable_dispatch<D, true, R, std::nullptr_t, Args...>)
    { return invoke_dispatch<D, R>(nullptr, std::forward<Args>(args)...); }
```

indirect_conv_dispatcher 间接调用转换（如指针类型），将 std::byte 的封装 self 恢复成 P 类型并调用 invoke_dispatch，使用 std::lauder(...) 来保证指针指向的是有效对象

direct_conv_dispatcher 直接调用转换，若 `Q == qualifier_type::rv`，则使用 `destruction_guard` 确保 `qp` 被正确销毁

default_conv_dispatcher 默认调用转换

### 拷贝调度器

```cpp
template <class P>
void copying_dispatcher(std::byte& self, const std::byte& rhs)
    noexcept(has_copyability<P>(constraint_level::nothrow)) {
  std::construct_at(reinterpret_cast<P*>(&self),
      *std::launder(reinterpret_cast<const P*>(&rhs)));
}
template <std::size_t Len, std::size_t Align>
void copying_default_dispatcher(std::byte& self, const std::byte& rhs)
    noexcept {
  std::uninitialized_copy_n(
      std::assume_aligned<Align>(&rhs), Len, std::assume_aligned<Align>(&self));
}
```

copying_dispatcher 拷贝调度，从 rhs 拷贝构造 self，使用 std::construct_at 显式构造，避免未定义行为。

### 移动调度器

```cpp
template <class P>
void relocation_dispatcher(std::byte& self, const std::byte& rhs)
    noexcept(has_relocatability<P>(constraint_level::nothrow)) {
  P* other = std::launder(reinterpret_cast<P*>(const_cast<std::byte*>(&rhs)));
  destruction_guard guard{other};
  std::construct_at(reinterpret_cast<P*>(&self), std::move(*other));
}
```

relocation_dispatcher 移动调度，使用  rhs 移动构造 self

### 析构调度器

```cpp
template <class P>
void destruction_dispatcher(std::byte& self)
    noexcept(has_destructibility<P>(constraint_level::nothrow))
    { std::destroy_at(std::launder(reinterpret_cast<P*>(&self))); }
inline void destruction_default_dispatcher(std::byte&) noexcept {}
```

destruction_dispatcher 调用 `std::destroy_at` 显式销毁 `self` 里的 `P` 对象

## overload_traits_impl

```cpp
template <class O> struct overload_traits : inapplicable_traits {};
template <qualifier_type Q, bool NE, class R, class... Args>
struct overload_traits_impl : applicable_traits {
  template <bool IsDirect, class D>
  struct meta_provider {
    template <class P>
    static constexpr auto get()
        -> func_ptr_t<NE, R, add_qualifier_t<std::byte, Q>, Args...> {
      if constexpr (!IsDirect &&
          invocable_dispatch_ptr_indirect<D, P, Q, NE, R, Args...>) {
        return &indirect_conv_dispatcher<D, P, Q, R, Args...>;
      } else if constexpr (IsDirect &&
          invocable_dispatch_ptr_direct<D, P, Q, NE, R, Args...>) {
        return &direct_conv_dispatcher<D, P, Q, R, Args...>;
      } else if constexpr (invocable_dispatch<
          D, NE, R, std::nullptr_t, Args...>) {
        return &default_conv_dispatcher<D, Q, R, Args...>;
      } else {
        return nullptr;
      }
    }
  };
  using return_type = R;
  using view_type = R(Args...) const noexcept(NE);

  template <bool IsDirect, class D, class P>
  static constexpr bool applicable_ptr =
      meta_provider<IsDirect, D>::template get<P>() != nullptr;
  static constexpr qualifier_type qualifier = Q;
};
template <class R, class... Args>
struct overload_traits<R(Args...)>
    : overload_traits_impl<qualifier_type::lv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) noexcept>
    : overload_traits_impl<qualifier_type::lv, true, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) &>
    : overload_traits_impl<qualifier_type::lv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) & noexcept>
    : overload_traits_impl<qualifier_type::lv, true, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) &&>
    : overload_traits_impl<qualifier_type::rv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) && noexcept>
    : overload_traits_impl<qualifier_type::rv, true, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const>
    : overload_traits_impl<qualifier_type::const_lv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const noexcept>
    : overload_traits_impl<qualifier_type::const_lv, true, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const&>
    : overload_traits_impl<qualifier_type::const_lv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const& noexcept>
    : overload_traits_impl<qualifier_type::const_lv, true, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const&&>
    : overload_traits_impl<qualifier_type::const_rv, false, R, Args...> {};
template <class R, class... Args>
struct overload_traits<R(Args...) const&& noexcept>
    : overload_traits_impl<qualifier_type::const_rv, true, R, Args...> {};
```

meta_provider 中的 get 函数，根据 IsDirect 和 invocable_dispatch_ptr_indirect 还有 invocable_dispatch_ptr_direct 选择不同的调度器，并返回调度器的函数指针

`view_type`：表示 **函数类型签名**，用于比较是否匹配

overload_traits 对不同的 qualifier_type 还有 NE 进行特化

## dispatcher_meta

```cpp
template <class MP>
struct dispatcher_meta {
  constexpr dispatcher_meta() noexcept : dispatcher(nullptr) {}
  template <class P>
  constexpr explicit dispatcher_meta(std::in_place_type_t<P>) noexcept
      : dispatcher(MP::template get<P>()) {}

  decltype(MP::template get<void>()) dispatcher;
};
```

MP 是 overload_traits

dispatcher_meta 对适配器函数指针的封装

## std::in_place_type_t 

**`std::in_place_type_t<T>`** 是一个 **空结构体**，它的作用是 **作为一个标签（tag）**，用于 **显式指示要构造的类型 `T`**。

## composite_meta_impl

```cpp
template <class... Ms>
struct composite_meta_impl : Ms... {
  constexpr composite_meta_impl() noexcept = default;
  template <class P>
  constexpr explicit composite_meta_impl(std::in_place_type_t<P>) noexcept
      : Ms(std::in_place_type<P>)... {}
};
```

Ms... 表示有多个类，composite_meta_impl 是一个多继承结构

`Ms(std::in_place_type<P>)...` 表示用 `std::in_place_type<P>` 构造 Ms 中的每一个类

例如：

```cpp
struct A { A(std::in_place_type_t<int>) { std::cout << "A constructed\n"; } };
struct B { B(std::in_place_type_t<int>) { std::cout << "B constructed\n"; } };

int main() {
    composite_meta_impl<A, B> meta(std::in_place_type<int>);
}
```

## composite_meta

```cpp
template <class O, class I> struct meta_reduction : std::type_identity<O> {};
template <class... Ms, class I> requires(!std::is_void_v<I>)
struct meta_reduction<composite_meta_impl<Ms...>, I>
    : std::type_identity<composite_meta_impl<Ms..., I>> {};
template <class... Ms1, class... Ms2>
struct meta_reduction<composite_meta_impl<Ms1...>, composite_meta_impl<Ms2...>>
    : std::type_identity<composite_meta_impl<Ms1..., Ms2...>> {};
template <class O, class I>
using meta_reduction_t = typename meta_reduction<O, I>::type;
template <class... Ms>
using composite_meta =
    recursive_reduction_t<meta_reduction_t, composite_meta_impl<>, Ms...>;
```

composite_meta 利用 recursive_reduction_t 和 meta_reduction_t 将 composite_meta_impl<>, Ms... 归约成 composite_meta_impl<Ms...>

注意：`meta_reduction<composite_meta_impl<>, A>` 匹配的是 struct meta_reduction<composite_meta_impl<Ms...>, I>， 其中 Ms... 为空，I 为 A，所以 `meta_reduction<composite_meta_impl<>, A>::type` 为 `composite_meta_impl<空, A>` 也就是  `composite_meta_impl<A>`

////=====;p[[pp           ]]