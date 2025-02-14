# 成员函数后置 const 修饰 (`Class xxx()  const`) 

语义行为，表示此函数的返回值不会被修改

标准库中会针对 const 和非 const 写两个函数

非 const 修饰函数语义为返回的对象可以被修改

const 修饰函数语义为返回的对象不能被修改

# std::string_view

作为 `std::string&` 的视图使用，在保证函数只查看字符串时，可以使用 `std::stirng_view` 来代替 `const std::string&`，`std::string_view` 本身并不是字符串，只是查看字符串的视图，所以使用 `std::string_view` 会比使用 `const&` 更快。

函数中常使用 `const std::string&` 来传入一个字符串，并且保证了传入的字符串不为空

```cpp
const std::string& get(const std::string& test) {
	return test;
}

int main() {
	const std::string& str1 = test("xxx");
	std::string str = "xxx";
	str = test(str);
}
```

在上面的代码中 `test("xxx")` 中的 "xxx" 是一个右值，可以用 `const &` 来接受一个右值，延长了右值的声明周期，但是 "xxx" 的生命周期只存在 `const std::string& str1 = test("xxx");` 中，下一行时会被销毁，此时 `str1` 会成为悬垂引用。