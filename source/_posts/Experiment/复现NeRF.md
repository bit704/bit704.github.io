---
title: 复现NeRF
layout: post
categories: [Experiment]
mathjax: true
---

Mildenhall B ,  Srinivasan P P ,  Tancik M , et al. NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis[J]. Springer, Cham, 2020.

<!-- more -->

## 1. 运行环境

### 1.1 ubuntu

#### 1.1.1 软硬件

设备：华硕魔霸新锐2021

系统：Ubuntu 21.10  

显卡：NVIDIA GeForce RTX 3060 Laptop GPU

python环境：（与项目原环境不同）

conda

```yaml
# 运行 conda env create -f environment.yml
# 以下是environment.yml中的内容
name: nerf2
channels:
    - conda-forge
dependencies:
    - python=3.7
    - pip
    - numpy
    - matplotlib
    - imageio
    - imageio-ffmpeg
    - configargparse
    - imagemagick
    - pip:
        - tensorflow-gpu==2.5.0
```

tensorflow-gpu==2.5.0也可以通过*pip install tensorflow-gpu==2.5.0 (-i https://pypi.doubanio.com/simple* ,指定安装渠道) 安装。

#### 1.1.2 备注

1. 该项目的依赖imagemagick仅有linux和osx版本，没有windows版本，因此使用ubuntu。

2. Ubuntu 21.10 之前的LTS版本没有合适此设备的键盘驱动和WIFI驱动。

3. 3060只支持cuda11.1版本及以上，不兼容tensorflow1及相应cuda，运行项目会报错误（[类似问题](https://github.com/bmild/nerf/issues/127)），因此使用tensorflow2（[参考](https://zhuanlan.zhihu.com/p/357652959))。

4. 由于使用tensorflow2，以脚本为主、手工为辅的方式（[参考](https://zhuanlan.zhihu.com/p/248557773)）将原代码由tensorflow1改造升级到tensorflow2。

5. 发现在conda中安装cudatoolkit和cudnn会找不到3060，因此手动安装（[参考](https://zhuanlan.zhihu.com/p/393023811)）。采用cudnn 8.1+cuda 11.2。nvdia-smi显示cuda版本大于等于nvcc显示的cuda版本即可。

6. tensorflow-gpu==2.5.0不可使用conda安装，单独安装（[参考](https://blog.csdn.net/QAQIknow/article/details/118858870)）

7. 测试gpu

   ```python
   import tensorflow as tf
   print("Num GPUs Available: ", len(tf.config.experimental.list_physical_devices('GPU')))
   ```

8. 关闭ubuntu的合盖挂起。在使用GPU训练的时候如果挂机会导致nvidia驱动损坏，无法使用nvidia-smi，需要重装nvidia驱动（但不影响nvcc）。如果使用nvidia官方的驱动重装非常麻烦，并且可能会导致开机黑屏的bug，此刻需要进入ubuntu高级选项-恢复模式，通过root进入命令行，卸载nvidia驱动（直接使用命令卸载可能卸载不掉，需要使用nvidia驱动的安装文件卸载）。所以推荐使用标准ubuntu仓库进行驱动的自动安装。

   ```bash
   sudo ubuntu-drivers devices
   sudo ubuntu-drivers autoinstall
   sudo reboot
   ```

### 1.2 windows

#### 1.2.1 软硬件

设备：华硕魔霸新锐2021

系统：windows 10

显卡：NVIDIA GeForce RTX 3060 Laptop GPU

python环境：（与项目原环境不同）

conda

```yaml
# 运行 conda env create -f environment_win.yml
# 以下是environment_win.yml中的内容
name: nerf2
channels:
    - conda-forge
dependencies:
    - python=3.7
    - pip
    - numpy
    - matplotlib
    - imageio
    - imageio-ffmpeg
    - configargparse
    - pip:
        - tensorflow-gpu==2.5.0
```

#### 1.2.2 备注

1. windows10中需安装好cudnn 8.1+cuda 11.2。
1. 代码需要升级到tensorflow2。
2. 相比在ubuntu中配置环境时去掉了原作者的imagemagick依赖。可以正常训练。
2. 在conda中激活建立好的nerf2环境后，使用命令python run_nerf_RE.py --config ./configs/config_fern.txt开启训练fern模型。



## 2.实际运行

### 2.1 本地验证

复现使用作者的[原代码](https://github.com/bmild/nerf)，在这里使用python run_nerf.py --config config_fern.txt 运行fern demo。（没有使用paper_configs里的llff_config，使用的作者提供的简化版config_fern.txt）

如果直接使用原来的参数运行，由于训练规模过大，在训练一定轮次后，会报**显存不够**的错误：

tensorflow.python.framework.errors_impl.ResourceExhaustedError: OOM when allocating tensor with shape[4194304,90] and type float on /job:localhost/replica:0/task:0/device:GPU:0 by allocator GPU_0_bfc [Op:ConcatV2] name: concat

![显存不足](https://tva2.sinaimg.cn/large/0077Un8Egy1h5fdwdq9wkj30y30dlqh2.jpg)

应该是因为我的3060的6G显存不如作者使用的nvidia v100。

因此，将config_fern.txt中的参数由：

```
N_samples = 64
N_importance = 64
```

降低为：

```
N_samples = 16
N_importance = 16
```

以减小训练规模，然后可以正常训练。

实际运行中的GPU情况：

![GPU使用情况](https://tvax3.sinaimg.cn/large/0077Un8Egy1h5fdwmdmocj30mu0etdp7.jpg)

**Terminal输出：**

![Terminal输出](https://tvax3.sinaimg.cn/large/0077Un8Egy1h5fdwt49h2j30ku0kb7g2.jpg)

**tesnorboard输出：**

使用命令tensorboard --logdir=logs/summaries --port=6006，查看网页

![tensorboard1](https://tva4.sinaimg.cn/large/0077Un8Egy1h5fdx08peoj30uk0rvagd.jpg)

![tensorboard2](https://tva2.sinaimg.cn/large/0077Un8Egy1h5fdx52bzbj30us0sfgua.jpg)

最后输出的视频位于logs/fern_test文件夹中

完成整个训练过程大概需要34个小时（跑完1000k  iterations，没有收敛）。因此减小训练参数效果不好。

### 2.2 复现论文

因为在本地6G显存的3060上无法复现论文，所以采用24G的3090进行复现。

**3060与3090可以使用完全相同的环境。**

实际复现时使用了17G显存，迭代了200k次，大概24个小时。PSNR达到了论文的效果。

### 2.3 训练细节

llff类型数据：

使用手机拍摄20张以上的照片，使用colmap自动生成相机参数。

我使用的是和作者一个型号的手机，因此拍照照片大小一样，均为4096*3024。拍完后分别做四分之一降采样和八分之一降采样的照片备用。作者在论文中使用的是四分之一降采样的照片做训练。

要求图片的长宽是16的整数倍，否则网络会做裁剪处理，可能会导致损失增大。

手动拍摄的照片之间基线不能太宽，角度变化不能太大。

```python
# render_rays函数的以下语句在处理360度环视场景时会报错
tf.debugging.check_numerics(ret[k], 'output {}'.format(k))
```

## 3.代码分析

以实际运行的fern demo为例子

### 3.1 输入参数列表

| 参数名         | 默认值           | 作用                                                         |
| :------------- | ---------------- | :----------------------------------------------------------- |
| config         |                  | 配置文件路径                                                 |
| expname        |                  | 实验名                                                       |
| basedir        | ./logs/          | 存放ckpts和logs的路径                                        |
| datadir        | ./data/llff/fern | 输入数据路径                                                 |
| netdepth       | 8                | 网络层数                                                     |
| netwidth       | 256              | 每层的通道数                                                 |
| N_rand         | 32\*32\*4        | 批大小 (每一梯度步的随即光线数)                              |
| lrate          | 5e-4             | 学习率                                                       |
| lrate_decay    | 250              | 指数学习率下降(1000s内)                                      |
| chunk          | 1024\*32         | 平行处理的光线数(内存不足时应减小)                           |
| netchunk       | 1024\*64         | 网络里平行发送的pts数(内存不足时应减小)                      |
| no_batching    | False            | 一次只从一张图片获取随机光线                                 |
| no_reload      | False            | 不从保存的模型中重载权重                                     |
| ft_path        | None             | 专为coarse网络重载的权重npy文件                              |
| random_seed    | None             | 确保结果可以复现的固定随机种子                               |
| precrop_iters  | 0                | 在central crops训练的步数                                    |
| precrop_frac   | 0.5              | 为central crops拿的图片部分                                  |
| N_samples      | 64               | 每道光线的粗采样数                                           |
| N_importance   | 0                | 每道光线的附加细采样数                                       |
| perturb        | 1.               | 0. for no jitter, 1. for jitter                              |
| use_viewdirs   | False            | 使用5D而不是3D输入                                           |
| i_embed        | 0                | 0对应默认位置编码，1对应None                                 |
| multires       | 10               | 位置编码的最大频率对数（3D位置）                             |
| multires_views | 4                | 位置编码的最大频率对数（2D方向）                             |
| raw_noise_std  | 0.               | std dev of noise added to regularize sigma_a output, 1e0 recommended |
| render_only    | False            | do not optimize, reload weights and render out render_poses path |
| render_test    | False            | render the test set instead of render_poses path             |
| render_factor  | 0                | downsampling factor to speed up rendering, set 4 or 8 for fast preview |
| dataset_type   | llff             | 三个选项：llff / blender / deepvoxels                        |
| testskip       | 8                | will load 1/N images from test/val sets, useful for large datasets like deepvoxels |
| shape          | greek            | 四个选项：armchair / cube / greek / vase                     |
| white_bkgd     | False            | set to render synthetic data on a white bkgd (always use for dvoxels) |
| half_res       | False            | load blender synthetic data at 400x400 instead of 800x800    |
| factor         | 8                | LLFF图片的降采样因子                                         |
| no_ndc         | False            | do not use normalized device coordinates (set for non-forward facing scenes) |
| lindisp        | False            | sampling linearly in disparity rather than depth             |
| spherify       | False            | set for spherical 360 scenes                                 |
| llffhold       | 8                | will take every 1/N images as LLFF test set, paper uses 8    |
| i_print        | 100              | 控制台打印频率                                               |
| i_img          | 500              | tesnorboard图片注册频率                                      |
| i_weights      | 1000             | 权重模型保存频率                                             |
| i_testset      | 5000             | 测试集保存频率                                               |
| i_video        | 5000             | 渲染姿态视频保存频率                                         |

### 3.2 训练过程

输入数据位于data/nerf_llff_data/fern

输出数据位于logs/fern_test

- 调用run_nerf.py train()函数
	- 调用config_parser()函数，获取外部参数args
	- 确定random_seed 
	- 确定数据类型，llff / blender / deepvoxels其中之一。
	- 复现使用数据类型为llff，因此调用load_llff.py  load_llff_data()加载数据。
	  - load_llff_data()传入数据路径（./data/nerf_llff_data/fern）、降采样因子（8）、recenter (True)，bd_factor(0.75)，spherify（False）
	  - 根据根据降采样因子处理图片得到新图片（如果不存在）
	  - 从poses_bounds.npy读取poses和bds，从图片文件夹读取imgs
	- 将使用的配置写入args.txt和config.txt
	- 使用create_nerf()建立模型
	- 如果render_only参数为真，从训练好的模型渲染出结果
	- 开启训练
	- 输出的视频为螺旋视角移动的视频

### 3.3 Pose生成

论文作者推荐使用LLFF code<sup><a class=n href="#ref3">[3]</a></sup> 里的脚本imgs2poses.py来生成Pose，然后将数据以llff类型传入模型。于是我先采用一下这个方法。

（以下过程是在ubuntu上编译安装colmap，比较麻烦。在windows上直接安装很简单，下载好之后配上环境变量就行了。）

我一开始使用conda直接安装colmap，但是运行时会报错：

```bash
Shader not supported by your hardware!
ERROR: SiftGPU not fully supported.
```

![SiftGPU](https://tvax2.sinaimg.cn/large/0077Un8Egy1h5fdxedtk9j30yd0j34ii.jpg)

这个问题我完全不知道怎么解决，显然3060及其驱动又和程序出现了兼容性问题。

可以从github的相关issue中看到[有人提到](https://github.com/colmap/colmap/issues/971) SiftGPU is rather old and not optimized with the latest GPU architecture, so it is very inefficient.

~~但是我的驱动已经是为了在3060上训练模型专门选择过了的，我不想再次更换。~~

于是我手动编译[安装colmap](https://colmap.github.io/install.html)，遇到了以下问题：

1. 编译 [Ceres Solver](http://ceres-solver.org/)这一步会把16G的内存跑满，导致编译失败甚至电脑重启。最好的解决办法是把交换区扩大。我是参考[这篇文章](https://www.linuxidc.com/Linux/2019-07/159580.htm)把交换区从2G扩大到10G。
2. 编译colmap过程中，因为安装了anaconda导致系统库的路径被覆盖掉。进行到这里时，把~/.bashrc里的conda的环境变量注释掉。
3. ubuntu21.10自带的gcc (Ubuntu 11.2.0-7ubuntu2) 11.2.0和g++ (Ubuntu 11.2.0-7ubuntu2) 11.2.0版本太高，无法正确编译colmap中的一些模块，因此安装9版本代替11版本（[参考](https://blog.csdn.net/Cxiazaiyu/article/details/106731687)）。

手动安装的colmap终于可用了，但我遇到了一个全新的问题。（[相同的issue](https://github.com/Fyusion/LLFF/issues/36)）

![error1](https://tvax4.sinaimg.cn/large/0077Un8Egy1h5fdxpb3p7j30k5066q6b.jpg)

根据我自己的实验，colmap在作者提供的数据集上运行的很完美，说明只是我只是拍的照片不够“好”，不足以让colmap匹配这些图片。（这种情况是，如果我对自己拍摄的照片进行降采样，就会使colmap无法匹配）

我自己用手机拍摄的一个数据集：

![toyset](https://tva4.sinaimg.cn/large/0077Un8Egy1h5fdxuvwc3j30k6099jw2.jpg)

![toy](https://tva4.sinaimg.cn/large/0077Un8Egy1h5fdy3h4cnj33402c0b29.jpg)

​			实验证明可以用colmap生成相机参数并输入nerf或grf训练。



## 4.评价标准

### 4.1 PSNR

Peak Signal to Noise Ratio 峰值信躁比

它是基于对应像素点间误差的图像质量评价。

> 由于并未考虑到人眼的视觉特性（人眼对空间频率较低的对比差异敏感度较高，人眼对亮度对比差异的敏感度较色度高，人眼对一个区域的感知结果会受到其周围邻近区域的影响等），因而经常出现评价结果与人的主观感觉不一致的情况。

$$
MSE=\frac{1}{H\times W} \sum_{i=1}^{H} \sum_{j=1}^{W}(X(i,j)-Y(i,j))^2
$$

$$
PSNR=10\log_{10}{(\frac{(2^{n}-1)^2}{MSE} )} 
$$

其中，MSE表示当前图像X和参考图像Y的均方误差（Mean Square Error)，H、W分别为图像的高度和宽度；n为每像素的比特数，一般取8，即像素灰阶数为256。

$PSNR$的单位是dB，数值越大表示失真越小。

一般来说，$PSNR$高于40dB说明图像质量几乎与原图一样好；在30-40dB之间通常表示图像质量的失真损失在可接受范围内；在20-30dB之间说明图像质量比较差；低于20dB说明图像失真严重。

### 4.2 SSIM

结构相似性

它分别从照明度(luminance)、对比度 (contrast) 和结构 (structure)三方面度量图像相似性。
$$
l(X,Y)=\frac{2\mu_{x}\mu_{y}+C_{1}}{\mu_{x}^2+\mu_{y}^2+C_{1}}
$$

$$
c(X,Y)=\frac{2\sigma_{x}\sigma_{y}+C_{2}}{\sigma_{x}^2+\sigma_{y}^2+C_{2}}
$$

$$
s(X,Y)=\frac{\sigma_{xy}+C_{3}}{\sigma_{x}\sigma_{y}+C_{3}}
$$

其中$\mu_{x}$、$\mu_{y}$分别表示图像X和Y的均值，$\sigma_{x}$、$\sigma_{y}$分别表示图像X和Y的方差，$\sigma_{xy}$表示图像X和Y的协方差。

$C_{1}$、$C_{1}$、$C_{3}$为常数，为了避免分母为0的情况，通常取$C_{1}=(K_{1}*L)^2$, $C_{2}=(K_{2}*L)^2$, $C_{3}=C_{2}/2$，一般地$K_{1}=0.01$, $K_{2}=0.03$, $L=255$。
$$
SSIM(X,Y)=l(X,Y)c(X,Y)s(X,Y)\sim[0,1]
$$
$SSIM$取值范围$[0,1]$，值越大，表示图像失真越小。

### 4.3 LPIPS

Learned Perceptual Image Patch Similarity 学习感知图像块相似度

它也称为“感知损失”(perceptual loss)，用于度量两张图像之间的差别。<sup><a class=n href="#ref2">[2]</a></sup> 

$LPIPS$的值越低表示两张图像越相似，反之，则差异越大。



## 参考文献

<a name="ref1">[1] Mildenhall B ,  Srinivasan P P ,  Tancik M , et al. NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis[J]. Springer, Cham, 2020.</a>

<a name="ref2">[2] Zhang R ,  Isola P ,  Efros A A , et al. The Unreasonable Effectiveness of Deep Features as a Perceptual Metric[J]. IEEE, 2018.</a>

<a name="ref3">[3]  Mildenhall B ,  Srinivasan P P ,  Ortiz-Cayon R , et al. Local Light Field Fusion: Practical View Synthesis with Prescriptive Sampling Guidelines[J]. ACM Transactions on Graphics, 2019, 38(4):1-14.</a>
