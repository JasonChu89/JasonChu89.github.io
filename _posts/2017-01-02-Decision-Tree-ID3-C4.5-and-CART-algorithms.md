---
layout: post
title: "Decision Tree, ID3, C4.5 and CART algorithms"
description: "决策树、ID3、C4.5以及CART算法"
categories: [Machine-Learning]
tags: [机器学习, 决策树, ID3, C4.5, CART]
redirect_from:
  - /2017/01/02/
---

* Kramdown table of contents
{:toc .toc}

# 决策树、ID3、C4.5以及CART算法

决策树模型在监督学习中非常常见，可用于分类和回归。虽然将多棵弱决策树的Bagging、Random Forest、Boosting等tree ensemble 模型更为常见，但是“完全生长”决策树因为其简单直观，具有很强的解释性，也有广泛的应用，而且决策树是tree ensemble 的基础，值得好好理解。[^f1]

一般而言一棵“完全生长”的决策树包含，特征选择、决策树构建、剪枝三个过程，这篇文章主要是简单梳理比较ID3、C4.5、CART算法。《统计学习方法》中有比较详细的介绍。

## **1.决策树的优点和缺点**

### **优点**
* 决策树算法中学习简单的决策规则建立决策树模型的过程非常容易理解，模型可以可视化，非常直观
* 应用范围广，可用于分类和回归，而且非常容易做多类别的分类
* 能够处理数值型和连续的样本特征

### **缺点**
* 很容易在训练数据中生成复杂的树结构，造成过拟合（overfitting）。剪枝可以缓解过拟合的负作用，常用方法是限制树的高度、叶子节点中的最少样本数量
* 学习一棵最优的决策树被认为是NP-Complete问题。实际中的决策树是基于启发式的贪心算法建立的，这种算法不能保证建立全局最优的决策树。Random Forest 引入随机能缓解这个问题
* 决策树模型无法表示类似异或（XOR），相乘的概念，神经网络可以很容易的表示出来

## **2.基本概念**
### **2.1 信息熵(information entropy)**
在1948年，克劳德·艾尔伍德·香农将热力学的熵，引入到信息论，因此它又被称为香农熵。一个系统越是有序，信息熵就越低；反之，一个系统越是混乱，信息熵就越高。信息熵也可以说是系统有序化程度的一个度量。
依据Boltzmann's H-theorem，香农把随机变量X的熵值 Η（希腊字母[Eta](https://zh.wikipedia.org/wiki/%CE%97)）定义如下，其值域为$\{x_{1}, ..., x_{n}\}$：
$$\mathrm{H} (X) = E[I(X)] = E[-\ln(P(X))]$$
其中，P为X的概率质量函数（probability mass function），E为期望函数，而I(X)是X的信息量（又称为自信息）。I(X)本身是个随机变数。
当取自有限的样本时，熵的公式可以表示为：
$$\mathrm{H}(X) = \sum_{i} \mathrm{P}(x_{i})\mathrm{I}(x_{i})= -\sum_{i} \mathrm{P}(x_{i})\log_{b}\mathrm{P}(x_{i})$$
在这里b是对数所使用的底，通常是2,自然常数e，或是10。当b = 2，熵的单位是bit；当b = e，熵的单位是nat；而当b = 10,熵的单位是Hart。[^f2]

### **2.2 条件熵(Conditional Entropy)与信息增益(Information Gain)**
如果用熵表示随机变量的不确定性，条件熵表示在一个条件下，随机变量的不确定性，则信息增益为：熵 - 条件熵。
条件熵的表达式:
1.当特征x被固定为值$x_{i}$时，条件熵为: $H(c|x=x_{i}) $
2.当特征X的整体分布情况被固定时，条件熵为:$H(c|X) $
那么因为特征X被固定以后，给系统带来的增益(或者说为系统减小的不确定度)为： $IG(X)=H(c)−H(c|X)$

### **2.3 基尼系数(基尼不纯度Gini impurity)**
Gini系数是一种与信息熵类似的做特征选择的方式，可以用来数据的不纯度。在CART(Classification and Regression Tree)算法中利用基尼指数构造二叉决策树。
Gini系数的计算方式如下：
$$Gini(D)=1−\sum_{i}^{n}p_{i}^{2}$$
其中，D表示数据集全体样本，pi表示每种类别出现的概率。取个极端情况，如果数据集中所有的样本都为同一类，那么有p0=1，Gini(D)=0，显然此时数据的不纯度最低。
与信息增益类似，我们可以计算如下表达式：
$$Gini(D|A)=\sum_{i=0}^{n}\frac{D_{i}}{D}$$
上面式子表述的意思就是，加入特征X以后，数据不纯度减小的程度。很明显，在做特征选择的时候，我们可以取ΔGini(X)最大的那个[^f3]

