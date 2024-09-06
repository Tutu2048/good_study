# c++细节收获

2024-05-29

**std::string无法被标准输入流输出问题(windows vs)**

```c++
#include <iostream>
using namespace std;
int main(){
	string str("hello world");
    cout <<str<<endl;
}
```

STL中的许多头文件都**间接地包含了`<xstring>`**（但不要试图直接包含`<xstring>`），这就保证了你可以仅`include`这些头文件（如本例的`#include <iostream>`）就可使用`std::string`类，**导致ide可以找到string不会爆红**，而`<xstring>`中没有实现操作符<<的重载，所以在编译时会报错。问题很隐蔽。

---

**explicit 关键字的运用**

```c++
class Integer {
public:
    explicit Integer(double d) {
        // 从double到Integer的转换
    }
};

Integer i = 10.5; // 错误：不能隐式转换
Integer i2 = Integer(10.5); // 正确：显式转换
```

---

**回调函数**

为什么需要回调函数

---

**std::vector**初始化问题

```c++
//方式1
vector<int> v(n,0);
//方式2
vector<int> v;
v.reserve(n);
```

方式1：分配内存的同时用0来初始化n个空间

方式2：仅分配内存

这就导致方式二在后续使用时要注意，先赋值再访问，否则会导致脏数据。

- 方式2是一种更加迅速和快捷的初始化方式，当在拥有自己的初始化方式的时候用方式二来提升一定的性能。
- 方式一是更加健壮的方法

---

##### **lock_guard unique_lock**

功能：防止忘记解锁，设计理念类似于智能指针的锁保护，在析构时自动解锁

用法：

```c++
lock_guard<mutex> guard(mt);
```

而`lock_guard`没有中途解锁的缺陷，所在地方的生命周期过长，会导致一直处于上锁状态，颗粒度过大，由此出现了`unique_lock`

用法：

```c++
unique_lock<mutex> unique(mt);
//...
unique.unlock()
```

总结：感觉用处不大

---

##### std::atomic

`std::atomic<T>` c++11引入提供原子操作，由硬件处理，非常高效

---

##### enable_shared_from_this(C++11)

这是一个类，内部拥有一个weak_ptr，继承该类可以实现在类函数获取指向自身的shared_ptr。



这个实现的前提，问题是

- 一旦原生指针交予智能指针管理，除却第一次初始化时的原生指针的使用，对于对象的操作都应该使用智能指针，对于原生指针的使用都应该视为非法操作，所以类内是无法获取shared_ptr的。

- weak_ptr 不拥有对象，用于观察shared_ptr的管理情况，可以通过lock()函数来获取一个shared_ptr，从此拥有对象，可以使用对象的function。

  



---

##### 大括号

*大括号可以用于限制作用域* --让对象在大括号} 时就结束生命周期进行析构

---

##### make_shared



- **内存分配**：
  - **`std::shared_ptr` 直接创建**：当你直接使用 `new` 关键字创建一个对象，然后将其包装在一个 `std::shared_ptr` 中时，对象和它的引用计数是分开分配的。这意味着分配两次内存：一次用于对象本身，一次用于引用计数器。
  - **`std::make_shared`**：这个函数会一次性分配足够的内存来存储对象和它的引用计数器（通常还包括一个弱引用计数器）。这种方式被称为“联合分配”（combined allocation），可以减少内存分配的次数，从而提高性能。
- **异常安全性**：
  - **`std::shared_ptr` 直接创建**：如果 `new` 操作符抛出异常，那么 `std::shared_ptr` 将不会被创建，但是 `delete` 操作仍然需要被调用以释放内存。这需要你手动管理，增加了代码的复杂性和出错的可能性。
  - **`std::make_shared`**：它提供了异常安全的保证。如果 `new` 操作符抛出异常，`make_shared` 会保证不会有内存泄漏，因为它会负责释放已经分配的内存。
- **性能**：
  - 使用 `std::make_shared` 通常比直接创建 `std::shared_ptr` 性能更好，因为它减少了内存分配的次数。
- **使用场景**：
  - **`std::shared_ptr` 直接创建**：适用于那些需要立即将一个裸指针转换为 `shared_ptr` 的情况，或者当你需要在创建 `shared_ptr` 时传递额外的参数给构造函数。
  - **`std::make_shared`**：适用于创建新的 `shared_ptr` 实例，特别是当你需要创建对象并立即管理它的生命周期时。

**总结：优先使用make_shared创建共享指针**

---

