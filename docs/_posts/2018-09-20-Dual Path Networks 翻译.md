Dual Path Networks 
=============

Abstract
-------------

本文介绍了简单、高效率、模块化的Dual Path Network (DPN) 来做图像分类，它展现了内部连接路径的新拓扑。通过揭露HORNN框架中resnet和densenet的等价关系，我们发现ResNet允许特征重用，而DenseNet允许新特征探索，这都是对学习好的特征很重要的方面。为了利用这两个网络的好处，我们提出的DPN在共享通用特征的同时，利用双路径架构保持了灵活的发现新特征的能力。
额外的实验使用三个基准数据集ImagNet-1k, Places365 and PASCAL VOC，清楚表明了DNP的优势。特别是在ImagNet-1k数据集上，一个比较浅的DPN以更小的模块大小(小26%)，更少的计算消耗(少25%)，和更少的内存使用(少8%)，超过了最好的ResNeXt-101(64 × 4d)，并且更深的DPN(DPN-131)表现更好，且训练速度快2倍。其他数据集上的实验也体现了稳定的好表现(对比DenseNet，ResNet和ResNeXt)。


INTRODUCTION
-------------

“网络引擎”(Network engineering)在视觉识别的研究中尤为重要。本文旨在用新的深度框架的路径拓扑来把表示学习(representation learning)推向更前沿。具体来说，我们聚焦于分析和改进skip connection（它在现代深度网络中大量使用，并且取得了卓越的成绩）【Resnet、wrn、inception-resnet、resNext】。

‘Skip connection’  创造了把信息直接从低层网络传到高层的路径，在前向传播时，它允许靠前的网络层从较远的底层获取信息；同时对于后向传播，它更利于对底层的梯度反向传播而不用减小量级，这有效解决了梯度消失问题，并使优化更容易。

Resnet是成功的使用skip connections的网络之一，它采用小块，也就是残差函数与skip connection相连，称之为残差路径。残差路径在相同的小块上，把输入特征的每个元素添加到输出，使它成为残差单元。基于对小块内部结构的设计，残差网络已经发展了一系列的框架，包括WRN、Inception-resnet和ResNeXt。

DenseNet与ResNet不同，它不是用残差路径，而是使用稠密连接路径(a densely connected path)来连接输入和输出特征。
它允许每个micro-block 从上一层接收raw information。和ResNet相似，DensNet也可以归为densely connected network。
虽然网络越深，其宽度也会线性增长，参数的数目也会平方性增长，但是其参数效率比resnet更高

本文吸收了前面两者的特点和局限，提出了双通道结构。为了用全新的方式理解densely connected network，这里提到了higher order recurrent neural network (HORNN，Higher Order RNN) ，并且探索DenseNet和resnet的联系。
更具体来讲，当每一步的权重是共享的时候，densenet实际是HORNNs 。受【Bridging the gaps between residual learning, recurrent neural networks and visual cortex】启发，本文证明了resnet在每一层连接共享时是densenet。通过对这些网络结构的分析，本文发现，resnet通过残差路径隐含重用features，densenet通过densely connected path 继续探索新features。

基于这些，本文提出全新的双路径结构，称作Dual Path Network （DPN），继承了以上的两个优点：feature re-usage and re-exploitation 
并且DPN有更高的参数效率，更低的运算损失和内存消耗，并且优化起来更友好。

DPN用ImageNet-1k和Places365-Standard数据集做验证，达到最高准确率，另外DPN在图片分类 物体检测 语义分割都有很好的效果


RELATED WORK
-------------

设计一个高级的网络来提高图片分类效果，是最具挑战但是又最有效的方法之一，并且可用于其他任务。

AlexNet和VGG展现了深度卷积网络的威力，他们展示了用小卷积核来建立深度网络是有效的提升网络学习能力的方法。

卷积网络极大的消除了优化的难度，并且通过skepping connections把网络层数推到更深。从那以后，各种卷积网络出现，他们致力于在内部结构中建造更有效的小块，或者探索怎样使用residual connections。

