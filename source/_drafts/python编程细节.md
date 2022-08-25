---
title: python编程细节
layout: draft
categories: [Essay]
tags: [Python]
mathjax: false
---

python编程的易错细节总结

<!--more-->

## 数组操作

当使用数组切片切下单列时，默认为一维数组，需用`reshape((-1, 1))`转换为二维数组，反可使用np.hstack等函数。（单行同理）