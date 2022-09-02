---
title: AI调参经验
categories: [Essay]
tags: [Python]
---

AI各种超参数调参经验总结

<!--more-->

## pytorch

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
# 此方法会自下而上递归调用model中所有层
model.apply(init_weight)
```

API:[torch.nn.init — PyTorch 1.12 documentation](https://pytorch.org/docs/stable/nn.init.html?highlight=nn init)



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