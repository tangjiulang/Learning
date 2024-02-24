## `shared_ptr` 和 `weak_ptr`

当 `shared_ptr` 中的 `use_count == 0` 时，资源释放

当 `weak_ptr` 中的 `use_count == 0` 时，内存释放

也就是说，当资源没有释放的时候（`shared_ptr` 没有全部解除），即使 `shared_ptr` 解除后，如果还存在 `weak_ptr` ，依旧可以通过 `weak_ptr::lock()` 重新获得内存的控制权

#### `shared_ptr -> weak_ptr`

```cpp
std::shared_ptr<int> a(new int(1));
std::weak_ptr<int> b = a;
std::cout << *a << '\n';
```

可以直接通过 `shared_ptr` 来构造一个 `weak_ptr` 

#### `weak_ptr -> shared_ptr`

```cpp
std::shared_ptr<int> a(new int(1));
std::shared_ptr<int> d = a;
std::weak_ptr<int> b = a;
std::cout << *a << '\n';
a.reset();
a = b.lock();
std::cout << *a << '\n';
```

#### 关于 `weak_ptr.lock()`

调用 `shared_ptr` 中的构造函数

在所有的 `shared_ptr` 和 `weak_ptr` 消亡之前，`_M_pi`的内存是不会被释放的，所以这里就算之前的`shared_ptr`已经全部消亡（即资源已释放），`_M_pi`还是有效的（因为`weak_ptr`还没有消亡）

#### 使用 `weak_ptr` 来表示临时所有权

- 当某个对象只有存在时才需要被访问，而且随时可能被他人删除时，可以使用 `std::weak_ptr` 来跟踪该对象。
- 需要获得临时所有权时，则将其转换为 `std::shared_ptr`，此时如果原来的 `std::shared_ptr`被销毁，则该对象的生命期将被延长至这个临时的 `std::shared_ptr` 同样被销毁为止。

#### 使用 `weak_ptr` 来打破 `shared_ptr` 造成的引用循环

- 若这种环被孤立（例如无指向环中的外部共享指针），则 `shared_ptr` 引用计数无法抵达零，而内存被泄露。
- 能令环中的指针之一为弱指针以避免此情况。

```cpp
#include <iostream>
#include <memory>


class Node {
public:
	std::shared_ptr<Node> next;
	Node() {
		std::cout << "hello node" << '\n';
	}
	~Node() {
		std::cout << "bye bye node" << '\n';
	}
};

using Nodeptr = std::shared_ptr<Node>;

int main() {
	Nodeptr a(new Node());
	Nodeptr b(new Node());
	a.get()->next = b;
	b.get()->next = a;
	std::cout << a.use_count() << '\n';
	std::cout << b.use_count() << '\n';
	return 0;
}
/*
hello node
hello node
2
2
*/
```

将 `Node` 中的 `shared_ptr`  改为 `weak_ptr` 后，就可以正常析构了

```cpp
#include <iostream>
#include <memory>


class Node {
public:
	std::weak_ptr<Node> next;
	Node() {
		std::cout << "hello node" << '\n';
	}
	~Node() {
		std::cout << "bye bye node" << '\n';
	}
};

using Nodeptr = std::shared_ptr<Node>;

int main() {
	Nodeptr a(new Node());
	Nodeptr b(new Node());
	a.get()->next = b;
	b.get()->next = a;
	std::cout << a.use_count() << '\n';
	std::cout << b.use_count() << '\n';
	return 0;
}
/*
hello node
hello node
1
1
bye bye node
bye bye node
*/
```

只要环中有一个是 `weak_ptr` 就可以打破循环了，析构顺序是先析构 `weak_ptr` 然后根据顺序析构

#### 总结

##### `weak_ptr`

`weak_ptr` 对 `shared_ptr` 是监察作用，`weak_ptr` 不影响引用计数，同时也不干预资源的释放，但我们应该避免使用失效的 `weak_ptr` （即对应的 `shared_ptr` 全部被释放

`weak_ptr` 主要使用场景是延迟加载，防止循环引用，缓存

在使用 `weak_ptr.lock()` 之前，可以先用 `expired()` 查看是否失效

##### `shared_ptr`

自动资源管理：`shared_ptr` 提供了自动管理动态分配的对象的能力。当最后一个 `shared_ptr` 对象超出作用域或被显式释放时，引用计数变为零，对象会被销毁，从而实现资源的自动释放。

多个拥有者：`shared_ptr` 允许多个智能指针同时拥有对同一对象的所有权。这样可以避免手动跟踪对象的使用情况，提高代码的可维护性和安全性。

安全共享：通过使用 `shared_ptr`，可以在多个地方共享对象，而无需手动管理其生命周期。只要存在至少一个 `shared_ptr` 指向对象，对象就会保持有效。

避免循环引用：循环引用是指两个或多个对象之间相互持有 `shared_ptr`，导致引用计数永远无法变为零，从而造成内存泄漏。为了避免循环引用，可以使用 `weak_ptr` 来打破循环引用关系，或者使用 `std::enable_shared_from_this` 来获取自身的 `shared_ptr`。

避免裸指针与 `shared_ptr` 混用：避免在代码中混用裸指针和 `shared_ptr`，这可能导致资源管理混乱和潜在的错误。尽量使用 `shared_ptr` 来管理对象的生命周期，避免手动释放资源。

注意避免悬空指针：当一个 `shared_ptr` 指向的对象被释放后，其他仍然持有该对象的 `shared_ptr` 可能会成为悬空指针。因此，在使用 `shared_ptr` 时要小心处理悬空指针的情况，确保不会访问已经释放的对象。

适度使用 `shared_ptr`：`shared_ptr` 带有引用计数的开销，每次增减引用计数都需要一定的开销。因此，对于生命周期较短、无需共享的对象，可以考虑使用 `unique_ptr` 来避免引用计数的开销。

## `auto_ptr`

c++11禁用

可以发现`std::auto_ptr`的失败在于CXX03中并不支持移动语义，而`std::auto_ptr` 却试图用复制构造函数来实现移动构造函数的功能，结果导致其无法与`vector` 等容器兼容，论为失败品。

## `unique_ptr`

1. 禁止复制构造函数、复制赋值的重载，即设置为`=delete`；
2. 实现各种移动构造函数；
3. 实现移动赋值重载，即`operator=(unique_ptr&&)`，需要先释放本身的资源，再将对方的资源移动过来；
4. 如果资源没有释放过，则会在析构函数中释放。

## 总结

|                                            | `auto_ptr` | `unique_ptr` | `shared_ptr` | `weak_ptr` |
| :----------------------------------------: | :--------: | :----------: | :----------: | :--------: |
|                是否持有资源                |     Y      |      Y       |      Y       |     Y      |
|            消亡是否影响资源释放            |     Y      |      Y       |      Y       |     N      |
|                是否独占资源                |     Y      |      Y       |      N       |     N      |
|           是否具有普通指针的行为           |     Y      |      Y       |      Y       |     N      |
| 能否转换为`shared_ptr`（有条件地转换也算） |     Y      |      Y       |      Y       |     Y      |
|        是否支持自定义释放内存的方法        |     N      |      Y       |      Y       |     N      |
|         是否完全支持容器的各种行为         |     N      |      N       |      Y       |     Y      |
|                                            |            |              |              |            |



