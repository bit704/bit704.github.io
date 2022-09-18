---
title: CMake笔记
index_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-CMake笔记.png
banner_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-CMake笔记.png
layout: post
categories: [Notebook]
tags: [Cmake,C++]
---

一个跨平台的安装（编译）工具

<!-- more -->

## 1.cmake-examples总结

项目地址：[GitHub - ttroy50/cmake-examples: Useful CMake Examples](https://github.com/ttroy50/cmake-examples)

中文辅助学习网址：[前言 · GitBook (sfumecjf.github.io)](https://sfumecjf.github.io/cmake-examples-Chinese/)（部分内容少于官方，比如01-E、01-L、03以后的部分）

### 01-basic

#### A 你好cmake

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5) #设置CMake最小版本
project (hello_cmake) #设置工程名
add_executable(hello_cmake main.cpp) #生成可执行文件
```

在windows上如果要生成makefile而不是sln，不能用MSVC，应该用**MinGW**。添加环境变量CMAKE_GENERATOR为MinGW Makefiles，或者命令行中使用 -G “MinGW Makefiles”。

```powershell
# 编译命令
# MinGW
mkdir build
cd build/
cmake ..
make
# 通用命令，符合modern cmake,支持Makefile、Ninja、MSVC等不同的底层。
cmake . -B build
cmake --build build
```

#### B 头文件

```tree
├── CMakeLists.txt
├── include
│   └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (hello_headers)
#创建一个变量，名字叫SOURCE。它包含了所有的cpp文件。
set(SOURCES
    src/Hello.cpp
    src/main.cpp
)
#等价于命令：add_executable(hello_headers src/Hello.cpp src/main.cpp)。建议直接罗列。
add_executable(hello_headers ${SOURCES}) 
#设置这个可执行文件hello_headers需要包含的库的路径
#PRIVATE指定了库的范围
target_include_directories(hello_headers
    PRIVATE 
        ${PROJECT_SOURCE_DIR}/include
)

#PROJECT_SOURCE_DIR指工程顶层目录
#PROJECT_Binary_DIR指编译目录
```

#### C 静态库

```tree
├── CMakeLists.txt
├── include
│   └── static
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project(hello_library)
############################################################
# Create a library
############################################################
#库的源文件Hello.cpp生成静态库hello_library
add_library(hello_library STATIC 
    src/Hello.cpp
)
# target_include_directories为一个目标（可能是一个库library也可能是可执行文件）添加头文件路径。
target_include_directories(hello_library
    PUBLIC 
        ${PROJECT_SOURCE_DIR}/include
)
############################################################
# Create an executable
############################################################
#生成可执行文件
add_executable(hello_binary 
    src/main.cpp
)
#链接可执行文件和静态库
target_link_libraries( hello_binary
    PRIVATE 
        hello_library
)
```

**target_include_directories范围**

- PRIVATE - 目录被添加到目标（库）的包含路径中。
- INTERFACE - 目录没有被添加到目标（库）的包含路径中，而是链接了这个库的其他目标（库或者可执行程序）包含路径中
- PUBLIC - 目录既被添加到目标（库）的包含路径中，同时添加到了链接了这个库的其他目标（库或者可执行程序）的包含路径中

#### D 共享库

```tree
├── CMakeLists.txt
├── include
│   └── shared
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project(hello_library)
############################################################
# Create a library
############################################################
#根据Hello.cpp生成动态库
add_library(hello_library SHARED 
    src/Hello.cpp
)
#给动态库hello_library起一个别的名字hello::library
add_library(hello::library ALIAS hello_library)
target_include_directories(hello_library
    PUBLIC 
        ${PROJECT_SOURCE_DIR}/include
)
############################################################
# Create an executable
############################################################
add_executable(hello_binary
    src/main.cpp
)
target_link_libraries( hello_binary
    PRIVATE 
        hello::library
)
```

#### E 安装

```tree
├── cmake-examples.conf
├── CMakeLists.txt
├── include
│   └── installing
│       └── Hello.h
└── src
    ├── Hello.cpp
    └── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project(cmake_examples_install)
############################################################
# Create a library
############################################################
#生成共享库
add_library(cmake_examples_inst SHARED
    src/Hello.cpp
)
target_include_directories(cmake_examples_inst
    PUBLIC 
        ${PROJECT_SOURCE_DIR}/include
)

############################################################
# Create an executable
############################################################
add_executable(cmake_examples_inst_bin
    src/main.cpp
)
target_link_libraries( cmake_examples_inst_bin
    PRIVATE 
        cmake_examples_inst
)

############################################################
# Install
############################################################

# Binaries
install (TARGETS cmake_examples_inst_bin
    DESTINATION bin)

# Library
# Note: may not work on windows
install (TARGETS cmake_examples_inst
    LIBRARY DESTINATION lib)

# Header files
install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/ 
    DESTINATION include)

