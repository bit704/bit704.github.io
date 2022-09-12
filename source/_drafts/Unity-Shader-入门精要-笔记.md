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

#### 附录

在传统的Shader中，我们仅可以编写特定类型的Shader，例如顶点着色器、片元着色器等。而在Unity Shader中，我们可以在同一个文件里同时包含需要的顶点着色器和片元着色器代码。

在传统的Shader中，我们无法设置一些渲染设置，例如是否开启混合、深度测试等，这些是开发者在另外的代码中自行设置的。而在Unity Shader中，我们通过一行特定的指令就可以完成这些设置。

在传统的Shader中，我们需要编写冗长的代码来设置着色器的输入和输出，要小心地处理这些输入输出的位置对应关系等。而在Unity Shader中，我们只需要在特定语句块中声明一些属性，就可以依靠材质来方便地改变这些属性。而且对于模型自带的数据（如顶点位置、纹理坐标、法线等）, Unity Shader也提供了直接访问的方法，不需要开发者自行编码来传给着色器。

### 第4章 学习Shader所需的数学基础

#### 4.8 Unity Shader的内置变量（数学篇）

![Unity内置的变换矩阵](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z3rw8l85j30rs0cigob.jpg)

![Unity内置的摄像机和屏幕参数](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5z3s12927j30rs0f7778.jpg)

#### 附录

通常在变换顶点时，我们都是使用右乘的方式来按列矩阵进行乘法。这是因为，Unity提供的内置矩阵（如UNITY_MATRIX_MVP等）都是按列存储的。但有时，我们也会使用左乘的方式，这是因为可以省去对矩阵转置的操作。

Unity在脚本中提供了一种矩阵类型——Matrix4x4。脚本中的这个矩阵类型则是采用列优先的方式。这与Unity Shader中的规定不一样。

> 矩阵在CG、DirectX中是**行优先**表示，在OpenGL中是**列优先**表示的。
>
> 更多细节可以参考我Essay分类下的文章 矩阵存储与表示。

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

