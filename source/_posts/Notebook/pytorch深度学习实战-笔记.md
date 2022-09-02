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

这本书的中文翻译不是很行。

[toc]

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

这章主要下载使用了一些不同种类的预训练网络。

在深度学习中，在新数据上运行训练过的模型的过程被称为**推理（inference）**。

```python
from torchvision import models
# 找到预定义的模型
dir(models)
# 首字母大写的名称指的是实现了许多流行模型的Python类，它们的体系结构不同，即输入和输出之间操作的编排不同。首字母小写的名称指的是一些便捷函数，它们返回这些类实例化的模型，有时使用不同的参数集。例如，resnet101表示返回一个有101层网络的ResNet实例，resnet152表示返回一个有152层网络的ResNet实例，以此类推。
```

训练时候开启`model.train()`，启用 BatchNormalization 和 Dropout，**将BatchNormalization和Dropout置为True**。测试（评估）时开启`model.eval()`。

## 第3章 从张量开始

张量可简单理解为多维数组。

Python列表或数字元组是在内存中单独分配的Python对象的集合，而PyTorch张量或NumPy数组通常是连续内存块的视图，这些内存块包含未装箱的数字类型，而不是Python对象。

tensor操作和numpy类似。

为了避免张量运算时粗心造成的对齐错误，可以对张量命名。

张量构造函数通过dtype参数指定包含在张量中的数字的数据类型：

- torch.float32或torch.float：32位浮点数。
- torch.float64或torch.double：64位双精度浮点数。
- torch.float16或torch.half：16位半精度浮点数。
- torch.int8：8位有符号整数。
- torch.uint8：8位无符号整数。
- torch.int16或torch.short：16位有符号整数。
- torch.int32或torch.int：32位有符号整数。
- torch.int64或torch.long：64位有符号整数。
- torch.bool：布尔型。

张量的默认数据类型是32位浮点数。

我们在大部分时间使用32位浮点数和64位有符号整数，原因如下：

1. 在神经网络中发生的计算通常是用32位浮点精度执行的。采用更高的精度，如64位，并不会提高模型精度，反而需要更多的内存和计算时间。16位半精度浮点数的数据类型在标准CPU中并不存在，而是由现代GPU提供的。如果需要的话，可以切换到半精度来减少神经网络占用的空间，这样做对精度的影响也很小。

2. 张量可以作为其他张量的索引，在这种情况下，PyTorch期望索引张量为64位的整数。创建一个将整数作为参数的张量，例如使用torch.tensor([2,2])，默认会创建一个64位的整数张量。

张量操作分类：

- 创建操作——用于构造张量的函数，如ones()和from_numpy()。
- 索引、切片、连接、转换操作——用于改变张量的形状、步长或内容的函数，如transpose()。
- 数学操作——通过运算操作张量内容的函数。
- 逐点操作——通过对每个元素分别应用一个函数来得到一个新的张量，如abs()和cos()。
- 归约操作——通过迭代张量来计算聚合值的函数，如mean()、std()和norm()。
- 比较操作——在张量上计算数字谓词的函数，如equal()和max()。
- 频谱操作——在频域中进行变换和操作的函数。
- 其他操作——作用于向量的特定函数（如cross()），或对矩阵进行操作的函数（如trace()）。
- BLAS和LAPACK操作——符合基本线性代数子程序（Basic Linear Algebra Subprogram，BLAS）规范的函数，用于标量、向量—向量、矩阵—向量和矩阵—矩阵操作。
- 随机采样——从概率分布中随机生成值的函数，如randn()和normal()。
- 序列化——保存和加载张量的函数，如load()和save()。
- 并行化——用于控制并行CPU执行的线程数的函数，如set_num_threads()。

张量是Storage实例的**视图**。

一个张量的值在存储区中从最右的维度开始向前排列被定义为连续张量。在PyTorch中一些张量操作只对连续张量起作用，如view()方法。在这种情况下，PyTorch将抛出一个提供有用信息的异常，并要求我们显式地调用contiguous()方法。值得注意的是，如果张量已经是连续的，那么调用contiguous()方法不会产生任何操作，也不会影响性能。一个连续张量转置后不再是连续张量。

