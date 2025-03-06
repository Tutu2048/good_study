## CMake学习

CMake是一个**构建系统生成器**。将描述构建系统(如：Unix Makefile、Ninja、Visual Studio等)应当如何操作才能编译代码。

**All Variables Are Strings**

**You Can Simulate a Data Structure using Prefixes**
cmake的变量可以嵌套定义定义

**Every Statement is a Command**

**Flow Control Commands**

**Lists are Just Semicolon-Delimited Strings**

**Functions Run In Their Own Scope; Macros Don’t**

**Including Other Scripts**

**Getting and Setting Properties**

---



1.现在为其设置一个默认值，同时也可以**从外部进行更改用一个选项替换上一个示例的`set(USE_LIBRARY OFF)`命令**。该选项将修改`USE_LIBRARY`的值，并设置其默认值为`OFF`：

`option(USE_LIBRARY "Compile sources into a library" OFF)`

2.具有依赖关系的变量

```cmake
include(CMakeDependentOption)	#加载依赖选项模块
cmake_dependent_option(<option> "<help_text>" <value> <depends> <force>) 
```

---

**!!!注意工具链架构，和所依赖的库是否匹配；x86的库 ，x64的编译器是找不到的！！**

---

#### 构建类型

1. **Debug**：用于在没有优化的情况下，使用带有调试符号构建库或可执行文件。
2. **Release**：用于构建的优化的库或可执行文件，不包含调试符号。
3. **RelWithDebInfo**：用于构建较少的优化库或可执行文件，包含调试符号。
4. **MinSizeRel**：用于不增加目标代码大小的优化方式，来构建库或可执行文件。

cmake内置编译参数标志：（大多数时候，**编译器有特性标示**。如GCC和Glang，其他供应商的编译器不确定是否会理解(如果不是全部)这些标志。如果项目是真正跨平台，那么这个问题就必须得到解决）
CMAKE_C_FLAGS_DEBUG、CMAKE_C_FLAGS_RELEASE、CMAKE_C_FLAGS_RELWITHDEBINFO、CMAKE_C_FLAGS_MINSIZEREL
CMAKE_CXX_FLAGS_DEBUG、CMAKE_CXX_FLAGS_RELEASE、CMAKE_CXX_FLAGS_RELWITHDEBINFO、CMAKE_CXX_FLAGS_MINSIZEREL



设置一个默认的构建类型(本例中是Release)，并打印一条消息。要注意的是，该变量被设置为缓存变量，可以通过缓存进行编辑：

```cmake
if(NOT CMAKE_BUILD_TYPE)    
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()
#效果约等于option
#option(<variable> "<help_text>" [value])
#option(USE_LIBRARY "Compile sources into a library" OFF)
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
```

-  `CACHE <type>`：变量的值被存储在 CMake 的缓存中

  - STRING :表示以字符串的形式，存入缓存
  - PATH ：以路径的方式，存入缓存

  - ...

  **简单来说**：变量缓存后，源码改变，重新cmake ，该变量依然存在，且可以使用

- `FORCE`：表示强制写入，即使缓存内该变量存在且有值。默认是不会修改缓存内变量的值的

---

#### 编译选项

常用默认变量flags

`flags +="-fPIC -Wall"`等价于`list(APPEND flags "-fPIC" "-Wall")`

```cmake
target_compile_options(compute-areas	#target
  PRIVATE	#可见性
    "-fPIC"	#参数
  )
```

可见性的含义如下:

- **PRIVATE**，编译选项会应用于给定的目标，**不会传递给与目标相关的目标**。我们的示例中， 即使`compute-areas`将链接到`geometry`库，`compute-areas`也不会继承`geometry`目标上设置的编译器选项。
- **INTERFACE**，不编译进程序，只在运行时调用时使用。
- **PUBLIC**，编译选项将应用于指定目标和使用它的目标。

*推荐为每个目标设置编译器标志。使用`target_compile_options()`不仅允许对编译选项进行细粒度控制，而且还可以更好地与CMake的更高级特性进行集成。*

---

#### 为语言设定标准

```cmake
#精准设置，只针对animal-farm
set_target_properties(animal-farm
  PROPERTIES
    CXX_STANDARD 14	#设置我们想要的标准
    CXX_EXTENSIONS OFF #告诉CMake只启用ISO C++标准的编译器标志，而不使用特定编译器的扩展
    CXX_STANDARD_REQUIRED ON #指定所选标准的版本。如果这个版本不可用，CMake将停止配置并出现错误。当这个属性被设置为OFF时，CMake将寻找下一个标准的最新版本，直到一个合适的标志。这意味着，首先查找C++14，然后是C++11，然后是C++98
  )
```

**TIPS**:*如果语言标准是**所有目标**共享的**全局属性**，那么可以将`CMAKE_<LANG>_STANDARD`、`CMAKE_<LANG>_EXTENSIONS`和`CMAKE_<LANG>_STANDARD_REQUIRED`变量设置为相应的值。所有目标上的对应属性都将使用这些设置。*

---

#### 循环控制流

`foreach-endforeach`用法

```cmake
foreach(loop_var arg1 arg2 ...)	#其中提供循环变量和显式项列表
foreach(loop_var range total)	#指定一个范围
foreach(loop_var range start stop [step]) #指定一个范围
foreach(loop_var IN LISTS [list1[...]])	#对列表值变量的循环 参数解释为列表，其内容就会自动展开。
foreach(loop_var IN ITEMS [item1 [...]])#对变量的循环，参数的内容没有展开。
```

//TODO:手打验证一下，各个方法的效果

---

#### 根据环境进行编译

##### 平台、操作系统

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")	#CMAKE_SYSTEM_NAME 内置变量
  target_compile_definitions(hello-world PUBLIC "IS_LINUX")#可以通过限定符PUBLIC来精细处理
endif()
add_definitions(-DIS_LINUX)	#整个项目  == cmake -D IS_LINUX=1
```

##### 编译器

```cmake
target_compile_definitions(hello-world PUBLIC "COMPILER_NAME=\"${CMAKE_CXX_COMPILER_ID}\"")
if(CMAKE_CXX_COMPILER_ID MATCHES GNU)	#CMAKE_CXX_COMPILER_ID 内置变量
  target_compile_definitions(hello-world PUBLIC "IS_GNU_CXX_COMPILER")
endif()
```

##### 架构ARCH（architecture）

32bit?or 64bit.

```cmake
if(CMAKE_SIZEOF_VOID_P EQUAL 8)	#CMake的CMAKE_SIZEOF_VOID_P变量,空指针大小
  target_compile_definitions(arch-dependent PUBLIC "IS_64_BIT_ARCH")
  message(STATUS "Target is 64 bits")
else()
  target_compile_definitions(arch-dependent PUBLIC "IS_32_BIT_ARCH")
  message(STATUS "Target is 32 bits")
endif()
```

处理器架构

```cmake
if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "i386")
    message(STATUS "i386 architecture detected")
elseif(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "i686")
    message(STATUS "i686 architecture detected")
