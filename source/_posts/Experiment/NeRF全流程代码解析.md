---
title: NeRF全流程代码解析
layout: post
categories: [Experiment]
mathjax: false
---

代码分析

<!-- more -->



## 1.相机参数生成

作者原代码见[Fyusion/LLFF: Code release for Local Light Field Fusion at SIGGRAPH 2019 (github.com)](https://github.com/Fyusion/LLFF)

可以参考[相机参数相关脚本](./相机参数相关脚本.md)

进入LLFF-master，运行指令python imgs2pose.py ./fern/，即可生成./fern/images/中照片的相机参数。

```flow
start=>start: gen_poses
cond1=>condition: 是否已经存在相机参数?
op1=>operation: run_colmap
op2=>operation: load_colmap_data
op3=>operation: 调用系统中安装好的colmap生成cameras.bin、images.bin、points3D.bin
op4=>operation: 加载colmap生成的bin文件,返回poses, pts3d, perm
op5=>operation: save_poses
end=>end: Done
start->cond1
cond1(no)->op1->op3(left)->op2
cond1(yes)->op2->op4->op5->end
```





### 1.1 入口函数(./imgs2pose.py)

它接受两个参数，必选参数是scenedir，照片必须放在./scenedir/images/下（此例中是./fern/）；可选参数是match_type，有exhaustive_matcher（默认）和sequential_matcher两种。然后调用gen_poses函数。



### 1.2 gen_poses函数(./llff/poses/pose_utils.py)

它首先判断是否已有相机参数。没有的话调用**run_colmap函数**(./llff/poses/colmap_wrapper.py)生成相机参数。

run_colmap函数通过开启子进程的方式调用系统中安装好的colmap，令其在./fern/文件夹下依次执行：

```bash
特征提取
colmap feature_extractor --database_path ./fern/database.db --image_path ./fern/images --ImageReader.single_camera 1
特征点匹配
colmap exhaustive_matcher --database_path ./fern/database.db
稀疏重建
colmap mapper --database_path  ./fern/database.db  --image_path  ./fern/images 
  --output_path  ./fern/sparse  # --export_path changed to --output_path in colmap 3.6
  --Mapper.num_threads 16
  --Mapper.init_min_tri_angle 4
  --Mapper.multiple_models 0
  --Mapper.extract_colors 0
```

colmap运行日志打印到./fern/colmap_output.txt。



### 1.3 load_colmap_data函数(./llff/poses/pose_utils.py)

它调用./llff/poses/colmap_read_model.py下的函数加载colmap生成的相机参数（./fern/sparse/0/下的cameras.bin、images.bin、points3D.bin），返回poses, pts3d, perm。

加载cameras.bin得到**相机内参**：

```json
{1: Camera(id=1, model='SIMPLE_RADIAL', width=4032, height=3024, params=array([3.32041042e+03, 2.01600000e+03, 1.51200000e+03, 1.39666458e-02]))}
```

以上参数对应相机编号, 相机模型, 宽, 高, ~~params[fx,fy,cx,cy]（fx、fy是焦距，cx、cy是光心）~~这个网上的解释是错误的，我翻了colmap的源码，应该是params[f,cx,cy,k] ，f是焦距，cx、cy是光心，k是畸变参数。这个相机模型SIMPLE_RADIAL是opencv相机模型的简化。

可以参考：[colmap 相机模型及参数 - 小小灰迪 - 博客园 (cnblogs.com)](https://www.cnblogs.com/xiaohuidi/p/15767477.html)

最终取hwf（高、宽、焦距）。

加载images.bin得到**相机外参**：

```json
Image(id=1, qvec=array([ 0.9977913 ,  0.01760456, -0.06277221,  0.01273779]), tvec=array([3.56130391, 1.68425   , 0.85253793]), camera_id=1, name='IMG_4026.JPG', xys=array([[ 157.9755249 ,    6.25280905],
       [  37.46022034,    8.62553501],
       [ 493.85971069,    8.32364368],
       ...,
       [1982.91186523,  725.0369873 ],
       [1232.66833496, 2057.39038086],
       [ 981.81091309, 2577.60693359]]), point3D_ids=array([   -1,    -1,    -1, ...,    -1,    -1, 10720]))

```

qvec为四元数表示的相机旋转信息，tvec为平移向量。相机编号对应上面相机内参中的相机编号。名字是照片名。xys是该照片中找到的特征点，point3D_ids对应每个特征点的3D索引，如果其为-1则代表该点在重建过程中没有被观测为3D点。

qvec2rotmath函数将四元数qvec转化为3\*3旋转矩阵。将3\*3旋转矩阵和3\*1平移向量和bottom位移向量[0,0,0,1]一起组装成**4\*4相机外参矩阵**。

所有照片的相机外参矩阵一起堆叠成w2c_mats，对其求逆矩阵得c2w_mats。（假设有20张照片，则shape为(20, 4, 4)）

(世界坐标系到相机坐标系的转换为w2c，相机坐标系到世界坐标系的转换为c2w）

对c2w_mats去掉第四行得到shape为(20, 3, 4)，再转置为(3, 4, 20)，得到poses。

将加载相机内参得到的hwf变形为(3, 1, 1),再沿axis=2复制扩张20倍（即poses.shape[-1]）得到hwf(3, 1, 20)。

将poses(3, 4, 20)和hwf(3, 1, 20)组装得到新的poses(3, 5, 20)。对新poses的axis=1(第二个轴（列）)还要进行重排。

[1,2,3,4,5]的列顺序**重排**为[2,1,-3,4,5]。实际上就是旋转了一下坐标系，左手系还是右手系不变。

(代码里的解释是must switch to [-u, r, -t] from [r, -u, t], NOT [r, u, -t])

加载points3D.bin得到**稀疏3D点**:

```json
Point3D(id=1, xyz=array([ -1.95661541, -14.04277513,  31.71051775]), rgb=array([0, 0, 0]), error=array(1.21389677), image_ids=array([7, 6]), point2D_idxs=array([63, 63]))
```

image_ids是该点存在于哪些照片之中（照片从1开始被编号，参见上面的相机外参）。point2D_idxs是该点在相应照片（image_ids）中对应的特征点的3D索引（即上面的相机外参中的point3D_ids）。

最后返回的perm是照片名数组的排序索引，poses是相机变换矩阵，pts3d是稀疏3D点。



### 1.4 save_poses函数(./llff/poses/pose_utils.py)

输入是load_colmap_data函数得到的poses, pts3d, perm。

经过处理，pts_arr存放所有点的xyz坐标，vis_arr存放所有点在每张照片中的存在情况（1：存在，0：不存在），zvals是计算得到的深度信息。对于perm中的每张照片，取出在其上可见的所有点的深度信息，按百分比0.1和99.9取最小深度和最大深度，将其和这张照片的poses数组展开拼接为一维数组（3*5+2=17）。假设有20张照片的情况下，最终得到(20,17)的save_arr数组，将其保存到poses_bounds.npy供NeRF使用。



## 2.NeRF代码解析

作者原代码见[bmild/nerf: Code release for NeRF (Neural Radiance Fields) (github.com)](https://github.com/bmild/nerf)

进入nerf-dev, 运行指令python run_nerf_RE.py --config ./configs/config_fern.txt开启训练fern模型。

```flow
start=>start: train
op1=>operation: load_llff_data/load_blender_data/load_dv_data
sub1=>subroutine: create_nerf
op2=>operation: run the model
end=>end: Done
start->op1->sub1->op2->end
```



### 2.1 入口函数train(./run_nerf.py)

读取设定好的参数，输入参数列表见《复现NeRF》。

首先根据不同的数据类型读入数据，这里以load_llff_data为例。

选取训练集i_train和测试集i_test、i_val。



### 2.2 load_llff_data部分(./load_llff.py)

读取生成相机参数代码生成的poses_bounds.npy，得到poses和bds；读照片文件夹，得到images。设照片数量为num，读取的poses、bds、imgs的shape为(num,3,5) (num,2) (num,768,1024,3)。

其中对poses矩阵的第二个轴（列）进行重排。[1,2,3,4,5]的列顺序**重排**为[2,1,-3,4,5]，实际上就是旋转了一下坐标系，左手系还是右手系不变。

生成render_poses:对于前向图片，得到螺旋渲染姿态；对于环视图片，得到旋转渲染姿态。


> recenter_poses函数：
>
>可视化之后大概知道是把全部相机的位置（以原点为中心）绕x轴旋转了一个角度。但是不知道是为了什么。


> spherify_poses函数：
>
> 可视化效果是把poses聚拢之后又转了一个小角度。

> render_path_spiral函数：
>
> 返回一组密集的环绕视角用于生成视频。环绕视角的中心轴垂直于xy平面。

求dists然后选i_test是为了选择居中的一张图片作为测试集。

其中有一些我不是很理解，因为我对相关理论还不熟：

<font color=red>bd_factor的用处是什么？</font>

<font color=red>path_zflat有什么作用？</font>



### 2.3 create_nerf部分(./run_nerf.py)

创建NeRF的MLP模型

>  get_embedder函数(./run_nerf_helpers.py)：入参是位置编码的最大频率的log2对数multires和位置编码方式i_embed，出参是一个函数embed_fn和输入维度input_ch。
>
>  embed_fn：操作一个1+multires*2长度的函数列表对输入数进行计算，返回一个拼接的tensor。这个函数列表第一个函数是y=x, 此后每两个函数分别是y=sin(x\*freq)和y=cos(x\*freq)。freq是从2^0到2^(multires-1)按对数采样的multires个数。
>
>  input_ch为(1+multires\*2)\*input_dims。input_dims在程序里是常数3。
>
>  对于3D方向输入，multires默认是10，因此input_ch=63；对于2D方向输入，multires默认是4，因此input_ch_views=27。

首先，调用get_embedder函数获得针对3D位置的embed_fn, input_ch，和针对2D方向的embeddirs_fn, input_ch_views（如果采用5D输入）。

设定output_ch = 4，skips = [4]，调用init_nerf_model函数生成模型model。

> init_nerf_model函数(./run_nerf_helpers.py)：入参是深度（默认8）、宽度（默认256）、input_ch、 input_ch_views、output_ch、skips、 use_viewdirs（是否采用5D输入），出参是model。
>
> model使用ReLU作为激活函数，输入层inputs的shape为(input_ch + input_ch_views)，然后再分割为inputs_pts [None, input_ch] 和inputs_views [None, input_ch_views]，分别针对3D位置和2D方向。输出层为outputs。
>
> 广度为全连接层的输出数量，深度为全连接层的数目。skips对应的第五层将[inputs_pts, outputs]连接起来做一次跳跃。
>
> 如果采用5D输入，还要再增加alpha_out层和bottleneck层，同样在中间做跳跃连接。

如果N_importance大于0 ，说明还要进行细采样，还要再生成一个模型model_fine。

将所有训练变量装入grad_vars。

定义函数**network_query_fn**，它调用**run_network**函数为网络准备输入。**这个函数很关键，它是体渲染代码调用神经网络的入口。**

> run_network函数(./run_nerf.py)：
>
> 对输入调用embed_fn映射到高维空间，然后调用fn(也就是MLP模型)依据配置的args.netchunk每次选择指定数目的**点**进行并行计算。

最后检查是否有已经训练好的参数可供加载，设置开始步数。

返回 render_kwargs_train, render_kwargs_test, start, grad_vars, models。分别是训练渲染参数，测试渲染参数，开始步数，训练变量，模型。

render_kwargs_train, render_kwargs_test的区别就是

```python
render_kwargs_test['perturb'] = False
render_kwargs_test['raw_noise_std'] = 0.
```



### 2.4 run the model部分(./run_nerf.py)



如果只是渲染的话，直接调用render_path函数渲染出结果，然后返回。

>render_path函数(./run_nerf.py)：
>
>入参gt_imgs函数是真实图像的意思
>
>调用render函数依据配置的args.chunk每次选择指定数目的**光线**进行并行渲染。输入render_kwargs_test。
>
>返回rgbs和disps，分别代表颜色和视差

设置好优化器和全局训练步数。

如果使用随机光线批（即每次从所有图片中选取随机光线而不是只从一张图片选取随机光线），对于每张照片的pose，调用get_rays_np函数得到光线信息，和图片拼接在一起，得到所有图片所有像素的光线rays_rgb，shape是(N\*H\*W, 3(即ro+rd+rgb), 3)，其中ro是光线原点，rd是光线方向，rgb对应的像素颜色。（只取i_train对应的训练图片 ，并对所有光线做shuffle打乱）。

> get_rays_np函数(./run_nerf_helpers.py)：入参是H、W、focal、c2w，出参是rays_d、rays_o（shape是(H,W,3)）。代表一张图片上每处光线的方向和起点。
>
> 和get_rays函数应该是一摸一样。

对于每一次训练，如果使用随机光线批，一次从rays_rgb中取N_rand条光线（默认是32 * 32 * 4）batch，切分为batch_rays (ro、rd)和 target_s（rgb）。

> render函数(./run_nerf.py)：输入参数是H, W, focal, chunk（一次处理的光线数量。默认1024\*32）, rays（batch_rays), render_kwargs_train（训练渲染参数）等，输出是rgb, disp, acc, extras。具体参数含义程序中有详细注释。
>
> 将rays分成rays_o和rays_d，rays_d除以二范数得到viewdirs。
>
> 对于forward facing scenes，调用ndc_rays函数将rays_o和rays_d转化为NDC坐标。
>
> 重新组装得到rays(ray origin(3), ray direction(3), min dist(1), max dist(1), normalized viewing direction(3))。为了防止OOM,调用batchify_rays函数按照chunk大小调用render_rays函数分批渲染得到结果。
>
> render_rays函数：
>
> 进行体积渲染，在每条光线上选取一定数量的点采样，用MLP计算点的颜色和密度，积分得到光线的颜色。
>
> 首先在粗模型上采样，获得输出；然后在精细模型上调用sample_pdf函数根据概率密度函数进行层次采样。
>
> 


核心步骤：**通过render函数渲染得到结果rgb与target_s对比，得到loss，输入优化器。**

之后的操作就是根据训练步数记录训练结果，输出图片和视频。训练直到指定步数结束。

## 3.NeRF思想

[NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis_Vpn_zc的博客-CSDN博客](https://blog.csdn.net/Vpn_zc/article/details/115729297)



