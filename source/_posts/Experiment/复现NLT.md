---
title: 复现NLT
layout: post
categories: [Experiment]
mathjax: false
---

Zhang X ,  Fanello S ,  Tsai Y T , et al. Neural Light Transport for Relighting and View Synthesis[J].  2020.

<!-- more -->

## 1.运行环境

原工程在ubuntu下测试，并且作者自己写的第三方库不兼容windows，因此其不能在windows上运行。

为了使其在3060、cuda11.2.0、cudnn8.1.1下运行，将原conda环境更改如下：

```yaml
name: nlt
channels:
    - conda-forge
    - anaconda
dependencies:
    - python=3.6
    - pip
    - numpy
    - ipython
    - pillow
    - opencv
    - tqdm
    - pip:
        - tensorflow==2.5.0
        - tensorflow-addons==0.13.0
        - tensorflow-probability==0.13.0
    - mpmath
    - matplotlib
```

## 2.实验内容

### 2.1 使用作者的数据复现

在作者的[项目主页](http://nlt.csail.mit.edu/)下载dragon_specular.zip，dragon_sss.zip。

可以在环境变量中添加作者自己的第三方库，也可以在pycharm的configurations中添加运行时的环境变量，注意ubuntu中的环境变量用:隔开。

原代码是在ubuntu上运行的，因此配置文件里在文件名中使用:，而这在windows下是不允许的，可以将其修改为$。

但是作者的自己写的第三方库并没有兼容windows，运行会报错:

![windows不兼容](https://bit704.oss-cn-beijing.aliyuncs.com/image/2022-09-18-windows%E4%B8%8D%E5%85%BC%E5%AE%B9.png)

<font color=red>非常坑的一点就是中断之后重新运行，不会加载原来保存的checkpoint而是直接覆盖输出目录重新从0开始训练，记得把配置文件里的overwrite改为False。</font>

作者配置文件里的bs对应batch_size，值是4。直接跑的话可能会因为显存不够进程挂掉（有时候不会），改为2进行训练。大概需要30个小时完成训练（3060,6G显存）。

dragon_specular.zip训练结果：

![dragon_specular_ckpt-100_pred_2fps](https://bit704.oss-cn-beijing.aliyuncs.com/image/2022-09-18-dragon_specular_ckpt-100_pred_2fps.gif)

dragon_sss.zip训练所需的神经网络学习率是前者的1/4，且深度是前者的4倍，可能会慢很多。并且将其batch size减半依然超出6G显存，暂时没有训练。