elseif(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")
    message(STATUS "x86_64 architecture detected")
else()
    message(STATUS "host processor architecture is unknown")
endif()

target_compile_definitions(arch-dependent
  PUBLIC "ARCHITECTURE=${CMAKE_HOST_SYSTEM_PROCESSOR}"#CMake定义了CMAKE_HOST_SYSTEM_PROCESSOR变量，以包含当前运行的处理器的名称
  )
```

除了`CMAKE_HOST_SYSTEM_PROCESSOR`, CMake还定义了`CMAKE_SYSTEM_PROCESSOR`变量。前者包含当前运行的CPU在CMake的名称，而后者将包含当前正在为其构建的CPU的名称。这是一个细微的差别，在交叉编译时起着非常重要的作用。

---

#### 处理器指令集

```cmake
cmake_host_system_information(RESULT VAR QUERY KEY)#查询KEY，value存放在VAR
```

KEY如下:
`NUMBER_OF_LOGICAL_CORES`、`    NUMBER_OF_PHYSICAL_CORES`、`    TOTAL_VIRTUAL_MEMORY`、`    AVAILABLE_VIRTUAL_MEMORY`、`    TOTAL_PHYSICAL_MEMORY`、`   AVAILABLE_PHYSICAL_MEMORY`、`    IS_64BIT`、`    HAS_FPU`、`    HAS_MMX`、`    HAS_MMX_PLUS`、`    HAS_SSE`、`    HAS_SSE2`、`    HAS_SSE_FP`、`   HAS_SSE_MMX`、`    HAS_AMD_3DNOW`、`    HAS_AMD_3DNOW_PLUS`、`    HAS_IA64`、`    OS_NAME`、`   OS_RELEASE`、`    OS_VERSION`、`    OS_PLATFORM`......

##### 生成source文件

```cmake
configure_file(input_file output_file [COPYONLY] [ESCAPE_QUOTES] [@ONLY])
#例
configure_file(config.h.in config.h @ONLY)
set(PROJECT_VERSION "1.0")
#config.h.in
#define PROJECT_VERSION   @PROJECT_VERSION@
#------------configure_file之后--------------
#config.h
#define PROJECT_VERSION   1.0
```

- `COPYONLY` 是一个可选参数，如果指定，将不执行变量替换，直接复制文件。
- `ESCAPE_QUOTES` 是一个可选参数，用于转义输出文件中的双引号。
- `@ONLY` 是一个可选参数，如果指定，只替换 `@VAR@` 形式的变量，而不是 `$(VAR)` 形式。

---

https://www.bookstack.cn/read/CMake-Cookbook/content-chapter2-2.6-chinese.md

本节涉及：

⭐`find_package(Eigen3 3.3 REQUIRED CONFIG)` #**找库文件、头文件、编译器宏、连接器宏、CMake模块文件.cmake**

`check_cxx_compiler_flag("-march=native" _march_native_works)`

`CheckCXXCompilerFlag`

//TODO：由于没有Eigen无法通过cmake

---

## 三、检测外部库和程序

⭐`find_package`命令是该章核心命令，是用于发现和设置包的CMake模块的命令

CMake模块文件称为`Find<name>.cmake`，当调用`find_package(<name>)`时，模块中的命令将会运行。

`find_package`命令的文档可以参考 https://cmake.org/cmake/help/v3.5/command/find_ackage.html 

可以在终端中直接浏览文档，本例中可使用`cmake --help-module FindPythonInterp`



⭐**tips:**`find_package`在找包时，会生成很多 `<PackageName>_Variable`变量

- _FOUND
- _VERSION
- _INCLUDE_DIRS
- _INCLUDE_DIR
- ...

---

##### 检测Python解释器

```cmake
find_package(PythonInterp REQUIRED)
```

- REQUIRED	必须要找到包，否则退出cmake

**在python未安装到标准路径，可以在执行时定义**

`$ cmake -D PYTHON_EXECUTABLE=/custom/location/python ..`

CMake 3.12版本中增加了新的Python检测模块，旨在解决这个棘手的问题。我们`CMakeLists.txt`的检测部分也将简化为

 **`find_package(Python COMPONENTS Interpreter Development REQUIRED)`** 命令的详细说明：

- `Python`：这是您要查找的包的名称。CMake 社区提供了一个 `PythonLibs` 模块来查找 Python。
- `COMPONENTS`：这个参数指定了您需要的 Python 包的组件。在这里，使用了两个组件：
  - `Interpreter`：查找 Python 解释器可执行文件的路径。
  - `Development`：查找 Python 开发相关的头文件和库文件，这对于编译 Python 扩展模块是必需的。

**打印调试，可用下面代码，代替message命令**

```cmake
include(CMakePrintHelpers)
cmake_print_variables(_status _hello_world)
```



**find_package + target_link_libraries**是更加**现代且优秀**的管理路径和链接的方式。

- 用包管理工具安装库，且库支持现代cmake
- link时不知道第二个参数填什么可以：
  1. 官网看
  2. *Config.cmake里面找add_library 第一个就是需要填的Target ，当然在后续的子cmake模块里面可能会对其取别名。



---

## 四、创建和运行测试

##### 简单的单元测试

用cmake管理单元测试

```cmake
enable_testing()
add_test(
  NAME cpp_test
  COMMAND $<TARGET_FILE:cpp_test>
  )
```

 `$<TARGET_FILE:cpp_test>` ---生成器表达式---`$ctest` 执行的是cpp_test

- TODO:学习生成器表达式，生成器表达式，CMake生成器表达式是一种特殊的语法，用于在**CMake构建系统中动态地生成构建规则**，比较难


生成器表达式在测试时非常方便，因为不必显式地将可执行程序的位置和名称

```shell
#执行cmake中的测试程序
$ctest 
```

- `--output-on-failure`:将测试程序生成的任何内容打印到屏幕上，以免测试失败。
- `-v`:将启用测试的详细输出。
- `-vv`:启用更详细的输出。

CTest提供了一个非常方快捷的方式，可以重新运行以前失败的测试；要使用的CLI开关是`--rerun-failed`，在调试期间非常有用。

*`cmake --help-manual ctest`**会将向屏幕输出完整的ctest手册。*

---

#### 第三方测试库

##### catch2

Catch2是一个单头文件测试框架，所以不需要定义和构建额外的目标。只需要确保CMake能找到`catch.hpp`

https://github.com/catchorg/Catch2 

```cmake
# Prepare "Catch" library for other executables 
set(CATCH_INCLUDE_DIR
${CMAKE_CURRENT_SOURCE_DIR}/catch) 
add_library(Catch
INTERFACE) 
target_include_directories(Catch INTERFACE
${CATCH_INCLUDE_DIR})

***
***

target_link_libraries(cpp_test Catch)
```

`INTERFACE`关键字标识接口库，是CMake提供的伪目标库，你可以为依赖这个库的目标提供编译器选项、定义宏、包含目录、编译器依赖等，**但不会生成实际的二进制文件**（.a .so）

**优势：**使用接口库的好处是，它允许你集中管理Catch的头文件路径，而不需要在每个需要Catch的测试目标中重复设置包含目录。

一旦设置好这个接口库，你可以在项目的其他部分通过以下方式使用Catch：

`target_link_libraries(your_target Catch)`

这样即可找到catch2.hpp

---

##### Google Test库

Google Test框架不仅仅是一个头文件，也是一个库，包含两个需要构建和链接的文件。可以将它们与我们的代码项目放在一起，但是**为了使代码项目更加轻量级**，我们将选择在配置时，**下载**一个定义良好的**Google Test**，然后构建框架并链接它。

```cmake
include(FetchContent)
# 声明Google Test作为外部依赖
FetchContent_Declare(
    googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG release-1.8.0
)

# 确保所有内容都已下载
FetchContent_GetProperties(googletest)
if(NOT googletest_POPULATED)
  FetchContent_Populate(googletest)
  add_subdirectory(${googletest_SOURCE_DIR} ${googletest_BINARY_DIR})
endif()
```

cmake：**`FetchContent`**模块支持通过**`ExternalProject`**模块，在配置时填充内容，并在其**3.11**版本中成为CMake的标准部分。

---

##### boost Test

---

##### valgrind

Valgrind( [http://valgrind.org](http://valgrind.org/) )是一个通用的工具，用来检测内存缺陷和内存泄漏。

```cmake
#通过名字查找可执行程序valgrind 放入 MEMORYCHECK_COMMAND 变量中
find_program(MEMORYCHECK_COMMAND NAMES valgrind)
#设置valgrind执行参数
set(MEMORYCHECK_COMMAND_OPTIONS "--trace-children=yes --leak-check=full")
```

`$ctest -T memcheck`

----

#### 测试策略⭐

##### 超时测试

`set_tests_properties(example PROPERTIES TIMEOUT 10)`

example -- test Name

**PROPERTIES** 属性关键字

TIMEOUT 属性名（有点类似于模板类的T）10 s以秒为单位



##### **并行测试**

并行工作可以减少测试时间，虽然cpu运行的总时间并不会减少

`$ ctest --parallel 4` 会**按照add_test定义的顺序**进行测试，未用到cpu并行分配的策略

`set_tests_properties(a b c d PROPERTIES COST 0.5)`

属性名称COST ：设置耗时估计值，手动设置估计值后，根据耗时大小，采用cpu并行分配的策略，**优先开始耗时大的测试**



##### 运行测试子集

并行策略可以减少测试时间，提升测试效率，但对于大型项目，我们并不希望运行整个测试集。那么我们就需要对每一个测试单位进行**打标签**，即可以实现**分类测试**

```cmake
//打标签
set_tests_properties(
  feature-a
  feature-b
  feature-c
  PROPERTIES
      LABELS "quick"
  )
set_tests_properties(
  feature-d
  benchmark-a
  benchmark-b
  PROPERTIES
      LABELS "long"
  )
```

```shell
//分类测试
$ctest -I 2,4 #I id 运行编号2-4的测试
$ctest -L long	#L lable 根据标签 运行被打赏long标签的测试
$ctest -R feature	#R  根据名称搜索，运行名字匹配的测试
```

- add_test会根据定义顺序自动编号（从1开始）
- 搜索方式都是**模糊匹配** `$ctest -L on`  ->依旧能匹配到"long"标签



##### 测试固件

- 类似于new和delete 空间的创建和释放被智能指针所包揽，**测试固件--在每次对于测试之前进行设置，测试完之后完成对垃圾的清理和设置的回归**。

对于大型项目来说，运行一次测试不论是单元测试、分类测试还是全测试，都可能会产生一定运行痕迹（比如一些日志和设置的更改）。那就需要在**测试前**进行**垃圾清理和配置的设置**，而这种操作一般是繁琐且耗时的，这就可以利用测试固件。

```cmake
#将setup test作为my-fixture测试固件的设置部分
set_tests_properties(
  setup
  PROPERTIES
      FIXTURES_SETUP my-fixture 
  )
#将feature-a、feature-b两个作为my-fixture测试固件的实际测试部分 
set_tests_properties(
  feature-a	
  feature-b 
  PROPERTIES 
      FIXTURES_REQUIRED my-fixture 
  )
  
#将clean test作为my-fixture测试固件的清理部分
set_tests_properties(
  cleanup
  PROPERTIES
      FIXTURES_CLEANUP my-fixture
  )
```

- `FIXTURES_SETUP fixtureName  `设置步骤
- `FIXTURES_REQUIRED fixtureName` 实际测试
- `FIXTURES_CLEANUP fixtureName`清理步骤

`$ ctest -R feature-a`执行效果：

```shell
Start 1: setup
1/3 Test #1: setup ............................ Passed 0.01 sec
Start 2: feature-a
2/3 Test #2: feature-a ........................ Passed 0.00 sec
Start 4: cleanup
3/3 Test #4: cleanup .......................... Passed 0.01 sec
```

---

## 五、平台交互（即和操作系统交互）

<u>CLI命令行界面（Command Line）</u>

#### CMake内置命令

```cmake
add_custom_target(unpack-eigen #target name
  ALL #告诉 CMake 在每次构建时都执行这个自定义目标中的命令
  COMMAND
      ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_SOURCE_DIR}/eigen-eigen-5a0156e40feb.tar.gz #使用cmake内置命令 tar
  COMMAND
      ${CMAKE_COMMAND} -E rename eigen-eigen-5a0156e40feb eigen-3.3.4
  WORKING_DIRECTORY #设置该target的工作目录
      ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT
      "Unpacking Eigen3 in ${CMAKE_CURRENT_BINARY_DIR}/eigen-3.3.4"
  )
