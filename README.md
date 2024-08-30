
准确来说，minizip其实是zlib提供的辅助工具，位于zlib库的contrib文件夹内。minizip提供了更为高级一点的接口，能直接操作文件进行压缩。不过，有点麻烦的是这个工具并没有提供CMake构建的方式。那么可以按照构建giflib的方式，自己组织CMakeList.txt，正好这个项目的代码量并不多。


另一个问题是，minizip其实是个可执行程序，Windows下不能直接将其构建成动态链接库，因为Windows下的动态链接库是需要设置导出的，否则就会提示找不到符号的问题。这种情况下最简便的方式就是将其组织成静态库了（[项目地址](https://github.com)），CMakeList.txt如下所示：



```
# 输出cmake版本提示
message(STATUS "The CMAKE_VERSION is ${CMAKE_VERSION}.")

# cmake的最低版本要求
cmake_minimum_required (VERSION 3.10)

# 工程名称、版本、语言
project(minizip VERSION 5.2.2)

# 支持当前目录
set(CMAKE_INCLUDE_CURRENT_DIR ON)

# 判断编译器类型
message("CMAKE_CXX_COMPILER_ID: ${CMAKE_CXX_COMPILER_ID}")

# 判断编译器类型
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    message(">> using Clang")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    message(">> using GCC")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
    message(">> using Intel C++")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    message(">> using Visual Studio C++")	  
    add_compile_options(/utf-8 /wd4996)  
else()
    message(">> unknow compiler.")
endif()

# 查找 ZLIB 模块
find_package(ZLIB REQUIRED)

# 源代码文件
set(PROJECT_SOURCES ioapi.c iowin32.c miniunz.c minizip.c mztools.c unzip.c zip.c)
set(PROJECT_HEADER crypt.h ioapi.h iowin32.h mztools.h unzip.h zip.h)

# 将源代码添加到此项目的可执行文件。
add_library(${PROJECT_NAME} STATIC ${PROJECT_SOURCES} ${PROJECT_HEADER})

#
target_link_libraries(${PROJECT_NAME} ZLIB::ZLIB)

# TODO: 如有需要，请添加测试

# 安装头文件到 include 目录
install(FILES ${PROJECT_HEADER} DESTINATION include/${PROJECT_NAME})

# 安装库文件到 lib 目录
install(TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION lib  # 对于共享库
        ARCHIVE DESTINATION lib  # 对于静态库
        RUNTIME DESTINATION bin  # 对于可执行文件
)

```

关键的构建指令如下所示：



```
# 配置CMake  
cmake .. -G "$Generator" -A x64 `
-DCMAKE_CONFIGURATION_TYPES=RelWithDebInfo `
-DCMAKE_PREFIX_PATH="$InstallDir" `
-DCMAKE_INSTALL_PREFIX="$InstallDir"

# 构建阶段，指定构建类型
cmake --build . --config RelWithDebInfo

# 安装阶段，指定构建类型和安装目标
cmake --build . --config RelWithDebInfo --target install

```

在最后谈谈动态库和静态库的问题。动态库和静态库各有优缺点，这里就不细致的论述了。但是在Windows下笔者还是倾向于优先使用动态库。一直以来，二进制兼容的问题一直是困扰C/C\+\+编程的重要问题。比如说，你用VS2010编译的动态库在VS2013的环境下可能是无法使用的，这还是同一家产品的不同版本就会造成这个二进制成果的差异性问题。但是，根据Microsoft提供的文档（参看：[https://learn.microsoft.com/zh\-cn/cpp/porting/binary\-compat\-2015\-2017](https://github.com):[楚门加速器p](https://tianchuang88.com) ），VS2015以后的版本就会开始提供二进制兼容的特性了，原理是标准库、运行时库（如 msvcp140\.dll）、C\+\+ 标准库保证了ABI（二进制接口）的稳定。笔者也确实发现很多产品的MSVC的预编译成果能够在MSVC环境中混用了。比如VS2017编译的Qt就能够在VS2019的环境下正常使用。不过这些能混用的成果一般都是动态库，也就是动态库的二进制兼容特性更好一点。至于静态库，文档中宣称静态库也可以做到，但是笔者实测至少这个基于VS2017的minizip静态库在VS2019中用不了。这一点就留待以后解决了。


