论文阅读 Realtime Multi-Person 2D Pose Estimation using Part Affinity Fields 
=============
对应开源项目：https://github.com/ZheC/Realtime_Multi-Person_Pose_Estimation


abstract
-------------
介绍一种高效的多人2D姿态识别，它使用非参数表示(non-parametric representation)，这里我们称之为:
Part Affinity Fields (PAFs)，来关联个人的身体区域。

整个框架对全局信息encode，使用贪心的从下到上的解析，不仅能保持高准确率，还能保持实时效果，效率和人数无关
框架通过两个连续的分支，联合学习局部位置和他们的关系


introduction
-------------
人体2D姿态检测问题，也就是关键点或部位定位问题，主要都聚焦在寻找个体的身体区域。
挑战有很多：图片大小、人体位置、个数、人体重叠引起的空间干扰、实时效率

通常的做法是先检测到单个人体，再在人体上做姿态检测。这种top-down的做法有两个弊端:人体检测可能因为遮挡失败，实时性根据人体个数而不同。

bottpm-up的做法并不是直接使用全局上下文。实际上，早期的很多做法，在最后一步需要全局inference，这样效率也不高。比如有人使用联合的标签候选区域，并把它们关联到单个人；然而这种在全连接图上的整数线性规划问题是NP问题。有人基于resnet使用强区域检测器(stronger part detectors)和image-dependent pairwise scores ，仍然需要几分钟，而且和proposal到的part的数量有关。pairwise的表示方法很难回归准确，因此还需要额外的逻辑回归。

我们提出了第一个通过Part Affinity Fields (PAFs) 来关联分数的bottom-up的方法，一组对肢体位置和方向encode的二维向量。



Method 
-------------
pipeline如下:
![](/blog/images/realtime_multi_person_estimation/1.jpg)
前馈网络同时预测2D的confidence maps of body part locations (fig.2b) 称为S
2D的part affinities fields, 它对degree of association between parts 进行encode(fig.2c)  称为L
最后S和L通过greedy inference解析(fig.2d) 、 输出所有人的2D关键点

### Simultaneous Detection and Association 
![](/blog/images/realtime_multi_person_estimation/2.jpg)

如上图，米黄色部分预测confidence map
蓝色部分预测affinity fields

图片最先使用经过fine tuned的VGG-19的前十层初始化，产生第一层特征表F。
F通过CNN分别产生S和L。
剩下的阶段使用前阶段产出的S L 和原始 F进行基尼异步精炼:
![](/blog/images/realtime_multi_person_estimation/2_2.jpg)

每个阶段精炼后的confidence map和affinity field如下图：
通过global inference， 渐渐区分出左右手:
![](/blog/images/realtime_multi_person_estimation/3.jpg)

每个阶段的末尾使用L2loss来分别计算maps和fields的误差。具体的计算方式如下:
![](/blog/images/realtime_multi_person_estimation/3_2.jpg)
其中W是binary mask，用来避免true positive预测的惩罚，
每一阶段周期地补充梯度，用来避免梯度消失

全局目标是:
![](/blog/images/realtime_multi_person_estimation/3_3.jpg)


### Confidence Maps for Part Detection 
为了评估confidence map，对图片中第k个人的可见的第j个身体部位的ground truth part confidence map做定义：
![](/blog/images/realtime_multi_person_estimation/3_4.jpg)

其中P是location，σ用来控制峰值的延伸，
网络中用来预测的confidence map是通过max运算符对单个confidence map的聚合:
![](/blog/images/realtime_multi_person_estimation/3_5.jpg)

只所以选用max，而不是average，是因为极大值能保存峰值相近时的不同:
![](/blog/images/realtime_multi_person_estimation/4.jpg)

身体部位的候选采用非极大值抑制

### Part Affinity Fields for Part Association 
![](/blog/images/realtime_multi_person_estimation/5.jpg)

对于检测到的身体区域点，怎样在不知道有几个人的情况下，连接它们，组成完整的身体呢？
我们需要有一套打分机制，给不同组合打分，来确定它们属于同一个人。

一种可能的做法是，检测肢体对的中间点，判断它的可能性。如fig(5b)。但是当人靠得比较近时，有可能出错，
就像fig(5b)中绿色的线。这种出错情况可归咎为(1)我们只encode位置，没有encode方向。(2)肢体的区域被缩小成了点

