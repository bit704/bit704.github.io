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

作者的部分源码摘自d3d教材源码，经过修改，目的是完全脱离旧的DirectX SDK，使用Windows SDK来实现DX11。[d3d教材网站](http://www.d3dcoder.net/)。

务必在项目属性页-C/C++ -命令行中添加`/utf-8`来强制指定代码页。否则会无端报C2065等错。原因是项目中存在不同代码页所致。

![属性配置](https://tva3.sinaimg.cn/large/0077Un8Egy1h5mgbxg0r8j30vv0lsjwd.jpg)

务必在`d3dApp.h`添加：

```C++
#pragma comment(lib, "d3d11.lib")
#pragma comment(lib, "dxgi.lib")
#pragma comment(lib, "dxguid.lib")
#pragma comment(lib, "D3DCompiler.lib")
#pragma comment(lib, "winmm.lib")
```

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

### DrawScene()

目前是GameApp类唯一的函数。

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

### 渲染管线

![渲染管线](https://tva3.sinaimg.cn/large/0077Un8Egy1h5fdj17h2pj30jf11s41b.jpg)

D3D设备可以创建出以上6种着色器

| 方法                               | 着色器类型           | 描述       |
| ---------------------------------- | -------------------- | ---------- |
| ID3D11Device::CreateVertexShader   | ID3D11VertexShader   | 顶点着色器 |
| ID3D11Device::CreateHullShader     | ID3D11HullShader     | 外壳着色器 |
| ID3D11Device::CreateDomainShader   | ID3D11DomainShader   | 域着色器   |
| ID3D11Device::CreateComputeShader  | ID3D11ComputeShader  | 计算着色器 |
| ID3D11Device::CreateGeometryShader | ID3D11GeometryShader | 几何着色器 |
| ID3D11Device::CreatePixelShader    | ID3D11PixelShader    | 像素着色器 |

本章仅考虑如下情况：

![初学者管线](https://directx11.tech/assets/02/02.png)

### 着色器

导入着色器文件后，按以下配置。

[HLSL编译着色器的三种方法](https://directx11.tech/#/misc/Compile)

一共3份着色器文件：

```c
// Triangle.hlsli

struct VertexIn
{
    float3 pos : POSITION;
    float4 color : COLOR;
};

struct VertexOut
{
    float4 posH : SV_POSITION;
    float4 color : COLOR;
};

//Triangle_VS.hlsl
#include "Triangle.hlsli"

// 顶点着色器
VertexOut VS(VertexIn vIn)
{
    VertexOut vOut;
    vOut.posH = float4(vIn.pos, 1.0f);
    vOut.color = vIn.color; // 这里alpha通道的值默认为1.0
    return vOut;
}

//Triangle_PS.hlsl
#include "Triangle.hlsli"

// 像素着色器
float4 PS(VertexOut pIn) : SV_Target
{
    return pIn.color;   
}
```

用到的语义如下：

| 语义名      | 具体含义                                                     |
| ----------- | ------------------------------------------------------------ |
| POSITION    | 描述该变量是一个坐标点                                       |
| SV_POSITION | 说明该顶点的位置在从顶点着色器输出后，后续的着色器都不能改变它的值，作为光栅化时最终确定的像素位置 |
| COLOR       | 描述该变量是一个颜色                                         |
| SV_Target   | 说明输出的颜色值将会直接保存到渲染目标视图的后备缓冲区对应位置 |

创建与HLSL中VertexIn结构体对应的C++结构体：

```c++
//GameApp.h
struct VertexPosColor
{
    DirectX::XMFLOAT3 pos;
    DirectX::XMFLOAT4 color;
    static const D3D11_INPUT_ELEMENT_DESC inputLayout[2];
};
//GameApp.cpp
const D3D11_INPUT_ELEMENT_DESC GameApp::VertexPosColor::inputLayout[2] = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0 },
    { "COLOR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D11_INPUT_PER_VERTEX_DATA, 0 }
};
```

```C++
typedef struct D3D11_INPUT_ELEMENT_DESC
{
    LPCSTR SemanticName;        // 语义名
    UINT SemanticIndex;         // 语义索引,若有多个相同的语义名，按语义索引区分，按从上到下所以分别为0,1,2...
    DXGI_FORMAT Format;         // 数据格式
    UINT InputSlot;             // 输入槽索引(0-15)
    UINT AlignedByteOffset;     // 初始位置(字节偏移量)      
    D3D11_INPUT_CLASSIFICATION InputSlotClass; // 输入类型
    UINT InstanceDataStepRate;  // 忽略
}     D3D11_INPUT_ELEMENT_DESC;
```

### InitEffect()

```C++
bool GameApp::InitEffect()
{
    ComPtr<ID3DBlob> blob;

    // 创建顶点着色器
    HR(CreateShaderFromFile(L"HLSL\\Triangle_VS.cso", L"HLSL\\Triangle_VS.hlsl", "VS", "vs_5_0", blob.ReleaseAndGetAddressOf()));
    HR(m_pd3dDevice->CreateVertexShader(blob->GetBufferPointer(), blob->GetBufferSize(), nullptr, m_pVertexShader.GetAddressOf()));
    // 创建并绑定顶点布局
    HR(m_pd3dDevice->CreateInputLayout(VertexPosColor::inputLayout, ARRAYSIZE(VertexPosColor::inputLayout),
        blob->GetBufferPointer(), blob->GetBufferSize(), m_pVertexLayout.GetAddressOf()));

    // 创建像素着色器
    HR(CreateShaderFromFile(L"HLSL\\Triangle_PS.cso", L"HLSL\\Triangle_PS.hlsl", "PS", "ps_5_0", blob.ReleaseAndGetAddressOf()));
    HR(m_pd3dDevice->CreatePixelShader(blob->GetBufferPointer(), blob->GetBufferSize(), nullptr, m_pPixelShader.GetAddressOf()));

    return true;
}
```

### InitResource()

```c++
bool GameApp::InitResource()
{
    // 设置三角形顶点
    VertexPosColor vertices[] =
    {
        { XMFLOAT3(0.0f, 0.5f, 0.5f), XMFLOAT4(0.0f, 1.0f, 0.0f, 1.0f) },
        { XMFLOAT3(0.5f, -0.5f, 0.5f), XMFLOAT4(0.0f, 0.0f, 1.0f, 1.0f) },
        { XMFLOAT3(-0.5f, -0.5f, 0.5f), XMFLOAT4(1.0f, 0.0f, 0.0f, 1.0f) }
    };
    // 设置顶点缓冲区描述
    D3D11_BUFFER_DESC vbd;
    ZeroMemory(&vbd, sizeof(vbd));
    vbd.Usage = D3D11_USAGE_IMMUTABLE;
    vbd.ByteWidth = sizeof vertices;
    vbd.BindFlags = D3D11_BIND_VERTEX_BUFFER;
    vbd.CPUAccessFlags = 0;
    // 新建顶点缓冲区
    D3D11_SUBRESOURCE_DATA InitData;
    ZeroMemory(&InitData, sizeof(InitData));
    InitData.pSysMem = vertices;
    HR(m_pd3dDevice->CreateBuffer(&vbd, &InitData, m_pVertexBuffer.GetAddressOf()));


    // ******************
    // 给渲染管线各个阶段绑定好所需资源
    //

    // 输入装配阶段的顶点缓冲区设置
    UINT stride = sizeof(VertexPosColor);	// 跨越字节数
    UINT offset = 0;						// 起始偏移量

    m_pd3dImmediateContext->IASetVertexBuffers(0, 1, m_pVertexBuffer.GetAddressOf(), &stride, &offset);
    // 设置图元类型，设定输入布局
    // 按一系列三角形进行装配，每三个顶点(或索引数组每三个索引对应的顶点)构成一个三角形
    m_pd3dImmediateContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
    m_pd3dImmediateContext->IASetInputLayout(m_pVertexLayout.Get());
    // 将着色器绑定到渲染管线
    m_pd3dImmediateContext->VSSetShader(m_pVertexShader.Get(), nullptr, 0);
    m_pd3dImmediateContext->PSSetShader(m_pPixelShader.Get(), nullptr, 0);

    // ******************
    // 设置调试对象名
    //
    D3D11SetDebugObjectName(m_pVertexLayout.Get(), "VertexPosColorLayout");
    D3D11SetDebugObjectName(m_pVertexBuffer.Get(), "VertexBuffer");
    D3D11SetDebugObjectName(m_pVertexShader.Get(), "Trangle_VS");
    D3D11SetDebugObjectName(m_pPixelShader.Get(), "Trangle_PS");

    return true;
}
```

### DrawScene()

使用以下方法进行绘制：

```C++
void ID3D11DeviceContext::Draw( 
    UINT VertexCount,           // [In]需要绘制的顶点数目
    UINT StartVertexLocation);  // [In]起始顶点索引
```

```c++
void GameApp::DrawScene()
{
    assert(m_pd3dImmediateContext);
    assert(m_pSwapChain);
   
    static float black[4] = { 0.0f, 0.0f, 0.0f, 1.0f };    // RGBA = (0,0,0,255)
    m_pd3dImmediateContext->ClearRenderTargetView(m_pRenderTargetView.Get(), black);
    m_pd3dImmediateContext->ClearDepthStencilView(m_pDepthStencilView.Get(), D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL, 1.0f, 0);

    // 绘制三角形
    m_pd3dImmediateContext->Draw(3, 0);
    HR(m_pSwapChain->Present(0, 0));
}
```

## 03 索引缓冲区、常量缓冲区

### InitResource()

顶点缓冲区的创建和使用和之前一样。

使用索引缓冲区可以有效减少顶点缓冲区的占用空间，避免大量重复顶点数据。因此在创建顶点缓冲区后不直接进行输入装配，先针对顶点数组创建索引数组，然后创建索引缓冲区。

```C++
// 设置索引缓冲区描述
D3D11_BUFFER_DESC ibd;
ZeroMemory(&ibd, sizeof(ibd));
ibd.Usage = D3D11_USAGE_IMMUTABLE;
ibd.ByteWidth = sizeof indices;
ibd.BindFlags = D3D11_BIND_INDEX_BUFFER;
ibd.CPUAccessFlags = 0;
// 新建索引缓冲区
InitData.pSysMem = indices;
HR(m_pd3dDevice->CreateBuffer(&ibd, &InitData, m_pIndexBuffer.GetAddressOf()));
```

在HLSL中，常量缓冲区的变量类似于C++的全局常量，供着色器代码使用。

```c
cbuffer ConstantBuffer : register(b0)
{
    matrix g_World; // matrix可以用float4x4替代。不加row_major的情况下，矩阵默认为列主矩阵，
    matrix g_View;  // 可以在前面添加row_major表示行主矩阵
    matrix g_Proj;  // 该教程往后将使用默认的列主矩阵，但需要在C++代码端预先将矩阵进行转置。
}
```

对应的C++结构体：

```c++
struct ConstantBuffer
{
    XMMATRIX world;
    XMMATRIX view;
    XMMATRIX proj;
};
```

### UpdateScene(float dt)

逐帧修改世界矩阵，旋转立方体。

```C++
void GameApp::UpdateScene(float dt)
{
    static float phi = 0.0f, theta = 0.0f;
    phi += 0.0001f, theta += 0.00015f;
    m_CBuffer.world = XMMatrixTranspose(XMMatrixRotationX(phi) * XMMatrixRotationY(theta));
    // 更新常量缓冲区，让立方体转起来
    D3D11_MAPPED_SUBRESOURCE mappedData;
    HR(m_pd3dImmediateContext->Map(m_pConstantBuffer.Get(), 0, D3D11_MAP_WRITE_DISCARD, 0, &mappedData));
    memcpy_s(mappedData.pData, sizeof(m_CBuffer), &m_CBuffer, sizeof(m_CBuffer));
    m_pd3dImmediateContext->Unmap(m_pConstantBuffer.Get(), 0);
}
```

## 作业 1 2

作业1 Rendering a 2D Hanzi

![01 Rendering a 2D Hanzi](https://tva4.sinaimg.cn/large/0077Un8Egy1h5okpl1gmvj318i0q4q95.jpg)

作业2 Rendering a 3D Hanzi

![02 Rendering a 3D Hanzi](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5oktdre8dj318i0q4jue.jpg)



## 04 变换

Direct3D使用**左手坐标系**，而OpenGL与我们平日接触到的数学使用**右手坐标系**。

纹理坐标系：

![纹理坐标系](https://directx11.tech/assets/04/02.png)

屏幕坐标系：

![屏幕坐标系](https://directx11.tech/assets/04/03.png)

的DirectXMath数学库中生成的矩阵都是行矩阵。这也意味着矩阵乘法通常被表示为行向量乘以行矩阵的形式。这不仅在编写C++的代码中有所体现，在HLSL中我们也将习惯写成上述形式。

**齐次坐标（homogeneous coordinate）**：

1. (x,y,z,0)**表示向量**
2. (x,y,z,1)**表示点**

线性映射（ linear mapping）是从一个向量空间V到另一个向量空间W的映射且保持加法运算和数量乘法运算，而线性变换（linear transformation）是线性空间V到其自身的线性映射。

仿射变换=线性变换+平移

投影部分可以看[GAMES101: 现代计算机图形学入门](https://sites.cs.ucsb.edu/~lingqi/teaching/games101.html)

## 05 DirectXMath数学库

即DirectXMath.h。

### SIMD与SSE2

**SIMD**(Single Instruction Multiple Data)即单指令流多数据流，是一种采用一个控制器来控制多个处理器，同时对一组数据（又称“数据向量”）中的每一个分别执行相同的操作从而实现空间上的并行性的技术。简单来说就是一个指令能够同时处理多个数据。

**SSE2**(Streaming SIMD Extensions 2）指令集是Intel公司在SSE指令集的基础上发展起来的。

### 向量

运算向量

```c++
typedef __m128 XMVECTOR;

typedef union __declspec(intrin_type) __declspec(align(16)) __m128 {
    float               m128_f32[4];
    unsigned __int64    m128_u64[2];
    __int8              m128_i8[16];
    __int16             m128_i16[8];
    __int32             m128_i32[4];
    __int64             m128_i64[2];
    unsigned __int8     m128_u8[16];
    unsigned __int16    m128_u16[8];
    unsigned __int32    m128_u32[4];
} __m128;
```

存储向量

```c++
2D向量: XMFLOAT2(常用), XMINT2, XMUINT2
3D向量: XMFLOAT3(常用), XMINT3, XMUINT3
4D向量: XMFLOAT4(常用), XMINT4, XMUINT4
```

在需要时进行相互转换。

对于自定义函数，如果需要传入`XMVECTOR`的话，前3个向量需要使用`FXMVECTOR`，第4个向量需要使用`GXMVECTOR`，第5-6个需要使用`HXMVECTOR`，从第7个开始则使用`CXMVECTOR`。这是为了使代码更具通用性，不受具体平台、编译器的影响，基于特定的平台和编译器，它们会被自动地定义为适当的类型。一定要把调用约定注解XM_CALLCONV加在函数名之前，它会根据编译器的版本确定出对应的调用约定属性。

```c++
#if _XM_VECTORCALL_
#define XM_CALLCONV __vectorcall
#else
#define XM_CALLCONV __fastcall
```

### 矩阵

`XMMATRIX`实际上里面是由4个`XMVECTOR`的数组构成的结构体。

## 06 使用ImGui

使用ImGui进行交互。

## 作业 3

作业3 Rendering Much 3D Hanzi

![03 Rendering Much 3D Hanzi](https://tvax2.sinaimg.cn/large/0077Un8Egy1h5vvv85vfjj318i0q4x2c.jpg)

## 07 光照、常用几何模型、光栅化状态

