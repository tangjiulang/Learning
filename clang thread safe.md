# Thread Safety Annotation

#### 修饰类的宏

```c++
// CAPABILITY 表明某个类对象可以当作 capability 使用，其中 x 的类型是 string，能够在错误信息当中指出对应的 capability 的名称
#define CAPABILITY(x) THREAD_ANNOTATION_ATTRIBUTE__(capability(x))

// SCOPED_CAPABILITY 用于修饰基于 RAII 实现的 capability。
#define SCOPED_CAPABILITY THREAD_ANNOTATION_ATTRIBUTE__(scoped_lockable)

```

`capability` 是 TSA 中的一个概念，用来为资源的访问提供相应的保护。这里的资源可以是数据成员，也可以是访问某些潜在资源的函数/方法。

`capability` 通常表现为一个带有能够获得或释放某些资源的方法的对象，最常见的就是 `mutex` 互斥锁。换言之，一个 `mutex` 对象就是一个 `capability`

#### 修饰数据成员的宏：

```cpp
// GUARD_BY 用于修饰对象，表明该对象需要受到 capability 的保护
#define GUARDED_BY(x) THREAD_ANNOTATION_ATTRIBUTE__(guarded_by(x))
// PT_GUARDED_BY(mutex) 用于修饰指针类型变量，在更改指针变量所指向的内容前需要加锁，否则发出警告
#define PT_GUARDED_BY(x) THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded_by(x))
```

##### 使用案例

```cpp
int *p1             GUARDED_BY(mu);
int *p2             PT_GUARDED_BY(mu);
unique_ptr<int> p3  PT_GUARDED_BY(mu);

int main() {
  p1 = 0;             // Warning!
  *p2 = 42;           // Warning!
  p2 = new int;       // OK.
  *p3 = 42;           // Warning!
  p3.reset(new int);  // OK.
}
```

#### 修饰 `capability` 的宏

```cpp
// ACQUIRED_BEFORE 和 ACQUIRED_AFTER 主要用于修饰 capability 的获取顺序，用于避免死锁
// _VA_ARGS__ 没有加锁的时候获取锁，加锁后 warning
#define ACQUIRED_BEFORE(...) THREAD_ANNOTATION_ATTRIBUTE__(acquired_before(__VA_ARGS__))

// _VA_ARGS__ 加锁后才能获取锁
#define ACQUIRED_AFTER(...) THREAD_ANNOTATION_ATTRIBUTE__(acquired_after(__VA_ARGS__))
```

#### 修饰函数/方法(成员函数)的宏：

```cpp
// REQUIRES 声明调用线程必须拥有对指定的 capability 具有独占访问权。可以指定多个 capabilities。函数/方法在访问资源时，必须先上锁，再调用函数，然后再解锁(注意，不是在函数内解锁)
#define REQUIRES(...) THREAD_ANNOTATION_ATTRIBUTE__(requires_capability(__VA_ARGS__))

// REQUIRES_SHARED 功能与 REQUIRES 相同，但是可以共享访问
#define REQUIRES_SHARED(...) THREAD_ANNOTATION_ATTRIBUTE__(requires_shared_capability(__VA_ARGS__))

//ACQUIRE 表示一个函数/方法需要持有一个 capability，但并不释放这个 capability。调用者在调用被 ACQUIRE 修饰的函数/方法时，要确保没有持有任何 capability，同时在函数/方法结束时会持有一个 capability(加锁的过程发生在函数体内)
#define ACQUIRE(...) THREAD_ANNOTATION_ATTRIBUTE__(acquire_capability(__VA_ARGS__))

//ACQUIRE_SHARED 与 ACQUIRE 的功能是类似的，但持有的是共享的 capability
#define ACQUIRE_SHARED(...) THREAD_ANNOTATION_ATTRIBUTE__(acquire_shared_capability(__VA_ARGS__))
```

