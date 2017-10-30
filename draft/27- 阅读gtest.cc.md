针对自己对C++代码还是相当生疏、而脑袋中又一直挥之不去的.NET BCL和FCL的情况，我决定在阅读googletest实现的基础上开始书写一些基于.NET Framework接口的C++实现。

于是有了新的repo：[Cpp11Playground](https://github.com/kiddliu/Cpp11Playground)

那么先前提到的gtest运行时会检测所有的位标志。每一个位标志都可以通过设置环境变量的方式影响程序的运行方式，否则加载默认的默认值。下边回顾一下所有位标志的定义和默认值：

```cpp
bool FLAGS_GTEST_ALSO_RUN_DISABLED_TESTS false
bool FLAGS_GTEST_BREAK_ON_FAILURE false
bool FLAGS_GTEST_CATCH_EXCEPTIONS true
string FLAGS_GTEST_COLOR "auto"
string FLAGS_GTEST_FILTER "*"
bool FLAGS_GTEST_LIST_TESTS false
string FLAGS_GTEST_OUTPUT ""
bool FLAGS_GTEST_PRINT_TIME true
int32 FLAGS_GTEST_RANDOM_SEED 0
int32 FLAGS_GTEST_REPEAT 1
bool FLAGS_GTEST_SHOW_INTERNAL_STACK_FRAMES false
bool FLAGS_GTEST_SHUFFLE false
int32 FLAGS_GTEST_STACK_TRACE_DEPTH 100
bool FLAGS_GTEST_THROW_ON_FAILURE false
string FLAGS_GTEST_STREAM_RESULT_TO ""
```

而获取环境变量的方式，标准库提供了`<cstdlib>`中的`char* getenv( const char* env_var );`。而我也通过实现.NET Framework中的`Environment`和`RegistryKey`接口实现了Windows平台上的C++实现。在实现的过程碰到了下边几个问题：

* pImpl + `std::unique_ptr<T>`的配合使用

  首先，选取了`std::unique_ptr<T>`而不是`std::shared_ptr<T>`是因为我们在访问系统和当前用户的环境变量是通过访问注册表实现的，而Win32 API都是通过创建句柄授权用户代码访问的。那么拷贝动作对于`RegistryKey`来说就没有移动有意义了，于是定义了移动构造函数和移动赋值构造函数。而`std::unique_ptr<T>`又保证了删除对象的时候释放句柄，最一开始我完全没有定义析构函数。但是这是错误的，因为定义了移动构造函数说明我们手动管理了资源，也一定需要定义析构函数，这里`RegistryKey::~RegistryKey() = default;`就足够了。

* 在扩展标准库的异常时，为什么不接受`std::wstring`？

  在C++中，接受`const char *`并不意味着只能用ANSI，而是指我们可以用单字节的方式处理字节流：那么UTF-8也是OK的。这样就完全足够应对各种多语言了不是？

* 从C++11开始标准库要求`std::string`或者说对应的`allocator`实现必须支持连续存储

  也就是说我们以后不必只能借助`std::vector`去对接C接口了，这对于封装许多Win32 API就非常方便了。

* [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)真的很混乱

* [Resharper C++](https://www.jetbrains.com/resharper-cpp/)对于我这种小白真的很好用（手动捂脸）