```

- `add_custom_target` 自定义构建目标
- `${CMAKE_COMMAND} -E` 即`$cmake -E`,-E选项表示在命令行使用CMake的内置命令
  - **文件和目录操作**：
    - `copy`：复制文件或目录。
    - `remove`：删除文件或目录。
    - `tar`：创建或提取 tar 压缩文件。
    - `rename`：重命名或移动文件或目录。
  - **系统操作**：
    - `environment`：查看或设置环境变量。
    - `echo`：输出文本到控制台。
    - `time`：测量命令的执行时间。
  - **其他操作**：
    - `md5sum`、`sha1`、`sha224`、`sha256`、`sha384`、`sha512`：计算文件的哈希值。
- `$cmake -E help`可以查看内置命令行命令的完整列表



显式地指定可执行目标对自定义目标的依赖关系

```cmake
add_dependencies(linear-algebra unpack-eigen)
```

`add_dependencies`不会改变目标本身的构建规则或命令，它只是确保在构建流程中，依赖目标会按照指定的顺序**先被处理**。

---

### 自定义命令

#### 配置时

```cmake
execute_process(
  COMMAND
      ${PYTHON_EXECUTABLE} "-c" "import ${_module_name}; print(${_module_name}.__version__)"
  OUTPUT_VARIABLE _stdout
  ERROR_VARIABLE _stderr
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_STRIP_TRAILING_WHITESPACE
  )
```

`execute_process`命令将从当前正在执行的CMake进程中**派生一个或多个子进程**，从而提供了在配置项目时运行任意命令的方法。

- WORKING_DIRECTORY，指定应该在哪个目录中执行命令。
- RESULT_VARIABLE将包含进程运行的结果。这要么是一个整数，表示执行成功，要么是一个带有错误条件的字符串。
- OUTPUT_VARIABLE和ERROR_VARIABLE将包含执行命令的标准输出和标准错误。由于命令的输出是通过管道传输的，因此只有最后一个命令的标准输出才会保存到OUTPUT_VARIABLE中。
- INPUT_FILE指定标准输入重定向的文件名
- OUTPUT_FILE指定标准输出重定向的文件名
- ERROR_FILE指定标准错误输出重定向的文件名
- 设置OUTPUT_QUIET和ERROR_QUIET后，CMake将静默地忽略标准输出和标准错误。
- 设置OUTPUT_STRIP_TRAILING_WHITESPACE，可以删除运行命令的标准输出中的任何尾随空格
- 设置ERROR_STRIP_TRAILING_WHITESPACE，可以删除运行命令的错误输出中的任何尾随空格。

#### 构建时

CMake提供了三个选项来在构建时执行自定义命令:

1. 使用`add_custom_command`编译目标，生成输出文件。
2. `add_custom_target`的执行没有输出。
3. 构建目标前后，`add_custom_command`的执行可以没有输出。

##### add_custom_command

```cmake
set(wrap_BLAS_LAPACK_sources
  ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxBLAS.hpp
  ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxBLAS.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxLAPACK.hpp
  ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxLAPACK.cpp
  )
add_custom_command(
  OUTPUT #该命令需要生成的目标
      ${wrap_BLAS_LAPACK_sources}
  COMMAND
      ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_SOURCE_DIR}/wrap_BLAS_LAPACK.tar.gz
  COMMAND
      ${CMAKE_COMMAND} -E touch ${wrap_BLAS_LAPACK_sources}
  WORKING_DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}
  DEPENDS
      ${CMAKE_CURRENT_SOURCE_DIR}/wrap_BLAS_LAPACK.tar.gz
  COMMENT
      "Unpacking C++ wrappers for BLAS/LAPACK"
  VERBATIM #逐字的，诉CMake为生成器和平台生成正确的命令，从而确保完全独立
  )
```

COMMAND 前缀，这些命令将在特定的时间执行

- **PRE_BUILD**：在执行与目标相关的任何其他规则之前执行的命令。
- **PRE_LINK**：使用此选项，命令在编译目标之后，调用链接器或归档器之前执行。Visual Studio 7或更高版本之外的生成器中使用`PRE_BUILD`将被解释为`PRE_LINK`。
- **POST_BUILD**：如前所述，这些命令将在执行给定目标的所有规则之后运行。



```cmake
#添加一个库，相较于add_library(math src1 src2 ...) 下面这种方法，更加直观，且可以通过PRIVATE、PUBLIC关键字来控制源文件的用途
add_library(math "")
target_sources(math
  PRIVATE
      ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxBLAS.cpp
      ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxLAPACK.cpp
  PUBLIC
      ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxBLAS.hpp
      ${CMAKE_CURRENT_BINARY_DIR}/wrap_BLAS_LAPACK/CxxLAPACK.hpp
  )

```

- `PRIVATE` 关键词指定的文件仅供这个库内部使用，不会通过 `PUBLIC` 或 `INTERFACE` 属性暴露给其他依赖这个库的目标。
- `PUBLIC` 关键词指定的文件可以被依赖这个库的其他目标所访问。
- target_link_libraries **优先链接动态库**

---

##### add_custom_target

由于add_custom_command存在一些缺陷：

- 只有在相同的`CMakeLists.txt`中，指定了所有依赖于其输出的目标时才有效。
- 对于不同的独立目标，使用`add_custom_command`的输出可以重新执行定制命令。这可能会导致冲突，应该避免这种情况的发生。

```cmake
add_custom_target(BLAS_LAPACK_wrappers
  WORKING_DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}
  DEPENDS
      ${MATH_SRCS}
  COMMENT
      "Intermediate BLAS_LAPACK_wrappers target"
  VERBATIM
  )
add_custom_command(
  OUTPUT
      ${MATH_SRCS}
  COMMAND
      ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_SOURCE_DIR}/wrap_BLAS_LAPACK.tar.gz
  WORKING_DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}
  DEPENDS
      ${CMAKE_CURRENT_SOURCE_DIR}/wrap_BLAS_LAPACK.tar.gz
  COMMENT
      "Unpacking C++ wrappers for BLAS/LAPACK"
  )
```

`add_custom_target`添加的目标没有输出，因此总会执行。因此，可以在子目录中引入自定义目标，并且仍然能够在主`CMakeLists.txt`中引用它

---

#### 探究编译与链接

对于大项目而言，依赖关系复杂，可以在cmake时，进行编译测试（编译器是否理解编译标志，依赖关系是否满足，进行测试）。

##### 1.如何确保代码能成功编译为可执行文件。

```cmake
set(_scratch_dir ${CMAKE_CURRENT_BINARY_DIR}/omp_try_compile)

#WAY ONE：try_compile
try_compile(
  omp_taskloop_test_1
  ${_scratch_dir}
  SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/taskloop.cpp
  LINK_LIBRARIES
    OpenMP::OpenMP_CXX
  )
message(STATUS "Result of try_compile: ${omp_taskloop_test_1}")

#WAY TWO：check_<lang>_source_compiles
include(CheckCXXSourceCompiles)
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/taskloop.cpp _snippet)#read "taskloop.cpp"'s content to _snippet 
set(CMAKE_REQUIRED_LIBRARIES OpenMP::OpenMP_CXX)
check_cxx_source_compiles("${_snippet}" omp_taskloop_test_2)
unset(CMAKE_REQUIRED_LIBRARIES)#卸载CMAKE_REQUIRED_LIBRARIES变量，以免影响到后续
message(STATUS "Result of check_cxx_source_compiles: ${omp_taskloop_test_2}")
```

TIPS：

- 检查并不总是万无一失的，并且可能产生假阳性和假阴性
- `try_compile`提供了灵活和干净的接口，特别是当编译的代码不是一个简短的代码时。我们建议在测试编译时，小代码片段时使用`check_<lang>_source_compile`。其他情况下，选择`try_compile`
- `$ cmake -U <variable-name>`将变量缓存清除
- 无论成功、失败，CMake也会删除由`try_compile`生成的所有文件，`cmake .. --debug-trycompile`将组织CMake进行删除

---

##### 2.如何确保编译器理解相应的标志。

```cmake
set(ASAN_FLAGS "-fsanitize=address -fno-omit-frame-pointer")

#设置CMAKE_REQUIRED_FLAGS变量。否则，作为第一个参数传递的标志将只对编译器使用。
set(CMAKE_REQUIRED_FLAGS ${ASAN_FLAGS})

#检测ASAN_FLAGS，结果放在asan_works
check_cxx_compiler_flag(${ASAN_FLAGS} asan_works)

unset(CMAKE_REQUIRED_FLAGS)                                         
if(asan_works) 
# 将ASAN_FLAGS替换空格 为 分号放入_asan_flags变量
  string(REPLACE " " ";" _asan_flags ${ASAN_FLAGS})
  
  add_executable(asan-example asan-example.cpp)
  
  target_compile_options(asan-example
    PUBLIC
      ${CXX_BASIC_FLAGS}
      ${_asan_flags}
    )
  target_link_libraries(asan-example
    PUBLIC
      ${_asan_flags}
    )
endif()

```



##### 3.如何确保特定代码能成功编译为运行可执行程序

`check_<lang>_source_runs`、`try_run`

```cmake
include(CheckCXXSourceRuns)

