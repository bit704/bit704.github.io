# 复现plenoxel

https://github.com/sxyu/svox2

[toc]

## 1.运行环境

注释掉原environment.yml中的pytorch、torchvision、cudatoolkit。直接pip install torch的话会装成cpu版本。应使用如下命令安装：

```bash
pip install torch==1.9.0+cu111 torchvision==0.10.0+cu111 -f https://download.pytorch.org/whl/torch_stable.html
```

使用以下命令安装库(svox2)：

```bash
pip install .
```

## 2.运行

下载作者训练好的模型，直接进行评估。

```bash
python .\opt\render_imgs_circle.py .\ckpt\ckpt.npz .\data\fern\
```

在RTX3060上，渲染一个40秒、30帧的视频需要20分钟左右的时间和4-5G显存。
