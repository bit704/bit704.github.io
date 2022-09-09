---
title: 《Unity Shader入门精要》笔记
categories: [Notebook]
tags: [Unity]
cover: https://tvax1.sinaimg.cn/large/0077Un8Egy1h5wngmmjh7j30rs140tad.jpg
banner: https://tvax1.sinaimg.cn/large/0077Un8Egy1h5wngmmjh7j30rs140tad.jpg
---

作者    冯乐乐

<!--more-->

人民邮电出版社

![封面](https://wfqqreader-1252317822.image.myqcloud.com/cover/473/22691473/t6_22691473.jpg)

[代码仓库](https://github.com/candycat1992/Unity_Shaders_Book)

[toc]

## 前言

概括来说，Unity Shader是Shader上层的一个抽象。

## 第一篇 基础篇

### 第一章 欢迎来到shader的世界

我们之所以会觉得学习Shader比学习C#这样的编程语言更加困难，一个原因是因为Shader需要牵扯到整个渲染流程。当学习C++、C#这样的高级语言时，我们可以在不了解计算机架构的情况下仍然编写出实现各种功能的代码，这样的高级语言更符合人类的思维方式。然而，Shader并不是这样的。我们之所以要学习Shader，是想要学习如何把物体按照自己的意愿渲染到屏幕上，但是，Shader只是整个渲染流程中的一个子部分。虽然它很关键，但想要学习它，我们就需要了解整个渲染流程是如何进行的。和C++这样的高级语言不同，尽管Shader的编写语言已经达到了我们可以理解的程度，但Shader更多地是面向GPU的工作方式，所以它的一些语法对我们来说并不那么直观。

### 第二章 渲染流水线

#### 2.1 综述

《Render-Time Rendering, Third Edition》一书中将一个渲染流程分成3个**概念**阶段：

- 应用阶段（Application Stage）

  这个阶段由应用主导，通常由CPU负责实现。

  - 准备好场景数据
  - culling
  - 设置材质、纹理、shader
  - 输出渲染图元

- 几何阶段（Geometry Stage）

  负责和每个渲染图元打交道，进行逐顶点、逐多边形的操作。通常在GPU上进行。

  - 通过对输入的渲染图元进行多步处理后，这一阶段将会输出屏幕空间的二维顶点坐标、每个顶点对应的深度值、着色等相关信息，并传递给下一个阶段。

- 光栅化阶段（Rasterizer Stage）

  决定每个渲染图元中的哪些像素应该被绘制在屏幕上，在GPU上运行。

  - 对上一个阶段得到的逐顶点数据（例如纹理坐标、顶点颜色等）进行插值，然后再进行逐像素处理。

![渲染流水线中的3个概念阶段](https://tvax2.sinaimg.cn/large/0077Un8Egy1h5xv2k3kn8j30s609yaal.jpg)

#### 2.2 CPU和GPU之间的通信

渲染流水线的起点是CPU，即应用阶段。应用阶段大致可分为下面3个阶段：

（1）把数据加载到显存中。

（2）设置渲染状态。

（3）调用Draw Call。

#### 2.3 GPU流水线

![GPU的渲染流水线实现](https://tvax2.sinaimg.cn/large/0077Un8Egy1h5xv2x3z20j30rs0c93za.jpg)

1. 顶点着色器（Vertex Shader）是完全可编程的，它通常用于实现顶点的**空间变换、顶点着色**等功能。一个最基本的顶点着色器必须完成的一个工作是，把顶点坐标**从模型空间转换到齐次裁剪空间**，接着通常再由硬件做透视除法后，最终得到归一化的设备坐标（Normalized Device Coordinates , NDC）。

   ![变换到齐次裁剪坐标空间](https://tva1.sinaimg.cn/large/0077Un8Egy1h5xwkmm53yj30rs0a8wex.jpg)

   **图中给出的坐标范围是OpenGL同时也是Unity使用的NDC，它的z分量范围在[-1, 1]之间，而在DirectX中，NDC的z分量范围是[0, 1]。**

2. 曲面细分着色器（Tessellation Shader）是一个可选的着色器，它用于细分图元。

3. 几何着色器（Geometry Shader）同样是一个可选的着色器，它可以被用于执行逐图元（Per-Primitive）的着色操作，或者被用于产生更多的图元。

4. 裁剪（Clipping）的目的是将那些不在摄像机视野内的顶点裁剪掉，并剔除某些三角图元的面片。这个阶段是可配置的。例如，我们可以使用自定义的裁剪平面来配置裁剪区域，也可以通过指令控制裁剪三角图元的正面还是背面。

5. 屏幕映射（Screen Mapping）是不可配置和编程的，它负责把每个图元的坐标转换到屏幕坐标系中。**OpenGL把屏幕的左下角当成最小的窗口坐标值，而DirectX则定义了屏幕的左上角为最小的窗口坐标值。**

6. 三角形设置（Triangle Setup）和三角形遍历（Triangle Traversal）阶段都是固定函数（Fixed-Function）的阶段。以三角形网格表示数据，找到哪些像素被三角网格覆盖。对应像素会生成一个片元，而片元中的状态是对3个顶点的信息迚行插值得到的。

7. 片元着色器（Fragment Shader）是完全可编程的，它用于实现逐片元（Per-Fragment）的着色操作。在DirectX中，片元着色器被称为**像素着色器**（Pixel Shader），但片元着色器是一个更合适的名字，因为此时的片元并不是一个真正意义上的像素。为了在片元着色器中进行**纹理采样**，我们通常会在顶点着色器阶段输出每个顶点对应的纹理坐标，然后经过光栅化阶段对三角网格的3个顶点对应的纹理坐标进行插值后，就可以得到其覆盖的片元的纹理坐标。其局限在于仅可以影响单个片元，不可以将自己的任何结果直接发送给它的邻居们。有一个情况例外，就是片元着色器可以访问到导数信息（gradient，或者说是derivative）。

8. 逐片元操作（Per-Fragment Operations）阶段负责执行很多重要的操作，例如修改颜色、深度缓冲、进行混合等，它不是可编程的，但具有很高的可配置性。**逐片元操作（Per-Fragment Operations）是OpenGL中的说法，在DirectX中，这一阶段被称为输出合并阶段（Output-Merger）。**

   片元 -> 模板测试-> 深度测试-> 混合 -> 颜色缓冲区

   ![模板测试和深度测试](https://tvax1.sinaimg.cn/large/0077Un8Egy1h5y6l8etzpj30rs0rbmz3.jpg)

   对于不透明物体，开发者可以关闭混合（Blend）操作。这样片元着色器计算得到的颜色值就会直接覆盖掉颜色缓冲区中的像素值。但对于半透明物体，我们就需要使用混合操作来让这个物体看起来是透明的。

   将深度测试提前执行的技术通常也被称为Early-Z技术。

9. 为了避免我们看到那些正在进行光栅化的图元，GPU会使用双重缓冲（Double Buffering）的策略。这意味着，对场景的渲染是在幕后发生的，即在后置缓冲（Back Buffer）中。一旦场景已经被渲染到了后置缓冲中，GPU就会交换后置缓冲区和前置缓冲（Front Buffer）中的内容，而前置缓冲区是之前显示在屏幕上的图像。

#### 2.4 一些容易困惑的地方

OpenGL和DirectX是图像应用编程接口，在硬件的基础上实现了一层抽象。这些接口用于渲染二维或三维图形。   

一个应用程序向这些接口发送渲染命令，而这些接口会依次向显卡驱动（Graphics Driver）发送渲染命令，这些显卡驱动是真正知道如何和GPU通信的角色，正是它们把OpenGL或者DirectX的函数调用翻译成了GPU能够听懂的语言，同时它们也负责把纹理等数据转换成GPU所支持的格式。

着色语言是专门用于编写着色器的，常见的着色语言有DirectX的HLSL（High Level Shading Language）、OpenGL的GLSL（OpenGL Shading Language）以及NVIDIA的CG（C for Graphic）。HLSL、GLSL、CG都是“高级（High-Level）”语言，但这种高级是相对于汇编语言来说的，而不是像C#相对于C的高级那样。这些语言会被编译成与机器无关的汇编语言，也被称为中间语言（Intermediate Language, IL）。这些中间语言再交给显卡驱动来翻译成真正的机器语言，即GPU可以理解的语言。

在游戏开发过程中，为了减少Draw Call的开销，有两点需要注意。（1）避免使用大量很小的网格。当不可避免地需要使用很小的网格结构时，考虑是否可以合并它们。（2）避免使用过多的材质。尽量在不同的网格之间共用同一个材质。

#### 2.5 那么，你明白什么是Shader了吗

Shader就是：

- GPU流水线上一些可高度编程的阶段，而由着色器编译出来的最终代码是会在GPU上运行的（对于固定管线的渲染来说，着色器有时等同于一些特定的渲染设置）；
- 有一些特定类型的着色器，如顶点着色器、片元着色器等；
- 依靠着色器我们可以控制流水线中的渲染细节，例如用顶点着色器来进行顶点变换以及传递数据，用片元着色器来进行逐像素的渲染。

### 第3章 Unity Shader基础

#### 3.1 Unity Shader概述

使用流程：

（1）创建一个材质；

（2）创建一个Unity Shader，并把它赋给上一步中创建的材质；

（3）把材质赋给要渲染的对象；

（4）在材质面板中调整Unity Shader的属性，以得到满意的效果。

在Unity 5.2及以上版本中，Unity一共提供了4种Unity Shader模板供我们选择——Standard Surface Shader, Unlit Shader, Image Effect Shader以及Compute Shader。

#### 3.2 Unity Shader的基础：ShaderLab

Unity Shader是Unity为开发者提供的高层级的渲染抽象层。

从设计上来说，ShaderLab类似于CgFX和Direct3D Effects（.FX）语言，它们都定义了要显示一个材质所需的所有东西，而不仅仅是着色器代码。

Unity在背后会根据使用的平台来把这些结构编译成真正的代码和Shader文件，而开发者只需要和Unity Shader打交道即可。

#### 3.3 Unity Shader的结构

一个Unity Shader的基础结构:

```c
//第一行都需要通过Shader语义来指定该Unity Shader的名字。
//通过在字符串中添加斜杠（“/”），可以控制Unity Shader在材质面板中出现的位置。
Shader  "category/ShaderName"  {            
    Properties  {              
        // 属性            
    }            
    SubShader  {                
        // 显卡A使用的子着色器            
    }            
    SubShader  {                
        // 显卡B使用的子着色器            
    }            
    Fallback  "VertexLit"        
}
```

这些属性将会出现在材质面板中。

```c
Shader  "Custom/ShaderLabProperties"  {
    Properties  {
        //  Name  ("display  name",  PropertyType)  =  DefaultValue
        //  Numbers  and  Sliders              
        _Int  ("Int",  Int)  =  2              
        _Float  ("Float",  Float)  =  1.5              
        _Range("Range",  Range(0.0,  5.0))  =  3.0              
        //  Colors  and  Vectors              
        _Color  ("Color",  Color)  =  (1,1,1,1)             
        _Vector  ("Vector",  Vector)  =  (2,  3,  6,  1)              
        //  Textures              
        _2D  ("2D",  2D)  =  ""  {}              
        _Cube  ("Cube",  Cube)  =  "white"  {}              
        _3D  ("3D",  3D)  =  "black"  {}            
    }            
    FallBack  "Diffuse"        
}
```

每一个Unity Shader文件可以包含多个SubShader语义块，但最少要有一个。当Unity需要加载这个Unity Shader时，Unity会扫描所有的SubShader语义块，然后选择第一个能够在目标平台上运行的SubShader。如果都不支持的话，Unity就会使用**Fallback**语义指定的Unity Shader。也可以使用Fallback Off关闭Fallback功能。

```c
SubShader  {            
    // 可选的            
    [Tags]            
    // 可选的            
    [RenderSetup]            
    Pass  {            
    }            
    //  Other  Passes        
}
```

SubShader中定义了一系列**Pass**以及可选的**状态**（[RenderSetup]）和**标签**（[Tags]）设置。

每个Pass定义了一次完整的渲染流程，但如果Pass的数目过多，往往会造成渲染性能的下降。因此，我们应尽量使用最小数目的Pass。

状态和标签同样可以在Pass声明。对于状态设置来说语法相同，SubShader得设置会用于所有的Pass。但是，SubShader中的一些标签设置是特定的，这些标签设置和Pass中使用的标签不一样。

**状态**

ShaderLab提供了一系列渲染状态的设置指令，这些指令可以设置显卡的各种状态，例如是否开启混合/深度测试等。

![常见的渲染状态设置选项](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z0e7pocoj30rs06cgme.jpg)

**标签**

标签是一个键值对，它的键和值都是字符串类型。这些键值对是SubShader和渲染引擎之间的沟通桥梁。

![SubShader的标签类型](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z0es2lkfj30rs0hhmz0.jpg)

**Pass**

```c
Pass  {            
    [Name]            
    [Tags]            
    [RenderSetup]            
    //  Other  code        
}
```

定义名称

```c
Name  "MyPassName"
```

通过这个名称，我们可以使用ShaderLab的UsePass命令来直接使用其他Unity Shader中的Pass。

```c
UsePass  "MyShader/MYPASSNAME"
```

由于Unity内部会把所有Pass的名称转换成大写字母的表示，因此，在使用UsePass命令时必须使用**大写形式的名字**。

![Pass的标签类型](https://tvax3.sinaimg.cn/large/0077Un8Egy1h5z0fwgruaj30rs04z0tj.jpg)

Unity Shader还支持一些特殊的Pass:

- UsePass：如我们之前提到的一样，可以使用该命令来复用其他Unity Shader中的Pass
- GrabPass：该Pass负责抓取屏幕并将结果存储在一张纹理中，以用于后续的Pass处理

#### 3.4 Unity Shader的形式

表面着色器是Unity对顶点/片元着色器的更高一层的抽象。

示例：

```c
Shader  "Custom/Simple  Surface  Shader"  {            
    SubShader  {
        Tags  {  "RenderType"  =  "Opaque"  }
        CGPROGRAM
        #pragma  surface  surf  Lambert
        struct  Input  {
        	float4  color  :  COLOR;
        };              
        void  surf  (Input  IN,  inout  SurfaceOutput  o)  {
        	o.Albedo  =  1;              
        }              
        ENDCG            
    }            
	Fallback  "Diffuse"        
}
```

表面着色器被定义在SubShader语义块（而非Pass语义块）中的CGPROGRAM和ENDCG之间。原因是，表面着色器不需要开发者关心使用多少个Pass、每个Pass如何渲染等问题，Unity会在背后为我们做好这些事情。

CGPROGRAM和ENDCG之间的代码是使用CG/HLSL编写的，它的语法和标准的CG/HLSL语法几乎一样，但还是有细微的不同，例如有些原生的函数和用法Unity并没有提供支持。

一个非常简单的顶点/片元着色器示例:

```c
Shader  "Custom/Simple  VertexFragment  Shader"  {
	SubShader  {              
    	Pass  {                
            CGPROGRAM                  
            #pragma  vertex  vert                  
            #pragma  fragment  frag                  
            float4  vert(float4  v  :  POSITION)  :  SV_POSITION  {                      
                return  mul  (UNITY_MATRIX_MVP,  v);                  
            }                  
            fixed4  frag()  :  SV_Target  {                      
                return  fixed4(1.0,0.0,0.0,1.0);                 
            }                  
            ENDCG              
        }            
    }        
}
```

#### 总结

在传统的Shader中，我们仅可以编写特定类型的Shader，例如顶点着色器、片元着色器等。而在Unity Shader中，我们可以在同一个文件里同时包含需要的顶点着色器和片元着色器代码。

在传统的Shader中，我们无法设置一些渲染设置，例如是否开启混合、深度测试等，这些是开发者在另外的代码中自行设置的。而在Unity Shader中，我们通过一行特定的指令就可以完成这些设置。

在传统的Shader中，我们需要编写冗长的代码来设置着色器的输入和输出，要小心地处理这些输入输出的位置对应关系等。而在Unity Shader中，我们只需要在特定语句块中声明一些属性，就可以依靠材质来方便地改变这些属性。而且对于模型自带的数据（如顶点位置、纹理坐标、法线等）, Unity Shader也提供了直接访问的方法，不需要开发者自行编码来传给着色器。

### 第4章 学习Shader所需的数学基础

#### 4.8 Unity Shader的内置变量（数学篇）

![Unity内置的变换矩阵](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z3rw8l85j30rs0cigob.jpg)

![Unity内置的摄像机和屏幕参数](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z3s12927j30rs0f7778.jpg)

#### 总结

通常在变换顶点时，我们都是使用右乘的方式来按列矩阵进行乘法。这是因为，Unity提供的内置矩阵（如UNITY_MATRIX_MVP等）都是按列存储的。但有时，我们也会使用左乘的方式，这是因为可以省去对矩阵转置的操作。

Unity在脚本中提供了一种矩阵类型——Matrix4x4。脚本中的这个矩阵类型则是采用列优先的方式。这与Unity Shader中的规定不一样。

> 矩阵在CG、DirectX中是**行优先**存储，在OpenGL中是**列优先**存储的。
>
> 行优先存储，点/向量用行向量表示，变换矩阵必须在✖的右边，称为**右乘**。
>
> 列优先可类比。

OpenGL和DirectX 10以后的版本认为**像素中心**对应的是浮点值中的0.5。VPOS/WPOS语义定义的输入是一个float4类型的变量。我们已经知道它的xy值代表了在屏幕空间中的像素坐标。如果屏幕分辨率为400 × 300，那么x的范围就是[0.5,400.5], y的范围是[0.5,300.5]。(VPOS是HLSL中对屏幕坐标的语义，而WPOS是CG中对屏幕坐标的语义。)

在Unity中，VPOS/WPOS的z分量范围是[0,1]，在摄像机的近裁剪平面处，z值为0，在远裁剪平面处，z值为1。对于w分量，我们需要考虑摄像机的投影类型。如果使用的是透视投影，那么w分量的范围是$[\frac{1}{Near},\frac{1}{Far}]$,Near和Far对应了在Camera组件中设置的近裁剪平面和远裁剪平面距离摄像机的远近；如果使用的是正交投影，那么w分量的值恒为1。

## 第2篇 初级篇

### 第5章 开始Unity Shader学习之旅

#### 5.1 本书使用的软件和环境

本书使用的Unity版本是Unity 5.2.1免费版。

本书工程编写的系统环境是Mac OS X 10.9.5。如果读者使用的是其他系统，绝大部分情况也不会有任何问题。但有时会由于图像编程接口的种类和版本不同而有一些差别，这是因为Mac使用的图像编程接口是基于OpenGL的，而其他平台如Windows，可能使用的是DirectX。例如，在OpenGL中，渲染纹理（Render Texture）的(0, 0)点是在左下角，而在DirectX中，(0, 0)点是在左上角。

这本书是2016年写的，比较老了。这里我使用**2020.3.26f1c1**。

#### 5.2 一个最简单的顶点/片元着色器

```c
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 5/Simple Shader" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
	}
	SubShader {
        Pass {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            
            uniform fixed4 _Color;
            //定义顶点着色器的输入，由使用该材质的Mesh Render组件提供
			struct a2v {
                //POSITION语义：顶点坐标
                float4 vertex : POSITION;
                //NORMAL语义：法线方向
				float3 normal : NORMAL;
                //TEXCOORD0语义：第一套纹理坐标
				float4 texcoord : TEXCOORD0;
            };
            
            struct v2f {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR0;
            };
            
            v2f vert(a2v v) {
            	v2f o;
            	o.pos = UnityObjectToClipPos(v.vertex);
            	o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
            	fixed3 c = i.color;
            	c *= _Color.rgb;
                return fixed4(c, 1.0);
            }

            ENDCG
        }
    }
}
```

#### 5.3 强大的援手：Unity提供的内置文件和变量

包含文件（include file），是类似于C++中头文件的一种文件。在Unity中，它们的文件后缀是．cginc。在编写Shader时，我们可以使用#include指令把这些文件包含进来，这样我们就可以使用Unity为我们提供的一些非常有用的变量和帮助函数。

CGIncludes文件夹在Mac上位于：/Applications/Unity/Unity.app/Contents/CGIncludes；在Windows上位于：Unity的安装路径/Data/CGIncludes。

![Unity中一些常用的包含文件](https://tva4.sinaimg.cn/large/0077Un8Egy1h5zl6tcyhmj30rs06aab7.jpg)

![UnityCG.cginc中一些常用的结构体](https://tva2.sinaimg.cn/large/0077Un8Egy1h5zl774z9ij30rs06m3zq.jpg)

![UnityCG.cginc中一些常用的帮助函数](https://tva4.sinaimg.cn/large/0077Un8Egy1h5zl7da9ztj30rs0bignw.jpg)

#### 5.4 Unity提供的CG/HLSL语义

语义可以让Shader知道从哪里读取数据，并把数据输出到哪里，它们在CG/HLSL的Shader流水线中是不可或缺的。需要注意的是，Unity并没有支持所有的语义。

在DirectX 10以后，有了一种新的语义类型，就是系统数值语义（system-value semantics）。这类语义是以SV开头的，SV代表的含义就是系统数值（system-value）。这些语义在渲染流水线中有特殊的含义。例如在上面的代码中，我们使用SV_POSITION语义去修饰顶点着色器的输出变量pos，那么就表示pos包含了可用于光栅化的变换后的顶点坐标（即齐次裁剪空间中的坐标）。用这些语义描述的变量是不可以随便赋值的，因为流水线需要使用它们来完成特定的目的，例如渲染引擎会把用SV_POSITION修饰的变量经过光栅化后显示在屏幕上。

> 一些Shader会使用POSITION而非SV_POSITION来修饰顶点着色器的输出。SV_POSITION是DirectX 10中引入的系统数值语义，在绝大多数平台上，它和POSITION语义是等价的，但在某些平台（例如索尼PS4）上必须使用SV_POSITION来修饰顶点着色器的输出，否则无法让Shader正常工作。同样的例子还有COLOR和SV_Target。因此，为了让我们的Shader有更好的跨平台性，对于这些有特殊含义的变量我们最好使用以SV开头的语义进行修饰。

![从应用阶段传递模型数据给顶点着色器时Unity支持的常用语义](https://tva4.sinaimg.cn/large/0077Un8Egy1h5zlblpluij30rs07ejse.jpg)

![从顶点着色器传递数据给片元着色器时Unity使用的常用语义](https://tvax3.sinaimg.cn/large/0077Un8Egy1h5zlbsud5tj30rs069757.jpg)

![片元着色器输出时Unity支持的常用语义](https://tva1.sinaimg.cn/large/0077Un8Egy1h5zlc4uuzoj30rs037glu.jpg)

#### 5.5 程序员的烦恼：Debug

Shader可以直接使用颜色代表数据进行可视化来Debug。也可以借助帧调试器一类的工具。