## **3.ID3算法**
ID3算法（Iterative Dichotomiser 3 迭代二叉树3代）是一个由Ross Quinlan在1986年发明的用于决策树的算法。[^f4]
ID3决策树可以有多个分支，但是不能处理特征值为连续的情况。决策树是一种贪心算法，每次选取的分割数据的特征都是当前的最佳选择，并不关心是否达到最优。在ID3中，每次根据“最大信息熵增益”选取当前最佳的特征来分割数据，并按照该特征的所有取值来切分，也就是说如果一个特征有4种取值，数据将被切分4份，一旦按某特征切分后，该特征在之后的算法执行中，将不再起作用，所以有观点认为这种切分方式过于迅速。ID3算法十分简单，核心是根据“最大信息熵增益”原则选择划分当前数据集的最好特征，信息熵是信息论里面的概念，是信息的度量方式，不确定度越大或者说越混乱，熵就越大。在建立决策树的过程中，根据特征属性划分数据，使得原本“混乱”的数据的熵(混乱度)减少，按照不同特征划分数据熵减少的程度会不一样。在ID3中选择熵减少程度最大的特征来划分数据（贪心），也就是“最大信息熵增益”原则。
ID3步骤为：
1. 计算数据集D的信息熵H(D)
2. 遍历所有特征，计算特征对于数据集D的条件熵
3. 选择对使得信息增益最大的特征，并从数据集D中去掉该特征

ID3算法缺点：

- ID3算法不能处理具有连续值的属性
- ID3算法不能处理属性具有缺失值的样本
- 算法会生成很深的树，容易产生过拟合现象
- 算法一般会优先选择有较多属性值的特征，因为属性值多的特征会有相对较大的信息增益

