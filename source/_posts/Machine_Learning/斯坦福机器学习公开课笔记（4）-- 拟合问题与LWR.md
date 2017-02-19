title: 斯坦福机器学习公开课笔记（4）-- 拟合问题与局部加权线性回归
tags: 斯坦福机器学习
categories: 斯坦福机器学习公开课笔记
----
###欠拟合与过拟合问题

* 欠拟合（underfitting）：拟合程度不高，部分数据过于远离拟合曲线。如下图最左所示。

* 过拟合（overfitting）：拟合程度过高，貌似拟合得很完美，但是丢失了信息规律，如下图最右所示，房价随着房屋面积的增长的规律丢失了：

![拟合问题](http://7pulhb.com1.z0.glb.clouddn.com/ml-subbandfitting.png)

上面中间部分的图像展示较好的拟合曲线。
*************************************************************
###LWR（Locally Weighted linear Regression）：局部加权线性回归
为了解决欠拟合以及过拟合问题，引入了LWR，
在一般的线性回归算法中，对于某个$x$，我们这样预测其对应的$y$：

1. Fit $\theta$ to minimize {% math \sum_{i}(y^{(i)}-\theta^Tx^{(i)})^2 %}

2. Output $\theta^Tx$

而在LWR中：

1. Fit $\theta$ to minimize {% math \sum_{i}w^{(i)}(y^{(i)}-\theta^Tx^{(i)})^2 %}

2. Output $\theta^Tx$

通过赋予x周围点不同的权值，来对最后的$y$进行预测，即LWR考虑了$x$周围的影响，也就是保住了 $x$ 周围的趋势，这样就带来了更好拟合效果。通常，$w^{(i)}$ 定义为指数衰减，即$w$近似服从高斯分布，如下：

{% math_block %}
w^{(i)}=exp(-\frac{(x^{(i)}-x)^2}{2\tau^2})
{% endmath_block %}

如下图所示：

![weights](http://7pulhb.com1.z0.glb.clouddn.com/ml-subband权值分布.jpg)

可以看到，随着距离不断远离$x$，权值不断降低，上面公式中的$\tau$控制了衰减范围，亦即衰减速度。
****************************************************************
LWR属于非参数化（non-paramatic）算法，因为其参数会随着训练集的变化而变化。

由于还有关注周围点，因而LWR的计算度较高。