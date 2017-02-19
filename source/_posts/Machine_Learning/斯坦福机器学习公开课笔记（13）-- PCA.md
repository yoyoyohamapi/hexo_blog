title: 斯坦福机器学习公开课笔记（13）-- PCA
tags: 斯坦福机器学习
categories: 斯坦福机器学习公开课笔记
----

## 数据压缩与特征降维
我们很希望有足够多的特征（知识）来保准学习模型的训练效果，尤其在图像处理这类的任务中，高维特征是在所难免的，但是，高维的特征也有几个如下不好的地方：

1. 学习性能下降（知识越多，吸收知识（输入），并且精通知识（学习）的速度就越慢）
2. 过多的特征难于分辨，你很难第一时间认识某个特征代表的意义。
3. 最大的问题：__特征冗余__，如下例子所示，厘米和英尺就是一对冗余特征，他们本身代表的意义是一样的，并且能够相互转换。

![特征冗余](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_特征冗余.png)

我们先来看两个特征降维的例子，如下图所示，原来我们有两个冗余特征---厘米和英尺，现在，我们构造如下的一条绿色直线，并且使每个样本投影到该直线，则原来需要横纵两个维度表示的样本（{% math x_1(厘米1,英尺1)，x_2(厘米2,英尺2) %}）现在仅需用直线上的相对位置（{% math x_1(位置1),x_2(位置2) %}）即可表示。

![2D->1D](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_2d3d.png)

而在下面这个例子中，我们同样通过构造一个二维平面对三维特征进行投影，成功将三维特征降到二维（3D->2D）

![3D->2D](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_2d3d.png)

综上，我们可以看到，特征降维的一般过程就是：__构造低维空间__ ---》 __将高维特征投影到低维空间__。

## PCA（主成成分分析法）
如下图所示，我们有将近几十个特征来描述国家的经济水平，但是你仔细观察发现，这些特征其实上可以融合压缩的，最终，我们仅需要两个特征就能反映经济水平：__（国家整体经济水平，个人经济状况）__

![国家经济水平](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_国家经济水平.png)

而__PCA(Principle Component Analysis，主成成分分析法)__就是来完成特征降维工作的，顾名思义，其从冗余的特征序列中保留了特征的主要成分，PCA在不太大影响训练效果的前提下大幅提高整个学习系统的性能。

![PCA](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_投影误差.png)

如上图所示，我们将__蓝色线段(样本到红色直线的垂直距离)__称作是投影误差（Projection Error），PCA做的就是找到低维空间并且投影特征，并使得各个特征的投影误差足够小。假设我们要将特征从{% math n %}维度降到{% math k %}维：PCA首先找寻$k$个$n$维向量，然后将特征投影到这些向量组成的$k$维空间，并保证投影误差足够小，下例中，为了将三维特征降到二维特征，我们就会先找寻两个三维向量{% math u^{(1)},u^{(2)} %},二者构成了一个二维平面，然后我们将原来的特征投影到该平面上。

![PCA,3D->2D](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_pca3d2d.png)

### PCA算法执行流程
(1) 数据__归一化处理（mean normalization）__：平均各特征尺度
	
{% math_block %}
x_j^{(i)}=\frac{x_j{(i)}-\mu_j}{s_j}
{% endmath_block %}

其中，{% math \mu_j %}为第{% math j %}个特征的__均值__，{% math s_j %}为第j个特征的__标准差__。

(2) 算法主体：假设我们将特征从{% math n %}维降到{% math k %}维

* 计算__协方差矩阵({% math \sum %})__：

{% math_block %}
\sum = \frac{1}{m}\sum\limits_{i=1}^{m}(x^{(i)})(x^{(i)})^T
{% endmath_block %}

上式的矩阵表述为：

{% math_block %}
\sum=\frac{1}{m} \cdot X^TX
{% endmath_block %}

* 进行__奇异值分解（SVD）__，奇异值的认识和入门推荐查看这篇博客：[强大的矩阵奇异值分解(SVD)及其应用]()

{% math_block %}
(U,S,V^T) = SVD(\sum)
{% endmath_block %}

* 从__U__中取出前{% math k %}个__左奇异向量__，构成一个约减矩阵{% math U_{reduce} %}:

{% math_block %}
U_{reduce} = (u^{(1)},u^{(2)},\cdots,u^{(k)})
{% endmath_block %}

* 如此，我们就得到了压缩后的特征向量{% math z %}为：

{% math_block %}
z^{(i)}=U_{reduce}^T \cdot x^{(i)}
{% endmath_block %}

### 从PCA压缩后的数据进行修复

因为PCA仅保留了特征的主成分，亦即PCA是一种__有损__的压缩方式:

{% math_block %}
z = U_{reduce}^Tx
{% endmath_block %}

修复{% math x %}:

{% math_block %}
x_{approx}=U_{reduce}z
{% endmath_block %}

如下图所示，我们不可能从压缩后的特征完美还原到原特征，但由于PCA尽可能地减小了投影误差，所以还原后的特征也近似于原特征了。

![PCA降维前](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_pca前.png)

![特征修复](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8p_修复.png)

### 降到多少维度才是合适的（k值的确定）

PCA做到了特征降维，但维度显然也不是被降得越小越好，亦即，在剩下的主成分特征中，各个特征间的差异需要足够大（亦即特征冗余足够小），我们通过如下方式评估特征差异性：

* 求各样本的投影__均方误差__:

{% math_block %}
\min\frac{1}{m}\sum\limits_{j=1}^{m}||x^{(i)}-x_{approx}^{(i)}||^2
{% endmath_block %}

* 求数据的总变差：

{% math_block %}
\frac{1}{m}\sum\limits_{j=1}^{m}||x^{(i)}||^2
{% endmath_block %}

* 评估下式是否成立:

{% math_block %}
\frac{\min\frac{1}{m}\sum\limits_{j=1}^{m}||x^{(i)}-x_{approx}^{(i)}||^2}{\frac{1}{m}\sum\limits_{j=1}^{m}||x^{(i)}||^2} \leq \epsilon
{% endmath_block %}

其中，{% math \epsilon %}的取值可以为{% math 0.01,0.05,0.10,\cdots %},假设{% math =0.01 %}，我们就说__“{% math 99\% %}的差异性得到保留”__。

### 不要滥用PCA
由于PCA减小了特征维度，因而也有可能带来过拟合的问题。所以PCA并不是必须的，所以在机器学习中，我们并不是一开始就需要做PCA，尽在学习算法运行速度不理想时再考虑使用PCA。