张量还有设备（device）的概念，即张量数据在计算机上的位置。

```python
points_gpu = torch.tensor([[4.0, 1.0], [5.0, 3.0], [2.0, 1.0]], device='cuda')
```

pytorch可以与numpy**互操作**：

```python
points = torch.ones(3, 4)
points_np = points.numpy()
points = torch.from_numpy(points_np)
```

PyTorch在内部使用pickle来序列化张量对象，并为存储添加专用的序列化代码。通过以下方法可以将张量points写到ourpoints.t文件中并读：

```python
with open('../data/p1ch3/ourpoints.t','wb') as f:
   torch.save(points, f)
with open('../data/p1ch3/ourpoints.t','rb') as f:
   points = torch.load(f)
```

## 第4章 使用张量表征真实数据

简单讲了下如何加载各种类型的数据。

## 第5章 学习的机制

### 手动求导并更新参数

```python
import numpy as np
import torch
torch.set_printoptions(edgeitems=2, linewidth=75)
# t_u是输入数据，t_c是输出真值
t_c = [0.5,  14.0, 15.0, 28.0, 11.0,  8.0,  3.0, -4.0,  6.0, 13.0, 21.0]
t_u = [35.7, 55.9, 58.2, 81.9, 56.3, 48.9, 33.9, 21.8, 48.4, 60.4, 68.4]
t_c = torch.tensor(t_c)
t_u = torch.tensor(t_u)
# 定义模型
def model(t_u, w, b):
    return w * t_u + b
# 定义损失函数，即输出拟合值和输出真值的平方差
def loss_fn(t_p, t_c):
    squared_diffs = (t_p - t_c)**2
    return squared_diffs.mean()
# 定义权重和偏差
w = torch.ones(())
b = torch.zeros(())
# 计算损失函数
t_p = model(t_u, w, b)
loss = loss_fn(t_p, t_c)
# 损失/权重变化率，损失/偏差变化率
delta = 0.1
loss_rate_of_change_w = \
    (loss_fn(model(t_u, w + delta, b), t_c) - 
     loss_fn(model(t_u, w - delta, b), t_c)) / (2.0 * delta)
loss_rate_of_change_b = \
    (loss_fn(model(t_u, w, b + delta), t_c) - 
     loss_fn(model(t_u, w, b - delta), t_c)) / (2.0 * delta)
# 设置学习率，更新权重
learning_rate = 1e-2
w = w - learning_rate * loss_rate_of_change_w
b = b - learning_rate * loss_rate_of_change_b
# 为了更好更新权重，使delta趋近无穷小，也就是计算导数
# loss_fn的导数
def dloss_fn(t_p, t_c):
    dsq_diffs = 2 * (t_p - t_c) / t_p.size(0)  # 除法来自均值的导数
    return dsq_diffs
# model关于权重和偏差的导数
def dmodel_dw(t_u, w, b):
    return t_u
def dmodel_db(t_u, w, b):
    return 1.0
# 关于w和b的损失梯度的函数
def grad_fn(t_u, t_c, t_p, w, b):
    dloss_dtp = dloss_fn(t_p, t_c)
    dloss_dw = dloss_dtp * dmodel_dw(t_u, w, b)
    dloss_db = dloss_dtp * dmodel_db(t_u, w, b)
    return torch.stack([dloss_dw.sum(), dloss_db.sum()])
# 循环训练
def training_loop(n_epochs, learning_rate, params, t_u, t_c):
    for epoch in range(1, n_epochs + 1):
        w, b = params

        t_p = model(t_u, w, b)  
        loss = loss_fn(t_p, t_c)
        grad = grad_fn(t_u, t_c, t_p, w, b)  

        params = params - learning_rate * grad

        print('Epoch %d, Loss %f' % (epoch, float(loss))) 
            
    return params

# 开始训练
t_un = 0.1 * t_u
params = training_loop(
    n_epochs = 5000, 
    learning_rate = 1e-2, 
    params = torch.tensor([1.0, 0.0]), 
    t_u = t_un, 
    t_c = t_c)

```

