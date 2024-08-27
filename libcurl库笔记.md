# libcurl库笔记

```c++
curl_easy_setopt(_curl, CURLOPT_WRITEDATA, &_Respond);//设置写入data的空间
curl_easy_setopt(_curl, CURLOPT_WRITEFUNCTION, *fun_ptr);//设置回调函数
size_t (*function)(char* ptr, size_t size, size_t nmemb, void* userdata)；//函数指针
```

回调中的`void* userdata`即为`&_Respond`传入的空间，**类型要保持一致，避免不必要的错误**

---

help

