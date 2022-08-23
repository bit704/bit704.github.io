---
title: 复现NeuTex
layout: post
categories: [Experiment]
mathjax: false

---

Xiang F ,  Xu Z ,  Haan M , et al. NeuTex: Neural Texture Mapping for Volumetric Neural Rendering[J].  2021.

<!-- more -->

[代码仓库](https://github.com/fbxiang/NeuTex)

## 1. 运行环境

### 1.1 ubuntu

系统版本：21.10

显卡：NVIDIA GeForce RTX 3060 Laptop GPU

python环境：pytorch gpu版本加上trimesh、hdf5、h5py

备注：直接pip install torch的话会装成cpu版本，特别是添加了其他channel的情况下。另外，系统中安装了cuda就不需要按照官网的安装命令在conda中安装cudatoolkit, 会发生冲突。我的显卡环境是cuda11.2+cudnn8.1，我的pytorch安装命令如下。

```bash
pip install torch==1.9.0+cu111 torchvision==0.10.0+cu111 torchaudio===0.9.0 -f https://download.pytorch.org/whl/torch_stable.html
```

### 1.2 windows

系统版本：windows10

显卡：NVIDIA GeForce RTX 3060 Laptop GPU

python环境：和ubuntu一摸一样

将原工程里的dtu.sh重写为dtu.bat即可在windows下运行。在Anaconda prompt中在配好的环境下执行 .\dtu.bat 114即可。

## 2.实验内容

### 2.1 运行作者提供的DTU scene

首先，程序根据入参的dataset_name（“dtu”）加载实现写好的dtu_dataset.py类，用这个类从入参的data_root（"DTU/scan%1%/trainData"，%1%为入参114）下加载训练数据。它加载的就是存在hdf5里的图片数据和一些相机参数以及点云，还有用来分离图片背景的mask。

运行没有问题。

要使用自己的输入数据的话，需要根据作者的要求写一个自己的数据加载类，加载指定需要的数据即可（而类中的加载方式怎么写都行，所以源数据格式不重要）。



