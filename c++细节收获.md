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

##### **lock_guard**

功能：防止忘记解锁，设计理念类似于智能指针的锁保护，在析构时自动解锁

用法：

```

```