最近提出的Dense Convolutional Networks, 通过把skip connections在输入和输出间连接，而不是添加的方式，使密集的连接路径(densely connected path)在宽度上跟随深度线性增长，从而导致参数数量平方增长，并且如果不进行特别的优化，将更耗GPU内存。这些是企图通过建造更深、更宽的densenet来提高准确率的限制。

除了设计新的结构以外，研究者也尝试重新探索已有的网络结构。【Identity mappings in deep residual networks】展示了卷积路径消除优化难度的重要性。【Bridging the gaps between residual learning, recurrent neural networks and visual cortex】中，卷积网络通过循环神经网络(RNNs)来桥接，使人更好的以RNN的视角理解深度卷积网络。【Sharing residual units through collective tensor factorization in deep neural networks】中，几种不同的卷积功能统一，试图对设计更高学习能力的小结构提供更好的理解。但是仍然对densely connected networks和一些对特征重用和效率梯度流的直观介绍还太少，不能深度理解。

本文致力于从HORNNs的视角对DenseNet(densely connected network)提供更深的理解，并且解释为什么残差网络是一种特殊的DenseNet。基于这些分析，本文提出了全新的Dual Path Network结构，它不仅实现了高精确度，而且有更高的参数和计算效率


Revisiting ResNet, DenseNet and Higher Order RNN
-------------

这段首先把DenseNet和HORNN桥接，来对DenseNet提供新的理解。证明了残差网络根本上是属于DenseNet家族，只不过残差网络的连接每一步共享。然后通过分析每个拓扑结构的优劣，促成了DPN的发展。

为了探索上述关系，本文展开新视野，从HORNNs的视角对DenseNet分析，解释了他们的关系，然后专门分析了残差网络。贯穿全文，我们用通用形式表达了HORNN：
![](/blog/images/dual_path_networks_1.jpg)

其中ht是在t步时循环网络的隐层状态，k是当前步骤的序号，xt是t步的输入，h0=x0。𝑓()表示特征提取函数，它以隐层状态做输入，输出提取信息。𝑔()表示转换函数，把提取的信息转换成当前隐层状态。
对于HORNNs，权重在每一步都共享，即：
![](/blog/images/dual_path_networks_2.jpg)

对于DenseNet，每一步(micro-block)有自己的参数，也就是𝑓()和𝑔()不共享。
这一发现表明，DenseNet的densely连接路径从根本上是个高阶路径，它可以从前一步状态分解新信息。

下图展示了DenseNet和高阶循环网络的关系：
![](/blog/images/dual_path_networks_3.jpg)

然后解释了残差网络是特殊的DenseNet，仅当：
![](/blog/images/dual_path_networks_4.jpg)

为了简便说明，这里引入rk来表示中间结果，并令r0=0，则上式可以写成：
![](/blog/images/dual_path_networks_5.jpg)

也可写成下面的迭代形式：
![](/blog/images/dual_path_networks_6.jpg)

显然，上式作为残差网络和循环网络都有相同的形式。特别的，当对任意k，∅ᴷ()≡∅()。上式退化成RNN；
当∅ᴷ()不共享，并且 Xᴷ = 0, k > 1时，上式变为残差网络。下图说明了它们的联系：
![](/blog/images/dual_path_networks_7.jpg)

另外，残差网络是在上述条件下导出的，意味着残差网络属于DensNet：
![](/blog/images/dual_path_networks_8.jpg)

Dual Path Networks
-------------

上面解释了残差网络和DenseNet的关系，展示了残差路径隐含重用了特征，但是并不善于发现新特征。DenseNet可以持续探索新特征但是存在高冗余。
这一节，我们会描述DPN的细节。

### Dual Path Architecture
基于上面的分析，提出了简单的双路径结构，它通过共享所有blocks的𝑓()，可以低冗余重用共同特征，并且保持densely connected path来使网络在学习新特征时更加灵活。表达式如下：
![](/blog/images/dual_path_networks_9.jpg)