![结果1](https://tva4.sinaimg.cn/large/0077Un8Egy1h61ci2jjdvj30q70o30wh.jpg)

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

#### 5.6 小心：渲染平台的差异

在OpenGL（OpenGL ES也是）中，(0, 0)点对应了屏幕的左下角，而在DirectX（Meta l也是）中，(0, 0)点对应了左上角。

大多数情况下，这样的差异并不会对我们造成任何影响。但当我们要使用**渲染到纹理技术**，把屏幕图像渲染到一张渲染纹理中时，如果不采取行任何措施的话，就会出现纹理翻转的情况。幸运的是，Unity在背后为我们处理了这种翻转问题——当在DirectX平台上使用渲染到纹理技术时，Unity会为我们翻转屏幕图像纹理，以便在不同平台上达到一致性。

如果我们需要同时处理多张渲染图像（前提是开启了抗锯齿），例如需要同时处理屏幕图像和法线纹理，这些图像在竖直方向的朝向就可能是不同的（只有在DirectX这样的平台上才有这样的问题）。

#### 5.7 Shader整洁之道

CG/HLSL中3种精度的数值类型

| 类型  | 精度                                           |
| ----- | ---------------------------------------------- |
| float | 最高精度，通常32位存储                         |
| half  | 中等精度，通常16位存储，精度范围-60000\~+60000 |
| fixed | 最低精度，通常11位存储，精度范围-2.0\~+2.0     |

Shader Model是由微软提出的一套规范，通俗地理解就是它们决定了Shader中各个特性（feature）的能力（capability）。这些特性和能力体现在Shader能使用的运算指令数目、寄存器个数等各个方面。Shader Model等级越高，Shader的能力就越大。

GPU使用了不同于CPU的技术来实现分支语句，在最坏的情况下，我们花在一个分支语句的时间相当于运行了所有分支语句的时间。因此，我们不鼓励在Shader中使用流程控制语句，因为它们会降低GPU的并行处理操作（尽管在现代的GPU上已经有了改进）。

### 第6章 Unity中的基础光照

#### 6.1 我们是如何看到这个世界的

**着色**（shading）指的是，根据材质属性（如漫反射属性等）、光源信息（如光源方向、辐照度等），使用一个等式去计算沿某个观察方向的出射度的过程。我们也把这个等式称为光照模型（Lighting Model）。

当给定模型表面上的一个点时，BRDF包含了对该点外观的完整的描述。在图形学中，**BRDF**（Bidirectional Reflectance Distribution Function）大多使用一个数学公式来表示，并且提供了一些参数来调整材质属性。通俗来讲，当给定入射光线的方向和辐照度后，BRDF可以给出在某个出射方向上的光照能量分布。

本章涉及的BRDF都是对真实场景进行理想化和简化后的模型，也就是说，它们并不能真实地反映物体和光线之间的交互，这些光照模型被称为是**经验模型**。

#### 6.2 标准光照模型

虽然光照模型有很多种类，但在早期的游戏引擎中往往只使用一个光照模型，这个模型被称为**标准光照模型**。实际上，在BRDF理论被提出之前，标准光照模型就已经被广泛使用了。

> 在1975年，著名学者裴祥风（Bui Tuong Phong）提出了标准光照模型背后的基本理念。标准光照模型只关心直接光照（direct light），也就是那些直接从光源发射出来照射到物体表面后，经过物体表面的一次反射直接进入摄像机的光线。

它把进入到摄像机内的光线分为4个部分，每个部分使用一种方法来计算它的贡献度:

- **自发光**（emissive）部分，本书使用cmissive来表示。这个部分用于描述当给定一个方向时，一个表面本身会向该方向发射多少辐射量。需要注意的是，如果没有使用全局光照（global illumination）技术，这些自发光的表面并不会真的照亮周围的物体，而是它本身看起来更亮了而已。

- **漫反射**（diffuse）部分，本书使用cdiffuse来表示。这个部分用于描述，当光线从光源照射到模型表面时，该表面会向每个方向散射多少辐射量。

  因为反射完全随机，可以认为在任何反射方向上的分布都是一样的，因此入射光线的角度很重要。

  漫反射光照符合兰伯特定律（Lambert's law）：反射光线的强度与表面法线和光源方向之间夹角的余弦值成正比。
  $$
  c_{diffuse}=(c_{light}·m_{diffuse})\max(0,\hat{n}·\hat{I})
  $$
  $c_{light}$是光源颜色，$m_{diffuse}$是材质的漫反射颜色， $\hat{n}$是表面法线， $\hat{I}$是指向光源的单位矢量。max防止物体被从后面来的光源照亮。

- **高光反射**（specular）部分，本书使用cspecular来表示。这个部分用于描述当光线从光源照射到模型表面时，该表面会在完全镜面反射方向散射多少辐射量。

  计算高光反射需要知道表面法线$\hat{n}$、视角方向$\hat{v}$、光源方向$\hat{l}$、反射方向$\hat{r}$。反射方向可以由表面法线和光源方向计算。
  $$
  \hat{r}=2(\hat{n}·\hat{I})\hat{n}-\hat{I}
  $$
  利用Phong模型来计算高光反射:
  $$
  c_{specular}=(c_{light}·m_{specular})\max(0,\hat{v}·\hat{r})^{m_{gloss}}
  $$
  $m_{gloss}$是材质的光泽度（gloss），也被称为反光度（shininess）。它用于控制高光区域的“亮点”有多宽，**gloss越大，亮点就越小**。

  Blinn模型引入了一个新的矢量$\hat{h}$，$\hat{h}=\frac{\hat{v}+\hat{I}}{ | \hat{v}+\hat{I} | }$：
  $$
  c_{specular}=(c_{light}·m_{specular})\max(0,\hat{n}·\hat{h})^{m_{gloss}}
  $$

- **环境光**（ambient）部分，本书使用cambient来表示。它用于描述其他所有的间接光照。

  通常是一个全局变量，即场景中的所有物体都使用这个环境光。

**逐像素光照**: 以每个像素为基础，得到它的法线（可以是对顶点法线插值得到的，也可以是从法线纹理中采样得到的），然后进行光照模型的计算。这种在面片之间对顶点法线进行插值的技术被称为Phong着色（Phong shading），也被称为Phong插值或法线插值着色技术。这不同于我们之前讲到的Phong光照模型。

**逐顶点光照**: 被称为高洛德着色（Gouraud shading）,每个顶点上计算光照，然后会在渲染图元内部进行线性插值，最后输出成像素颜色。由于顶点数目往往远小于像素数目，因此逐顶点光照的计算量往往要小于逐像素光照。但是，由于逐顶点光照依赖于线性插值来得到像素光照，因此，当光照模型中有非线性的计算（例如计算高光反射时）时，逐顶点光照就会出问题。而且，由于逐顶点光照会在渲染图元内部对顶点颜色进行插值，这会导致渲染图元内部的颜色总是暗于顶点处的最高颜色值，这在某些情况下会产生明显的棱角现象。

> 标准光照模型有很多不同的叫法。例如，一些资料中称它为Phong光照模型，因为裴祥风（Bui Tuong Phong）首先提出了使用漫反射和高光反射的和来对反射光照进行建模的基本思想，并且提出了基于经验的计算高光反射的方法（用于计算漫反射光照的兰伯特模型在那时已经被提出了）。而后，由于Blinn的方法简化了计算而且在某些情况下计算更快，我们把这种模型称为Blinn-Phong光照模型。

**局限**: 首先，有很多重要的物理现象无法用Blinn-Phong模型表现出来，例如菲涅耳反射（Fresnel reflection）。其次，Blinn-Phong模型是各项同性（isotropic）的，也就是说，当我们固定视角和光源方向旋转这个表面时，反射不会发生任何改变。但有些表面是具有各向异性（anisotropic）反射性质的，例如拉丝金属、毛发等。

#### 6.3 Unity中的环境光和自发光

环境光在window->Rendering->Lighting->Environment中调节。

计算自发光只需要在片元着色器输出最后的颜色之前，把材质的自发光颜色添加到输出颜色上。

#### 6.4 在Unity Shader中实现漫反射光照模型

逐顶点的漫反射光照着色器

```c
// Upgrade NOTE: replaced '_World2Object' with 'unity_WorldToObject'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level" {
	Properties {
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
	}
	SubShader {
		Pass { 
            //定义该Pass在Unity的光照流水线中的角色
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Diffuse;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				fixed3 color : COLOR;
			};
			
			v2f vert(a2v v) {
				v2f o;
				// Transform the vertex from object space to projection space
				o.pos = UnityObjectToClipPos(v.vertex);
				
				// Get ambient term
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				// Transform the normal from object space to world space
				fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
				// Get the light direction in world space
				fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
				// Compute diffuse term
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
				
				o.color = ambient + diffuse;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				return fixed4(i.color, 1.0);
			}
			
			ENDCG
		}
	}
	FallBack "Diffuse"
}
```

注意，管线会自动在顶点着色器和片元着色器之间进行插值。

逐像素的漫反射光照着色器类似，只是把颜色的计算放在了片元着色器中。

半兰伯特光照模型：
$$
c_{diffuse}=(c_{light}·m_{diffuse})(0.5(\hat{n}·\hat{I})+0.5)
$$
不使用max，而是把$\hat{n}·\hat{I}$从$[-1,1]$映射到$[0,1]$。没有任何物理依据，仅仅是一个视觉加强技术。

左起逐顶点漫反射光照、逐像素漫反射光照、逐像素半兰伯特光照：

![结果2](https://tva4.sinaimg.cn/large/0077Un8Egy1h61vvfs78hj30pw0ng78e.jpg)

#### 6.5 在Unity Shader中实现高光反射光照模型

该部分主干代码与上节相同，把计算公式换掉即可。

#### 6.6 召唤神龙：使用Unity内置的函数

```c
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 6/Blinn-Phong Use Built-in Functions" {
	Properties {
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(1.0, 500)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float4 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
                // o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld, v.vertex);
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				fixed3 worldNormal = normalize(i.worldNormal);
				// fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
				// fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

### 第7章 基础纹理

纹理最初的目的就是使用一张图片来控制模型的外观。使用**纹理映射**（texture mapping）技术，我们可以把一张图“黏”在模型表面，逐纹素（texel）（纹素的名字是为了和像素进行区分）地控制模型的颜色。

在美术人员建模的时候，通常会**在建模软件中利用纹理展开技术把纹理映射坐标（texture-mapping coordinates）存储在每个顶点上**。纹理映射坐标定义了该顶点在纹理中对应的2D坐标。通常，这些坐标使用一个二维变量(u, v)来表示，其中u是横向坐标，而v是纵向坐标。因此，纹理映射坐标也被称为**UV坐标**，通常都被归一化到[0, 1]范围内。对于不在[0, 1]范围内的纹理坐标，与之关系紧密的是纹理的平铺模式。

#### 7.1 单张纹理

通常会使用一张纹理来代替物体的漫反射颜色。

```c
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 7/Single Texture" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white" {}
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {		
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"
			
			fixed4 _Color;
			sampler2D _MainTex;
            //在Unity中，我们需要使用纹理名_ST的方式来声明某个纹理的属性。其中，ST是缩放（scale）和平移（translation）的缩写。_MainTex_ST可以让我们得到该纹理的缩放和平移（偏移）值，_MainTex_ST.xy存储的是缩放值，而_MainTex_ST.zw存储的是偏移值。这些值可以在材质面板的纹理属性中调节。
			float4 _MainTex_ST;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				float2 uv : TEXCOORD2;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				// Or just call the built-in function o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				
				// Use the texture to sample the diffuse color
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

![结果3](https://tva3.sinaimg.cn/large/0077Un8Egy1h62zwtiy2tj30ek0jijvw.jpg)

Wrap Mode : 一种是Repeat，在这种模式下，如果纹理坐标超过了1，那么它的整数部分将会被舍弃，而直接使用小数部分进行采样，这样的结果是纹理将会不断重复；另一种是Clamp，在这种模式下，如果纹理坐标大于1，那么将会截取到1，如果小于0，那么将会截取到0。

Filter Mode : Point模式使用了最近邻（nearest neighbor）滤波，在放大或缩小时，它的采样像素数目通常只有一个，因此图像会看起来有种像素风格的效果。而Bilinear滤波则使用了线性滤波，对于每个目标像素，它会找到4个邻近像素，然后对它们进行线性插值混合后得到最终像素，因此图像看起来像被模糊了。而Trilinear滤波几乎是和Bilinear一样的，只是Trilinear还会在多级渐远纹理之间进行混合。如果一张纹理没有使用多级渐远纹理技术，那么Trilinear得到的结果是和Bilinear就一样的。

#### 7.2 凹凸映射

**凹凸映射**（bump mapping）的目的是使用一张纹理来修改模型表面的法线，以便为模型提供更多的细节。这种方法不会真的改变模型的顶点位置，只是让模型看起来好像是“凹凸不平”的，但可以从模型的轮廓处看出“破绽”。

有两种主要的方法可以用来进行凹凸映射

- 使用一张高度纹理（height map）来模拟表面位移（displacement），然后得到一个修改后的法线值，这种方法也被称为高度映射（height mapping）
- 使用一张法线纹理（normal map）来直接存储表面法线，这种方法又被称为法线映射（normal mapping）。

由于法线方向的分量范围在[-1, 1]，而像素的分量范围为[0, 1]，因此我们需要做一个映射将法线方向存储到法线纹理中。

法线纹理分类 : 

- **模型空间的法线纹理**（object-space normal map）

  - 实现简单

  - 这是因为模型空间下的法线纹理存储的是同一坐标系下的法线信息，因此在边界处通过插值得到的法线可以平滑变换 

- **切线空间的法线纹理**（tangent-space normal map）

  > 对于模型的每个顶点，它都有一个属于自己的切线空间，这个切线空间的原点就是该顶点本身，而z轴是顶点的法线方向（n）, x轴是顶点的切线方向（t），而y轴可由法线和切线叉积而得，也被称为是副切线（bitangent, b）或副法线。
  
  - 自由度很高，可以重用。模型空间下的法线纹理记录的是绝对法线信息，仅可用于创建它时的那个模型，而应用到其他模型上效果就完全错误了。而切线空间下的法线纹理记录的是相对法线信息，这意味着，即便把该纹理应用到一个完全不同的网格上，也可以得到一个合理的结果。
  - 可进行UV动画。比如，我们可以移动一个纹理的UV坐标来实现一个凹凸移动的效果，但使用模型空间下的法线纹理会得到完全错误的结果。原因同上。这种UV动画在水或者火山熔岩这种类型的物体上会经常用到。
  - 可压缩。由于切线空间下的法线纹理中法线的Z方向总是正方向，因此我们可以仅存储XY方向，而推导得到Z方向。而模型空间下的法线纹理由于每个方向都是可能的，因此必须存储3个方向的值，不可压缩。

我们需要在计算光照模型中统一各个方向矢量所在的坐标空间。由于法线纹理中存储的法线是切线空间下的方向，因此我们通常有**两种方法**：

1. 在切线空间下进行光照计算，此时我们需要把光照方向、视角方向变换到切线空间下。
2. 在世界空间下进行光照计算，此时我们需要把采样得到的法线方向变换到世界空间下，再和世界空间下的光照方向和视角方向进行计算。

从效率上来说，第一种方法往往要优于第二种方法，因为我们可以在顶点着色器中就完成对光照方向和视角方向的变换，而第二种方法由于要先对法线纹理进行采样，所以变换过程必须在片元着色器中实现，这意味着我们需要在片元着色器中进行一次矩阵操作。

但从通用性角度来说，第二种方法要优于第一种方法，因为有时我们需要在世界空间下进行一些计算，例如在使用Cubemap进行环境映射时，我们需要使用世界空间下的反射方向对Cubemap进行采样。如果同时需要进行法线映射，我们就需要把法线方向变换到世界空间下。

对于第一种方法，原代码有点问题：[Issue #45 · candycat1992/Unity_Shaders_Book · GitHub](https://github.com/candycat1992/Unity_Shaders_Book/issues/45)

```c
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 7/Normal Map In Tangent Space" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white" {}
		_BumpMap ("Normal Map", 2D) = "bump" {}
		_BumpScale ("Bump Scale", Float) = 1.0
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				// 和法线方向normal不同，tangent的类型是float4，而非float3，这是因为我们需要使用tangent.w分量来决定切线空间中的坐标轴——副切线的方向性。
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
				float3 lightDir: TEXCOORD1;
				float3 viewDir : TEXCOORD2;
			};

			// Unity doesn't support the 'inverse' function in native shader
			// so we write one by our own
			// Note: this function is just a demonstration, not too confident on the math or the speed
			// Reference: http://answers.unity3d.com/questions/218333/shader-inversefloat4x4-function.html
			float4x4 inverse(float4x4 input) {
				#define minor(a,b,c) determinant(float3x3(input.a, input.b, input.c))
				
				float4x4 cofactors = float4x4(
				     minor(_22_23_24, _32_33_34, _42_43_44), 
				    -minor(_21_23_24, _31_33_34, _41_43_44),
				     minor(_21_22_24, _31_32_34, _41_42_44),
				    -minor(_21_22_23, _31_32_33, _41_42_43),
				    
				    -minor(_12_13_14, _32_33_34, _42_43_44),
				     minor(_11_13_14, _31_33_34, _41_43_44),
				    -minor(_11_12_14, _31_32_34, _41_42_44),
				     minor(_11_12_13, _31_32_33, _41_42_43),
				    
				     minor(_12_13_14, _22_23_24, _42_43_44),
				    -minor(_11_13_14, _21_23_24, _41_43_44),
				     minor(_11_12_14, _21_22_24, _41_42_44),
				    -minor(_11_12_13, _21_22_23, _41_42_43),
				    
				    -minor(_12_13_14, _22_23_24, _32_33_34),
				     minor(_11_13_14, _21_23_24, _31_33_34),
				    -minor(_11_12_14, _21_22_24, _31_32_34),
				     minor(_11_12_13, _21_22_23, _31_32_33)
				);
				#undef minor
				return transpose(cofactors) / determinant(input);
			}

			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;

				///
				/// Note that the code below can handle both uniform and non-uniform scales
				///

				// Construct a matrix that transforms a point/vector from tangent space to world space
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
				fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
				fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 

				/*
				float4x4 tangentToWorld = float4x4(worldTangent.x, worldBinormal.x, worldNormal.x, 0.0,
												   worldTangent.y, worldBinormal.y, worldNormal.y, 0.0,
												   worldTangent.z, worldBinormal.z, worldNormal.z, 0.0,
												   0.0, 0.0, 0.0, 1.0);
				// The matrix that transforms from world space to tangent space is inverse of tangentToWorld
				float3x3 worldToTangent = inverse(tangentToWorld);
				*/
				
				//wToT = the inverse of tToW = the transpose of tToW as long as tToW is an orthogonal matrix.
				// xyz轴
				float3x3 worldToTangent = float3x3(worldTangent, worldBinormal, worldNormal);

				// Transform the light and view dir from world space to tangent space
				o.lightDir = mul(worldToTangent, WorldSpaceLightDir(v.vertex));
				o.viewDir = mul(worldToTangent, WorldSpaceViewDir(v.vertex));

				///
				/// Note that the code below can only handle uniform scales, not including non-uniform scales
				/// 

				// Compute the binormal
//				float3 binormal = cross( normalize(v.normal), normalize(v.tangent.xyz) ) * v.tangent.w;
//				// Construct a matrix which transform vectors from object space to tangent space
//				float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
				// Or just use the built-in macro
//				TANGENT_SPACE_ROTATION;
//				
//				// Transform the light direction from object space to tangent space
//				o.lightDir = mul(rotation, normalize(ObjSpaceLightDir(v.vertex))).xyz;
//				// Transform the view direction from object space to tangent space
//				o.viewDir = mul(rotation, normalize(ObjSpaceViewDir(v.vertex))).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {				
				fixed3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentViewDir = normalize(i.viewDir);
				
				// Get the texel in the normal map
				fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
				fixed3 tangentNormal;
				// If the texture is not marked as "Normal map"
//				tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale;
//				tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
				
				// Or mark the texture as "Normal map", and use the built-in funciton
				tangentNormal = UnpackNormal(packedNormal);
				tangentNormal.xy *= _BumpScale;
				tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
				
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));

				fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

第二种方法：

```c
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Unity Shaders Book/Chapter 7/Normal Map In World Space" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white" {}
		_BumpMap ("Normal Map", 2D) = "bump" {}
		_BumpScale ("Bump Scale", Float) = 1.0
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
                // 从切线空间到世界空间的变换矩阵
                // 一个插值寄存器最多只能存储float4大小的变量，对于矩阵这样的变量，我们可以把它们按行拆成多个变量再进行存储。
				float4 TtoW0 : TEXCOORD1;  
				float4 TtoW1 : TEXCOORD2;  
				float4 TtoW2 : TEXCOORD3; 
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;
				
				float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
				fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
				fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 
				
				// Compute the matrix that transform directions from tangent space to world space
				// Put the world position in w component for optimization
				o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
				o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
				o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				// Get the position in world space		
				float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
				// Compute the light and view dir in world space
				fixed3 lightDir = normalize(UnityWorldSpaceLightDir(worldPos));
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
				
				// Get the normal in tangent space
				fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));
				bump.xy *= _BumpScale;
				bump.z = sqrt(1.0 - saturate(dot(bump.xy, bump.xy)));
				// Transform the normal from tangent space to world space
				bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
				
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(bump, lightDir));

				fixed3 halfDir = normalize(lightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(bump, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}

```

![结果4](https://tvax4.sinaimg.cn/large/0077Un8Egy1h63002yv2wj30cf0jkagx.jpg)

#### 7.3 渐变纹理





### 第18章 基于物理的渲染



