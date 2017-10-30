24-26 阅读googletest源码之gtest.h
===================================================

十九号的面试失败其实是不出意外的：四个月的C++/Windows准备期间由于基础薄弱以及自我认知的一些偏差，侧重点主要放到了我认为的基础知识方面。那么从这周开始我的转型任务是在余下三本进阶C++书籍之外，开始着重阅读一些实际工程的代码。在Google了一番之后，大部分来自网络上的建议是从Google自己的C++源码看起，例如[googletest](https://github.com/google/googletest)、[protocol buffer](https://github.com/google/protobuf)还有[chromium](https://github.com/chromium/chromium)。

那么今天就从googletest开始，用Visual Studio打开[工程](https://github.com/google/googletest/tree/master/googletest/msvc/2010)。工程分为4部分：gtest，gtest_main, gtest_prod_test和gtest_unittest，顾名思义。

gtest只有单个文件[gtest-all.cc](https://github.com/google/googletest/blob/master/googletest/src/gtest-all.cc)，包含了最核心的几行代码：
```cpp
// This line ensures that gtest.h can be compiled on its own, even
// when it's fused.
#include "gtest/gtest.h"

// The following lines pull in the real gtest *.cc files.
#include "src/gtest.cc"
#include "src/gtest-death-test.cc"
#include "src/gtest-filepath.cc"
#include "src/gtest-port.cc"
#include "src/gtest-printers.cc"
#include "src/gtest-test-part.cc"
#include "src/gtest-typed-test.cc"
```

根据我目前的理解，那[gtest.h](https://github.com/google/googletest/blob/master/googletest/include/gtest/gtest.h)应该是核心了。打开文件，可以看到包含了用到的其他内部头文件
```cpp
#include "gtest/internal/gtest-internal.h"
#include "gtest/internal/gtest-string.h"
#include "gtest/gtest-death-test.h"
#include "gtest/gtest-message.h"
#include "gtest/gtest-param-test.h"
#include "gtest/gtest-printers.h"
#include "gtest/gtest_prod.h"
#include "gtest/gtest-test-part.h"
#include "gtest/gtest-typed-test.h"
```

稍后这里有段注释非常有意思，是说在Linux系统上如果`GTEST_HAS_GLOBAL_STRING`被定义，gtest也可以使用与`::std::string`有着相同接口的`::string`。确实在[gtest-port.h](https://github.com/google/googletest/blob/master/googletest/include/gtest/internal/gtest-port.h#L1131-L1135)找到了定义，但是Google了一圈也没搞懂`::string`到底是个什么样的东东。

之后，声明了一系列运行时设置，包括
```cpp
bool FLAGS_GTEST_ALSO_RUN_DISABLED_TESTS
bool FLAGS_GTEST_BREAK_ON_FAILURE
bool FLAGS_GTEST_CATCH_EXCEPTIONS
string FLAGS_GTEST_COLOR
string FLAGS_GTEST_FILTER
bool FLAGS_GTEST_LIST_TESTS
string FLAGS_GTEST_OUTPUT
bool FLAGS_GTEST_PRINT_TIME
int32 FLAGS_GTEST_RANDOM_SEED
int32 FLAGS_GTEST_REPEAT
bool FLAGS_GTEST_SHOW_INTERNAL_STACK_FRAMES
bool FLAGS_GTEST_SHUFFLE
int32 FLAGS_GTEST_STACK_TRACE_DEPTH
bool FLAGS_GTEST_THROW_ON_FAILURE
string FLAGS_GTEST_STREAM_RESULT_TO
```

接下来是一些重要的数据类型的定义:
* 为存储`AssertionSuccess`和`AssertionSuccess`结果定义的`AssertionResult`。包含`bool success_`和`internal::scoped_ptr< ::std::string> message_;`，可以进行取反、转化为布尔值、取结果信息
* 基类`Test`是所有测试的核心，从属于一个`TestCase`，而一个单元测试程序至少包含一个`TestCase`。包含了一些列静态方法用于设置`TestCase`、统计失败和记录上下文属性
```cpp
 static void SetUpTestCase();
 static void TearDownTestCase();
 static bool HasFatalFailure();
 static bool HasNonfatalFailure();
 static bool HasFailure();
 static void RecordProperty(const std::string& key, const std::string& value);
 static void RecordProperty(const std::string& key, int value);
 ```

  各个扩展类负责具体建立、销毁测试环境，定义测试内容。`Test`提供判断环境是否变化与测试的执行序列。
  这里需要注意的是SetUp方法的命名，gtest定义了Setup从而当用户使用Setup时会得到一个解释非常详尽的错误。

* `TestProperty`用来存储测试过程中的相关属性，可以理解一个键值对
* `TestResult`用来存放测试结果，包括总共完成的部分数，每一部分的结果，总共的测试属性个数，具体每个属性为何，测试是否通过，测试耗时时长等等
* `TestInfo`用来记录`TestCase`名称、`Test`名称、`Test`是否应该被执行、用来创建`Test`对象的函数指针，以及`TestResult`
* `TestCase`用来管理一系列的`TestInfo`，一样包含类似`TestInfo`的相关信息，可以被继承
* `Environment`接口，用`SetUp`和`TearDown`而不是构造函数和析构函数来管理环境的建立与销毁
* `TestEventListener`接口，用来跟踪测试执行过程，按下列顺序触发

  ```cpp
  OnTestProgramStart
  OnTestIterationStart
  OnEnvironmentsSetUpStart
  OnEnvironmentsSetUpEnd
  OnTestCaseStart
  OnTestStart
  OnTestPartResult
  OnTestEnd
  OnTestCaseEnd
  OnEnvironmentsTearDownStart
  OnEnvironmentsTearDownEnd
  OnTestIterationEnd
  OnTestProgramEnd  
  ```
* `EmptyTestEventListener`是`TestEventListener`的一个空白实现：当只需要重写接口中某一个或某几个方法的时候，继承它而不是接口本身会更加方便。
* `TestEventListeners`用来管理所有的`TestEventListener`，其中包括默认的输出到控制台的`default_result_printer`、输出到XML文件的`default_xml_generator_`、广播转发事件给所有注册的`TestEventListener`的`repeater`。
* `UnitTest`是一个单例类，包含了一系列的`TestCase`。只要调用得当，那么这个类就是线程安全的。通过公开方法可以获取当前的工作目录，当前的`TestCase`，当前的`TestInfo`等等相关的各种信息。而私有部分暴露添加测试结果的方法，gtest提供的各种断言宏最终都会调用。核心的实现，则是通过pimpl模式实现的。

  在声明中，多次使用了空白的宏，比如`GTEST_LOCK_EXCLUDED_(mutex_)`从而起到注解作用。

在这之后是`testing::internal`命名空间的各种辅助函数模板，一系列的`IsSubString`全局函数，对数值参数化测试（Value Parameterized Tests）的支持，一系列生成结果的宏（`AddFailure`、`ADD_FAILURE_AT`，`GTEST_FAIL`、`GTEST_SUCCEED`），一系列的断言宏（`EXPECT_*`、`GTEST_ASSERT_*`）。

最后是整个gtest的核心定义
```cpp
#define GTEST_TEST(test_case_name, test_name)
#define TEST_F(test_fixture, test_name)

inline int RUN_ALL_TESTS()
```

至此gtest.h头文件就结束了。