# Config
install (FILES cmake-examples.conf
    DESTINATION etc)
```

windows上**CMAKE_INSTALL_PREFIX**默认是C:\Program Files (x86)\。如果要修改安装路径，使用 -DCMAKE_INSTALL_PREFIX='D:\code\cmake\install'。

#### F 构建类型

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
#如果没有指定则设置默认编译方式
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  #在命令行中输出message里的信息
  message("Setting build type to 'RelWithDebInfo' as none was specified.")
  #不管CACHE里有没有设置过CMAKE_BUILD_TYPE这个变量，都强制赋值这个值为RelWithDebInfo
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE)
  # 当使用cmake-gui的时候，设置构建级别的四个可选项
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

project (build_type)
add_executable(cmake_examples_build_type main.cpp)
```

**cmake优化级别**

- Release —— 不可以打断点调试，程序开发完成后发行使用的版本，占的体积小。 它对代码做了优化，因此速度会非常快，

  在编译器中使用命令： `-O3 -DNDEBUG` 可选择此版本。

- Debug ——调试的版本，体积大。

  在编译器中使用命令： `-g` 可选择此版本。

- MinSizeRel—— 最小体积版本

  在编译器中使用命令：`-Os -DNDEBUG`可选择此版本。

- RelWithDebInfo—— 既优化又能调试。

  在编译器中使用命令：`-O2 -g -DNDEBUG`可选择此版本。


命令行中使用-DCMAKE_BUILD_TYPE=Release可以设置

#### G 编译选项

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
#强制设置默认C++编译标志变量为缓存变量。该缓存变量被定义在文件中（CMakeCache.txt），相当于全局变量，源文件中也可以使用这个变量。是全局的。
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DEX2" CACHE STRING "Set C++ Compiler Flags" FORCE)
project (compile_flags)
add_executable(cmake_examples_compile_flags main.cpp)
#为可执行文件添加私有编译定义。只为这个目标设置编译选项。
target_compile_definitions(cmake_examples_compile_flags 
    PRIVATE EX3
)
```

除了以上两种方法设置编译选项，还可以在命令行中使用-DCMAKE_CXX_FLAGS=（是全局的）。

#### H 包含第三方库

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (third_party_include)
#使用库文件系统和系统查找boost install
find_package(Boost 1.46.1 REQUIRED COMPONENTS filesystem system)
#这是第三方库，而不是自己生成的静态动态库
#是否找到
if(Boost_FOUND)
    message ("boost found")
else()
    message (FATAL_ERROR "Cannot find Boost")
endif()

add_executable(third_party_include main.cpp)

target_link_libraries( third_party_include
    PRIVATE
        Boost::filesystem
)
```

#### I 使用clang编译工程

```tree
├── CMakeLists.txt
├── main.cpp
├── pre_test.sh
├── run_test.sh
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (hello_cmake)
add_executable(hello_cmake main.cpp)
```

