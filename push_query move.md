#### push和query移植操作

1. taf基础编译时缺失头文件

   arch标准库中可能没有包含functional头文件，给需要使用functional的taf源文件增加头文件

   

   特殊：

   在编译servant时没有包含aarch

   ```makefile
   #if defined(__x86_64__) || defined(__x86_64) \
       || defined(__amd64__) || defined(__amd64) \
       || defined(_M_X64) || defined(_M_AMD64)  \
   ++  || defined(__aarch64__) || defined(__aarch64)
   #  include "tc_fcontext_x86_64.h"
   ```

   Tip:

   ​	**tc_fcontext_x86_64.h**好像是由上下文切换的汇编转义的，要增加支持平台需要重新生成**tc_fcontext_arrch.h**

2. 高精度计时器实现

   Rdtsc是x86下一条读取TSC的指令。在ARM64平台下，我们可以通过mrs指令来读取CNTVCT_EL0计时器来实现，具体实现如下。
   原x86嵌汇编实现：

   ```makefile
   #define rdtsc(low,high) \
          __asm__ __volatile__("rdtsc" : "=a" (low), "=d" (high))
   ```

   支持ARM64平台后的实现：

   ```makefile
   #if defined(__aarch64__)
    #define rdtsc(var) \
          asm volatile("mrs %0, CNTVCT_EL0" : "=r"(var))
    #elif defined(__x86_64__)
    #define rdtsc(low,high) \
          __asm__ __volatile__("rdtsc" : "=a" (low), "=d" (high))
    #endif
   ```

3. 协程堆栈初始化和寄存器上下文切换操作

   添加tc_jump_arm64_aapcs_elf_gas.s、tc_make_arm64_aapcs_elf_gas.s 汇编文件

   https://liu-paopao.github.io/posts/4059c7aa.html#x86-64%E6%9E%B6%E6%9E%84%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2

   并在cmake添加判断

   ```cmake
   if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
       list(APPEND DIR_SRCS "tc_jump_x86_64_sysv_elf_gas.s")
       list(APPEND DIR_SRCS "tc_make_x86_64_sysv_elf_gas.s")
   elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
       list(APPEND DIR_SRCS "tc_make_arm64_aapcs_elf_gas.s")
       list(APPEND DIR_SRCS "tc_jump_arm64_aapcs_elf_gas.s")
   endif ()
   ```

   ---

   cmake通过，jce2cpp工具编译成功,libparse.a  libtafframework.a  libtafservant.a  libtafutil.a库编译成功

   ---

   

4. taf整体编译时无法连接`libmysqlclient.a`静态库

    需要重新编译arch版本

    1. 需要boost1.59

    2. 缺失Curses库

       `sudo yum install ncurses-devel`

    注意：编译完需要安装 `make install`

    ```
    [100%] Built target libmysql_api_test
    make[1]: Leaving directory '/home/mysql/mysql-5.7.17/BUILD'
    Install the project...
    -- Install configuration: "RelWithDebInfo"
    -- Installing: /usr/local/mysql/lib/libmysqlclient.a
    -- Installing: /usr/local/mysql/lib/libmysqlclient.so.20.3.4
    -- Installing: /usr/local/mysql/lib/libmysqlclient.so.20
    -- Installing: /usr/local/mysql/lib/libmysqlclient.so
    ```

    ---

    ​					taf-src 整体编译成功， `make install`安装taf库

    ---

5. 编译PushServer arch机器无法识别-m64编译选项

    直接去掉，由编译器自动识别，或修改为**`-march=armv8-a`**

6. 链接upevent失败

    在依赖库/event内，重新编译安装

7. 找不到main的定义

    问题原因在于 makefile.taf.new  include 了`+ 平台判断里面过滤掉了.o文件的生成

    暂时解决：注释掉判断

8. 出现jce.h大量truncated截断警告，log字符超过字符数组长度

    扩大字符数组长度64-128

9. Push找不到虚表

    `rm /home/tafjce/IndicatorSys/LIBS/isutils/*`,重新编译了LIBS/isutils 工具

10. ld libtafutil.a (tc_encoder.cpp.o)失败，提示未定义libiconv

     暂时：make PushServer的时候 -liconv  连个动态的先

11. 编译出来的可执行文件，链接不上libmysqlclient.so.20

     ```cmake
     #taf-src CMakeLists.txt
     link_directories(/usr/local/lib)
     ```

     重新编译后，可以链接了(???)

     libmysqlclient.so.20放到了/usr/lib64下

    ---

      							PushServer编译成功

    ---

     

     

       

       







