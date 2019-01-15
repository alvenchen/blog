Dense Human Pose Estimation In The Wild 翻译
=============

介绍前
-------------
这是facebook的一篇研究，项目地址在https://github.com/facebookresearch/DensePose。
本文只翻译重要的内容，实验结果什么的就不翻了，要更深入的了解，最好还看下论文引用的相关文章。

在理解这篇文章之前，需要了解两个重要的名词：
UV映射(uv mapping)：把2D图像细节投影到3D表面上的映射。(UV是坐标轴,之所以叫UV，是因为XYZ都已经被坐标系用掉了)
稠密相关(dense correspondences):在3D重建领域，从原图到目标图的重建。举个直观的例子:
把图二按照图一的稠密相关性展开成图三:
![](/blog/images/dense_human_pose/1.jpg)


Abstract
-------------

茂密的人体姿态检测：把人体从RGB图片转换为基于表面的表示。
本文介绍一种高效的打标签流程，然后训练基于CNN的网络，可以用于各种场景(in the wild)，即存在遮挡、大小变换等。


INTRODUCTION
-------------
本工作旨在更好的在图片中封装人体，把2D转换成3D，基于表面的表示(surface based representation)。
可以类比物体检测、姿态估计、实例分割等。可以有助于增强现实、人机交互等的应用。

其他的解决方法通常都是基于depth sensor，我们是在RGB图像的基础上建立表面点和图像像素上的联系。

一些其他的任务的目的是用非监督的方式恢复一组目标的dense correspondences。最近也有些用同变型原理(equivariance principle)来把一组图片对齐到同一个坐标系下。

我们把任务缩小到人体上，可以简化模型成探索参数可变的表面模型，例如 Skinned Multi-Person Linear (SMPL) model。或者图片到表面的映射：先用CNN找出人体关键点，然后通过最小化迭代拟合参数可变表面模型(parametric deformable surface model)。

我们的方法论和上面的都不同。我们使用成熟的监督学习的方法，并且收集图片到parametric surface model 的真实映射数据。仅使用SMPL 作为训练时定义问题的方法，而不是在测试时使用它。

Fashionista, PASCAL-Parts, and Look-Into-People (LIP)数据集提供了人体分段数据，可以作为image-to-surface 的粗估计使用(非连续，是离散的坐标点)。其他也有些合成的表面级别的数据。
我们介绍一种全新的标记流程，以助于收集50k COCO数据集的标签。

介绍了一些成果。4-5帧  1080GPU，800×1100的图片。

本文总体有三个成就：
* 打标签的系统
* 利用CNN回归人体坐标(基于Deeplab的全连接网络， 和基于区域识别的Mask-RCNN都有尝试，后者更好)。
* 探索了不同的利用结构化真实数据(ground truth)的方法。

COCO-DensePose Dataset
-------------
50k humans   5 million manually annotated correspondences 

### Annotation System
目标：2D images到surface-based的转换。
简单的做法：寻找图片中的”顶点”，然后做表面旋转。但是这样的做法效率很低。
我们提出的方法：
如图：
![](/blog/images/dense_human_pose/2.jpg)

* 先框出范围,包括头、躯干、上下手臂、大小腿、手掌、脚。
为了简化UV参数，把躯体和肢体分为上下、前后的相同部分。
对于头、手掌、脚，使用SMPL model中的UV域。对于其他部分，使用多维展开的，按对匹配测量距离后的UV域，如下图：
![](/blog/images/dense_human_pose/3.jpg)
这里要注意的是，我们对躯干的估计是假设去掉衣服后的估计。

* 对每个区域采用k-means估计后粗等距的点，然后要求标记者把这些点放到相应的表面空间。
采样点的个数和区域的大小有关，但个数的最大值是14。
为了简化任务，把区域表面打开成身体同区域的6个预渲染的的视图，让标记者可以把标记放置于其中任意一个比较方便的图上，而不用手动旋转。

实验统计这两部分打标记的时间是差不多的。

### Accuracy of human annotators
传统的估计准确度的方法是，让多个标记者标记同一份数据，然后看方差
我们使用渲染后的真实位置，和标记者估计的位置对比。
具体来说，我们用同一个surface model生成的图片，给标记者，让他们用标记工具把合成的图片标记到表面坐标中去。
通过计算每个标记点与正确点之间的距离来衡量误差。
如图，大面积范围躯体的误差更大：
![](/blog/images/dense_human_pose/4.jpg)

### Evaluation Measures 
逐点评估
分人体实例评估

Learning Dense Human Pose Estimation
-------------
本文综合了Dense Regression (DenseReg) system 和Mask-RCNN。

### Fully-convolutional dense pose regression
简单的做法是用全卷积网络，类似DenseReg，结合分类和回归。
第一步，分类像素点到背景、不同的身体区域。使用交叉熵做损失函数。
第二步，回归方法来表示像素点的具体坐标。使用smooth L1 

### Region-based Dense Pose Regression
我们采用这种基于区域的估计方法，它由层级连接的regions-of-interest (ROI) 组成。
通过RIO pooling提取region-adapted的特征。
如下图：
![](/blog/images/dense_human_pose/5.jpg)
在ROI-pooling 之前，使用全连接网络产生分类和回归的预测。

### Multi-task cascaded architectures
受iterative refinement的灵感，我们使用级层的框架。它的好处是给下一步骤提供上下文，和有益于深度挖掘。
如下图：
![](/blog/images/dense_human_pose/6.jpg)
挖掘的信息不受限于单一任务。

### Distillation-based ground-truth interpolation
在训练之前，训练一个”teacher”网络来重建那些没有标记的值。如图：
![](/blog/images/dense_human_pose/7.jpg)

Experiments
-------------

Conclusion
-------------