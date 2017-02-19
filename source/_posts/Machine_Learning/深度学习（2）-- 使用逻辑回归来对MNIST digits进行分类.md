title: 深度学习（2）-- 使用逻辑回归来对MNIST digits进行分类
tags: 深度学习
categories: 深度学习
------
[MNIST digits](http://yann.lecun.com/exdb/mnist/)是一个手写字符图像数据集,拥有包含60000个样本的训练集以及10000个样本的测试集。
###逻辑回归（Logic Regression）模型
逻辑回归是一个概率线性分类器，其参数包括有一个权值矩阵$W$以及一个偏置向量$b$。逻辑回归通过将输入向量投影到一个超平面（每个超平面代表一种类型）集合中来完成分类，那么输入向量与超平面间的距离远近就反映了某个输入属于某个类型的可能性大小。
在逻辑回归模型中，一个输入向量$x$属于某个分类i的概率如下：

{% math_block %}
P(Y=i|x,W,b)=softmax_i(Wx+b)=\frac{e^{W_ix+b_i}}{\sum\limits_j e^{W_jx+b_j}}
{% endmath_block %}

该模型的预测$y_pred$，亦即具有最大概率的分类定义如下：

{% math_block %}
y_{pred}=argmax_iP(Y=i|x,W,b)
{% endmath_block %}

在Theano中完成上述模型的代码如下：

```python
# 初始化权值矩阵W，其大小为n_in行，n_out列
self.W = theano.shared(
    value=numpy.zeros(
        (n_in, n_out),
        dtype=theano.config.floatX
    ),
    name='W',
    borrow=True
)
# 初始化偏置向量b，其维度为n_out维
self.b = theano.shared(
    value=numpy.zeros(
        (n_out,),
        dtype=theano.config.floatX
    ),
    name='b',
    borrow=True
)
# Where:
# 权值矩阵W的第k列描述了第k个类型的分割超平面
# 输入向量构成的矩阵x中，第j行描述了输入的第j个训练样本
# 偏置向量b的第k个元素描述了第k个超平面的自由参数
# p_y_given_x亦即给定x是y取某值的概率
self.p_y_given_x = T.nnet.softmax(T.dot(input, self.W) + self.b)
# y_pred为模型的预测分类,axis=1亦即最大值的索引号为1
self.y_pred = T.argmax(self.p_y_given_x, axis=1)
```

[borrow](http://deeplearning.net/software/theano/tutorial/aliasing.html#borrowing-when-creating-shared-variables)参数的解释:如果Borrow参数为True，则获得的shared varible将会随value赋值的矩阵变化而变化