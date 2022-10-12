---
title: DX11-Practice
categories: [Notebook]
tags: [DirectX,CG]
---

DX11练手
<!-- more -->

[toc]

## 前言

参考教程：[DirectX11 With Windows SDK](https://directx11.tech/#/)

教程代码仓库：[MKXJun/DirectX11-With-Windows-SDK: 现代DX11系列教程：使用Windows SDK(C++)开发Direct3D 11.x](https://github.com/MKXJun/DirectX11-With-Windows-SDK)

练习仓库：[bit704/DX11-Practice: DX11练习](https://github.com/bit704/DX11-Practice)

使用代码版本：Commits on Jun 29, 2022

commit SHA：fd2745d2b9ca89347b1255276728e0e51ef848f3

使用IDE：VS2019

作者的部分源码摘自d3d教材源码，经过修改，目的是完全脱离旧的DirectX SDK，使用Windows SDK来实现DX11。[d3d教材网站](http://www.d3dcoder.net/)。

务必在项目属性页-C/C++ -命令行中添加`/utf-8`来强制指定代码页。否则会无端报C2065等错。原因是项目中存在不同代码页所致。

![项目属性设置](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E9%A1%B9%E7%9B%AE%E5%B1%9E%E6%80%A7%E8%AE%BE%E7%BD%AE.jpg)

务必在`d3dApp.h`添加：

```C++
#pragma comment(lib, "d3d11.lib")
#pragma comment(lib, "dxgi.lib")
#pragma comment(lib, "dxguid.lib")
#pragma comment(lib, "D3DCompiler.lib")
#pragma comment(lib, "winmm.lib")
```

或者在项目->属性->链接器->输入->附加依赖项中加入。

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

在初始化Direct3D时和窗口大小发生改变后调用该函数，大致流程如下：

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

![渲染流水线的各个阶段](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E6%B8%B2%E6%9F%93%E6%B5%81%E6%B0%B4%E7%BA%BF%E7%9A%84%E5%90%84%E4%B8%AA%E9%98%B6%E6%AE%B5.jpg)

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

![初学者管线](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%88%9D%E5%AD%A6%E8%80%85%E7%AE%A1%E7%BA%BF.png)

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

![01 Rendering a 2D Hanzi](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-01%20Rendering%20a%202D%20Hanzi.png)

作业2 Rendering a 3D Hanzi

![02 Rendering a 3D Hanzi](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-02%20Rendering%20a%203D%20Hanzi.png)



## 04 变换

Direct3D使用**左手坐标系**，而OpenGL与我们平日接触到的数学使用**右手坐标系**。

纹理坐标系：

![纹理坐标系](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E7%BA%B9%E7%90%86%E5%9D%90%E6%A0%87%E7%B3%BB.png)

屏幕坐标系：

![屏幕坐标系](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%B1%8F%E5%B9%95%E5%9D%90%E6%A0%87%E7%B3%BB.png)

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

使用ImGui库编写UI。

## 作业 3 4

作业3 Rendering Much 3D Hanzi

每个汉字有两个汉字围绕其旋转

![03 Rendering Much 3D Hanzi](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-03%20Rendering%20Much%203D%20Hanzi.png)

作业4 Roaming 

实现漫游

WASD 水平前后左右移动     QE 滚筒旋转      视角跟随鼠标移动

![04 Roaming](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-04%20Roaming.png)

## 07 光照、常用几何模型、光栅化状态

一定要注意[HLSL常量缓冲区打包规则](https://directx11.tech/#/misc/Packing?id=hlsl常量缓冲区打包规则)。

**颜色向量**：$(red, green, blue, alpha)$。对于RGB，0代表没有颜色，1代表纯色。对于alpha，0代表透明，1代表不透明。

若是立方体，则一个顶点同时在三个面上，则拥有三个**法向量**。

```c++
//C++
struct VertexPosNormalColor
{
    DirectX::XMFLOAT3 pos;
    DirectX::XMFLOAT3 normal;
    DirectX::XMFLOAT4 color;
    static const D3D11_INPUT_ELEMENT_DESC inputLayout[3];
};
const D3D11_INPUT_ELEMENT_DESC VertexPosNormalColor::inputLayout[3] = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0 },
    { "NORMAL", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12, D3D11_INPUT_PER_VERTEX_DATA, 0},
    { "COLOR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 24, D3D11_INPUT_PER_VERTEX_DATA, 0}
};
//HLSL
struct VertexIn
{
    float3 posL : POSITION;
    float3 normalL : NORMAL;
    float4 color : COLOR;
};
```

**材质**属性决定了各种颜色分量的反射系数是多少，其中红绿蓝每个分量的取值范围为`[0.0f, 1.0f]`。其中分别代表环境光、漫反射光、高光的反射率。

```c++
//C++
struct Material
{
    DirectX::XMFLOAT4 ambient;
    DirectX::XMFLOAT4 diffuse;
    DirectX::XMFLOAT4 specular;
    DirectX::XMFLOAT4 reflect;
};
//HLSL
struct Material
{
    float4 Ambient;
    float4 Diffuse;
    float4 Specular;
    float4 Reflect;
};
```

三种光源：

方向光：

- 所有位置方向一样
- 环境光直接用材质反射率乘光源颜色，漫反射光用兰伯特模型，高光用blinn-phong模型

点光：

- 每个位置的方向需要单独求
- 方向光基础上漫反射光和高光随距离d衰弱：$I(d)=\frac{I_0}{a_0+a_1d+d^2}$，并通过灯光范围测试减少计算量

聚光灯：

- 点光基础上整个光照乘一个强度因子：$k_{spot}=max(-L\cdot d,0)^{spot}$，L是光向量，d是照射强度，spot是聚光程度

**光栅化阶段**在像素着色器之前，是不可编程的，但是可以设置光栅化状态。

```c
typedef struct D3D11_RASTERIZER_DESC
{
    D3D11_FILL_MODE FillMode;          // 填充模式
    D3D11_CULL_MODE CullMode;          // 裁剪模式
    BOOL FrontCounterClockwise;        // 是否三角形顶点按逆时针排布时为正面
    INT DepthBias;                     // 深度偏移相关，目前忽略
    FLOAT DepthBiasClamp;              // 深度偏移相关，目前忽略
    FLOAT SlopeScaledDepthBias;        // 深度偏移相关，目前忽略
    BOOL DepthClipEnable;              // 是否允许深度测试将范围外的像素进行裁剪，默认TRUE
    BOOL ScissorEnable;                // 是否允许指定矩形范围的裁剪，若TRUE，则需要在RSSetScissor设置像素保留的矩形区域
    BOOL MultisampleEnable;            // 是否允许多重采样
    BOOL AntialiasedLineEnable;        // 是否允许反走样线，仅当多重采样为FALSE时才有效
}D3D11_RASTERIZER_DESC;
```

| 枚举值                   | 含义         |
| ------------------------ | ------------ |
| D3D11_FILL_WIREFRAME = 2 | 线框填充方式 |
| D3D11_FILL_SOLID = 3     | 面填充方式   |

| 枚举值               | 含义                                                   |
| -------------------- | ------------------------------------------------------ |
| D3D11_CULL_NONE = 1  | 无背面裁剪，即三角形无论处在视野的正面还是背面都能看到 |
| D3D11_CULL_FRONT = 2 | 对处在视野正面的三角形进行裁剪                         |
| D3D11_CULL_BACK = 3  | 对处在视野背面的三角形进行裁剪                         |

创建及设置光栅化状态：

```c++
HRESULT ID3D11Device::CreateRasterizerState( 
    const D3D11_RASTERIZER_DESC *pRasterizerDesc,    // [In]光栅化状态描述
    ID3D11RasterizerState **ppRasterizerState) = 0;  // [Out]输出光栅化状态

void ID3D11DeviceContext::RSSetState(
  ID3D11RasterizerState *pRasterizerState);  // [In]光栅化状态，若为nullptr，则使用默认光栅化状态
```

## 作业 5

这里为了省事对于在多个面上的顶点只赋了一条法线。朝向屏幕外侧的面的顶点赋(0.f,0.f,-1.f)，内侧的赋(0.f,0.f,1.f)。

作业5 Light

通过数字键控制3种光源开关。1：开关方向光（猩红）；2：开关点光（板岩暗蓝灰）； 3：开关聚光（金）。

 三种光源默认开启，开关情况在左上角的UI界面底部显示。

方向光1个，方向固定。

点光3个，在字符矩阵上做随机运动。

聚光1个，跟随相机照亮前方。

使用Blinn-Phong替换Phong计算光照。

![05 Lighting](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-20-05%20Lighting.png)

## 08 几何着色器

在上一节的基础上添加几何着色器，需要以下步骤：

1. 创建Light_GS.hlsl文件，在文件的属性中配置好着色器编译后对应的**对象文件名**HLSL\%(Filename).cso、**入口点名称**GS、**着色器类型**/gs、**着色器名称**/5_0。

2. 在C++端创建几何着色器

   ```c++
   // 创建几何着色器
   HR(CreateShaderFromFile(L"HLSL\\Light_GS.cso", L"HLSL\\Light_GS.hlsl", "GS", "gs_5_0", blob.ReleaseAndGetAddressOf()));
   HR(m_pd3dDevice->CreateGeometryShader(blob->GetBufferPointer(), blob->GetBufferSize(), nullptr, m_pGeometryShader.GetAddressOf()));
   ```

3. 在C++绑定几何着色器到渲染管线上并设置它的常量缓冲区

   ```c++
   // 绑定几何着色器
   m_pd3dImmediateContext->GSSetShader(m_pGeometryShader.Get(), nullptr, 0);
   // GS常量缓冲区对应HLSL寄存于b0的常量缓冲区
   m_pd3dImmediateContext->GSSetConstantBuffers(0, 1, m_pConstantBuffers[0].GetAddressOf());
   ```

4. 设置调试对象名

   ```c++
   D3D11SetDebugObjectName(m_pGeometryShader.Get(), "Light_GS");
   ```

5. 编写shader

   ```c
   #include "Light.hlsli"
   
   /* 
       对任意平面 Ax+By+Cz+D=0 求镜像点
   */
   void Symmetry(in float3 p, out float3 p_s)
   {
       float A = 0.f, B = 0.f, C = 1.f, D = -2.f;
       float f1 = A * A + B * B + C * C;
       float f2 = A * p.x + B * p.y + C * p.z + D;
       p_s.x = p.x - 2 * A * (f2) / f1;
       p_s.y = p.y - 2 * B * (f2) / f1;
       p_s.z = p.z - 2 * C * (f2) / f1;
   }
   
   
   [maxvertexcount(6)]
   void GS(
   	triangle VertexOut input[3],
   	inout TriangleStream<VertexOut> output
   )
   {
       VertexOut vOri[3], vNew[3];
       
   	[unroll]
       for (uint i = 0; i < 3; i++)
   	{
           vOri[i] = input[i];
           output.Append(vOri[i]);
           
           //镜像顶点
           Symmetry(input[i].posW, vNew[i].posW);
           
           //世界到裁剪矩阵
           matrix viewProj = mul(g_View, g_Proj);
           vNew[i].posH = mul(float4(vNew[i].posW, 1.0f), viewProj);
           
           //顶点自身颜色由白色改为 LimeGreen 酸橙绿
           vNew[i].color = float4(0.2f, 0.8f, 0.2f, 1.f);
           
           //镜像法线
           float3 normalWSta = input[i].posW;
           float3 normalWEnd = input[i].posW + input[i].normalW;
           Symmetry(normalWEnd, normalWEnd);
           vNew[i].normalW = normalWEnd - vNew[i].posW;
           
       }
       
       //在以线段或者三角形作为图元的时候，默认是以strip的形式输出的，如果我们不希望下一个输出的顶点与之前的顶点构成新图元，则需要调用此方法来重新开始新的strip。若希望输出的图元类型也保持和原来一样的TriangleList，则需要每调用3次Append方法后就调用一次RestartStrip。
       output.RestartStrip();
       
       [unroll]
       for (uint i = 0; i < 3; i++)
       {
           output.Append(vNew[i]);
       }
   }
   ```

## 作业6 

利用几何着色器对字符森林关于z=2平面作镜像。

![06 Geometry](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-26-06%20Geometry.png)

## 09 纹理

包含`DDSTextureLoader11.h`和`WICTextureLoader11.h`进项目，以读取创建纹理。

> 第一种：在[DirectXTex](https://github.com/Microsoft/DirectXTex)中找到`DDSTextureLoader`文件夹和`WICTextureLoader`文件夹中分别找到对应的头文件和源文件(不带12的)，并加入到你的项目中。
>
> 第二种：将[DirectXTK](https://github.com/Microsoft/DirectXTK)库添加到你的项目中。

为了方便贴图，仅渲染一个平面贴上照片。

汉字使用的顶点类型为`VertexPosNormalColor`，平面使用的顶点类型为 `VertexPosNormalTex`，因此有两套着色器和C++端输入装配阶段。

对于 `VertexPosNormalTex`类型顶点：

1. 在着色器端增加纹理和采样器状态

```c
Texture2D g_Tex : register(t0);
SamplerState g_SamLinear : register(s0);
```

2. 在C++端初始化纹理和采样器状态

```c++
//GameApp.h
ComPtr<ID3D11ShaderResourceView> m_pPhoto;			    // 照片纹理
ComPtr<ID3D11SamplerState> m_pSamplerState;				// 采样器状态

//GameApp.cpp 
//bool GameApp::InitResource()

// 初始化照片纹理
HR(CreateWICTextureFromFile(m_pd3dDevice.Get(), L"Texture\\photo.png", nullptr, m_pPhoto.GetAddressOf()));

// 初始化采样器状态
D3D11_SAMPLER_DESC sampDesc;
ZeroMemory(&sampDesc, sizeof(sampDesc));
sampDesc.Filter = D3D11_FILTER_MIN_MAG_MIP_LINEAR;
sampDesc.AddressU = D3D11_TEXTURE_ADDRESS_WRAP;
sampDesc.AddressV = D3D11_TEXTURE_ADDRESS_WRAP;
sampDesc.AddressW = D3D11_TEXTURE_ADDRESS_WRAP;
sampDesc.ComparisonFunc = D3D11_COMPARISON_NEVER;
sampDesc.MinLOD = 0;
sampDesc.MaxLOD = D3D11_FLOAT32_MAX;
HR(m_pd3dDevice->CreateSamplerState(&sampDesc, m_pSamplerState.GetAddressOf()));

// PS设置采样器和纹理
m_pd3dImmediateContext->PSSetSamplers(0, 1, m_pSamplerState.GetAddressOf());
m_pd3dImmediateContext->PSSetShaderResources(0, 1, m_pPhoto.GetAddressOf());

//调试
D3D11SetDebugObjectName(m_pSamplerState.Get(), "SSLinearWrap");
```

3. 每次渲染平面时单独设置着色器和输入装配

```c++
//设置Plane
void GameApp::SetPlane()
{
    // ******************
    // 设置顶点缓冲区和索引缓冲区
    //
    
    // 释放旧资源
    m_pVertexBuffer.Reset();
    m_pIndexBuffer.Reset();

    // 创建Plane顶点
    VertexPosNormalTex* vertices = nullptr;
    Plane::CreateVertexs_PosNormalTex(&vertices, m_VertexCount_Plane);

    // 设置顶点缓冲区描述
    D3D11_BUFFER_DESC vbd;
    ZeroMemory(&vbd, sizeof(vbd));
    vbd.Usage = D3D11_USAGE_IMMUTABLE;
    vbd.ByteWidth = m_VertexCount_Plane * sizeof VertexPosNormalTex;
    vbd.BindFlags = D3D11_BIND_VERTEX_BUFFER;
    vbd.CPUAccessFlags = 0;
    // 新建顶点缓冲区
    D3D11_SUBRESOURCE_DATA InitData;
    ZeroMemory(&InitData, sizeof(InitData));
    InitData.pSysMem = vertices;
    HR(m_pd3dDevice->CreateBuffer(&vbd, &InitData, m_pVertexBuffer.GetAddressOf()));

    // 输入装配阶段的顶点缓冲区设置
    UINT stride = sizeof(VertexPosNormalTex);	// 跨越字节数
    UINT offset = 0;							// 起始偏移量

    m_pd3dImmediateContext->IASetVertexBuffers(0, 1, m_pVertexBuffer.GetAddressOf(), &stride, &offset);

    // 创建Plane索引
    DWORD* indices = nullptr;
    Plane::CreateIndices(&indices, m_IndexCount_Plane);

    // 设置索引缓冲区描述
    D3D11_BUFFER_DESC ibd;
    ZeroMemory(&ibd, sizeof(ibd));
    ibd.Usage = D3D11_USAGE_IMMUTABLE;
    ibd.ByteWidth = m_IndexCount_Plane * sizeof(DWORD);
    ibd.BindFlags = D3D11_BIND_INDEX_BUFFER;
    ibd.CPUAccessFlags = 0;
    // 新建索引缓冲区
    InitData.pSysMem = indices;
    HR(m_pd3dDevice->CreateBuffer(&ibd, &InitData, m_pIndexBuffer.GetAddressOf()));
    // 输入装配阶段的索引缓冲区设置
    m_pd3dImmediateContext->IASetIndexBuffer(m_pIndexBuffer.Get(), DXGI_FORMAT_R32_UINT, 0);

    // 设置调试对象名
    D3D11SetDebugObjectName(m_pVertexBuffer.Get(), "VertexBuffer");
    D3D11SetDebugObjectName(m_pIndexBuffer.Get(), "IndexBuffer");

    //释放堆内存
    delete [] vertices;
    delete [] indices;

    // ******************
    // 给渲染管线各个阶段绑定好所需资源
    //

    // 设置图元类型，设定输入布局
    m_pd3dImmediateContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
    m_pd3dImmediateContext->IASetInputLayout(m_pVertexLayout_Plane.Get());

    // 绑定VS
    m_pd3dImmediateContext->VSSetShader(m_pVertexShader_Plane.Get(), nullptr, 0);
    // VS常量缓冲区对应HLSL寄存于b0的常量缓冲区
    m_pd3dImmediateContext->VSSetConstantBuffers(0, 1, m_pConstantBuffers[0].GetAddressOf());
    // 绑定PS
    m_pd3dImmediateContext->PSSetShader(m_pPixelShader_Plane.Get(), nullptr, 0);
    // PS常量缓冲区对应HLSL寄存于b1的常量缓冲区
    m_pd3dImmediateContext->PSSetConstantBuffers(1, 1, m_pConstantBuffers[1].GetAddressOf());
    // PS设置采样器和纹理
    m_pd3dImmediateContext->PSSetSamplers(0, 1, m_pSamplerState.GetAddressOf());
    m_pd3dImmediateContext->PSSetShaderResources(0, 1, m_pPhoto.GetAddressOf());

    // ******************
    // 设置调试对象名
    //
    D3D11SetDebugObjectName(m_pVertexLayout_Plane.Get(), "VertexPosNormalColorLayout_Plane");
    D3D11SetDebugObjectName(m_pVertexShader_Plane.Get(), "VS_Plane");
    D3D11SetDebugObjectName(m_pPixelShader_Plane.Get(), "PS_Plane");

    //更新常量缓冲区
    UpdateConstantBuffer();
    // 绘制顶点
    m_pd3dImmediateContext->DrawIndexed(m_IndexCount_Plane, 0, 0);
}
```

## 作业7

添加一个正方形面片，贴上照片纹理，放置在x=1平面处，两侧字符森林通过GS以X=1平面对称。

在PS常量缓冲区增加g_Time变量传入时间，随时间在PS里移动纹理坐标实现平移动画。

![07 Texture](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-10-08-07%20Texture.png)