## **4.C4.5算法**
C4.5算法是由Ross Quinlan在1993年开发的用于产生决策树的算法。该算法是对Ross Quinlan之前开发的ID3算法的一个扩展。C4.5算法产生的决策树可以被用作分类目的，因此该算法也可以用于统计分类。[^f5]
C4.5中使用信息增益比率(Infomation Gain Ratio)来作为选择分支的准则。信息增益比率通过引入一个被称作分裂信息(Split information)的项来惩罚取值较多的特征。
$$GR(D|A)=\frac{G(D|A)}{SI(D|A)}$$
其中，$SI(D|A)$表示分裂信息值，它的定义为(实际就分类熵)：
$$SI(D|A)=-\sum_{i=i}^{n}\frac{N_{i}}{N}\log_{2}\frac{N_{i}}{N}$$
除此之外，C4.5还弥补了ID3中不能处理特征属性值连续的问题。但是，对连续属性值需要扫描排序，会使C4.5性能下降，有兴趣可以参考[博客](http://leijun00.github.io/2014/09/decision-tree/)。
C5.0是一个商业软件，对于公众是不可得到的。它是在C4.5算法做了一些改进，具体请参考：[C5.0算法改进](http://cse-wiki.unl.edu/wiki/index.php/Decision_Trees,_Overfitting,_and_Occam%27s_Razor#C5.0)
C5.0主要增加了对Boosting的支持，它同时也用更少地内存。它与C4.5算法相比，它构建了更小地规则集，因此它更加准确。

## **5.CART算法**
CART（Classification and Regression tree）分类回归树由L.Breiman,J.Friedman,R.Olshen和C.Stone于1984年提出。
### 5.1 划分
ID3中根据属性值分割数据，之后该特征不会再起作用，这种快速切割的方式会影响算法的准确率。CART算法要检查每个变量和该变量所有可能的划分值来发现最好的划分，对**离散值** 如{x,y,x}，则在该属性上的划分有三种情况$\{\{x,y\},\{z\}\},\{\{x,z\},y\},\{\{y,z\},x\}$，空集和全集的划分除外；对于**连续值**处理引进“分裂点”的思想，假设样本集中某个属性共n个连续值，则有n-1个分裂点，每个“分裂点”为相邻两个连续值的均值 (a[i] + a[i+1]) / 2。[^f6]
CART是一棵二叉树，采用二元切分法，每次把数据切成两份，分别进入左子树、右子树。而且每个非叶子节点都有两个孩子，所以CART的叶子节点比非叶子多1。相比ID3和C4.5，CART应用要多一些，既可以用于分类也可以用于回归。
CART分类时，使用基尼指数（Gini）来选择最好的数据分割的特征，gini描述的是纯度，与信息熵的含义相似。CART中每一次迭代都会降低GINI系数。下图显示信息熵增益的一半，Gini指数，分类误差率三种评价指标非常接近。
![这里写图片描述](/upload_imgs/754644-20160411201334707-543610148.png)
在实际应用中，Gini index和熵最终会得到非常相似的结果
### 5.2 终止条件
一个节点产生左右孩子后，递归地对左右孩子进行划分即可产生分类回归树。那么在什么时候节点就可以停止分裂？直观的情况，当节点包含的数据记录都属于同一个类别时就可以终止分裂了。这只是一个特例，更一般的情况我们计算$x^{2}$值来判断分类条件和类别的相关程度，当$x^{2}$很小时说明分类条件和类别是独立的[^f7]，即按照该分类条件进行分类是没有道理的，此时节点停止分裂。注意这里的“分类条件”是指按照GINI_Gain最小原则得到的“分类条件”。[^f8]

## **6.scikit-learn 中的决策树**
scikit-learn中的决策树使用CTRX算法实现，详细阅读参考:[http://scikit-learn.org/stable/modules/tree.html](http://scikit-learn.org/stable/modules/tree.html)
代码示例及API参考：[scikt-learn 官网](http://scikit-learn.org/stable/modules/generated/sklearn.tree.DecisionTreeClassifier.html#sklearn.tree.DecisionTreeClassifier)

## **7.分类树 VS 回归树**
提到决策树算法，很多想到的就是上面提到的ID3、C4.5、CART分类决策树。其实决策树分为分类树和回归树，前者用于分类，如晴天/阴天/雨天、用户性别、邮件是否是垃圾邮件，后者用于预测实数值，如明天的温度、用户的年龄等。
作为对比，先说分类树，我们知道ID3、C4.5分类树在每次分枝时，是穷举每一个特征属性的每一个阈值，找到使得按照feature<=阈值，和feature>阈值分成的两个分枝的熵最大的feature和阈值。按照该标准分枝得到两个新节点，用同样方法继续分枝直到所有人都被分入性别唯一的叶子节点，或达到预设的终止条件，若最终叶子节点中的性别不唯一，则以多数人的性别作为该叶子节点的性别。
回归树总体流程也是类似，不过在每个节点（不一定是叶子节点）都会得一个预测值，以年龄为例，该预测值等于属于这个节点的所有人年龄的平均值。分枝时穷举每一个feature的每个阈值找最好的分割点，但衡量最好的标准不再是最大熵，而是最小化均方差--即（每个人的年龄-预测年龄）^2 的总和 / N，或者说是每个人的预测误差平方和 除以 N。这很好理解，被预测出错的人数越多，错的越离谱，均方差就越大，通过最小化均方差能够找到最靠谱的分枝依据。分枝直到每个叶子节点上人的年龄都唯一（这太难了）或者达到预设的终止条件（如叶子个数上限），若最终叶子节点上人的年龄不唯一，则以该节点上所有人的平均年龄做为该叶子节点的预测年龄。



[^f1]: [http://www.cnblogs.com/wxquare/p/5379970.html)](http://www.cnblogs.com/wxquare/p/5379970.html)

[^f2]: [https://zh.wikipedia.org/wiki/熵_(信息论)](https://zh.wikipedia.org/wiki/%E7%86%B5_(%E4%BF%A1%E6%81%AF%E8%AE%BA))

[^f3]: [http://blog.csdn.net/bitcarmanlee/article/details/51488204](http://blog.csdn.net/bitcarmanlee/article/details/51488204)

[^f4]: [https://en.wikipedia.org/wiki/ID3_algorithm](https://en.wikipedia.org/wiki/ID3_algorithm)

[^f5]: [https://zh.wikipedia.org/wiki/C4.5算法](https://zh.wikipedia.org/wiki/C4.5%E7%AE%97%E6%B3%95)

[^f6]: [http://www.cnblogs.com/happyblog/archive/2011/09/30/2196901.html](http://www.cnblogs.com/happyblog/archive/2011/09/30/2196901.html)

[^f7]: [$x^2$计算方法参考](http://www.cnblogs.com/zhangchaoyang/articles/2642032.html)

[^f8]: [http://blog.csdn.net/b_h_l/article/details/9214745](http://blog.csdn.net/b_h_l/article/details/9214745)