---
title: 《pytorch深度学习实战》笔记
categories: [Essay]
tags: [C++,DirectX]
cover: https://tva4.sinaimg.cn/large/0077Un8Egy1h5k6z703otj30rs14079r.jpg
banner: https://tva4.sinaimg.cn/large/0077Un8Egy1h5k6z703otj30rs14079r.jpg
---

作者  *Eli Stevens   Luca Antiga   Thomas Viehmann*

<!--more-->

Deep learning with pytorch

译者 牟大恩

人民邮电出版社



![封面](https://wfqqreader-1252317822.image.myqcloud.com/cover/589/43134589/t6_43134589.jpg)

[代码仓库](https://github.com/deep-learning-with-pytorch/dlwpt-code)

## 第1章 深度学习和PyTorch库简介

深度学习使用大量数据来近似**输入和输出相距很远的复杂函数**，如输入是图像，输出是对输入进行描述的一行文本；或输入是书面文字，输出是朗读该文字的自然语音。

深度学习是一个巨大的空间，本书只覆盖该空间的一小部分。具体来说，包括一些使用PyTorch进行较小范围的图像分类和分割的项目，通过一些示例处理二维和三维的图像数据集。

为什么用PyTorch？

- Theano是最早的深度学习框架之一，目前它已经停止开发。
- TensorFlow：
  - 完全对Keras进行封装，将其提升为一流的API；
  - 提供了一种立即执行的“急切模式（eager mode）”，这种模式有点儿类似于PyTorch处理计算的方式；
  - TensorFlow 2.0默认采用急切模式。
- JAX是谷歌的一个库，它是独立于TensorFlow开发的，作为一个与GPU、Autograd和JIT编译器具有对等功能的NumPy库，它已经开始获得关注。
- PyTorch：
  - Caffe2完全并入PyTorch，作为其后端模块；
  - 替换了从基于Lua的Torch项目重用的大多数低级别代码；
  - 增加对开放式神经网络交换（Open Neural Network Exchange，ONNX）的支持，这是一种与外部框架无关的模型描述和交换格式；
  - 增加一种称为“TorchScript”的延迟执行的“图模型”运行环境；
  - 发布了1.0版本；
  - 取代CNTK和Chainer成为各自企业赞助商选择的框架。


随着TorchScript和eager模式的出现，PyTorch和TensorFlow的特征集开始趋同，尽管在这些特征的呈现和整体体验上仍然存在很大的差异。

事实上，由于性能原因，PyTorch大部分是用C++和CUDA编写的，CUDA是一种来自英伟达的类C++的语言，可以被编译并在GPU上以并行方式运行。有一些方法可以直接在C++环境中运行PyTorch，我们将在第15章中讨论这些方法。此功能的动机之一是为生产环境中部署模型提供可靠的策略。但是，大多数情况下我们都是使用Python来与PyTorch交互的，包括构建模型、训练模型以及使用训练过的模型解决实际问题等。

在PyTorch中，将运算从CPU转移到GPU不需要额外的函数调用。

## 第2章 预训练网络

在深度学习中，在新数据上运行训练过的模型的过程被称为**推理（inference）**。

```python
from torchvision import models
# 找到预定义的模型
dir(models)
# 首字母大写的名称指的是实现了许多流行模型的Python类，它们的体系结构不同，即输入和输出之间操作的编排不同。首字母小写的名称指的是一些便捷函数，它们返回这些类实例化的模型，有时使用不同的参数集。例如，resnet101表示返回一个有101层网络的ResNet实例，resnet152表示返回一个有152层网络的ResNet实例，以此类推。
```

训练时候开启`model.train()`，启用 BatchNormalization 和 Dropout，将BatchNormalization和Dropout置为True。测试（评估）时开启`model.eval()`。







