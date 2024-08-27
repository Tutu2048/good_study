# rapidjson

官方文档：https://rapidjson.org/zh-cn/md_doc_tutorial_8zh-cn.html

**AddMember**：增加元素到DOM结构中

```c++
/** 格式
GenericValue& AddMember(
    GenericValue& name, 
    GenericValue& value, 
    Allocator& allocator)
**/

//...
_document.AddMember("key","value",._document.GetAllocator());
_document.AddMember("key",123,._document.GetAllocator());
//不会报错	{"key":"value","key":123}
```

**注意！**JSON的**key不具有唯一性**，添加同名key值，会导致1key，2个value，占据2个元素位置。在查找时也只能找到第一个key对于的位置。导致许多错误

---

### 转移语义及临时值

有时候，我们想直接构造一个 Value 并传递给一个“转移”函数（如 `PushBack()`、`AddMember()`）。由于临时对象是不能转换为正常的 Value 引用，我们加入了一个方便的 `Move()` 函数：

```c++
Value a(kArrayType);
Document::AllocatorType& allocator = document.GetAllocator();
// a.PushBack(Value(42), allocator);       // 不能通过编译
a.PushBack(Value().SetInt(42), allocator); // fluent API
a.PushBack(Value(42).Move(), allocator);   // 和上一行相同
```