set(_test_uuid
  "
#include <uuid/uuid.h>

int main(int argc, char * argv[]) {
  uuid_t uuid;

  uuid_generate(uuid);

  return 0;
}
  ")
                                                                         
set(CMAKE_REQUIRED_LIBRARIES PkgConfig::UUID)
check_cxx_source_runs("${_test_uuid}" _runs)
unset(CMAKE_REQUIRED_LIBRARIES)

if(NOT _runs)
  message(FATAL_ERROR "Cannot run a simple C executable using libuuid!")
endif()

add_executable(use-uuid use-uuid.cpp)

target_link_libraries(use-uuid
  PUBLIC
    PkgConfig::UUID
  )
```

正如`check_<lang>_source_compiles`是`try_compile`的包装器，`check_<lang>_source_runs`是CMake中另一个功能更强大的命令的包装器:`try_run`。

---

##### **生成器表达式🔺**

- **逻辑表达式**，基本模式为`$<condition:outcome>`。基本条件为0表示false, 1表示true，但是只要使用了正确的关键字，任何布尔值都可以作为条件变量。
- **信息表达式**，基本模式为`$<information>`或`$<information:input>`。这些表达式对一些构建系统信息求值，例如：包含目录、目标属性等等。这些表达式的输入参数可能是目标的名称，比如表达式`$<TARGET_PROPERTY:tgt,prop>`，将获得的信息是tgt目标上的prop属性。
- **输出表达式**，基本模式为`$<operation>`或`$<operation:input>`。这些表达式可能基于一些输入参数，生成一个输出。它们的输出可以直接在CMake命令中使用，也可以与其他生成器表达式组合使用。例如,`- I$<JOIN:$<TARGET_PROPERTY:INCLUDE_DIRECTORIES>, -I>`将生成一个字符串，其中包含正在处理的目标的包含目录，每个目录的前缀由`-I`表示。

逻辑表达式常用于其他两个表达式的嵌套使用

```cmake
target_link_libraries(example
  PUBLIC
    $<$<BOOL:${MPI_FOUND}>:MPI::MPI_CXX>   
  )
```

tips：不要认准基本模式，因为生成器表达式很抽象，例如$<BOOL:${SOME_FEATURE} > 并不遵循基本模式`$<condition:outcome>`,

- `${SOME_FEATURE}` 是条件。它是一个 CMake 变量，其值用于评估真假。如果 `${SOME_FEATURE}` 变量被定义且不为 "0"（零）、空字符串或任何被视为假的值，那么条件就被认为是真的（true）。
- `BOOL` 是逻辑运算符，它将条件（变量的值）转换为布尔值。如果条件为真，`BOOL` 运算符返回 1；如果条件为假，返回 0。
- `$<BOOL:${SOME_FEATURE}>` 整个表达式根据 `${SOME_FEATURE}` 的值输出 0 或 1。这个输出可以用作生成器表达式的一部分，来决定是否包含某些编译选项或定义。

简单来说，条件是 `${SOME_FEATURE}` 变量的值，而 `BOOL` 运算符是用于决定这个条件是真还是假的工具。生成器表达式 `$<BOOL:${SOME_FEATURE}>` 的输出（0 或 1）基于条件变量的值。



在 `$<BOOL:${SOME_FEATURE}>` 的情况下，可以认为 `${SOME_FEATURE}` 是条件，而 `1`（或 `0`）是可能的结果。然而，这种表达式的简洁形式直接将条件和结果结合在一起，所以它不直接包含一个显式的 "outcome" 部分，而是直**接输出条件的布尔值。**这种形式的表达式通常用于与其他生成器表达式结合使用，以构建更复杂的条件逻辑。

---

## 六、生成源码

某些情况下，我们使用构建系统在配置或构建步骤时生成源代码。根据配置步骤中收集的信息，对源代码进行微调

##### 配置时（cmake ..）

```cmake
# $whoami > _user_name
execute_process(
  COMMAND
    whoami
  TIMEOUT
    1
  OUTPUT_VARIABLE
    _user_name
  OUTPUT_STRIP_TRAILING_WHITESPACE
  )

#查询主机名
cmake_host_system_information(RESULT _host_name QUERY HOSTNAME) 
#查询时间
string(TIMESTAMP _configuration_time "%Y-%m-%d %H:%M:%S [UTC]" UTC)

#`@`开头和结尾的字符串被替换，其余字符，生成 `print_info.c`
configure_file(print_info.c.in print_info.c @ONLY)

add_executable(example "")
target_sources(example
  PRIVATE
    main.c
    ${CMAKE_CURRENT_BINARY_DIR}/print_info.c
  )
#...
#print_info.c.in
#...
void print_info(void) {                                                     
  printf("Who compiled                | %s\n", "@_user_name@");
  printf("Compilation hostname        | %s\n", "@_host_name@");
  printf("Configuration time          | %s\n", "@_configuration_time@");
                                                              
  fflush(stdout);
  }
```

- **`${CMAKE_CURRENT_SOURCE_DIR}`**,与项目的根目录相

  **`${CMAKE_CURRENT_BINARY_DIR}`**,项目**构建目录**的位置,即xx_project/**cmake_build**

  **`${CMAKE_CURRENT_BINARY_DIR}`**，对于多级CMakeLists.txt，配置过程中，配置到哪个CMakeLists.txt，该path对应该文件的目录

- 用值替换占位符时，CMake中的变量名应该与将要配置的文件中使用的变量名完全相同，并放在`@`之间.输入和输出文件作为参数时，CMake不仅将配置`@VAR@`变量，还将配置`${VAR}`变量。如果`${VAR}`是语法的一部分，并且不应该修改(例如在shell脚本中)，那么就很不方便。

**TIPS**:*通过使用**`CMake --help-variable-list`**，可以从CMake手册中获得完整的内部CMake变量列表。*

---

通过python脚本替代`configure_file`(不重要)

局限性：需要程序员手动编辑，更花时间，不能完全模拟`configure_file()`，无法联动构建时的动态生成

---

##### 构建时（build:make）

```cmake
#创建一个目录
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/generated)
```

`file`根据关键字，对文件进行增删改查

官网：https://cmake.org/cmake/help/v3.5/command/file.html

note:`file(GLOB…)`在**配置时执行**，而代码生成是在**构建时发生的**。因此可能需要一个间接操作，将`file(GLOB…)`命令放在一个单独的CMake脚本中，使用`${CMAKE_COMMAND} -P`执行该脚本，以便在构建时获得生成的文件列表。

```cmake
#add_custom_command+python脚本，生成源码
add_custom_command(
  OUTPUT
      ${CMAKE_CURRENT_BINARY_DIR}/generated/primes.hpp
  COMMAND
      ${PYTHON_EXECUTABLE} generate.py ${MAX_NUMBER} ${CMAKE_CURRENT_BINARY_DIR}/generated/primes.hpp
  WORKING_DIRECTORY
      ${CMAKE_CURRENT_SOURCE_DIR}
  DEPENDS
      generate.py
  )
```

- `OUTPUT`当设置输出选项时，若**不存在输出的目标**，则会执行`COMMAND`
- `DEPENDS`当设置依赖时，若依赖文件**发生变化**，则会执行`COMMAND`

---

##### 版本记录

Git记录**源代码**的版本控制，获取githash，这同样可以用`configure_file`，生成头文件，源文件引用的方式实现

- `execute_process`模拟命令行`$git log -1 --pretty=format:%h`，**配置时**生成

  ```cmake
  # in case Git is not available, we default to "unknown"
  set(GIT_HASH "unknown")
  # find Git and if available set GIT_HASH variable
  find_package(Git QUIET)
  if(GIT_FOUND)
    execute_process(
      COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%h
      OUTPUT_VARIABLE GIT_HASH
      OUTPUT_STRIP_TRAILING_WHITESPACE
      ERROR_QUIET
      WORKING_DIRECTORY
          ${CMAKE_CURRENT_SOURCE_DIR}
    )
  endif()
  configure_file(version.hpp.in generated/version.hpp @ONLY)
  ```

​	:warning:：这一种方法有一个令人不满意的地方，如果在**配置代码（cmake ..）之后**更改分支或提交更改，则源代码中包含的版本记录可能指向错误的Git Hash值

- **构建时**生成：用`add_custom_command`模拟`$cmake -P xx.cmake`,xx.cmake脚本文件模拟命令行`$git log -1 --pretty=format:%h`

  ```cmake
  #将上面代码框内容放入git-hash.cmake，修改configure_file
  configure_file(
    ${CMAKE_CURRENT_LIST_DIR}/version.hpp.in
    ${TARGET_DIR}/generated/version.hpp
    @ONLY
    )
  
  #CMakeLists.txt...
  add_custom_command(
   OUTPUT
       ${CMAKE_CURRENT_BINARY_DIR}/generated/version.hpp
   ALL #表示每次构建的时候都会执行自定义命令中的`COMMAND`
   COMMAND
       ${CMAKE_COMMAND} -D TARGET_DIR=${CMAKE_CURRENT_BINARY_DIR} -P ${CMAKE_CURRENT_SOURCE_DIR}/git-hash.cmake
   WORKING_DIRECTORY
       ${CMAKE_CURRENT_SOURCE_DIR}
   )
  
  # rebuild version.hpp every time
  add_custom_target(
   get_git_hash
   ALL #表示get_git_hash为总是需要构建的目标，也是将get_git_hash目标加入到ALL目标中
   DEPENDS
       ${CMAKE_CURRENT_BINARY_DIR}/generated/version.hpp
   )
   
  # version.hpp has to be generated
  # before we start building example
  add_dependencies(example get_git_hash)
  ```

  **Notes：**

  - `cmake  -P <file>` = Process script mode   执行 脚本文件，以这种方式执行xx.cmake文件**不会修改`CMAKE_CURRENT_LIST_DIR`变量**。如果 `.cmake` 文件是作为模块通过 `include` 命令或 `find_package` 命令包含在 `CMakeLists.txt` 文件中，那么 `CMAKE_CURRENT_LIST_DIR` 将被设置为包含或调用该模块的 `CMakeLists.txt` 文件所在的目录。
    简单来说，还是**以`CMakeLists.txt` 文件为准**。
  - `ALL`属性关键字，在`add_custom_target`，`add_custom_command`意义不同，需要区别。
  - `ALL`目标，make的默认目标，`add_custom_target`中添加ALL关键字，即使不显示构建自定义目标get_git_hash，它也会构建，但不会保证构建顺序
  - `add_dependencies(example get_git_hash)`，添加依赖后，则得以保证
  - `git describe --abbrev=7 --long --always --dirty --tags`检测构建环境是否“污染”(即是否包含未提交的更改和未跟踪的文件)，或者“干净”

---

然而，不仅需要对源代码进行版本控制，而且**可执行文件**还需要记录项目版本，以便将其打印到代码输出或用户界面上。

- **CMakeLists.txt**

  ```cmake
  project(recipe-04 VERSION 2.0.1 LANGUAGES C)
  configure_file(version.h.in generated/version.h @ONLY)
  ```

- **额外文件 VERSION 保存版本信息**

  将版本保存在单独文件中的动机，是允许其他构建框架或开发工具使用独立于CMake的信息，而无需将信息复制到多个文件中

  ```cmake
  if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION")
      file(READ "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" PROGRAM_VERSION)
      string(STRIP "${PROGRAM_VERSION}" PROGRAM_VERSION)
  else()
      message(FATAL_ERROR "File ${CMAKE_CURRENT_SOURCE_DIR}/VERSION not found")
  endif()
  ```

---

## 七、构建项目

如何组合这些构建块，并引入抽象，并最小化代码重复、全局变量、全局状态和显式排序，以免CMakeLists.txt文件过于庞大。将讨论一些策略，也将帮助我们控制中大型代码项目的CMake代码复杂性。



##### 宏和函数

```cmake
#定义：
#macro(<name> [<arg1> ...])
#  <commands>
#endmacro()

