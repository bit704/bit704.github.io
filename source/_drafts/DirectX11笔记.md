---
title: DirectX11笔记
categories: [Notebook]
tags: [C++,DirectX]
---

目前最常用的DX

<!--more-->

[toc]

## 前言

参考教程：[DirectX11 With Windows SDK](https://directx11.tech/#/)

代码仓库：[MKXJun/DirectX11-With-Windows-SDK: 现代DX11系列教程：使用Windows SDK(C++)开发Direct3D 11.x](https://github.com/MKXJun/DirectX11-With-Windows-SDK)

使用代码版本：Commits on Jun 29, 2022

commit SHA：fd2745d2b9ca89347b1255276728e0e51ef848f3

作者的部分源码摘自龙书源码，经过修改，目的是完全脱离旧的DirectX SDK，使用Windows SDK来实现DX11。[龙书网站](http://www.d3dcoder.net/)。

务必在项目属性页-C/C++ -命令行中添加`/utf-8`来强制指定代码页。否则会无端报C2065等错。原因是项目中存在不同代码页所致。

![属性配置](https://tva3.sinaimg.cn/large/0077Un8Egy1h5mgbxg0r8j30vv0lsjwd.jpg)

## 01 DirectX11初始化

注意，Direct3D 11.1的API命名要在尾部比Direct3D 11多个1。

| 头文件     | 功能                                                   |
| ---------- | ------------------------------------------------------ |
| d3dApp.h   | Direct3D应用程序框架类                                 |
| d3dUtil.h  | 包含一些常用头文件及自己编写的函数                     |
| DXTrace.h  | 包含了HR宏与`DXTraceW`函数                             |
| GameApp.h  | 游戏应用程序扩展类，游戏逻辑在这里实现，继承自D3DApp类 |
| CpuTimer.h | 游戏计时器类                                           |
| WinMin.h   | 定义宏，去掉Windows中那些没用的组件                    |

d3dApp.cpp持有一个全局变量`D3DApp*  g_pd3dApp`，在构造函数中初始化指向this。它的意义是将成员函数MsgProc的返回值给到函数MainWndProc，后者才能绑定到WNDCLASS::lpfnWndProc上。

### Direct3D初始化

`bool D3DApp::InitDirect3D()`

1. D3D设备与D3D设备上下文的创建。

```C++
HRESULT WINAPI D3D11CreateDevice(
    IDXGIAdapter* pAdapter,             // [In_Opt]适配器
    D3D_DRIVER_TYPE DriverType,         // [In]驱动类型
    HMODULE Software,                   // [In_Opt]若上面为D3D_DRIVER_TYPE_SOFTWARE则这里需要提供程序模块
    UINT Flags,                         // [In]使用D3D11_CREATE_DEVICE_FLAG枚举类型
    D3D_FEATURE_LEVEL* pFeatureLevels,  // [In_Opt]若为nullptr则为默认特性等级，否则需要提供特性等级数组
    UINT FeatureLevels,                 // [In]特性等级数组的元素数目
    UINT SDKVersion,                    // [In]SDK版本，默认D3D11_SDK_VERSION
    ID3D11Device** ppDevice,            // [Out_Opt]输出D3D设备
    D3D_FEATURE_LEVEL* pFeatureLevel,   // [Out_Opt]输出当前应用D3D特性等级
    ID3D11DeviceContext** ppImmediateContext ); //[Out_Opt]输出D3D设备上下文
```

将`IDXGIAdapter*`设为nullptr，使用默认显卡。

注意，**特性等级**和**D3D设备的版本**并不是互相对应的。可以打特定的系统补丁来支持高版本的DX，比如让Win7支持DX12的部分。

2. 检测MSAA支持的质量等级。

3. 创建DXGI交换链。

### OnResize()

在窗口大小发生改变后调用该函数，大致流程如下：

1. 获取交换链后备缓冲区的`ID3D11Texture2D`接口对象
2. 为后备缓冲区创建渲染目标视图`ID3D11RenderTargetView`
3. 通过D3D设备创建一个`ID3D11Texture2D`用作深度/模板缓冲区，要求与后备缓冲区等宽高
4. 创建深度/模板视图`ID3D11DepthStrenilView`，绑定刚才创建的2D纹理
5. 通过D3D设备上下文，在渲染管线的输出合并阶段设置渲染目标
6. 在渲染管线的光栅化阶段设置好渲染的视口区域

### GameApp类

继承自D3DApp类，目前仅实现DrawScene()函数。

```C++
void GameApp::DrawScene()
{
    assert(m_pd3dImmediateContext);
    assert(m_pSwapChain);
    static float blue[4] = { 0.0f, 0.0f, 1.0f, 1.0f };    // RGBA = (0,0,255,255)
    //对后备缓冲区(R8G8B8A8)使用蓝色进行清空
    m_pd3dImmediateContext->ClearRenderTargetView(m_pRenderTargetView.Get(), blue);
    //每一次清空我们需要将深度值设为1.0f，模板值设为0.0f。其中深度值1.0f表示距离最远处
    m_pd3dImmediateContext->ClearDepthStencilView(m_pDepthStencilView.Get(), D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL, 1.0f, 0);

    HR(m_pSwapChain->Present(0, 0));
}
```

## 02 顶点/像素着色器的创建、索引缓冲区

渲染管线如图所示：

![渲染管线](https://tva3.sinaimg.cn/large/0077Un8Egy1h5fdj17h2pj30jf11s41b.jpg)

本章仅考虑如下情况：

![初学者管线](https://directx11.tech/assets/02/02.png)