此示例与[hello-cmake](https://sfumecjf.github.io/cmake-examples-Chinese/A-hello-cmake)示例相同，不同之处在于它使用脚本将编译器从默认的gcc更改为[clang](http://clang.llvm.org/)的最基本方法。

CMake提供了控制程序编译以及链接的选项，选项如下:

- CMAKE_C_COMPILER - 用于编译c代码的程序。

- CMAKE_CXX_COMPILER - 用于编译c++代码的程序。

- CMAKE_LINKER - 用于链接二进制文件的程序。

#### J 使用ninja编译工程

```tree
├── CMakeLists.txt
├── main.cpp
├── pre_test.sh
├── run_test.sh
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (hello_cmake)
add_executable(hello_cmake main.cpp)
```

使用cmake -h查看支持的构建工具生成器。有三种，分别是命令行构建工具生成器、IDE构建工具生成器、其它生成器。使用-G选择生成器。

#### K 导入目标

与H相似。

#### L C++标准

1. [common-method](https://sfumecjf.github.io/cmake-examples-Chinese/01-basic/i-common-method). 可以与大多数版本的CMake一起使用的简单方法。

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 2.8)
project (hello_cpp11)

# try conditional compilation
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)

# check results and add flag
if(COMPILER_SUPPORTS_CXX11)#
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)#
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
    message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

add_executable(hello_cpp11 main.cpp)

```

2. [cxx-standard](https://sfumecjf.github.io/cmake-examples-Chinese/01-basic/ii-cxx-standard). 使用CMake v3.1中引入的CMAKE_CXX_STANDARD变量。

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.1)
project (hello_cpp11)

# set the C++ standard to C++ 11
set(CMAKE_CXX_STANDARD 11)

add_executable(hello_cpp11 main.cpp)
```

它会选择最近的正确标准而不会报错。

