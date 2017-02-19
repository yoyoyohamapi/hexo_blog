title: 斯坦福机器学习公开课笔记（12）-- K-Means
tags: 斯坦福机器学习
categories: 斯坦福机器学习公开课笔记
----
## 无监督学习（Unsupervised Learning）
从本节开始，将正式进入到__无监督学习（Unsupervised Learning）__部分，无监督学习，顾名思义，就是不受监督的学习，一种自由的学习方式，他并不需要先验知识进行指导，而是在不断地自我认知，自我巩固，自我归纳，在机器学习中，这种学习方式可以简单理解为不为训练集提供对应于特征向量的__类别标识__,即：

有监督学习（Supervised Learning）下的__训练集（Training Set）__：

{% math_block %}
{(x^{(1)},y^{(1)}),(x^{2},y^{(2)}),\cdots}
{% endmath_block %}

无监督学习（Supervised Learning）下的__训练集（Training Set）__：

{% math_block %}
{(x^{(1)}),(x^{(2)}),(x^{3})}
{% endmath_block %}

## 聚类
在有监督学习中，我们对样本进行分类的过程称之为__分类（Classification）__，而在无监督学习中，我们将物体被划分到不同集合的过程称为__聚类（Clustering）__，__聚__这个词十分贴切，他好像就描述各个物体“自主”地向属于自己的集合靠拢的过程。在聚类中，把物体所在集合称为__簇__。

## K-Means算法步骤
那么K-Means是如何完成聚类过程的呢？注意，他的名字已经暗示了我们该算法重要的两个部分，（1）__K__---描述__簇的数量__,也就是该聚合成多少个类；（2）__Means__：求某个__均值__将会是该算法的核心。

具体看下K-Means算法的步骤：

（1） 根据设定的聚类数__K__，随机地选择__K__个__聚类中心(Cluster Centroid)__。（古代乱世，诸侯并起而逐鹿）。

![聚类中心](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_聚类中心.png)

（2） 评估__每个__样本到各聚类中心的距离，如果样本距离第{% math i %}个聚类中心更近，则认为其属于第{% math i %}簇（四方义士纷纷投奔起义诸侯，形成不同势力）。

![义士投奔](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_分配簇.png)

（3） 计算每个簇中样本的__平均(Mean)位置__，将聚类中心移动至该位置（诸侯选择战略根据地以达到最强的内聚力）。

![中心转移](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_移动聚类中心.png)

重复以上步骤至各聚类中心位置不在改变。

如上，可以看到，K-Means算法主要分为如下两步：

（1）分配：将样本分配到某簇当中。

（2）移动：移动聚类中心至簇中样本平均位置。

> 注！在第二步中，有些聚类中心可能没有被分配到样本，这些聚类中心就会被淘汰（意味着最终的类数也会减少）。

K-Means的算法伪码也就能够很快写出：

设定簇的个数 {% math K %}（显然，{% math K \lt m %}）;

随机设定 {% math K %} 个聚类中心：{% math \mu_1,\mu_2,\cdots,\mu_k \in R^n %} ;

重复至聚类中心不再改变{
	
__// 分配过程__：

for {% math i=1 %} to {% math m %}:

{% math_block %}
c^{(i)}= 距 x^{(i)} 最近的聚类中心
{% endmath_block %}

亦即我们会求取下面过程：

{% math_block %}
\min_k||x^{(i)}-\mu_k||^2
{% endmath_block %}	
	
__// 移动过程__：

for {% math k=1 %} to {% math K%}:

{% math_block %}
\mu_k(第k个聚类中心的新位置) = 第k簇的平均位置
{% endmath_block %}
	
}


例如，{% math \mu_2 %}聚类中心下分配了4个样本：

{% math_block %}
x^{(1)},x^{(5)},x^{(6)},x^{(10)}
{% endmath_block %}

亦即：

{% math_block %}
c^{(1)}=c^{(5)}=c^{(6)}=c^{(10)}=2
{% endmath_block %}

