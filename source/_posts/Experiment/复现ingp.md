---
title: 复现ingp
layout: post
categories: [Experiment]
mathjax: false
---

ingp

<!-- more -->

[github网址](https://github.com/NVlabs/instant-ngp)

## 1.实验环境

RTX3090

CUDA 11.4

VS2019

编译过程中可能会遇到很多问题，具体参考github里的issue

远程控制主机，若主机不连接显示器，opengl渲染会出现白屏。

经过一次更新之后（https://github.com/NVlabs/instant-ngp/issues/36#issuecomment-1036198048），ingp对显存的需求量大大降低。

## 2.使用即时神经图形基元训练NeRF模型的技巧

[link](https://github.com/NVlabs/instant-ngp/blob/master/docs/nerf_dataset_tips.md)

我们的NeRF实现期望初始相机参数以 transforms. json的格式提供，和原始的NeRF代码库兼容。我们提供了一个方便的脚本，scripts/colmap2nerf.py，它可以用来处理视频文件或图像序列，使用开源的COLMAP（ structure from motion，SFM软件）来提取必要的摄像机数据。

训练过程可能对数据集非常挑剔。例如，重要的是数据集要有良好的覆盖率，不要包含错误标记的相机数据，不要包含模糊的帧(运动模糊和离焦模糊都是有问题的)。这篇问题试图给出一些技巧。一个很好的经验法则是，如果你的NeRF模型在20秒左右没有收敛，那么在长时间的训练之后，它就不太可能变得更好。因此，我们建议调整数据，以在培训的早期阶段获得明确的结果。对于大型的真实世界场景，可以通过最多几分钟的训练获得一点额外的清晰度。几乎所有的收敛都发生在最初的几秒钟。

数据集最常见的问题是相机位置的不正确比例或偏移（更多细节见下面）。第二个最常见的问题是图像太少，或者图像的相机参数不准确(例如，如果COLMAP失败)。在这种情况下，您可能需要获取更多的图像，或者调整计算相机位置的过程。这超出了instant-npg实现的范围。

### 2.1 已有数据集

默认情况下，instant-ngp的NeRF实现只将射线通过从[0,0,0]到[1,1,1]的单元边界框。默认情况下，数据加载器在输入JSON文件中接受摄像机转换，并将位置缩放0.33，偏移量[0.5,0.5,0.5]，以便将输入数据的原点映射到立方体的中心。缩放因子的选择是为了适应原始NeRF论文中的合成数据集，以及我们的脚本/colmap2nerf.py脚本的输出。

通过在UI的 "Debug visualization"中选择 "Debug visualization"和 "Visualize unit cube" 来检查你的摄像机与这个包围框的对齐方式是值得的，如下图所示:

![](https://github.com/NVlabs/instant-ngp/blob/master/docs/assets/nerfbox.jpg)

对于单位立方体外有可见背景的自然场景，需要在 transforms.json文件中把参数aabb_scale设置为2次幂(即1、2、4、8或16)，在最外层的作用域(与现有的camera_angle_x参数相同的嵌套)。data/nerf/fox/transforms.json是个例子。

效果如下图所示:

![](https://github.com/NVlabs/instant-ngp/blob/master/docs/assets/nerfboxrobot.jpg)

摄像机仍然以单位立方体中的“感兴趣的对象”为中心;然而，aabb_scale参数，在这里设置为16，导致NeRF实现跟踪射线到一个更大的边界框(边长16)，包含背景元素，中心在[0.5,0.5,0.5]。

### 2.2 扩展现有的数据集

如果你有一个transform.json格式的数据集，它应该以原点为中心，并和原始的NeRF synthetic datasets规模近似。当你把它加载到NGP中时，如果你发现它没有收敛，首先要检查的是相机相对于立方体单元的位置，使用上面描述的调试功能。如果数据集不在单元立方体中占主导地位，则值得将其移到那里。你可以通过调整转换本身来做到这一点，或者你可以在json的外部范围中添加全局参数。

您可以设置以下任何参数，其中列出的值为默认值。

```json
{
	"aabb_scale": 16,
	"scale": 0.33,
	"offset": [0.5, 0.5, 0.5],
	...	
}
```

实现细节和附加选项参见nerf_loader. cu。

### 2.3 准备新的NeRF数据集

确保您已经安装了COLMAP，并且它在您的PATH中可用。如果您使用一个视频文件作为输入，也要确保安装FFmpeg，并确保它在您的PATH中可用。要检查这种情况，从一个终端窗口，您应该能够运行colmap和ffmpeg -?并从中看到一些帮助文本。

如果您正在从视频文件中进行训练，请在包含视频的文件夹中运行scripts/colmap2nerf.py脚本，并使用以下推荐参数:

```bash
data-folder$ python [path-to-instant-ngp]/scripts/colmap2nerf.py --video_in <filename of video> --video_fps 2 --run_colmap --aabb_scale 16
```

上面假设输入一个视频文件，然后按指定的帧率(2)提取帧。建议选择帧率在50-150左右的图像。对于一分钟的视频，——video_fps 2是理想的。

如果从图像中进行训练，将它们放在名为images的子文件夹中，然后使用合适的选项，如下面的选项:

```bash
data-folder$ python [path-to-instant-ngp]/scripts/colmap2nerf.py --colmap_matcher exhaustive --run_colmap --aabb_scale 16
```

脚本将根据需要运行FFmpeg和/或COLMAP，然后执行转换到所需transforms.json格式的步骤，该步骤将写入当前目录。

默认情况下，脚本使用“顺序匹配器”调用colmap，它适用于从平滑变化的摄像机路径中拍摄的图像，比如视频。如果图像没有特定的顺序，则穷举匹配器更合适，如上面的图像示例所示。要了解更多选项，可以使用——help运行脚本。有关COLMAP更高级的使用或具有挑战性的场景，请参阅COLMAP文档;您可能需要修改脚本/colmap2nerf.py脚本本身。

aabb_scale参数是最重要的instant-ngp特定参数。它指定场景的范围，默认值为1;也就是说，场景被缩放，使得摄像机位置距离原点的平均距离为1个单位。对于小型的合成场景，如原始NeRF数据集，默认的aabb_scale为1是理想的，可以获得最快的训练。NeRF模型假设训练图像可以完全被包含在这个边界框中的场景所解释。然而，在自然场景中，背景超出了这个边界框，NeRF模型会在框的边界出现“飞蚊”。通过将aabb_scale设置为2的更大的幂(最高为16)，NeRF模型将射线扩展到一个更大的边界框。注意，这可能会轻微影响训练速度。如果有疑问，对于自然场景，首先将aabb_scale设置为16，然后如果可能的话将其缩小。值可以在 transforms. json输出文件直接编辑，无需重新运行scripts/colmap2nerf.py脚本。

假设成功，你现在可以训练你的NeRF模型如下，从instant-ngp文件夹开始:

```bash
instant-ngp$ ./build/testbed --mode nerf --scene [path to training data folder containing transforms.json]
```

### 2.4 NeRF训练数据的技巧

NeRF模型在50-150张图像之间训练得最好，这些图像需要表现出最小的场景移动，运动模糊或其他人为模糊。COLMAP能够从图像中提取准确的相机参数，从而保证了重建的质量。有关如何验证这一点的信息，请参阅前面的部分。

colmap2nerf.py脚本假设所有训练图像都大约指向一个共享的兴趣点，它将这个点放置在原点。这个点是通过所有对训练图像的中心像素对光线之间最近的接近点取加权平均来找到的。在实践中，这意味着当训练图像指向感兴趣的对象时，脚本工作得最好，尽管它们不需要完成该对象的360度全景。如果将aabb_scale设置为大于1的数值，则感兴趣对象后面的任何可见背景仍然会被重建，如上所述。