macro(add_catch_test _name _cost)

  math(EXPR num_macro_calls "${num_macro_calls} + 1")#expression,调用后自动+1
  
  #宏自动填充了${ARGC}(参数数量)和${ARGV}(参数列表)
  message(STATUS "add_catch_test called with ${ARGC} arguments: ${ARGV}")
  
  #------------que  1 start-------------
  set(_argn "${ARGN}")
  if(_argn)
      message(STATUS "oops - macro received argument(s) we did not expect: ${ARGN}")
  endif()
  #------------que  1 end--------------
  
  add_test(
    NAME
      ${_name}
    COMMAND
      $<TARGET_FILE:cpp_test>
    [${_name}] --success --out
    ${PROJECT_BINARY_DIR}/tests/${_name}.log --durations yes
    WORKING_DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}
    )
  set_tests_properties(
    ${_name}
    PROPERTIES
        COST ${_cost}
    )
endmacro()

set(num_macro_calls 0)
```

- `COMMAND $<TARGET_FILE:cpp_test>`：指定测试用例执行的命令。`$<TARGET_FILE:cpp_test>`是一个CMake的生成器表达式，它将展开为目标 `cpp_test` 的可执行文件路径。这意味着测试将运行名为 `cpp_test` 的程序。
  - `--success`：可能用于指示程序在测试成功后采取的某些行为。
  - `--out`：后面跟着的是输出日志文件的路径，这里是 `${PROJECT_BINARY_DIR}/tests/${_name}.log`，表示测试结果将被写入到项目二进制目录下的 `tests` 子目录中，文件名为测试用例名称加上 `.log` 后缀。
  - `--durations yes`：指示程序输出每个测试用例的持续时间。
- **`ARGN`**变量存储额外参数，对于额外参数的判断**que 1**  ：这个`if`语句中，引入一个新变量，但不能直接查询`ARGN`，因为它**不是通常意义上的CMake变量**。
- 让耗时长的测试在耗时短测试之前启动，这要归功于**`COST`**属性。



**TIPS：**可以用一个函数来实现上述操作

```cmake
function(add_catch_test _name _cost)
    ...
endfunction()
```

宏和函数之间的区别在于它们的变量范围。**宏在调用者的范围内执行**，而**函数有自己的变量范围**。换句话说，如果我们使用**宏**，需要**设置或修改**对调用者可用的**变量**。如果不去设置或修改输出变量，最好使用函数。我们注意到，可以在函数中修改父作用域变量，但这必须使用`PARENT_SCOPE`显式表示

`set(variable_visible_outside "some value" PARENT_SCOPE)`



```cmake
#.
#设置输出目录
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
#CMAKE_INSTALL_LIBDIR、CMAKE_INSTALL_BINDIR 有默认值
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

#src/CMakeLists
set(CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE ON) 
```

细节：**CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE**，开启该选项，该CMakeLists内的所有target作为接口，供其他target使用。简单来说，其他目标无需再`target_include_directory`添加这份CMakeLists的头文件了，减少许多重复操作

---

##### .cmake模块包含宏

前面对于宏的定义和调用都放置在了一个CMakeLists中，推荐的做法是在模块中定义宏或函数，然后调用宏或函数，让调用方更加清晰和简单。



一个简单的彩色输出示例：

```cmake
#cmake/colors.cmake
macro(define_colors){
  	set(ColourReset "${Esc}[m")
  	set(Red "${Esc}[31m")
}

#CMakeLists

#list告诉CMake去哪里查找宏，include指示CMake搜索${CMAKE_MODULE_PATH}，查找名称为colors.cmake的模块
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(colors)
#调用
define_colors()
message(STATUS "${Red}This is a red${ColourReset}")
```

也可以显示地`include(cmake/colors.cmake)`模块

实际的`include`命令不应该定义或修改变量，其原因是重复的`include`(可能是偶然的)不应该引入任何不想要的副作用

优雅习惯：除了定义函数和宏以及查找程序、库和路径之外，包含模块不应该做更多的事情

---

##### 用指定参数定义函数或者宏

上面我们用到了`ARGN`变量，现在可以结合`cmake_parse_arguments`函数，对输入的参数进行选择性的使用。

```cmake
#cmake_parse_arguments(<PREFIX> <OPTIONS> <ONE_VALUE_KEYWORDS> <MULTI_VALUE_KEYWORDS> args...)
function(add_catch_test)
  set(options)
  set(oneValueArgs NAME COST)
  set(multiValueArgs LABELS DEPENDS REFERENCE_FILES)
  cmake_parse_arguments(add_catch_test
    "${options}"
    "${oneValueArgs}"                                    
    "${multiValueArgs}"
    ${ARGN}
    )
message(STATUS "    NAME: ${add_catch_test_NAME}")
message(STATUS "    LABELS: ${add_catch_test_LABELS}")
message(STATUS "    COST: ${add_catch_test_COST}")
message(STATUS "    REFERENCE_FILES: ${add_catch_test_REFERENCE_FILES}")
endfunction()
```

- `<PREFIX>` 是一个前缀，用于区分解析后的变量名。
- `<OPTIONS>` 是一个选项列表，这些选项是布尔值，如果指定了这些选项，相应的变量会被设置为 TRUE。
- `<ONE_VALUE_KEYWORDS>` 是一个关键字列表，每个关键字对应一个单一值的参数。
- `<MULTI_VALUE_KEYWORDS>` 是一个关键字列表，每个关键字可以对应多个值。

注：

- 解析后的变量名=前缀+解析前的变量名。
- cmake中列表是通过 **分号；** 来区分的
- 当你只需要多值参数时，也不**能省略cmake_parse_arguments前面的输入参数**，需要用“”空值来占位

---

##### 包含保护

​	对于一个项目而言很多头文件可能会反复包含，我们可以利用`#pragma`、`#ifdef`来避免重复包含，而对于cmake的模块来说，自然也有这样的机制---“包含保护”机制（min version 3.10）

```cmake
#当然我们可以兼容3.10以下版本的cmake，将其包含在一个宏中
macro(include_guard)
  if (CMAKE_VERSION VERSION_LESS "3.10")
    message(STATUS "calling our custom include_guard")

    if(NOT DEFINED included_modules)
        set(included_modules)
    endif()
	
	#CMAKE_CURRENT_LIST_FILE当前list文件,cmake文件可以视为list文件
    if ("${CMAKE_CURRENT_LIST_FILE}" IN_LIST included_modules)
      message(WARNING "module ${CMAKE_CURRENT_LIST_FILE} processed more than once")
    endif()

    list(APPEND included_modules ${CMAKE_CURRENT_LIST_FILE})
  else()
    message(STATUS "calling the built-in include_guard")

    _include_guard(${ARGV})
  endif()
endmacro()     
```

**注意：**cmake version>= 3.10时，cmake有内置的`include_guard`函数，上述操作涉及到了**cmake的重写机制**（不论是自定义的，还是内置的函数和宏），而`_include_guard(${ARGV})`指向的时内置的`include_guard`函数

---

##### 废弃

“废弃”是在不断发展的项目开发过程中一种重要机制，它向开发人员发出信号，表明将来某个函数、宏或变量将被删除或替换。在一段时间内，函数、宏或变量将继续可访问，但会发出警告，最终可能会上升为错误。



```cmake
macro(somemacro)
  message(DEPRECATION "somemacro is deprecated")
  _somemacro(${ARGV})
endmacro()

function(deprecate_variable _variable _access)
  if(_access STREQUAL "READ_ACCESS")
      message(DEPRECATION "variable ${_variable} is deprecated")
  endif()
endfunction()
variable_watch(included_modules deprecate_variable)#监控included_modules变量的变化，如果变化调用
```

- 实际操作，就是利用了上面提到的重写机制，调用它的同时，仅仅加上message，就达到了警告、暂时可用、不立刻废除的效果。
- If `<command>` is given, this command will be executed instead. The command will receive the following arguments: `COMMAND(<variable> <access> <value> <current_list_file> <stack>)`  ，variable_watch按顺序为function提供参数

---

##### 限定范围

经过前面的学习，我们已经能够很好的构建项目了。下面还有一些限定范围技巧来，减少构建时的风险。