如果学习率过大，会导致参数更新过度、无法收敛；但如果学习率过小，则更新非常小，所以损失下降得非常慢，最终停滞不前。可以通过学习率自适应来解决这个问题，也就是说，根据更新的大小进行更改。

对于上例，权重的梯度大约是偏置梯度的50倍。这意味着权重和偏置存在于不同的比例空间中，在这种情况下，如果学习率足够大，能够有效更新其中一个参数，那么对于另一个参数来说，学习率就会变得不稳定，而一个只适合于另一个参数的学习率也不足以有意义地改变前者。可以采用归一化输入的方法使输入靠近输出，对于上例可以将t_u乘0.1。

### 自动求导

PyTorch张量可以记住它们自己从何而来，根据产生它们的操作和父张量，它们可以根据输入自动提供这些操作的导数链。这意味着我们不需要手动推导模型，给定一个前向表达式，无论嵌套方式如何，PyTorch都会**自动提供**表达式相对其输入参数的梯度。

调用backward()将导致导数在叶节点上累加。再次调用backward()（就像在任何训练循环中一样），每个叶节点上的梯度将在上一次迭代中计算的梯度之上累加（求和），这会导致梯度计算不正确。为了防止这种情况发生，我们需要在每次迭代时明确地将梯度归零。

```python
params = torch.tensor([1.0, 0.0], requires_grad=True)

def training_loop(n_epochs, learning_rate, params, t_u, t_c):
    for epoch in range(1, n_epochs + 1):
        # 在调用loss.backward()之前的任何时间完成,清空累积的梯度
        if params.grad is not None: 
            params.grad.zero_()
        
        t_p = model(t_u, *params) 
        loss = loss_fn(t_p, t_c)
        # PyTorch反向遍历计算图以计算梯度
        loss.backward()
        
        # 在with块中对参数进行更新，在这里PyTorch自动求导机制将不起作用
        with torch.no_grad(): 
            params -= learning_rate * params.grad

        if epoch % 500 == 0:
            print('Epoch %d, Loss %f' % (epoch, float(loss)))
            
    return params
```

### 自动更新参数

使用优化器模块封装整个过程。

```python
params = torch.tensor([1.0, 0.0], requires_grad=True)
learning_rate = 1e-2
optimizer = optim.SGD([params], lr=learning_rate)

def training_loop(n_epochs, optimizer, params, t_u, t_c):
    for epoch in range(1, n_epochs + 1):
        t_p = model(t_u, *params) 
        loss = loss_fn(t_p, t_c)
        # 清空累积梯度
        optimizer.zero_grad()
        # 计算梯度
        loss.backward()
        # 查看params.grad并更新params，从中减去学习率乘梯度，就像手动求导并更新参数
        optimizer.step()

        if epoch % 500 == 0:
            print('Epoch %d, Loss %f' % (epoch, float(loss)))
            
    return params
```

### 分割数据集

为了避免过拟合，可以单独分出一部分数据作为验证集。

