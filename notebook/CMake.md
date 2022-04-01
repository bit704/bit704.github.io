# CMake

[toc]

## 1.cmake-examples学习

项目地址：[GitHub - ttroy50/cmake-examples: Useful CMake Examples](https://github.com/ttroy50/cmake-examples)

中文辅助学习网址：[前言 · GitBook (sfumecjf.github.io)](https://sfumecjf.github.io/cmake-examples-Chinese/)



01-A 在windows上如果要生成makefile而不是sln，不能用MSVC，应该用**MinGW**。添加环境变量CMAKE_GENERATOR为MinGW Makefiles，或者使用 -G “MinGW Makefiles”。

01-E windows上**CMAKE_INSTALL_PREFIX**默认是C:\Program Files (x86)\。如果要修改安装路径，使用 -DCMAKE_INSTALL_PREFIX='D:\code\cmake\install'。

01-F 使用-DCMAKE_BUILD_TYPE=Release来设置构建类型。具体set_property没有完全理解。

01-G 编译选项没有完全理解。

学习到 01-I

