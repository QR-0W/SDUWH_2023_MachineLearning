[TOC]



# 结果预览

![image-20230130021643762](/asset/image-20230130021643762.png)

测试环境：

CPU: Intel Core i7-12700KF

GPU: NVIDIA RTX 3080TI

RAM: 32GB



# 一、OCR简述

## 	1.1 OCR是什么

OCR（Optical Character  Recognition，光学字符识别）是计算机视觉重要方向之一。传统定义的OCR一般面向扫描文档类对象，现在我们常说的OCR一般指场景文字识别（Scene Text Recognition，STR），主要面向自然场景，如下图中所示的牌匾等各种自然场景可见的文字。



## 	1.2 OCR流程

文字识别的全流程包括：图像预处理，OCR识别，恢复版面，后处理。

OCR主要分为两部分: 

1. 文本检测, 即找出文本所在位置。
2. 文本识别, 将文字区域进行识别。



## 	1.3 OCR技术挑战

在OCR技术中，有几种最常见的问题：透视变换、尺寸过小、文字扭曲、背景复杂、字体多样、语言混合、字体模糊、光照复杂。这些问题，在本次的轮胎OCR识别中都有着或多的体现。





# 二、图像预处理

## 	2.1 原始方法

在本次项目中，我个人使用的点云转换为图像的具体步骤如下：

1. 原始数据提取高度数据

2. 高度数据转化为高度图

3. 裁切高度图

4. 修复图像

5. 展平图像

6. 规格化

7. 去噪

8. 直方图均衡化

9. 直方图裁切

   

### 		2.1.1 原始数据提取高度数据

原始数据是一个（X，Z，Y）格式的三维点云数据，但是根据扫描线的流程来看，可以只采用Z坐标，因为扫描线是逐行的。



### 			2.1.2 图像修复

图像中存在着本不该存在的点，因此需要去除这些点。采用滤波的方式填充、去除这些点。参考资料，可采用的方案有：

1. 去零均值滤波：虽然会在一定程度上破坏图像的细节，但是有效的抑制噪声。而且实际检验，对于细节的丢失可以接受。
2. 形态学运算：主要是腐蚀与膨胀。这一操作可以快速提取形态特征，但是效果并不如上面的滤波。



### 			2.1.3 展平图像

利用Open3D生成的点云观察可得，轮胎本省并不是平整的，而是一个曲面。这就需要展平图像。

![image-20230130142811133](/asset/image-20230130142811133.png)

本次采用的方法是：估计曲面的高度，然后用原本的点云减去估计的高度。估计高度的方法有：

1. 扫描线均值估计：求出轮胎每一行的高度平均值，依次作为曲面的估计值。

2. 均值估计：利用均值模糊的原理，消除高度差。

根据生成的高度图来看，扫描线均值估计的结果虽然更精确，但是因为轮胎本身不够平整，会产生块状伪影；而均值估计则是会出现条状伪影。因此决定将这两种方法结合使用。



## 2.2 另一种方法

![picture_1](Original_Images\picture_1.PNG)



### 	2.2.1 预处理：样条插值网格化（可选）

对读入的数据进行样条插值网格化。这一步骤是可选的。经过测试，个人觉得对生成的图片质量影响不大，但是用了之后跑起来会比较慢。

直接读入数据，从原始数据提取高度数据也是可行的一种方法。



### 	2.2.2 解决轮胎弯曲

下一个问题是，如何克服轮胎弯曲的问题？

本次提出的想法是：展平图像，并且还需去除轮胎的弯曲。

获取图像的长宽。设置一些预设的参数。创建一个空的平面。

设置一个1*40，值全为0.25的滤波器。

接下来，通过对每一行循环，从图像的开头到结尾，取图像的一部分作为patch。

在这里，patch的大小是20。当然，patch的大小是要参考实际情况而定的。

![image-20230130213527005](asset\image-20230130213527005.png)

然后，将所有为0的值重新设置为patch里最大的值。

接下来，将patch里所有行堆叠在一起，取其中的最小值，生成一个一行数组。这个数组里面的值就是该patch里的最小值。

![image-20230130215404976](asset\image-20230130215404976.png)

在取最小值之后，利用上文提到的1*40滤波器对其滤波，让他变得平滑，这样是为了方便接下来的展平。

滤波器的大小不是固定的，可以根据效果调整。

![image-20230130215648634](\asset\image-20230130215648634.png)

在卷积完之后，可以看到曲线变得更加平滑。这样就意味着文字的部分被消去了。