1. **add_subdirectory**

   cmake模块文件和子目录的CMakeLists.txt，都可以视为list文件，但是两个文件处理的方式不一样，cmake模块文件通过`include`，而CMakeLists.txt 常常通过`add_subdirectory`来找到。

   文件在主`CMakeLists.txt`中被引用。这意味着使用`CMakeLists.txt`文件，构建我们的项目。这种方法对于许多项目来说是可用的，并且它可以扩展到更大型的项目，而不需要在目录间的全局变量中包含源文件列表。`add_subdirectory`方法的另一个好处是它**隔离了作用范围**，因为子目录中定义的变量在父范围中不能访问。

   

   除了程序员自身对项目的管理，也可以增加一些提示来帮助构建，将构建目录和源文件目录区分开，保持源代码树和构建树是分开的，是一个很基础，也很重要的方式。可以：

   ```cmake
   if(${PROJECT_SOURCE_DIR} STREQUAL ${PROJECT_BINARY_DIR})
       message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there.")
   endif()
   ```

   CMake可以使用**Graphviz图形**可视化软件([http://www.graphviz.org](http://www.graphviz.org/) )生成项目的依赖关系图:

   ```shell
   $ cd build
   $ cmake --graphviz=example.dot ..
   $ dot -T png example.dot -o example.png
   ```

   ![7.7 add_subdirectory的限定范围 - 图2](https://static.sitestack.cn/projects/CMake-Cookbook/images/chapter7/7-7-2.png)

   注：cmake 3.12引入**OBJECT**的库类型，对象库--源文件将被编译成目标文件：既不存档到静态库中，也不链接到动态库中。优势有：

   - 避免重复编译，一份源文件仅编译一次
   - 减少编译依赖：当使用对象库时，如果源文件没有变化，即使依赖于这些源文件的可执行文件或库需要重新构建，这些源文件也不会被重新编译。这可以提高增量构建的效率。

2. **target_sources**

   在前面的学习中我们已经用到了`target_sources`这个函数，而它可以帮助我们减少全局变量，从而缩小变量的作用范围，减少风险。

   在顶层CMakeLists.txt我们设置应用等一系列设置，而对应应用所依赖的库和源文件的编译可以交给下一层的模块CMakeLists.txt。

   ```cmake
   #project ./CMakeLists.txt	
   #...
   add_executable(automata "")
   include(src/CMakeLists.txt)
   #...
   
   #./src/CMakeLists.txt
   target_sources(automata 
   	PRIVATE	
   		${CMAKE_CURRENT_LIST_DIR}/initial.cpp
   	PUBLIC
       	${CMAKE_CURRENT_LIST_DIR}/parser.hpp
   )
   ```

   这样对initial.cpp来说直接编译到了automata应用中，不需要再添加编译成库，来供automata链接。

   **tips：**对于可见性，常用private cpp文件，开放public h头文件，这里的权限是指--是否仅用于构建目标本身，是否还需要用于构建目标依赖的编译

   **注：**这种方式减少了目标的生成，变相减少了全局变量的增加，但

   - 对于一些重用性较高的源文件，就意味着多次编译，这里可以考虑使用对象库来提升编译效率。
   - 对于源文件路径尽量使用绝对路径，可以避免找不到，因为在实际生成目标的时候，目录是以包含`add_executable`命令的上层CMakeLists.txt为准

==TODO:了解对象库==

---

## 八 超级构建模式

依赖项缺失怎么办？当不满足依赖关系，我们只能使配置失败，并向用户警告失败的原因，目前为止我们一直使用这种模式。但我们会发现，市面上很多项目，它是能够自动去下载缺失的依赖项的，这样的方式也更加人性化。然而，使用CMake可以组织我们的项目，如果在系统上找不到依赖项，就可以自动获取和构建依赖项。

- `ExternalProject.cmake`---在**构建时**检索项目的依赖项

- `FetchContent.cmake`---在**配置时**检索依赖项(CMake的3.11版本后添加)

通过上面两个标准模块，使用超级构建模式，我们可以利用CMake作为包管理器：相同的项目中，将以相同的方式处理依赖项，无论依赖项在系统上是已经可用，还是需要重新构建。

官网参考文档：https://cmake.org/cmake/help/latest/module/ExternalProject.html

https://cmake.org/cmake/help/latest/module/FetchContent.html



##### ExternalProject模块

`ExternalProject_Add`是该节最重要的命令，有许多选项，可用于外部项目的配置和编译等所有方面。这些选择可以分为以下几类:

- **Directory**：它们用于调优源码的结构，并为外部项目构建目录。

  通过设置`EP_BASE`目录属性，CMake将按照以下布局为各个子项目设置所有目录

  - `TMP_DIR = <EP_BASE>/tmp/<name>`
  - `STAMP_DIR = <EP_BASE>/Stamp/<name>`
  - `DOWNLOAD_DIR = <EP_BASE>/Download/<name>`
  - `SOURCE_DIR = <EP_BASE>/Source/<name>`
  - `BINARY_DIR = <EP_BASE>/Build/<name>`
  - `INSTALL_DIR = <EP_BASE>/Install/<name>`

- **Update**和**Patch**：可用于定义如何更新外部项目的源代码或如何应用补丁。

- **Download**：外部项目的代码可能需要从在线存储库或资源处下载。

- **Configure**：默认情况下，CMake会假定外部项目是使用CMake配置的。可以通过这些参数进行传递：

  - CMAKE_ARGS：命令行参数直接传递
  - CMAKE_CACHE_ARGS：通过CMake脚本文件传递

- **Build**：可用于调整外部项目的实际编译

- **Install**：这些选项用于配置应该如何安装外部项目。

- **Test**：测试是否正确安装，正常使用

---



1. **本地编译依赖项目**

   ```cmake
   set_property(DIRECTORY PROPERTY EP_BASE ${CMAKE_BINARY_DIR}/subproject
   
   include(ExternalProject)                              
                                                         
   ExternalProject_Add(${PROJECT_NAME}_core                              
     SOURCE_DIR
       ${CMAKE_CURRENT_LIST_DIR}/src
     CMAKE_ARGS
       -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
       -DCMAKE_CXX_STANDARD=${CMAKE_CXX_STANDARD}
       -DCMAKE_CXX_EXTENSIONS=${CMAKE_CXX_EXTENSIONS}
       -DCMAKE_CXX_STANDARD_REQUIRED=${CMAKE_CXX_STANDARD_REQUIRED}
     CMAKE_CACHE_ARGS
       -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
     BUILD_ALWAYS
       1 #每次编译都会编译
     INSTALL_COMMAND
       ""
     )
   #检索外部项目的属性
   ExternalProject_Get_Property(${PROJECT_NAME}_core CMAKE_ARGS)
   message(STATUS "CMAKE_ARGS of ${PROJECT_NAME}_core ${CMAKE_ARGS}")
   ```

   注：外部项目的属性是在首次调用`ExternalProject_Add`命令时设置的。

   ==TODO:ExternalProject_Add_Step、ExternalProject_Add_StepTargets、ExternalProject_Add_StepDependencies这三个函数能够提供更细致化的配置==

   ---

   

2. **联网获取依赖项目（自定义构建系统）**

   ```cmake
     include(ExternalProject)
     ExternalProject_Add(boost_external
       URL
         https://sourceforge.net/projects/boost/files/boost/1.61.0/boost_1_61_0.zip
       URL_HASH #保证下载完整性
      SHA256=02d420e6908016d4ac74dfc712eec7d9616a7fc0da78b0a1b5b937536b2e01e8
       DOWNLOAD_NO_PROGRESS #禁止打印下载进度信息
         1
       UPDATE_COMMAND
         ""
       CONFIGURE_COMMAND #配置编译环境
         <SOURCE_DIR>/bootstrap.sh #<SOURCE_DIR> 源目录的path
         --with-toolset=${_toolset}
         --prefix=${STAGED_INSTALL_PREFIX}/boost
         ${_bootstrap_select_libraries}
       BUILD_COMMAND #编译构建
         <SOURCE_DIR>/b2 -q
              link=shared
              threading=multi
              variant=release
              toolset=${_toolset}
              ${_b2_select_libraries}
       LOG_BUILD #编译过程写入日志文件
         1
       BUILD_IN_SOURCE #在源目录构建
         1
       INSTALL_COMMAND 
         <SOURCE_DIR>/b2 -q install
              link=shared
              threading=multi
              variant=release
              toolset=${_toolset}
              ${_b2_select_libraries}
       LOG_INSTALL #安装过程写入日志文件
         1
       BUILD_BYPRODUCTS #允许ExternalProject_Add在后续构建中，跟踪新构建的Boost库
         "${_build_byproducts}"
       )
   ```

   - `BUILD_IN_SOURCE`默认为off，会在源目录外创建一个构建目录，用于构建

   - ⭐`BUILD_BYPRODUCTS`：指定构建过程中生成的中间或最终产物的路径，可以做到：

     - **依赖性管理**：CMake 可以更准确地确定何时需要重新构建外部项目，基于这些生成的文件是否存在。
     - **增量构建**：如果外部项目的源代码没有变化，但是构建产物丢失或损坏，CMake 可以使用 `BUILD_BYPRODUCTS` 来检测这一点，并触发重新构建。
     - **避免构建**：如果构建产物已经存在，并且是最新的，CMake 可以避免不必要的构建步骤，从而节省时间。

     简单来说，就是编译出的库，作为整个编译过程的中间产物，被后续的构建所依赖时，即可使用

   - `CMAKE_BINARY_DIR`：当前运行的目录，**构建目录常用**这个变量为基础，做到构建、源目录隔离

   - `CMAKE_CURRENT_LIST_DIR`:当前CMakeLists目录，常用作**查看源文件**的基础

   - `<SOURCE_DIR>`:是ExternalProject_Add中特有的占位符，可以替换成cmake标准的`${SOURCE_DIR}`进行替换，当然这时候需要加上“ ”

   - ⭐`CONFIGURE_COMMAND`：当不需要cmake配置环境，且不需要特殊配置时，请设置为空（已踩坑）

   ---

   

3. **联网获取依赖项目（CMake构建系统）**

   ```cmake
   ExternalProject_Add(fftw3_external
       URL
         http://www.fftw.org/fftw-3.3.8.tar.gz
       URL_HASH
         MD5=8aac833c943d8e90d51b697b27d4384d
       DOWNLOAD_NO_PROGRESS
         1
       UPDATE_COMMAND
         ""
       CMAKE_ARGS
         -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
         -DBUILD_TESTS=OFF
       CMAKE_CACHE_ARGS
         -DCMAKE_C_FLAGS:STRING=$<$<BOOL:WIN32>:-DWITH_OUR_MALLOC>
       )
   ```

   如果是cmake来管理的依赖项目，就会很简单。configure、build、install都可以交由CMake自动完成，只需要传递相应参数即可。

   

   可以在可执行文件的CMakeLists，通过find_package来进行依赖检测，进一步保障依赖关系。

   ```cmake
   find_package(FFTW3 CONFIG REQUIRED)#CONFIG模式，告诉CMAKE通过使用 Find<PackageName>.cmake 或 <PackageName>Config.cmake 文件来查找包
   get_property(_loc TARGET FFTW3::fftw3 PROPERTY LOCATION)
   message(STATUS "Found FFTW3: ${_loc} (found version ${FFTW3_VERSION})")
   ```



`ExternalProject_Add`的一些选项：

- **GIT_REPOSITORY**：定包含依赖项源的存储库的URL，通过git clone的方式下载依赖项目
- **GIT_TAG**：默认情况下，CMake将检出给定存储库的默认分支，常填写版本和分支信息
- 测试命令：**TEST_COMMAND**
  - **TEST_AFTER_INSTALL**：依赖项很可能有自己的测试套件，您可能希望运行测试套件，以确保在超级构建期间一切顺利。此选项将在安装步骤之后立即运行测试
  - **TEST_BEFORE_INSTALL**：将在安装步骤之前运行测试套件
  - **TEST_EXCLUDE_FROM_MAIN**：

---

##### FetchContent模块

```cmake
include(FetchContent)
#声明第三方库，并确定获取路径、版本
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git           
  GIT_TAG        release-1.8.0
)

FetchContent_GetProperties(googletest)
if(NOT googletest_POPULATED)
  FetchContent_Populate(googletest)#获取并构建
  add_subdirectory(	#增加目录
    ${googletest_SOURCE_DIR}
    ${googletest_BINARY_DIR}
endif()
```

可以发现我们在获取构建时，只需要`FetchContent_Populate`这一个函数，然后增加googletest路径作为项目的子目录，非常简单。
我们还可以使用`FetchContent_MakeAvailable(googletest)`下载googletest，使得下载和编译分离。

注：

- `FetchContent_Populate`不会检查是否已经存在、已经编译，所以需要用if包裹起来，
- `FetchContent_MakeAvailable`先检查依赖是否已经构建完成，不会重复构建
- `<PackageName>_SOURCE_DIR`：指向外部项目的源代码的目录`<PackageName>_BINARY_DIR`：使用了 `add_subdirectory`，这个变量会指向构建外部项目的二进制目录。这些**变量都在`FetchContent_GetProperties`时已经获取**

**区别​：**

|          | FetchContent                                                 | ExternalProject                                              |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 集成方式 | 直接将外部项目的源代码作为**子模块获取到你的构建目录**中，然后作为子目录添加到你的项目中。 | 可以指定详细的下载、更新、配置、构建和安装步骤，包括使用系统外部的构建系统。 |
| 版本适用 | 3.11以上                                                     | 2.8以上                                                      |
| 时机     | 配置时                                                       | 编译时                                                       |
| 适用场景 | 适用于大多数现代 **CMake 项目**，特别是当外部项目提供了 CMake 构建系统时，很好用也很简单。 | 适用于需要详细控制外部项目构建过程的场景，即使用**非 CMake** 构建系统的项目或需要**复杂构建步骤**的项目。 |

FetchContent模块将很多依赖项的配置和其他设置都交由CMake自动处理，封装起来，只暴露了几个简单的接口

---

## 九 混合语言编译

我们经常需要利用一些专业库来解决一些专业问题，而这些专业库很可能与你正在编写代码的语言并不一致。每种语言有自己的解释方式，这时候需要一些额外的库（翻译）来，帮助接口的兼容实现“交流”，也就是调用。

当然就像人可能会多种语言一样，计算机语言本身也会提供一些对其他的语言的了解，正如C/C++库对于Fortran语言（科学）的兼容。而在编译过程中，也需要这种语言的转变，CMake也提供了相应的管理的方法





==TODO:学习==

在，**C/C++和解释语言Python**正占据着语言中心舞台。将编译语言代码与解释语言的代码集成在一起，变得确实越来越普遍，这样做有以下好处:

- 用户可以需要进行定制和扩展功能，以满足需求。
- 可以将Python等语言的表达能力与编译语言的性能结合起来，后者在内存寻址方面效率接近于极致，达到两全其美的目的。

---

## 十 、安装

对于一个对外的项目而言，项目的代码的布局和安装都是需要被用户所看见的，一个优秀的源码布局和安装流程是很重要的。



##### 安装设置流程

1. 对**`CMAKE_INSTALL_PREFIX`**进行设置，该变量默认值--Windows上的`C:\Program Files`和Unix上的`/usr/local`，这里未改变默认路径。注：后续安装命令内使用的是若不已/根目录开始，就是相对路径，相对的正是`CMAKE_INSTALL_PREFIX`，使用相对路径也是推荐做法

   ```cmake
   message(STATUS "Project will be installed to ${CMAKE_INSTALL_PREFIX}")
   ```

2. 使用标准CMake的`GNUInstallDirs.cmake`模块，告诉CMake在何处构建可执行、静态和动态库目标。便于在用户**不打算安装项目**的情况下，访问这些构建目标

   ```cmake
   include(GNUInstallDirs)
   #≈  xx/build/bin  xx/build/lib
   set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY
       ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
   set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
       ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
   set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
       ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
   ```

3. **安装的标准路径**设置，可以和非安装项目采用一致的布局，区别就是`PROJECT_BINARY_DIR`和`CMAKE_INSTALL_PREFIX`的**前缀的区别**。

   ```cmake
   # Offer the user the choice of overriding the installation directories
   set(INSTALL_LIBDIR ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
   set(INSTALL_BINDIR ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for executables")
   set(INSTALL_INCLUDEDIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files")
   if(WIN32 AND NOT CYGWIN)
       set(DEF_INSTALL_CMAKEDIR CMake)
   else()
       set(DEF_INSTALL_CMAKEDIR share/cmake/${PROJECT_NAME})
   endif()
   set(INSTALL_CMAKEDIR ${DEF_INSTALL_CMAKEDIR} CACHE PATH "Installation directory for CMake files")
   ```

   注：

   - `CMAKE_INSTALL_LIBDIR`、`CMAKE_INSTALL_BINDIR`、`CMAKE_INSTALL_INCLUDEDIR`是CMake的内置变量，和`CMAKE_INSTALL_PREFIX`一起为项目提供安装的默认值。这里仅是做复制操作，当然也通过这种用新变量承接内置变量内容，修改自定义变量的方式，防止污染内置变量也是不错的选择

     *https://cmake.org/cmake/help/v3.6/module/GNUInstallDirs.html*详细 GNUInstallDirs的内置变量

   - `DEF_INSTALL_CMAKEDIR`是自定已的cmake安装路径，意为存放`Config.cmake`：包含库的配置信息，**用于 `find_package()` 命令**。`Targets.cmake`：包含目标链接信息，用于现代的 CMake 项目依赖管理。`Version.cmake`：包含库的版本信息。CMake**没有内置变量提供默认路径**，不强制设置，而这是遵循 CMake 社区推荐的最佳实践之一。

   可以做一个简单的展示：

   ```cmake
   # Report to user
   foreach(p LIB BIN INCLUDE CMAKE)
     file(TO_NATIVE_PATH ${CMAKE_INSTALL_PREFIX}/${INSTALL_${p}DIR} _path )
     message(STATUS "Installing ${p} components to ${_path}")
     unset(_path)
   endforeach()
   ```

   :wind_face:  

   - file({TO_CMAKE_PATH | TO_NATIVE_PATH} <path> <out-var>)作用是将路径转化成cmake格式|当前系统的本地路径，存入out-var变量中。

   - 在循环内`unset`，限制作用范围是一个好习惯

   

   **运行时路径设置** :warning:

   设置该路径的原因是因为，某些依赖库非标准位置，设置运行时路径以免找不到

   ```cmake
   # Prepare RPATH
   file(RELATIVE_PATH _rel ${CMAKE_INSTALL_PREFIX}/${INSTALL_BINDIR} ${CMAKE_INSTALL_PREFIX})
   if(APPLE)
     set(_rpath "@loader_path/${_rel}")
   else()
     set(_rpath "\$ORIGIN/${_rel}")
   endif()
   file(TO_NATIVE_PATH "${_rpath}/${INSTALL_LIBDIR}" message_RPATH)
   
   ```

   :cake:file(RELATIVE_PATH <out-var> <directory> <file>)------计算从directory到file的相对路径，并存入 out-var变量中

   tip：`\$ORIGIN`，该特殊变量可以在非mac地址表示可执行文件的所在位置

   解决方案：构建树和安装目录中的可执行文件设置了不同的`RPATH`，因此它总是指向“有意义”的位置

4. 安装

   ```cmake
   install(
     TARGETS
       message-shared                              
       hello-world_wDSO
     ARCHIVE
       DESTINATION ${INSTALL_LIBDIR}
       COMPONENT lib
     RUNTIME
       DESTINATION ${INSTALL_BINDIR}
       COMPONENT bin
     LIBRARY
       DESTINATION ${INSTALL_LIBDIR}
       COMPONENT lib
     PUBLIC_HEADER
       DESTINATION ${INSTALL_INCLUDEDIR}/message
       COMPONENT dev
     )
   ```

   - 关键字COMPONENT 效果类似于一个标签，当在命令行

     `$ cmake -D COMPONENT=lib -P cmake_install.cmake`
     就可以仅安装COMPONENT为lib的组件

     注：cmake_install.cmake在cmake配置时生成

   - 我们会发现，这里对静态库、运行时目录、动态库、头文件目录进行了设置，前面设置的`DEF_INSTALL_CMAKEDIR`变量并没有被使用，可以对cmake配置文件单独设置：

   ```cmake
   install(
   	FILES
   		MyProjectConfig.cmake 
   	DESTINATION 
   		${DEF_INSTALL_CMAKEDIR})
   ```

   

总结：简单来说，也可以区分为**路径设置和安装指令**两个阶段

注：可以通过`set_target_properties`命令修改目标属性，来区分构建路径和安装路径，设置是否开启运行路径的属性，等很多操作
https://cmake.org/cmake/help/v3.6/manual/cmake-properties.7.html可以在这查询。



---

##### 共享库构建

为减少库的大小并提高链接器优化的机会，**构建共享库**时推荐的做法：动态库定义的所有符号都对外隐藏，只留必要的符号。

```cmake
#step1
set_target_properties(message-shared
  PROPERTIES
    POSITION_INDEPENDENT_CODE 1
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN 1
    SOVERSION ${PROJECT_VERSION_MAJOR}
    OUTPUT_NAME "message"
    DEBUG_POSTFIX "_d"
    PUBLIC_HEADER "Message.hpp;${CMAKE_BINARY_DIR}/${INSTALL_INCLUDEDIR}/messageExport.h"
    MACOSX_RPATH ON
  )
```

- `CXX_VISIBILITY_PRESET`：`hidden`：默认情况下，所有符号都不可见。相当于为目标添加`-fvisibility=hidden`标志
  - `hidden`：默认情况下，所有符号都**不可见**。这是构建共享库时推荐的做法，因为它可以减少库的大小并提高链接器优化的机会。
  - `protected`：所有符号默认为protected可见性，这意味着它们对于直接链接到库的程序可见，但对于其他共享库不可见。
  - `default`：符号的可见性由编译器默认设置决定，这通常意味着所有符号都是**可见**的。
- `VISIBILITY_INLINES_HIDDEN`：编译器标志，控制内联函数的可见性。这对应于`-fvisibility-inlines-hidden`



从第一步将目标的编译属性总体设为了hidden，也就意味着非显示设置对外隐藏了。

如何设置可见：`__attribute__((visibility("default")))` 在源码加载加上可见符号，可以被编译器所看见。

当然我们手动去设置没有问题，但是这样的修改，对于源码而言是一种污染，需要知道这是`__attribute__`关键字是编译器所识别的符号，不属于项目的符号。我们也可以通过宏函数的方式进行将其包含，通过下面这种方式：

```cmake
#step2
include(GenerateExportHeader)
generate_export_header(message-shared
  BASE_NAME "message"
  EXPORT_MACRO_NAME "message_EXPORT"
  EXPORT_FILE_NAME "${CMAKE_BINARY_DIR}/${INSTALL_INCLUDEDIR}/messageExport.h"
  DEPRECATED_MACRO_NAME "message_DEPRECATED"
  NO_EXPORT_MACRO_NAME "message_NO_EXPORT"
  STATIC_DEFINE "message_STATIC_DEFINE"
  NO_DEPRECATED_MACRO_NAME "message_NO_DEPRECATED"
  DEFINE_NO_DEPRECATED
  )
```

CMake提供了导出头文件的方法`generate_export_header`，然后进行设置一些宏的名字，就会在配置时生成所需的头文件，记得在源文件内使用该宏，将方法、变量暴露出来。

官网：https://cmake.org/cmake/help/latest/module/GenerateExportHeader.html查看函数的参数使用方法

例：:warning:`STATIC_DEFINE`：If the same sources are used to c**reate both a shared and a static library**, the uppercased symbol `${BASE_NAME}_STATIC_DEFINE` should be used when building the static library.



**注：**因为是同一套源码(也就意味着里面有显式的message_NO_EXPORT和message_EXPORT)，而静态库不同于动态库，其源码符号应该全部开放，所以需要`STATIC_DEFINE`进行控制，然后在设置静态库属性

```cmake
set_target_properties(message-static
	PROPERTIES
	COMPILE_FLAGS -Dmessage_STATIC_DEFINE
)
```

也可以`target_compile_definitions`进行控制，这样即使是通过`message_NO_EXPORT`显式设置为`hidden`的变量、函数，也依旧会开放其可见性



**more notes:**

构建动态库时，隐藏内部符号是一个很好的方式。这意味着库会缩小，因为向用户公开的内容要小于库中的内容。这定义了应用程序**二进制接口(ABI)**，通常情况下应该与**应用程序编程接口(API)**一致。这分两个阶段进行：

1. 使用适当的编译器标志。
2. 使用预处理器变量(示例中是`message_EXPORT`)标记要导出的符号。编译时，将解除这些符号(类和函数)的隐藏。

**静态库只是目标文件的归档**。因此，可以将源代码编译成目标文件，然后归档器将它们捆绑到归档文件中。这时没有ABI的概念：所有符号在默认情况下都是可见的，编译器的可见标志不影响静态归档。但是，如果要从相同的源文件构建动态和静态库，则需要一种方法来赋予`message_EXPORT`预处理变量意义，这两种情况都会出现在代码中。这里使用`GenerateExportHeader.cmake`模块，它定义一个包含所有逻辑的头文件，用于给出这个预处理变量的正确定义。对于动态库，它将给定的平台与编译器相组合。注意，根据构建或使用动态库，宏定义也会发生变化。幸运的是，CMake为我们解决了这个问题。对于静态库，它将扩展为一个空字符串，执行我们期望的操作——什么也不做。

构建此处所示的静态和共享库实际上需要**编译源代码两次**。对于我们的简单示例来说，这不是一个很大的开销，但会显得相当麻烦，即使对于只比示例稍大一点的项目来说，也是如此。为什么我们选择这种方法，而不是：`OBJECT`库负责编译库的第一步：从源文件到对象文件。该步骤中，预处理器将介入并计算`message_EXPORT`。由于对象库的编译只发生一次，`message_EXPORT`被计算为构建动态库库或静态库兼容的值。因此，为了避免歧义，我们选择了**更健壮**的方法，即编译两次，为的就是让预处理器**正确地评估变量的可见性**。



---

##### 导出目标

用户希望，当他们编译并安装了库，库就能更容易找到。这个示例将展示CMake如何让我们导出目标，以便其他使用CMake的项目可以轻松地获取它们(通过FindPackage)。



设置完成库的可见性等一系列设置后，只需要2步导出目标供他人使用：

1. 生成导出目标文件，在**install中增加`EXPORT messageTargets`**即可。并为生成的`messageTargets.cmake`设置属性、安装路径。这一步即可让其他项目find 到message

   ```cmake
   install(
     TARGETS
       message-shared
       message-static
     EXPORT #generate xx.cmake
         messageTargets
     ARCHIVE
     #...
   )
   #set install rules
   install(
     EXPORT
         messageTargets
     NAMESPACE
         "message::"
     DESTINATION
         ${INSTALL_CMAKEDIR}
     COMPONENT
         dev
     )
   ```

2. 使用`CMakePackageConfigHelpers.cmake`标准模块生成CMake配置文件，并为两个配置文件设置安装规则：

   ```cmake
   include(CMakePackageConfigHelpers)
   write_basic_package_version_file(#生成版本配置文件
     ${CMAKE_CURRENT_BINARY_DIR}/messageConfigVersion.cmake
     VERSION ${PROJECT_VERSION}
         COMPATIBILITY SameMajorVersion #大版本相同即可
     )
   configure_package_config_file(#Create a config file for a project
     ${PROJECT_SOURCE_DIR}/cmake/messageConfig.cmake.in
     ${CMAKE_CURRENT_BINARY_DIR}/messageConfig.cmake
     INSTALL_DESTINATION ${INSTALL_CMAKEDIR}
     )
     
   install(#安装上面的两个配置文件
     FILES
         ${CMAKE_CURRENT_BINARY_DIR}/messageConfig.cmake
         ${CMAKE_CURRENT_BINARY_DIR}/messageConfigVersion.cmake
     DESTINATION
         ${INSTALL_CMAKEDIR}
     )
   ```

注：messageConfig.cmake.in，配置文件的源文件是需要自己设置的，实质还是cmake文件。这里需要弄清楚的其实是这个配置文件究竟需要做些什么：

- 首先版本控制，上面的配置文件已经完成了。
- 自动化查找，我们会发现前面安装的路径,是标准路径，也无需特意设置
- 命名空间管理，第一步已经设置
- **条件化安装**：组件安装可以通过COMPONENT来控制，额外需要安装较特殊的组件搭配，也可以在配置文件进行测试
- **依赖管理**：本身库依赖的其他库，可以在这配置文件进行精细化设置
- 可移植性：可以在配置环境内配置不同平台的设置
- 减少手动配置产生的错误，比如路径和链接



官网：https://cmake.org/cmake/help/latest/module/CMakePackageConfigHelpers.html#generating-a-package-version-file



---

##### 安装超级构建

场景：你已经用自己的代码附带依赖库的源代码，实现了超级构建，而环境内已经安装了依赖库，修改了依赖库，又不想单独去安装依赖库，该怎么办呢？

:question:CMAKE_ARCHIVE_OUTPUT_DIRECTORY，可以通过ExternalProject传递参数，来控制Makefile的生成路径吗？

```cmake
#尝试
ExternalProject_Add(
#...
    CMAKE_ARGS
        -DCMAKE_ARCHIVE_OUTPUT_DIRECTORY=${CMAKE_BINARY_DIR}/external_libs
#...
)
```

答案：是不可用



==TODO：这里只能是由cmake管理的项目，才可以用cmake进行安装管理==

如果超级构建的项目由其他管理工具管理的，可能需要写脚本进行安装







## 打包

1. cmake还支持项目打包`CMakeCPack.cmake`标准模块提供了比CPack指令更简单的命令，打包成`.deb`  `.rpm` `.dmg` ...各式的格式
2. 利用第三方库、软解来打包，PyPI打包标准python包，Anaconda 打包混合项目



## 测试面板

- 将测试部署到CDash面板
- CDash面板显示测试覆盖率
- 使用AddressSanifier向CDash报告内存缺陷
- 使用ThreadSaniiser向CDash报告数据争用



## 第15章 使用CMake构建已有项目


使用CMake构建已有项目，其实也就是将其他管理的项目，转移成cmake管理。这其中可能会涉及到一些移植项目的问题，本章也会提供关于移植项目或将CMake添加到遗留代码的建议



首先使用原生的配置文件，对环境进行配置

`$ ./configure --enable-gui=no`

使用原生编译方式，记录日志，分析内容

`$ make > build.log`



---

## 简单积累

CMake的`file`命令可以用来直接调用Makefile。这通常用于简单的场景，其中Makefile可以独立于CMake构建系统运行。

```cmake
file(MAKE <directory> <target>)
```



判断语句

if(<variable|string> **STREQUAL** <variable|string>) 会自动检索变量，可以不${} ，但是尽量规范一些，用值对比值，字符串加引号。

**EQUAL**是对比数字的



#查看所有编译选项

```shell
$cmake .. -LH
```















