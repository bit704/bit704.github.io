---
title: Win32笔记
categories: [Notebook]
tags: [C++,API]
index_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Win32笔记.jfif
banner_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Win32笔记.jfif
---

使用 C++ 和 Win32 API 为Windows电脑生成桌面应用

<!--more-->



## 0 前言

由于需要用到windows窗口，参考以下文档:

[使用 Win32 API 生成桌面Windows应用 - Win32 apps ](https://docs.microsoft.com/zh-cn/windows/win32/)

进入其中的：[Win32 和 C++ 入门 - Win32 apps ](https://docs.microsoft.com/zh-cn/windows/win32/learnwin32/learn-to-program-for-windows)

都是机翻的，得看英文原版。

安装Windows SDK即可。注意其不支持硬件驱动程序开发。

匈牙利表示法虽然仍在文档中存在，但已经不再使用。

## 1 第一个窗口程序

### 窗口简介

窗口之间有两种关系，拥有关系和亲子关系：

![窗口关系](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E7%AA%97%E5%8F%A3%E5%85%B3%E7%B3%BB.png)

窗口句柄的数据类型是 **HWND**，通过将其作为参数的函数来操作它。句柄不是指针。

原点始终在左上角：

![屏幕和窗口坐标](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%B1%8F%E5%B9%95%E5%92%8C%E7%AA%97%E5%8F%A3%E5%9D%90%E6%A0%87.png)

每个窗口程序都包含一个名为 **WinMain** 或 **wWinMain** 的入口点函数。

```c++
int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow);
```

- *hInstance* 称为“实例句柄”或“模块句柄”。操作系统使用此值在内存中加载可执行文件时标识可执行文件 (EXE) 。 某些Windows函数需要实例句柄，例如加载图标或位图。
- *hPrevInstance* 没有意义。 它在 16 位Windows中使用，但现在始终为零。
- *pCmdLine* 包含作为 Unicode 字符串的命令行参数。
- *nCmdShow* 是一个标志，指示主应用程序窗口是最小化、最大化还是正常显示。

**WinMain** 函数与 **wWinMain** 相同，但命令行参数作为 ANSI 字符串传递。

**WINAPI** 是调用约定。 *调用约定*定义函数如何从调用方接收参数。 例如，它定义参数在堆栈上显示的顺序。 

### 程序示例

创建一个窗口。

代码来自[microsoft/Windows-classic-samples](https://github.com/microsoft/Windows-classic-samples/)中的**Samples/Win7Samples/begin/LearnWin32/HelloWorld/cpp**。此后的程序示例均位于LearnWin32中。

```c++
#ifndef UNICODE
#define UNICODE
#endif 

#include <windows.h>

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam);

int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow)
{
    //填写WNDCLASS结构，注册窗口类。此类不是 C++ 意义上的“类”。相反，它是操作系统在内部使用的数据结构。 
    const wchar_t CLASS_NAME[]  = L"Sample Window Class";
    
    WNDCLASS wc = { };

    wc.lpfnWndProc   = WindowProc; //窗口过程,定义窗口的大部分行为
    wc.hInstance     = hInstance;  //应用程序实例的句柄
    wc.lpszClassName = CLASS_NAME; //标识窗口类的字符串
	//向操作系统注册窗口类
    RegisterClass(&wc);

    //创建窗口类
    HWND hwnd = CreateWindowEx(
        0,                              //可选窗口行为
        CLASS_NAME,                     //窗口类名称
        L"Learn to Program Windows",    //窗口文本
        WS_OVERLAPPEDWINDOW,            //窗口样式

        //位置和大小， CW_USEDEFAULT表示使用默认值。
        CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT,

        NULL,       //父窗口或所有者窗口，NULL代表顶级窗口   
        NULL,       // 定义菜单
        hInstance,  // 应用程序实例的句柄
        NULL        // 附加数据， void*
        );

    if (hwnd == NULL)
    {
        return 0;
    }

    //显示窗口。nCmdShow 参数可用于最小化或最大化窗口。
    ShowWindow(hwnd, nCmdShow);

    //程序将保留在此循环中，直到用户关闭窗口并退出应用程序。
    MSG msg = { };
    //GetMessage函数从队列的头中删除第一条消息。后三个参数用于筛选消息，几乎所有情况都默认为0。
    while (GetMessage(&msg, NULL, 0, 0) > 0)
    {
        //将击键转换为字符
        TranslateMessage(&msg);
        //告知操作系统调用消息的目标窗口的窗口过程
        DispatchMessage(&msg);
    }

    return 0;
}

//入参：窗口句柄，消息代码，附加数据（后两个参数）
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
    case WM_DESTROY:
        //退出应用程序并中断消息循环。PostQuitMessage 函数在消息队列上放置WM_QUIT消息。 WM_QUIT 是一条特殊消息：它会导致 GetMessage 返回零，发出消息循环的结束信号。
        PostQuitMessage(0);
        return 0;

    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hwnd, &ps);

            // 所有绘制发生在BeginPaint和EndPaint之间
            FillRect(hdc, &ps.rcPaint, (HBRUSH) (COLOR_WINDOW+1));

            EndPaint(hwnd, &ps);
        }
        return 0;

    }
    //执行默认操作。对于 WM_CLOSE，DefWindowProc 会自动调用 DestroyWindow。
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
```

[关于消息和消息队列 - Win32 apps](https://docs.microsoft.com/zh-CN/windows/win32/winmsg/about-messages-and-message-queues)

关闭窗口的消息流程：

![关闭窗口流程](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%85%B3%E9%97%AD%E7%AA%97%E5%8F%A3%E6%B5%81%E7%A8%8B.png)

[管理应用程序状态 - Win32 apps ](https://docs.microsoft.com/zh-cn/windows/win32/learnwin32/managing-application-state-)

## 2 使用COM

### COM简介

COM 是 *二进制标准*，而不是语言标准：它定义应用程序和软件组件之间的二进制接口。 作为二进制标准，COM 是语言中立的，尽管它自然映射到某些 C++ 构造。

COM接口的概念与C#、Java中的接口和C#中的纯虚类类似。使用名为“接口定义语言” (IDL) 的语言定义 COM 接口。 IDL 文件由 IDL 编译器处理，该编译器生成 C++ 头文件。

### 初始化

使用 COM 的任何Windows程序都必须通过调用 [**CoInitializeEx**](https://docs.microsoft.com/zh-CN/windows/desktop/api/combaseapi/nf-combaseapi-coinitializeex) 函数来初始化 COM 库。 使用 COM 接口的每个线程都必须单独调用此函数。

对于 CoInitializeEx 的每个成功调用，必须在线程退出之前调用 [**CoUninitialize**](https://docs.microsoft.com/zh-CN/windows/desktop/api/combaseapi/nf-combaseapi-couninitialize) 。 此函数不采用任何参数，并且没有返回值。

```c++
HRESULT CoInitializeEx(LPVOID pvReserved, DWORD dwCoInit);
CoUninitialize();
```

CoInitializeEx第一个参数是保留的，必须为 **NULL**。 第二个参数指定程序将使用的线程模型。

| 标志                         | 描述       |
| :--------------------------- | :--------- |
| **COINIT_APARTMENTTHREADED** | 单元线程。 |
| **COINIT_MULTITHREADED**     | 多线程。   |

如果使用单元线程，你应该保证：

- 你将从单个线程访问每个 COM 对象；不会在多个线程之间共享 COM 接口指针。
- 线程将有一个消息循环。

COM 方法和函数返回 **HRESULT** 类型的值要指示成功或失败。 **HRESULT** 是一个 32 位整数，高阶位表示成功或失败， 0表示成功，1 表示失败。少量 COM 方法不返回 **HRESULT** 值。 例如， [**AddRef**](https://docs.microsoft.com/zh-CN/windows/desktop/api/unknwn/nf-unknwn-iunknown-addref) 和 [**Release**](https://docs.microsoft.com/zh-CN/windows/desktop/api/unknwn/nf-unknwn-iunknown-release) 方法返回无符号长值。 Windows SDK 标头提供两个宏，[**SUCCEEDED**](https://docs.microsoft.com/zh-CN/windows/desktop/api/winerror/nf-winerror-succeeded) 宏和 [**FAILED**](https://docs.microsoft.com/zh-CN/windows/desktop/api/winerror/nf-winerror-failed) 宏，检查 COM 方法是否成功。

### 创建对象

[**CoCreateInstance**](https://docs.microsoft.com/zh-cn/windows/desktop/api/combaseapi/nf-combaseapi-cocreateinstance) 函数提供用于创建对象的泛型机制。两个 COM 对象可以实现相同的接口，一个对象可以实现两个或多个接口。 因此，创建对象的泛型函数需要两段信息：

- 要创建的对象。
- 要从对象中获取的接口。

在 COM 中，通过为其分配 128 位数字来标识对象或接口，该数字称为*全局唯一标识符* (GUID) 。GUID 以有效唯一的方式生成。 GUID 是解决如何在没有中央注册机构的情况下创建唯一标识符的问题。 GUID 有时称为 *通用唯一标识符* (UUID) 。 在 COM 之前，它们用于 DCE/RPC (分布式计算环境/远程过程调用) 。 存在多个用于创建新 GUID 的算法，并非所有算法都严格保证唯一性，但意外创建同一 GUID 值的概率极小，实际上为零。

### 程序示例

创建一个“打开”对话框。

```c++
#include <windows.h>
#include <shobjidl.h> 

int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE, PWSTR pCmdLine, int nCmdShow)
{
    //初始化COM库
    HRESULT hr = CoInitializeEx(NULL, COINIT_APARTMENTTHREADED | 
        COINIT_DISABLE_OLE1DDE);
    if (SUCCEEDED(hr))
    {
        IFileOpenDialog *pFileOpen;

        //创建FileOpenDialog对象。参数依次为：类标识符，NULL，执行上下文，接口标识符，指向接口的指针
        hr = CoCreateInstance(CLSID_FileOpenDialog, NULL, CLSCTX_ALL, 
                IID_IFileOpenDialog, reinterpret_cast<void**>(&pFileOpen));
		//创建成功
        if (SUCCEEDED(hr))
        {
            //展示对话框
            hr = pFileOpen->Show(NULL);

            // 从对话框获取文件名
            if (SUCCEEDED(hr))
            {
                //实现 IShellItem 接口的 Shell 项表示用户选择的文件
                IShellItem *pItem;
                hr = pFileOpen->GetResult(&pItem);
                if (SUCCEEDED(hr))
                {
                    PWSTR pszFilePath;
                    // 此方法以字符串的形式获取文件路径
                    hr = pItem->GetDisplayName(SIGDN_FILESYSPATH, &pszFilePath);

                    // Display the file name to the user.
                    if (SUCCEEDED(hr))
                    {
                        //显示显示文件路径的消息框
                        MessageBoxW(NULL, pszFilePath, L"File Path", MB_OK);
                        CoTaskMemFree(pszFilePath);
                    }
                    pItem->Release();
                }
            }
            pFileOpen->Release();
        }
        //取消初始化COM库
        CoUninitialize();
    }
    return 0;
}
```

### 管理对象的生存期

 每个 COM 接口都必须直接或间接地从名为 [**IUnknown**](https://docs.microsoft.com/zh-CN/windows/desktop/api/unknwn/nn-unknwn-iunknown) 的接口继承。 此接口提供所有 COM 对象必须支持的一些基线功能，它定义了三种方法：

- [**QueryInterface**](https://docs.microsoft.com/zh-CN/windows/desktop/api/unknwn/nf-unknwn-iunknown-queryinterface(q))
- [**AddRef**](https://docs.microsoft.com/zh-CN/windows/desktop/api/unknwn/nf-unknwn-iunknown-addref)
- [**Release**](https://docs.microsoft.com/en-us/windows/desktop/api/unknwn/nf-unknwn-iunknown-release)

为了释放资源，C++ 使用自动析构函数，而 C# 和 Java 则使用垃圾回收。COM 使用称为 *引用计数*的方法。

每个 COM 对象都维护内部计数。 这称为引用计数。 引用计数跟踪对象当前处于活动状态的引用数。 当引用数降至零时，对象将删除自身。 最后一部分值得重复：对象删除自身。 程序永远不会显式删除对象。

![COM管理过程](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-COM%E7%AE%A1%E7%90%86%E8%BF%87%E7%A8%8B.png)

```C++
HRESULT QueryInterface(REFIID riid, void **ppvObject);
```

查询某个组件是否支持某个特定的接口。第一个参数标识客户所需的接口；第二个参数是指向接口的指针。



[COM 中的内存分配 - Win32 apps ](https://docs.microsoft.com/zh-cn/windows/win32/learnwin32/memory-allocation-in-com)