取出该patch里的第一行，也就是该patch里要处理的行，与卷积完之后的图像叠放在一起，取最小值，可以凸显文字。

![image-20230130220015723](/asset/image-20230130220015723.png)

这里可以更加清楚的看到。

![image-20230130220050329](/asset/image-20230130220050329.png)

但是这样也会产生一个问题：卷积完之后，边缘的值会变得异常。因此需要再处理边缘值。

![image-20230130224016405](/asset/image-20230130224016405.png)

处理完之后，再将patch中的第一行与生成的值对比，将其相减，这样保留下来的就是文字了。

![image-20230130224434310](/asset/image-20230130224434310.png)

最后再将其加入上文生成的平面中。

如此循环就能生成图像。

但是生成的图像对比度不明显，原因是这样的：![image-20230130224829009](/asset/image-20230130224829009.png)

在这其中存在着诸多小于0的元素，因此要把这些元素变成0。

除此之外，因为边缘过大，所以还需要去除0.5以上的部分。把这些地方的元素钳制到0.5即可。

![image-20230130225149165](/asset/image-20230130225149165.png)

得到的效果如上图所示。



### 	2.2.3 去除起伏

观察生成的平面，发现生成的平面实际上还是有起伏的。之前所作的操作在转置之后相当于是对每一列滤波。

![image-20230130225711744](/asset/image-20230130225711744.png)

而现在要做的就是再对每一行滤波。同样的设置一个1*40的矩阵，值全部为0.025，对其卷积。

这是卷积之前的图像（部分）。

![image-20230130232332716](/asset/image-20230130232332716.png)

这是卷积之后的图像（部分）。

![image-20230130232428616](/asset/image-20230130232428616.png)

![image-20230130232449109](/asset/image-20230130232449109.png)

可以看到蓝色线（之前），橘红色（之后）。取两者之间的最大值，将其送回。

除此之外，轮胎上存在毛刺，这也是需要去除的部分。方法是：寻找图像中的峰值（代表着毛刺），将他们提取出来，通过形态学操作使其膨胀，然后对其取反，去掉这些洞。效果如图。

![image-20230130233914877](/asset/image-20230130233914877.png)

但问题就是，这样也会影响文字主体。不过就整体而言，这样的代价是可以接受的。下一步就是进行文字检测。

![image-20230130233949493](/asset/image-20230130233949493.png)



# 三、文本检测

## 3.1 概述

文本检测的任务是定位出输入图像中的文字区域。文字检测是文字识别过程中的一个重要环节。对于简单场景，可以采用：形态学操作、MSER+NMS等算法；对于复杂场景可以采用：CTPN、SegLink、EAST、DBnet等。



## 3.2 MSER

在初期尝试的时候，尝试使用MSER+NMS算法，但是存在着将杂讯区域识别成文字区域的问题。MSER（Maximally Stable Extremal Regions，最大稳定极值区域）是一个较为流行的文字检测传统方法。

同时，一般会采用 NMS 方法（Non Maximum Suppression，非极大值抑制）来辅助去除检测到的重合区域。这一方法抑制非极大值的元素，即抑制不是最大尺寸的框，相当于去除大框中包含的小框，达到去除重复区域，找到最佳检测位置的目的。

然而，尽管MSER+NMS 尽管检测速度非常快，且能满足一定的文字识别场景。但当在复杂的自然场景中，特别是有复杂背景的，其检测效果也不尽人意，会将一些无关的因素也检测出来。因此，尝试使用其他方法来进行文字检测。



## 3.3 EAST

### 3.3.1 EAST模型简介

EAST算法的中间过程只有：全卷积网络FCN、非极大值抑制NMS。而且支持文本行、单词的多角度检测。

EAST的检测过程如下图所示：

