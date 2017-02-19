title: 斯坦福机器学习公开课笔记（9）-- BP神经网络
tags: 斯坦福机器学习
categories: 斯坦福机器学习公开课笔记
----
###定义
BP（Back Propagation）神经网络，顾名思义，其性质仍然是一个神经网络，是一个数学模型，其目的仍然是解决分类或者回归问题。然而，一般的神经网络并不具有向后的过程，仅只是向前传递刺激，BP神经网络过加入了隐层神经元，也就是加入了更多处理单元，更加精细了处理过程，亦即多了处理逻辑，故而需要评价每一层上产生的误差，纠正每一层产生的误差，BP神经网络的向后过程正是为了逐层纠正误差，达到全局的分类最优。

![BP神经网络](http://7pulhb.com1.z0.glb.clouddn.com/ml-神经网络.jpg)

通常，隐层越多，分类能力越强，因为隐层增多，意味着优化，处理就越多。一般在设计BP神经网络时，要注意各隐层上的神经元数目最好保持一致。

*********************************************************
###过程
首先分析BP神经网络的一个局部：

![BP局部](http://7pulhb.com1.z0.glb.clouddn.com/ml-BP神经网络的一个局部.jpg)

设$a^(k)$为第$k$层神经输入向量，$y$为真实输出向量。
令第$k$层误差为：

$e^{(k)}=\frac{1}{2}\sum_i(a_i^{(k)}-y_i)^2$

我们沿着梯度下降方向来减小每层误差,则权值矩阵$W$的更新如下:

$\Delta W^{(k)}=-\alpha\frac{1}{m}\frac{\partial e^{(k+1)}}{\partial W^{(k+1)}}$ 其中，$m$为样本容量,$\alpha$为下降速度

易求得：

$\frac{\partial e^{(k+1)}}{\partial W^{(k+1)}}=d^{(k+1)}a^{(k)}$

其中:

$d^{(k)}=\begin{cases} a^{(K)}\cdot(1-a^{(K)})\cdot(a^{(K)}-y) & \mbox{if $k=K$,K is the final input layer} \\\ a^{(k)}\cdot(1-a^{(k)})\cdot(W^Td^{(k+1)}) & \mbox{otherwise} \end{cases}$

那么我们定义
将第$k$层的权值的修正为:

$W(k)=W(k)-\alpha[(\frac{1}{m}\Delta W^{(k)})+\lambda W^{(k)}]$ 其中，$\lambda$用于进行regularization以防止过拟合

为了评价BP的分类成绩，我们如下定义全局代价函数:

{% math_block %}
J(W,b)=\frac{1}{m}\sum\limits_{i=1}^{m}[\frac{1}{2}||h_w,b(x)-y||]^2+\frac{\lambda}{2}\sum\limits_{l=1}\sum\limits_{i=1}\sum\limits_{j=1}[W_{ji}^{(l)}]^2
{% endmath_block %}

*****************************************************************

###训练流程，BP神经网络的实现伪码：
Begin：

1. 各层权值矩阵及楼偏置向量赋随机初值，通常会赋值0到1间的小值切勿将权值矩阵赋0，否则要导致神经网络的训练过程呈僵死状态。

2. for t in max_loop(max_loop为最大迭代限制):

(1) 初始化权值梯度增量，初始化偏置向量梯度增量
$\Delta W=0$
$\Delta b=0$

(2) for i in m(m为样本数):

前向(forward propagation)计算各层输出:

$a^{(1)}=x$
$z^{(2)}=W^{(1)}a^{(1)}+b^{(1)}$
$a(2)=sigmoid(z^{(2)})$
$\cdots \cdots$

后向(backward propagation)评估各层误差,获得梯度下降所需增量$d^{(k)}$:

若$k=K$,即第$k$层位输出层，则

$d^{(k)}=[a^{(k)}-y]\cdot g'(z^{(k)})$
其中,$g'(z^{k})=a^{(k)}\cdot [1-a^{(k)}]$
则$d^{(k)}=[a^{(k)}-y]\cdot a^{(k)} \cdot [1-a^{(k)}]$

若$k\neq K$,则

$d^{(k)}=[(W^{(k)})^T d^{(k+1)}]\cdot f'(z^{(k)})$
即$d^{(k)}=[(W^{(k)})^T d^{(k+1)}]\cdot [a^{(k)}[1-a^{(k)}]]$

得到各层梯度下降增量:

$\Delta W^{(k)}=d^{(k+1)}[a^{(k)}]^T$
$\Delta b^{(k)}=d^{(k+1)}$

更新各层权值及偏置:

$W^{(k)}=W^{(k)}-\alpha [\frac{1}{m}\Delta W^{(k)}+\lambda W^{(k)}]$
$b^{(k)}=b^{(k)}-\alpha [\frac{1}{m}\Delta b^{(k)}]$

(3) 此时若系统的预测代价足够小（$J(W,b)<eplison$）,则停止训练

*************************************************
以下是实现代码，并没有采用regularization（采用了，反而导致分类失败，也许是我的代码仍存在一定问题）

```python
#coding:utf-8

'''
    - Created with Sublime Text 2.
    - User: yoyoyohamapi
    - Date: 2015-01-27
    - Time: 14:34:01
    - Contact: yoyoyohamapi.com
'''

import numpy as np
import random
import os
# 定义sigmoid函数
def sigmoid(z):
	return 1.0/(1.0+np.exp(-z))



#############################
#------BP神经网络建模-------#
#############################
class BPNN:
	"""
	构造函数:
		:param x:输入特征向量
		:param hidden_num:隐层数
		:param y:分类向量
		
	"""
	def __init__(self,x,y,hidden_num):
		self.x = x
		self.y = y
		# 训练样本数
		self.m = x.shape[0]
		self.hidden_num = hidden_num
		# 各层需要添加偏置向量
		self.weights = []
		self.b = []
		# 隐藏层神经元数目默认比特征数多1,且添加偏置
		self.hidden_node_num = self.x.shape[1] + 1		
		# BP网络层数
		self.K = self.hidden_num + 2
		# 各层神经元数目
		self.layer_num = [self.hidden_node_num for i in range(self.K)]
		self.layer_num[0] = self.x.shape[1]
		self.layer_num[self.K-1] = self.y.shape[0]
		self.layer_num = tuple(self.layer_num)

		# 初始化权值矩阵,以及偏置向量
		self.initWeightsB()
		
	"""
	训练函数:
		:param options:训练参数
				1. rate:学习率
				2. alpha:权值修正系数
				3. max_loop:最大迭代次数
				4. eplison:收敛精度
				5. theLambda:for regularazation
	"""
	def	train(self,options=None):
		# 初始化训练选项
		rate = options['rate']
		max_loop = options['max_loop']
		eplison = options['eplison']
		theLambda = options['theLambda']
		for t in range(max_loop):
			# delta_W计算权值矩阵梯度偏导数所用的增量
			delta_W = [0 for num in range(self.K-1)]
			# delta_b计算偏置向量梯度偏导数所用的增量
			delta_b = [0 for num in range(self.K-1)]
			for i in range(self.m):
				# 各层输出
				a = []

				# 各层误差
				d = [1000 for num in range(self.K)]
				# Forward Propagation 计算各层输出
				for k in range(self.K):
					if k==0 :
						a.append(self.x[i].T)
					else:
						z = np.dot(self.weights[k-1],a[k-1])+self.b[k-1]
						a.append( sigmoid(z) )
				# Back Propagation 修正误差
				for k in range(self.K-1,0,-1):
					if k==self.K-1:
						d[k] = np.multiply(np.multiply(a[k],(1.0-a[k])),(a[k]-self.y[:,i]))
					else:
						d[k] = np.multiply(np.multiply(a[k],(1.0-a[k])),np.dot(self.weights[k].T,d[k+1]))
					# 修正权值
					delta_W[k-1] = delta_W[k-1] + np.dot(d[k],a[k-1].T)
					delta_b[k-1] = delta_b[k-1] + d[k]
			
			for k in range(self.K-1):
				self.weights[k] = self.weights[k] - rate*((1.0/self.m)*delta_W[k]+theLambda*self.weights[k])
				self.b[k] = self.b[k] - rate*(1.0/self.m*delta_b[k])
			sys_error = self.J(theLambda)
			if abs(sys_error) < eplison:
				break
		print 'The iteration has done %d times'%t

	"""  
	初始化权值矩阵，各权值落在随机分布在0附近
	"""
	def initWeightsB(self):
		for i in range(self.K - 1):
			m = self.layer_num[i+1]
			n = self.layer_num[i]
			weights = np.matrix(np.random.normal(scale=0.01,size=[m,n]))
			# 权值阵最后一行为阀值theta
			self.weights.append( weights)
			b = (np.random.normal(scale=0.01,size=[m,1]))
			self.b.append(b)
	
	"预测函数"
	def h(self,x):
		a = []
		for k in range(self.K):
			if k==0 :
				a.append(x)
			else:
				z = np.dot(self.weights[k-1],a[k-1])+self.b[k-1]
				a.append( sigmoid(z) )
		return a[self.K-1]

	"""
	整体代价函数
	"""
	def J(self,theLambda):
		result =0.0
		for i in range(self.m):
			error = self.h(self.x[i].T) - self.y[:,i]
			result = result + 1.0/2.0*(np.multiply(error,error)).sum()
		w_sum = 0.0
		for i in range(len(self.weights)):
			w_sum = w_sum + np.sum(np.multiply(self.weights[i],self.weights[i]))
		return 1.0/self.m*result-theLambda/(2.0*self.m)*w_sum


if __name__ == '__main__':  
	train_b = np.loadtxt("train.txt")
	train_x = train_b[:,0:4]
	train_y = train_b[:,4:7]
	train_x = np.matrix(train_x)
	train_y = np.matrix(train_y)
	train_y = train_y.T

	bpnn = BPNN(train_x,train_y,2)

	# 定义训练参数
	options = {
		'rate':5,
		'max_loop':2000,
		'eplison':0.02,
		'theLambda':0.0
	}
	###############################
	########    训练开始        ####
	###############################
	bpnn.train(options)
	test_b = np.loadtxt("test.txt")
	test_x = test_b[:,0:4]
	test_y = test_b[:,4:7]
	test_x = np.matrix(test_x)
	test_y = np.matrix(test_y)
	test_y = test_y.T


	print 'the error of test 1 is:'
	print abs(bpnn.h(test_x[10].T) - test_y[:,10])
	print 'the error of test 2 is:'
	print abs(bpnn.h(test_x[30].T) - test_y[:,30])
	print 'the error of test 3 is:'
	print abs(bpnn.h(test_x[40].T) - test_y[:,40])
	print 'the error of test 4 is:'
	print abs(bpnn.h(test_x[60].T) - test_y[:,60])
	print 'the error of test 5 is:'
	print abs(bpnn.h(test_x[70].T) - test_y[:,70])
```

测试结果也表明训练结果不错：

![训练结果](http://7pulhb.com1.z0.glb.clouddn.com/ml-BP训练结果.png)