为了解决(address)这些问题，我们提出了全新的表示方式，称之为Part Affinity Fields (PAFs)。
在候选的区域内既保存位置，又保存方向。如图fig(5c)。
PAF是一个每个肢体(limb)都有的2维向量。对于肢体区域中的每个像素点，2维向量都编码了肢体一部分到另一部分的方向。对不同的肢体类型，都有对应的PAF与它相关的两个身体位置相联系

考虑下图的单个肢体：
![](/blog/images/realtime_multi_person_estimation/6.jpg)

X是ground truth，如果P是肢体上的一点，
![](/blog/images/realtime_multi_person_estimation/7.jpg)

如果P点在k人的c肢体上，则PAF的值为v，否则为0. 这里  
![](/blog/images/realtime_multi_person_estimation/8.jpg)
是肢体c方向上的单位向量。
肢体上的点就被定义为在一定距离阈值内的线段，也就是点P满足：
![](/blog/images/realtime_multi_person_estimation/9.jpg)

ground truth PAF是图像中所有人PAF的均值:
![](/blog/images/realtime_multi_person_estimation/10.jpg)

其中Nc(P)是所有k个人的P点的非零向量数(也就是不同人的肢体在像素点上交错的平均值)

在测试时，我们用计算相关PAF的线性积分与线段连接的方式，来度量候选区域的关联度。
换句话说，我们连接检测到的身体部分组成的肢体，和预测到的PAF，通过判断两者是否对齐，来测量准确性、
具体来说，对于候选区域Dj1和Dj2，对线段上的PAF采样来测量他们相关信的置信度。
![](/blog/images/realtime_multi_person_estimation/11.jpg)

这里P(u)是Dj1和Dj2的中间插入值: P(u) = (1-u)Dj1 + uDj2
在实际中，我们采样和累加等间隔的u，来近似求积分


### Multi-Person Parsing using PAFs 
我们使用非极大值抑制，从检测出的confidence map中来获得一组离散的候选区域位置。
对每个位置而言，因为存在多个人，或者预测错，都可能有多个候选。

![](/blog/images/realtime_multi_person_estimation/12.jpg)

这些候选点可以组成很多种不同的肢体组合。我们使用上面式子10，来计算不同肢体组合线性积分的置信度。
可是这个寻找最优解的问题是一个K维匹配问题，是NP难的。(fig.6c)
本文提出了一个贪心解法：因为PAF网络庞大的接收域，我们推测配对分数隐含编码在全局context中。

正式来说，我们获取一组多人的身体区域的候选集Dj。这些候选集需要与同一个人的其他区域联合起来，也就是需要配对组合成肢体。我们定义变量
![](/blog/images/realtime_multi_person_estimation/13.jpg)
来表示两个区域是否连接。对单个人而言，问题归纳为：有权重的二分图匹配问题。如图fig6b，其中每条边的权重都是式子10：
![](/blog/images/realtime_multi_person_estimation/14.jpg)
![](/blog/images/realtime_multi_person_estimation/15.jpg)

式子13/14 确保没有两条边共享一个节点的情况
我们可以用匈牙利算法求解最佳匹配

为了解决NP难问题，我们添加了两个松弛(参见线性规划的松弛)。
首先，选择最小数量的边缘来取得人体姿态的生成树骨架，而不是使用整个图，参见fig.6c
然后，进一步分解匹配问题成一系列双边匹配子问题，然后分别判定邻近树节点的匹配，参见fig.6d。
结果表明效果还是不错的，因为邻近树节点的关系被显式模块化在PAFs中，但是在内部，非邻近节点之间的关系又被隐式模块化在CNN中。因为CNN是通过大接收域来训练的，而且非邻近树节点的PAFs同样影响着被预测的PAF。

有了这两个松弛方法，最终的优化问题分解成:
![](/blog/images/realtime_multi_person_estimation/16.jpg)


Results 
-------------
两个benchmark数据集:the MPII human multi-person dataset 和 COCO 2016 keypoints challenge dataset 
评价指标是对所有的body parts使用平均精度均值(mean Average Precision  mAP)


Discussion 
-------------