那么 {% math \mu_2 %} 将会移到这4个样本的中心位置：

{% math_block %}
\mu_2=\frac{1}{4}(x^{(1)}+x^{(5)}+x^{(6)}+x^{(10)})
{% endmath_block %}

## 优化目标（Optimization Objective）
和其他学习算法一样，K-Means也要评估并且最小化聚类代价，在引入K-Means的代价函数之前，先引入如下定义：

{% math_block %}
\mu_{c^{(i)}}=样本x^{(i)}被分配到的聚类中心
{% endmath_block %}

引入代价函数：

{% math_block %}
J(c^{(1)},c^{(2)},\cdots,c^{(m)};\mu_1,\mu_2,\cdots,\mu_k)=\frac{1}{m}\sum\limits_{i=1}^{m}||x^{(i)}-\mu_{c^{i}}||^2
{% endmath_block %}

{% math J %} 也被称为__失真代价函数（Distortion Cost Function）__:
 
 实际上，K-Means的两步已经完成了__最小化代价函数__的过程：
 
 __“样本分配时”：__
 
 我们固定住了 {% math (\mu_1,\mu_2,\cdots,\mu_k) %} 而关于 {% math (c^{(1)},c^{(2)},\cdots,c^{(m)}) %}最小化了{% math J %} 。
 
 __“中心移动”：__
 
 我们再关于 {% math (\mu_1,\mu_2,\cdots,\mu_k) %} 最小化了 {% math J %} 。
 
 由于我们每次迭代过程都在最小化{% math J %},所以下面的变化曲线不会出现:
 
 ![错误的变化曲线](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_错误的代价曲线.png)
 
## 如何初始化聚类中心
通常，我们会随即是选取 {% math K %} 个样本作为 {% math K %} 个聚类中心（{% math K \lt m %}）。

但是，如下图所示，不同的初始化会带来不同的聚类结果，达到__全局最优（global optimal）__固然是好的，但是，我们往往也可能只获得了__局部最优（local optimal）__，那么如何避免不好的聚类结果呢？当然，没什么好办法，只能尝试不同的初始化:

![全局最优](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_全局最优.png)

![局部最优](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_局部最优.png)

For {% math i=1 %} to 100 {

* 随机初始化

* 执行K-Means，得到每个样本所属簇，以及各聚类中心位置：

{% math_block %}
c^{(1)},c^{(2)},\cdots,c^{(m)},\mu_1,\cdots,\mu_k
{% endmath_block %}


* 计算失真函数 {% math J %}

}

选择 {% math J %} 最小的聚类结果。

> 这种方法适用于 {% math K %} 较小的场景。

## 如何选择适当的聚类数K

实际上，一开始很难确定聚类的数目，如下图所示，似乎这两种聚类的方式都是OK的：

![聚类方式1](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_聚类方式1.png)

![聚类方式2](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_聚类方式2.png)

但是，也存在着一种称为__肘部法则（Elbow Method）__的方法来选定适当的K值：

![肘部法则](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_肘部法则.png)

如上图所示，该曲线类似于人的手肘，__”肘关节“__部分对应的 {% math K %} 值就是最恰当的{% math K %}值。但是并不是所有曲线都存在明显的”肘关节“，当我们面临如下所示的曲线时，肘部法则就不是那么好用了。

![肘部法则失效](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_肘部失效.png)

一般来说，K-Means得到的聚类结果是服务于我们的后续目的（如通过聚类进行市场分析），故而更合理更客观地评价{% math K %}的选择是否恰当的方式我们的后续目的是否受益。

也就是说，脱离最终目的进行 {% math K %} 的选择是不推崇的，如在下面这个例子中，假定我们的衣服想要是分为 S,M,L 三个尺码，就设定 {% math K=3 %} ，如果我们想要 XS,S,M,L,XL 5个衣服的尺码，就设定 {% math K=5 %} 。

![衣服尺码问题](http://7pulhb.com1.z0.glb.clouddn.com/ml-w8k_衣服尺码.png)
 