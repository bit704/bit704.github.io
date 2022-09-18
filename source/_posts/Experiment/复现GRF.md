---
title: 复现GRF
layout: post
categories: [Experiment]
mathjax: false
---

Trevithick A ,  Yang B . GRF: Learning a General Radiance Field for 3D Scene Representation and Rendering[J].  2020.

<!-- more -->



## 1. 运行环境

可以使用与《复现NeRF》完全相同的环境。GRF的代码一部分基于NeRF。

## 2.实验内容

参考论文中的实验部分。一共四个实验，前三个实验分别验证在不同物体、不同类别、不同场景的泛化能力，最后一个实验验证其在单个场景上的表现也胜过以往。

### 2.1 不同物体泛化

使用[SRN cars and chairs datasets](https://drive.google.com/drive/folders/1OkYgeRcIcLOFu1ft5mRODWNQaPJ0ps90)，在cars和chairs这两个数据集上分别训练单独的模型。对每个训练好的模型：

1. 喂入一个新物体的2个视角的图片，然后推断其他251个视角的图片。
2. 同上，但只喂入一个视角的图片。

### 2.2 不同类别泛化

使用[DISN MultiShapenet dataset](https://github.com/Xharlie/ShapenetRender_more_variation),在 {chair,bench,car,airplane,table,speaker} 6个类别的数据集上训练，在{cabinet,display,lamp,phone,rifle,sofa,watercraft} 7个新类别上验证。对于训练好模型：

1. 喂入属于新类别的一个新物体的2个视角的图片，然后推断全部36个视角的图片。
2. 同上，喂入6个视角的图片。

### 2.3 不同场景泛化

使用[NeRF dataset](https://drive.google.com/drive/folders/128yBriW1IG_3NJ5Rp7APSTZsJqdJdfc1)，对于其中的8个合成场景，随机选取4个场景进行训练。对于训练好的模型：

1. 将它直接在其它4个新场景上进行验证。
2. 将它分别在其他4个新场景上进行微调（分别训练100,1k,10k次）,这样就得到3\*4=12个新模型。使用NeRF分别在这4个新场景上训练（分别训练100,1k,10k次），然后与之对比。

### 2.4 单个场景表现

使用[NeRF dataset](https://drive.google.com/drive/folders/128yBriW1IG_3NJ5Rp7APSTZsJqdJdfc1)，在单个场景上进行训练，与NeRF对比。

## 3. 实际运行

### 3.1 不同物体泛化

因为chairs数据集太大了，这里只使用cars数据集。将cars_test.zip，cars_train.zip，cars_train_test.zip，cars_train_val.zip，cars_val.zip五个压缩包下载下来，解压并把文件夹名字**去掉“cars_”前缀**。

使用python run.py --config  config_shapenet_cars.txt运行程序。这里使用和作者一摸一样的配置。

**注意事项：**

1. 如果在windows下运行，config文件中数据路径写的windows路径，记得将load_shapenet_rewrite_lowmem.py里的load_shapenet_data函数中对文件路径操作时用的分隔符从'/'换成'//'。

2. 数据集中有部分数据不合法，不满足load_shapenet_rewrite_lowmem.py里的read_scene函数中的

   ```python
   assert len(rgb_files) == len(pose_files), scene_path
   ```

   因此加上if判定忽视掉不合法数据使程序正常运行。

3. run.py中的render_rays函数最后的检查语句会报错，因此将其注释掉：

   ```python
   tf.debugging.check_numerics(ret[k], 'output {}'.format(k))
   ```

   这很奇怪，因为我采用与作者一摸一样的配置。

   备注：GRF的render_rays函数与NeRF中的render_rays函数基本相同。NeRF中这个检查语句在我训练forward-facing场景时不会报错，训练360度场景时则会报错，但训练结果都是正常的。

4. 第一次训练后发现没有tensorboard输出。后续有空给代码加上。

**训练情况：**

使用24G的3090，显存基本占满。根据作者的说法他将32G的V100占满了。

GRF开始的训练是正常的，过了一段时间之后acm_loss全部变成nan。

根据作者在Github Issue中的说法，训练400k次应该是足够的，因此我训练了30个小时完成400次。