3. [compile-features](https://sfumecjf.github.io/cmake-examples-Chinese/01-basic/iii-compile-features). 使用CMake v3.1中引入的target_compile_features函数。

```tree
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.1)
project (hello_cpp11)
add_executable(hello_cpp11 main.cpp)

# set the C++ standard to the appropriate standard for using auto
target_compile_features(hello_cpp11 PUBLIC cxx_auto_type)

# Print the list of known compile features for this version of CMake
message("List of compile features: ${CMAKE_CXX_COMPILE_FEATURES}")

```

### 02-sub-projects

```tree
├── CMakeLists.txt 最高层
├── subbinary
│   ├── CMakeLists.txt  生成可执行文件
│   └── main.cpp
├── sublibrary1
│   ├── CMakeLists.txt 生成静态库
│   ├── include
│   │   └── sublib1
│   │       └── sublib1.h
│   └── src
│       └── sublib1.cpp
└── sublibrary2
    ├── CMakeLists.txt 生成仅有头文件的库
    └── include
        └── sublib2
            └── sublib2.h
```

将头文件移至每个项目include目录下的子文件夹可以防止文件名冲突

```cmake
cmake_minimum_required (VERSION 3.5)
project(subprojects)

# Add sub directories
add_subdirectory(sublibrary1)
add_subdirectory(sublibrary2)
add_subdirectory(subbinary)
```

CMake中有一些变量会自动创建:

| Variable           | Info                                                         |
| ------------------ | ------------------------------------------------------------ |
| PROJECT_NAME       | 当前project（）设置的项目的名称。                            |
| CMAKE_PROJECT_NAME | 由project（）命令设置的第一个项目的名称，即顶层项目。        |
| PROJECT_SOURCE_DIR | 当前项目的源文件目录。                                       |
| PROJECT_BINARY_DIR | 当前项目的构建目录。                                         |
| name_SOURCE_DIR    | 在此示例中，创建的源目录为 `sublibrary1_SOURCE_DIR`, `sublibrary2_SOURCE_DIR`, and `subbinary_SOURCE_DIR` |
| name_BINARY_DIR    | 本工程的二进制目录是`sublibrary1_BINARY_DIR`, `sublibrary2_BINARY_DIR`,和 `subbinary_BINARY_DIR` |

```cmake
project(subbinary)
add_executable(${PROJECT_NAME} main.cpp)

# Link the static library from subproject1 using its alias sub::lib1
# Link the header only library from subproject2 using its alias sub::lib2
# This will cause the include directories for that target to be added to this project
target_link_libraries(${PROJECT_NAME}
    sub::lib1
    sub::lib2
)
```

```cmake
project (sublibrary1)

add_library(${PROJECT_NAME} src/sublib1.cpp)
add_library(sub::lib1 ALIAS ${PROJECT_NAME})

target_include_directories( ${PROJECT_NAME}
    PUBLIC 
    	${PROJECT_SOURCE_DIR}/include
)
```

```cmake
project (sublibrary2)

add_library(${PROJECT_NAME} INTERFACE)
add_library(sub::lib2 ALIAS ${PROJECT_NAME})

target_include_directories(${PROJECT_NAME}
    INTERFACE
        ${PROJECT_SOURCE_DIR}/include
)
```

### 03-code-generation

代码生成是一个很好用的功能，它可以使用一份公共的描述文件，生成不同语言下的源代码。这个功能使得需要人工编写的代码大幅减少，同时也增加了互操作性。

#### 1. [configure-file](https://sfumecjf.github.io/cmake-examples-Chinese/03-code-generation/configure-files)

- 使用CMake中的configure_file函数注入CMake变量

```tree
├── CMakeLists.txt
├── main.cpp
├── path.h.in
├── ver.h.in
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (cf_example)

# set a project version
set (cf_example_VERSION_MAJOR 0)
set (cf_example_VERSION_MINOR 2)
set (cf_example_VERSION_PATCH 1)
set (cf_example_VERSION "${cf_example_VERSION_MAJOR}.${cf_example_VERSION_MINOR}.${cf_example_VERSION_PATCH}")

# Call configure files on ver.h.in to set the version.
# Uses the standard ${VARIABLE} syntax in the file
configure_file(ver.h.in ${PROJECT_BINARY_DIR}/ver.h)

# configure the path.h.in file.
# This file can only use the @VARIABLE@ syntax in the file
configure_file(path.h.in ${PROJECT_BINARY_DIR}/path.h @ONLY)

add_executable(cf_example
    main.cpp
)

# include the directory with the new files
target_include_directories( cf_example
    PUBLIC
        ${CMAKE_BINARY_DIR}
)
```

#### 2. [Protocol Buffers](https://sfumecjf.github.io/cmake-examples-Chinese/03-code-generation/protobuf)

- 使用Google Protocol Buffers来生成C++源码

```tree
├── AddressBook.proto
├── CMakeLists.txt
├── main.cpp
```

```cmake
cmake_minimum_required(VERSION 3.5)
project (protobuf_example)

# find the protobuf compiler and libraries
find_package(Protobuf REQUIRED)

# check if protobuf was found
if(PROTOBUF_FOUND)
    message ("protobuf found")
else()
    message (FATAL_ERROR "Cannot find Protobuf")
endif()

# Generate the .h and .cxx files
PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS AddressBook.proto)

# Print path to generated files
message ("PROTO_SRCS = ${PROTO_SRCS}")
message ("PROTO_HDRS = ${PROTO_HDRS}")

# Add an executable
add_executable(protobuf_example
    main.cpp
    ${PROTO_SRCS}
    ${PROTO_HDRS})

target_include_directories(protobuf_example
    PUBLIC
    ${PROTOBUF_INCLUDE_DIRS}
    ${CMAKE_CURRENT_BINARY_DIR}
)

# link the exe against the libraries
target_link_libraries(protobuf_example
    PUBLIC
    ${PROTOBUF_LIBRARIES}
)
```

### 04-static-analysis