![img](https://oscimg.oschina.net/oscnet/e0c69cf042328840c3312b25619d6fe4b76.jpg)



### 	3.3.2 EAST模型网络结构

![img](https://oscimg.oschina.net/oscnet/6c65fbe3cb38b2ee7f5e760fc59abd71e68.jpg)

EAST 模型的网络结构分为特征提取层、特征融合层、输出层三大部分。



#### 		3.3.3.1 特征提取层

基于 PVANet（一种目标检测的模型）作为网络结构的骨干，分别从 stage1，stage2，stage3，stage4  的卷积层抽取出特征图，卷积层的尺寸依次减半，但卷积核的数量依次增倍，这是一种 “金字塔特征网络”（FPN，feature pyramid  network）的思想。

通过这种方式，可抽取出不同尺度的特征图，以实现对不同尺度文本行的检测（大的 feature map 擅长检测小物体，小的 feature map 擅长检测大物体）。



#### 		3.3.3.2 特征融合层

将前面抽取的特征图按一定的规则进行合并，这里的合并规则采用了 U-net 方法，规则如下：

- 特征提取层中抽取的最后一层的特征图（f1）被最先送入 unpooling 层，将图像放大 1 倍

- 接着与前一层的特征图（f2）串起来（concatenate）

- 然后依次作卷积核大小为 1x1，3x3 的卷积

- 对 f3，f4 重复以上过程，而卷积核的个数逐层递减，依次为 128，64，32

- 最后经过 32 核，3x3 卷积后将结果输出到 “输出层”

  

#### 3.3.3.3 输出层

  最终输出以下 5 部分的信息，分别是：

- score map：检测框的置信度，1 个参数；

- text boxes：检测框的位置（x, y, w, h），4 个参数；

- text rotation angle：检测框的旋转角度，1 个参数；

- text quadrangle coordinates：任意四边形检测框的位置坐标，(x1, y1), (x2, y2), (x3, y3), (x4, y4)，8 个参数。

其中，text boxes 的位置坐标与 text quadrangle coordinates 的位置坐标看起来似乎有点重复，其实不然，这是为了解决一些扭曲变形文本行，如下图：

![img](https://oscimg.oschina.net/oscnet/916a05899248e4fa2fc4be096f45fad0537.jpg)

如果只输出 text boxes 的位置坐标和旋转角度（x, y, w, h,θ），那么预测出来的检测框就是上图的粉色框，与真实文本的位置存在误差。而输出层的最后再输出任意四边形的位置坐标，那么就可以更加准确地预测出检测框的位置（黄色框）。



## 3.4 DB

Differentiable Binarization（DB），可以在分割网络中执行二值化的过程，可以自适应的设置二值化阈值，不仅可以简化后处理，并且提高了文本检测的性能。

在本项目所使用的框架PP中，集成了DB。也一并使用了这一算法来识别作为对照。



# 四、文本识别

## 4.1 概述

对于一个文本识别模型，需要有一下要素：

1、读取输入的图像，提取图像特征，因此，需要有个卷积层用于读取图像和提取特征。

2、由于文本序列是不定长的，因此在模型中需要引入 RNN（循环神经网络），一般是使用双向 LSTM 来处理不定长序列预测的问题。

3、为了提升模型的适用性，最好不要要求对输入字符进行分割，直接可进行端到端的训练，这样可减少大量的分割标注工作，这时就要引入 CTC  模型（Connectionist temporal classification， 联接时间分类），来解决样本的分割对齐的问题。

4、最后根据一定的规则，对模型输出结果进行纠正处理，输出正确结果。



## 4.2 CRNN

CRNN（Convolutional Recurrent Neural Network，卷积循环神经网络），是华中科技大学在发表的论文《An  End-to-End Trainable Neural Network for Image-based Sequence Recognition and ItsApplication to Scene Text  Recognition》提出的一个识别文本的方法，该模型主要用于解决基于图像的序列识别问题，特别是场景文字识别问题。



### 4.1.1 CRNN基本结构

![img](https://oscimg.oschina.net/oscnet/bbae80a01e6a26bd946bdd819fd24fe5440.jpg)

CRNN 模型主要由以下三部分组成：

（1）卷积层：从输入图像中提取出特征序列；

（2）循环层：预测从卷积层获取的特征序列的标签分布；

（3）转录层：把从循环层获取的标签分布通过去重、整合等操作转换成最终的识别结果。



### 4.2.2 卷积层

#### 	4.2.2.1 预处理

CRNN 对输入图像先做了缩放处理，把所有输入图像缩放到相同高度，默认是 32，宽度可任意长。



#### 	4.2.2.2 卷积运算

由标准的 CNN 模型中的卷积层和最大池化层组成，结构类似于 VGG，如下图：

![img](https://oscimg.oschina.net/oscnet/21ea89cc57c0040b6b2c8b3afbe358f84eb.jpg)

卷积层是由一系列的卷积、最大池化、批量归一化等操作组成的。



#### 	4.2.2.3 提取序列特征

提取的特征序列中的向量是在特征图上从左到右按照顺序生成的，用于作为循环层的输入，每个特征向量表示了图像上一定宽度上的特征，默认的宽度是 1，也就是单个像素。由于 CRNN 已将输入图像缩放到同样高度了，因此只需按照一定的宽度提取特征即可。如下图所示：

![img](https://oscimg.oschina.net/oscnet/6216a1de5bcb3bb8d2182c56dd7c33e7a8e.jpg)



### 4.2.3 循环层

循环层由一个双向 LSTM 循环神经网络构成，预测特征序列中的每一个特征向量的标签分布。

由于 LSTM 需要有个时间维度，在本模型中把序列的 width 当作 LSTM 的时间 time steps。

其中，“Map-to-Sequence” 自定义网络层主要是做循环层误差反馈，与特征序列的转换，作为卷积层和循环层之间连接的桥梁，从而将误差从循环层反馈到卷积层。



### 4.2.4 转录层

转录层是将 LSTM 网络预测的特征序列的结果进行整合，转换为最终输出的结果。

在 CRNN 模型中双向 LSTM 网络层的最后连接上一个 CTC  模型，从而做到了端对端的识别。所谓 CTC 模型（Connectionist Temporal  Classification，联接时间分类），主要用于解决输入数据与给定标签的对齐问题，可用于执行端到端的训练，输出不定长的序列结果。

由于输入的自然场景的文字图像，由于字符间隔、图像变形等问题，导致同个文字有不同的表现形式，但实际上都是同一个词，如下图：

![img](https://oscimg.oschina.net/oscnet/1efa6a0d2e74092807d8d72c8c1b812c1ab.jpg)

而引入 CTC 就是主要解决这个问题，通过 CTC 模型训练后，对结果中去掉间隔字符、去掉重复字符（如果同个字符连续出现，则表示只有 1 个字符，如果中间有间隔字符，则表示该字符出现多次），如下图所示：

![img](https://oscimg.oschina.net/oscnet/f1edb15eb6bbb1b802199cb764104ab805b.jpg)



## 4.3 SVTR_LCNet

SVTR_LCNet 是一种新设计的轻量级文本识别网络，它结合了基于Transformer的算法 SVTR (Du 等人 2022)  和基于卷积的算法 PP-LCNet (Cui 等人 2021)，被用作PP-OCRv2 识别器的骨干，以结合它们在准确性和速度方面的优势。

SVTR_Tiny 网络结构如下所示：![svtr_tiny](/asset/svtr_tiny.png)

在本项目所使用的框架PP中，集成了SVTR_LCNet。也一并使用了这一算法来识别作为对照。



# 五、训练与测试

## 5.1 数据标注

利用PaddleOCR Label标注数据。由于轮胎中存在着多样字体等问题，标注数据采取手工方式进行。

![image-20230130234900590](/asset/image-20230130234900590.png)



## 5.2 训练

Paddle提供了对于图像的增强方式，包括Resize, CenterCrop, ColorJitter, RandomHorizontalFlip, RandomRotation, Transpose, Normalize等。除此之外，我们还定义了Random Cutout。利用这些方法，我们生成了训练集。

训练使用的是PaddleOCR已经预训练的模型。经过测试，可以顺利在3080TI GPU上训练的只有ch_PP-OCRv3_det_distill_train的student模型。因为只有英文出现，所以使用的是英文字典。训练参数如下：

```python
epoch_num: 5000

输出层: 96

优化器：Adam，默认参数

学习率: 0.0001

防过拟合：L2正则化
```



## 5.3 测试

进行5次随机挑选图片进行测试，得到的结果如下：

| 评价指标 | 平均值 | 最大值 | 最小值 |
| -------- | ------ | ------ | ------ |
| 识别率   | 86.7%  | 95.5%  | 81.8%  |
| 准确率   | 70.12% | 91.0%  | 43.3%  |
| 召回率   | 75.3%  | 96.0%  | 33.3%  |

### 	5.3.1 超长句识别困难

![image-20230131002104733](/asset/image-20230131002104733.png)

在测试中发现，对于超长句的识别存在困难。



### 	5.3.2 准确度低

![image-20230131002406619](/asset/image-20230131002406619.png)

在测试中发现，对于 7与/，O与0，1与I的区分存在困难。即使是更换了自己训练的模型之后仍然存在这个问题。



# 六、总结与反思

## 6.1 点云转化图像

目前的效果总体还不错。可以基本上提取出字符用来识别。

但是只考虑了高度，得到的结果还是不够精确。



## 6.2 文本检测

通过EAST、DB算法，可以自动化的标识出绝大部分的句子了。但是轮胎上都或多或少带有花样字体，而无论是哪种算法都无法准确检测出这部分字体。



## 6.3 文本识别

CRNN对于大部分的文字有着良好的识别能力。但是面对部分字体会认错字，比如7与/这种。除此之外，对于长句的识别效果也不好。
