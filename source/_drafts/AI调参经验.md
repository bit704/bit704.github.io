---
title: AI调参经验
categories: [Essay]
tags: [Python]
---

AI各种超参数调参经验总结

<!--more-->

## loss nan

出现loss nan可能有以下原因

- 训练数据中含有nan

- 计算过程中溢出
  -  上溢出，进行譬如$exp(x)$之类的运算。
  - 下溢出，进行譬如$\log(0)$之类的运算。

- 梯度爆炸
  - 每个batch前梯度没有零，`optimizer.zero_grad()`
  - 学习率过大
  - batchsize过大