其中(5)表示DenseNet，(6)表示残差网络，(7)表示dual path，(8)是加上了最后一层转换函数g()，
DPN网络图如下：
![](/blog/images/dual_path_networks_10.jpg)

更一般的，DPN是残差路径和稠密连接路径的卷积网络组合。可以自定义micro-block以供特殊用途，或者做进一步的性能提升

### Dual Path Networks
本文使用的DNP结构如上图所示，每一个micro-block被设计成bottleneck style【Deep residual learning for image recognition】，
它以1x1的卷积层开始，中间是3x3，最后以1x1结束。输出层的1x1卷积分成两部分：第一部分是以逐个元素(element-wisely)的方式加入卷积路径，第二部分和稠密连接路径相连。为了增加每个micro-block的学习能力，我们像ResNeXt一样在第二层使用了组合的卷积层。

考虑到卷积网络比稠密连接网络更常用，我们选择了卷积网络作为主干，然后添加了稀疏的稠密连接路径来建立DPN。这种设计也减少了稠密连接路径的宽度增长和GPU内存的消耗。下图展示了详细的结构设置：
![](/blog/images/dual_path_networks_11.jpg)

其中G是group的数量，k是稠密连接路径的通道增加数，+k是新发表的DPNs的宽度增加数。
实现DPN可以通过在已有残差网络的基础上增加“切片层”和“连接层”。在一个较好的优化的深度学习平台上，这些操作都不会增加额外的计算消耗和内存消耗。
网络模型效率对比如下（尽量小的模型，超参数依据经验，没有使用网格搜索）：
![](/blog/images/dual_path_networks_12.jpg)

Experiments
-------------

为了评估DPN，做了大量的实验。具体来说有三类：图片分类、物体检测、语义分割。用了三个对应的基准数据集：the ImageNet-1k dataset, Places365-Standard dataset 和 the PASCAL VOC datasets

### Experiments on image classification task
我们在MXnet上实现DPNs，用了40个K80显卡。根据【Sharing residual units through collective tensor factorization in deep neural networks】采用标准数据增强方法，使用mini-batch大小为32的SGD训练。比较深的网络如DPN-131，因为12G的显存有限，mini-batch设置为24。DPN-92和DPN-131的学习率从√0.1，DPN-98从0.4开始。每隔一定周期递减0.1倍。根据【Deep residual learning for image recognition】batch normalization layers are refined after training

* ImageNet-1k dataset

图片分类表现，看图说话（分别是不同模型的准确度、模型大小、训练时间等）：
![](/blog/images/dual_path_networks_13.jpg)
![](/blog/images/dual_path_networks_14.jpg)
![](/blog/images/dual_path_networks_15.jpg)

* Place365-Standard dataset

我们使用Places365-Standard dataset做场景分类任务，它是高分辨率的场景理解数据集，由365个场景总共180万图片组成。不同于物体图片，场景图片没有明确的离散物体，它需要更高层次的场景理解能力。

### Experiments on the object detection task
物体检测任务是依据【Faster r-cnn: Towards real-time object detection with region proposal networks】在VOC 2007 trainval和VOC 2012 trainval下训练，在VOC 2007 test set下评估：

![](/blog/images/dual_path_networks_16.jpg)

我们所有的实验都是以残差网络为基础的R-CNN框架，通过替换残差网络来做对比，其他部分都不变。

### Experiments on the semantic segmentation task
这个实验对DPN的密度检测性能做评估，也就是语义分割，其训练目标是对输入图像的每个像素点预测语义标签。我们使用PASCAL VOC 2012做基准数据集，使用DeepLab-ASPP-L做框架。
下图的比较是把conv4 and conv5的3*3卷积层用atrous convolution替换的，并且在最后的conv5的特征表中插入了一系列Atrous Spatial Pyramid Pooling (ASPP)

![](/blog/images/dual_path_networks_17.jpg)


conclusions
-------------

略