```python
# 整个样本和验证集大小
n_samples = t_u.shape[0]
n_val = int(0.2 * n_samples)

# 打乱顺序
shuffled_indices = torch.randperm(n_samples)

# 划分训练集和验证集
train_indices = shuffled_indices[:-n_val]
val_indices = shuffled_indices[-n_val:]
train_t_u = t_u[train_indices]
train_t_c = t_c[train_indices]
val_t_u = t_u[val_indices]
val_t_c = t_c[val_indices]
train_t_un = 0.1 * train_t_u
val_t_un = 0.1 * val_t_u

def training_loop(n_epochs, optimizer, params, train_t_u, val_t_u,
                  train_t_c, val_t_c):
    for epoch in range(1, n_epochs + 1):
        train_t_p = model(train_t_u, *params)
        train_loss = loss_fn(train_t_p, train_t_c)
		
        # 对于val_loss，关闭自动求导，不构建自动求导图。
        # 这不是必要的，因为后面并未对val_loss调用backward()。但是可以提高性能。
        with torch.no_grad():
            val_t_p = model(val_t_u, *params)
            val_loss = loss_fn(val_t_p, val_t_c)
            assert val_loss.requires_grad == False 
        
        optimizer.zero_grad()
        # 只在训练集上反向传播并更新参数
        train_loss.backward() 
        optimizer.step()

        if epoch <= 3 or epoch % 500 == 0:
            print(f"Epoch {epoch}, Training loss {train_loss.item():.4f},"
                  f" Validation loss {val_loss.item():.4f}")
            
    return params
```

使用相关的`set_grad_enabled()`，还可以根据一个布尔表达式设定启用或禁用自动求导的条件。

## 第6章 使用神经网络拟合数据

神经元即线性变换+非线性激活函数。

激活函数两个作用：

1. 允许输出函数在不同的值上有不同的斜率，这是线性函数无法做到的。
2. 将前面的线性运算的输出集中到给定的范围内。

### 线性模型

```python
import torch
import torch.optim as optim

# 这里继续使用第5章的数据，但是需要添加以下代码，在轴1处扩充一个维度。nn中的所有模块都被编写为可以同时为多个输入产生输出。
t_c = torch.tensor(t_c).unsqueeze(1) 
t_u = torch.tensor(t_u).unsqueeze(1) 

# 入参：输入张量大小，输出张量大小，偏置（默认为True）
linear_model = nn.Linear(1, 1) 
optimizer = optim.SGD(linear_model.parameters(), lr=1e-2)

def training_loop(n_epochs, optimizer, model, loss_fn, t_u_train, t_u_val,
                  t_c_train, t_c_val):
    for epoch in range(1, n_epochs + 1):
        t_p_train = model(t_u_train)
        loss_train = loss_fn(t_p_train, t_c_train)

        t_p_val = model(t_u_val) 
        loss_val = loss_fn(t_p_val, t_c_val)
        
        optimizer.zero_grad()
        loss_train.backward() 
        optimizer.step()

        if epoch == 1 or epoch % 1000 == 0:
            print(f"Epoch {epoch}, Training loss {loss_train.item():.4f},"
                  f" Validation loss {loss_val.item():.4f}")
```

可以用`nn.MSELoss()`代替前面自定义的`loss_fn`做损失函数入参。

### 神经网络

```python
# seq_model可以直接替换上面的linear_model作为入参
seq_model = nn.Sequential(
            nn.Linear(1, 13), 
            nn.Tanh(),
            nn.Linear(13, 1)) 
'''
Sequential(
  (0): Linear(in_features=1, out_features=13, bias=True)
  (1): Tanh()
  (2): Linear(in_features=13, out_features=1, bias=True)
)
'''
# 调用model.parameters()将从第1个和第2个线性模块收集权重和偏置
for name, param in seq_model.named_parameters():
    print(name, param.shape)
'''
0.weight torch.Size([13, 1])
0.bias torch.Size([13])
2.weight torch.Size([1, 13])
2.bias torch.Size([1])
'''
# 可以用OrderedDict对模块进行命名
from collections import OrderedDict
seq_model = nn.Sequential(OrderedDict([
    ('hidden_linear', nn.Linear(1, 8)),
    ('hidden_activation', nn.Tanh()),
    ('output_linear', nn.Linear(8, 1))
]))
'''
Sequential(
  (hidden_linear): Linear(in_features=1, out_features=8, bias=True)
  (hidden_activation): Tanh()
  (output_linear): Linear(in_features=8, out_features=1, bias=True)
)
'''
```

下图是seq_model的结构：

![第一个简单神经网络结构](https://tvax3.sinaimg.cn/large/0077Un8Egy1h5reteg4vwj30rs0i3gmn.jpg)



