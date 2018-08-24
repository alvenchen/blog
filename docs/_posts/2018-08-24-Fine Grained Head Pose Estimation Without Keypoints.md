Fine-Grained Head Pose Estimation Without Keypoints 翻译
=============
Abstract
-------------

估计头部姿态可用来辅助目光估计、注意力建模、拟合3D视频模型、实现人脸对齐等。
传统的方法是通过估计关键点计算，然后用平均人类头部模型解决2D到3D的映射问题。
但本文认为该方法完全依赖landmark、外来的头部模型和ad-hoc拟合步骤, 不是很靠谱

本文提出了，通过训练a multi-loss convolutional neural network on 300W-LP,
来直接预测 intrinsic Euler angles (yaw, pitch and roll)
在测试集common in-the-wild pose benchmark datasets上表现出色


INTRODUCTION
-------------

以前主要是两种方法，除了上面介绍的landmark估计以外，还有PAMs..
然后说了很多通过人脸关键点估计的缺点—误差、转3D模型的依赖和计算量

在应用场景中，通常的解决方案使用RGBD (depth) cameras
但是深度摄像机受光线影响大，户外使用困难。（另外说了两点电池和存储的问题）

这篇文章的关键工作是：
* 提出用多损失网络预测头部姿态欧拉角度的方法
每个角度(yaw, pitch, roll)有一个loss
每个loss由两部分组成：角度二进制分类，和回归

* 通过在大量合成数据集上的训练，在几个测试集上取得了不错的效果，展示了模型的通用能力
* ablation studies？ 展示卷积网络结构
* 详细说明了通过2D脸部特征点预测的精度和它的缺点，以及我们是怎样解决的
* 研究了低分辨率下不同方法的影响。说明了本文的方法的有效

RELATED WORK
-------------

介绍了其他的一些方式
其中有通过潜的神经网络在AFLW数据集上训练的
有通过修改GoogleNet把预测面部关键点和pose结合起来的
Hyperface通过使用基于R-CNN的方式，用修改的AlexNet把卷积网络的输出连接起来，并且分别加上全连接层，来预测不同的子任务(微笑、性别、面部识别) 相当于all in one
也有不通过人脸特征点，通过使用简单CNN回归3D头部姿态实现的
比较相似的是以个用VGG网络来回归头部姿态欧拉角的，不同的是他们用RNN加了时间维度，并且没有用外部数据来finetune

METHOD
-------------

##### Advantages of Deep Learning for Head Pose Estimation
deep networks 比 landmark-to-pose 好的例子：
不依赖head model、 bla bla ..  (就是abstract里面的东东反复讲)

##### The Multi-Loss Approach
以前的工作都是直接对三个方向的error loss的均方值进行回归
本文用三个不同的loss，每个主网络都与三个全连接层相连，用来预测角度* （划重点） *

![](images/multi-loss?raw=true)

整体的loss函数就是交叉熵loss + 均方根loss

#####  Datasets for Fine-Grained Pose Estimation
本文使用了AFLW2000 dataset
BIWI dataset(RGB-D video) 做测试

##### Training on a Synthetically Expanded Dataset
300W-LP数据集(自然条件下的2D 特征点数据)

##### The Effects of Low-Resolution
低分辨率下人脸特征点会消失，
本文讨论了在低分辨率下简单有效的方法：通过随机下采样和上采样来增强数据集
用模糊来增强数据集的方法也做了实验


EXPERIMENTAL RESULTS
-------------

##### Fine-Grained Pose Estimation on the AFLW2000 and BIWI Datasets
￼
￼
##### Landmark-To-Pose Study
￼
*68特征点居然有最大的错误率*

￼
￼
*特征点少，受抖动的影响更大*

##### 其他的实验不翻了，没什么干货


CONCLUSIONS AND FUTURE WORK
-------------