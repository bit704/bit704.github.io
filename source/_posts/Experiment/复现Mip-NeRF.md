---
title: 复现Mip-NeRF
layout: post
categories: [Experiment]
mathjax: false

---

Barron J T, Mildenhall B, Tancik M, et al. Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields[J]. arXiv preprint arXiv:2103.13415, 2021.

<!-- more -->

## 1. 运行环境

### 1.1 ubuntu

系统版本：21.10

显卡：NVIDIA GeForce RTX 3060 Laptop GPU

python环境：按照作者github的原环境安装即可，记得使用国内源加速 -i https://pypi.doubanio.com/simple。建议手动安装，直接pip install -r requirements.txt会下载很多多余的包，并且手动安装可以选择适配自己环境的。我安装的是tensorflow-gpu==2.5.0。

tensorflow 2.x不再区分是否gpu，当检测到gpu并安装cuda后，自动调用gpu。



## 2.实验内容

作者使用[NeRF官方数据集](https://drive.google.com/drive/folders/128yBriW1IG_3NJ5Rp7APSTZsJqdJdfc1)。使用以下命令从nerf_synthetic生成multiscale数据集:

```bash
python scripts/convert_blender_data.py --blenderdir /nerf_synthetic --outdir /multiscale
```

batch_size默认4096 ，我的显存6G，将batch_size调小为512。

直接运行，会报错：

```bash
E external/org_tensorflow/tensorflow/stream_executor/cuda/cuda_dnn.cc:364] Could not create cudnn handle: CUDNN_STATUS_INTERNAL_ERROR
```

这个怀疑是因为显存分配有问题。

并且，偶尔进程会被系统kill，怀疑是因为16G内存不够用。
