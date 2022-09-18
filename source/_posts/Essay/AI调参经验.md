---
title: AI调参经验
categories: [Essay]
tags: [Python,AI]
---

训练AI的调参经验总结

<!--more-->

## pytorch

### 训练

训练时使用`model.train()`，测试时使用`model.eval()`。BatchNormalization在测试时参数固定，Dropout在测试时不再生效。

### 权重初始化

使用方法：

```python
class Model(torch.nn.Module):
    ...
    
def init_weight(layer):
    # or if isinstance(layer, nn.Conv2d):
    if type(layer)==nn.Conv2d:
        nn.init.normal_(layer.weight,mean=0,std=0.5)
    elif type(layer)==nn.Linear:
        nn.init.uniform_(layer.weight,a=-1,b=0.1)
        nn.init.constant_(layer.bias,0.1)

model=Model()
# 此方法会自下而上递归调用model中所有层,初始化其权重
model.apply(init_weight)
```

不同的初始化API:[torch.nn.init — PyTorch 1.12 documentation](https://pytorch.org/docs/stable/nn.init.html?highlight=nn init)

Pytorch线性层采取的默认初始化方式是kaiming_uniform_初始化。

### BN 

```python
torch.nn.BatchNorm1d(num_features, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True, device=None, dtype=None)
```

[Paper](https://arxiv.org/abs/1502.03167)
$$
y=\frac {x-E[x]} {\sqrt{Var[x]+\epsilon} } *\gamma+\beta
$$
在最小批的每个维度上计算均值和标准差，$\gamma$ 和$\beta$则是可学习参数向量，大小为输入的特征数或通道数。$\gamma$ 默认为1，$\beta$默认为0。

为了将输入调整到激活函数的敏感区，BatchNorm层要加在激活函数前面。

### Dropout

```python
torch.nn.Dropout(p=0.5, inplace=False)
```

在训练过程中，按伯努利分布采样概率p随机将输入张量的元素置零。在每次调用中每个通道将被独立地置零。这已经被证明是正则化的有效手段。

注意，训练时输出会乘以$\frac{1}{1-p}$。测试时模块简单计算一个恒等函数。

BN层的加入可以起到抑制过拟合的作用，在训练过程对每个单个样本的forward均引入多个样本（Batch个）的统计信息，相当于自带一定噪音，起到正则效果，无需添加Dropout。BN和Dropout同时使用会使精度下降，参考论文：

> Li X, Chen S, Hu X, et al. Understanding the disharmony between dropout and batch normalization by variance shift[C]//Proceedings of the IEEE/CVF conference on computer vision and pattern recognition. 2019: 2682-2690.

Dropout层一般放在激活函数层之后。

### 保存模型

1. 保存整个模型

   ```python
   # 保存
   model = Model()
   torch.save(model, 'model_name.pth')
   # 读取
   model = torch.load('model_name.pth')
   ```

2. 保存模型参数

   ```python
   # 保存
   model = Model()
   # 可以加个键，特别是保存多个模型的时候
   torch.save({'model': model.state_dict()}, 'model_name.pth')
   # 读取
   model = Model()
   state_dict = torch.load('model_name.pth')
   model.load_state_dict(state_dict['model'])
   ```
   
   第一种方法可以直接保存模型，加载模型的时候直接把读取的模型给一个参数就行。它包含四个键，分别是model,optimizer,scheduler,iteration。
   
   第二种方法只保存model参数，在读取模型参数前要先定义一个模型（模型必须与原模型相同的构造），然后对这个模型导入参数。
   



## loss nan

出现loss nan可能有以下原因

- 训练数据中含有nan

- 计算过程中溢出
  -  上溢出，进行譬如$exp(x)$之类的运算。
  - 下溢出，进行譬如$\log(0)$之类的运算。

- 梯度爆炸
  - 每个batch前梯度没有零，`optimizer.zero_grad()`(pytorch)
  - 学习率过大
  - batchsize过大