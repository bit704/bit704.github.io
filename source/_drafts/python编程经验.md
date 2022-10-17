---
title: python编程经验
categories: [Essay]
tags: [Python]
---

python编程的经验和易错细节总结

<!--more-->

[toc]

## numpy

### 深浅复制

默认浅复制，使用`.copy()`进行深复制。

### 修改

`resize()` 就地操作，可以修改尺寸。

`reshape()` 非就地操作，只能修改形状。

入参都是一组数字或者元组。但前者入参不能有-1。

### 切片

当使用数组切片切下单列时，默认为一维数组，需用`reshape(-1, 1)`转换为二维数组，才可使用np.hstack等函数或同二维数组进行运算。（单行同理）

可以直接对切片赋值，如对一维切片进行赋值，需要将赋值对象拉平`reshape(-1)`。

### 生成

`np.meshgrid()` 生成网格，输出为网格的横坐标矩阵和纵坐标矩阵。

```python
x = np.array([0, 1, 2])
y = np.array([0, 1])
X, Y = np.meshgrid(x, y)
'''
[[0 1 2]
 [0 1 2]]
[[0 0 0]
 [1 1 1]]
'''
```



## 可视化

### matplotlib

常用例子

```python
import matplotlib.pyplot as plt

# 设置图像大小,像素为figsize*dpi
fig = plt.figure(figsize=(30, 10), dpi=50)  
ax1 = fig.add_subplot(121)

X = np.arange(10)
Y = np.square(X)
ax1.set_xlabel('X')
ax1.set_ylabel('Y')
ax1.plot(X, Y,
         color='grey',
         marker='o',
         markersize=8, 
         linewidth=2,
         markerfacecolor='red',
         markeredgecolor='grey',
         markeredgewidth=2)

XX, YY = np.meshgrid(X, Y)
Z = np.full_like(XX, 0)
for i in range(X.size):
    for j in range(Y.size):
        # 注意这里Z的shape是(Y.size,X.size)
        Z[j][i] = X[i] + Y[j]

ax2 = fig.add_subplot(122, projection='3d')
ax2.plot_surface(XX, YY, Z, cmap='rainbow')
ax2.set_xlabel('X')
ax2.set_ylabel('Y')
ax2.set_zlabel('Z')
# visible：表示是否显示网格。若提供其他关键字参数，则b参数设为True。
# which：表示显示网格的类型，支持major、minor、both这3种类型，默认为major。
# axis：表示显示哪个方向的网格，该参数支持both、x和y这3个选项，默认为both。
# linewidth或lw：表示网格线的宽度。
# ax1.grid(visible=True,which='both', axis='both', linewidth=0.3)
# 在show之前调用
plt.savefig('loss_image.jpg')
plt.show()
plt.close()
```

![可视化结果](https://bit704.oss-cn-beijing.aliyuncs.com/image/2022-09-18-%E5%8F%AF%E8%A7%86%E5%8C%96%E7%BB%93%E6%9E%9C.png)



