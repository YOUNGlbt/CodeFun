既然代码贴出来了，大家又这么喜欢问，那么我就应该写点什么。几天下来，洋洋洒洒竟有几千余字。遂理之，而又恐小子之言徒惹发笑，思忖再三，终究还是落了笔。翻了下大家开的几百条[issues](https://github.com/YunYang1994/tensorflow-yolov3/issues)，其中的吐槽大致可以总结成以下三点:

- **YOLOv3 算法的前向传播过程怎么进行的，如何理解画网格？** 
- **YOLOv3 到底是怎么训练的，损失函数理解太难了，代码写得跟一坨屎一样!** 
- **为什么我在训练的时候loss出现了Nan，有什么办法解决它吗？** 

哈哈，本文的目的，就在于此。

--------------------
# 1. YOLOv3算法的前向传播过程
<p align="center">
    <img width="60%" src="https://pjreddie.com/media/image/Screen_Shot_2018-03-24_at_10.48.42_PM.png" style="max-width:80%;">
    </a>
</p>

假设我们想对上面这张 `416 X 416` 大小的图片进行预测，把图中`dog`、`bicycle`和`car`三种物体框出来，其实涉及到以下三个过程：

- [x] 怎么在图片上找出很多有价值的候选框？
- [x] 接着判断候选框里有没有物体？
- [x] 如果有物体的话，那么它属于哪个类别？

听起来就像把大象装进冰箱，分三步走。事实上，目前的 anchor 机制算法例如 RCNN、Faster rcnn 以及 YOLO 算法都是这个思想。最早的时候， [RCNN](https://arxiv.org/abs/1311.2524) 是这么干的，首先利用 Selective Search 的方法通过图片上像素之间的相似度和纹理特征进行区域合并，然后提出很多候选框并喂给 CNN 网络提取特征映射(feature map)，最后利用 feature map 训练SVM来对目标和背景进行分类.

![image](https://user-images.githubusercontent.com/30433053/62198083-944b6e00-b3b3-11e9-9cd5-a7230ced3762.png)

这是最早利用神经网络进行目标检测的开山之作，虽然现在看来有不少瑕疵，例如：<br>

- Selective Search 会在图片上提取2000个候选区域，每个候选区域都会喂给 CNN 进行特征提取，这个过程太冗余啦，其实这些候选区域之间很多特征其实是可以共享的；<br>
- 由于 CNN 最后一层是全连接层，因此输入图片的尺寸大小也有限制，只能进行 Crop 或者 Warp，这样一来图片就会扭曲、变形和失真；<br>
- 在利用 SVM 分类器对候选框进行分类的时候，每个候选框的特征向量都要保留在磁盘上，很浪费空间！<br>

**尽管如此，但仍不可否认它具有划时代的意义，至少告诉后人我们是可以利用神经网络进行目标检测的。**

后面，一些大神们在此基础上提出了很多改进，从 Fast RCNN 到 Faster RCNN 再到 Mask RCNN, 目标检测的 region proposal 过程变得越来越有针对性，并提出了著名的 RPN 网络去学习如何给出高质量的候选框，然后再去判断所属物体的类别。简单说来就是: 提出候选框 ---> 然后分类，这就是我们常说的 two-stage 算法。two-stage 算法的好处就是精度较高，但是检测速度满足不了实时性(real time)的要求。在这样的背景下，YOLO 算法应运而生。

## 1.1 不妨先给图片画网格

YOLO算法最重要的思想就是**画网格**，由于本人做过一点点关于计算流体力学(Computational Fluid Dynamics, 简称CFD)的研究，所以听到网格(grid cells)这个单词感觉特别亲切。emm，先来看看[YOLOv1论文](https://arxiv.org/abs/1506.02640)里的这张图:
<p align="center">
    <img width="70%" src="https://user-images.githubusercontent.com/30433053/62187018-97863000-b39a-11e9-84ff-d7d3166f0407.png" style="max-width:90%;">
    </a>
</p>

初学者咋一看，这特么什么东西？只想说，看不懂！事实上，网上很多关于YOLO系列算法的教程也喜欢拿这张图去忽悠。好了，我想从这张图片出发，讲一讲 YOLO 算法的**画网格**思想。在讲这个之前，我们先来了解一下什么是 feature map 和 ROI， 以及它们之间的关系。

### 1.1.1 什么是 feature map 

当我们谈及 CNN 网络，总能听到 feature map 这个词。它也叫特征映射，简单说来就是输入图像在与卷积核进行卷积操作后得到图像特征。在输入层: 如果是灰度图片，那就只有一个feature map；如果是彩色图片（RGB），一般就是3个 feature map（红绿蓝）。一般而言，CNN 网络在对图像自底向上提取特征时，feature map 的数量(其实也对应的就是卷积核的数目) 会越来越多，而空间信息会越来越少，其特征也会变得越来越抽象。比如著名的 VGG16 网络，它的 feature map 变化就是这个样子。

<p align="center">
    <img width="70%" src="https://raw.githubusercontent.com/YunYang1994/TensorFlow2.0-Examples/master/3-Neural_Network_Architecture/vgg16/docs/vgg16.png" style="max-width:70%;">
    </a>
</p>

> feature map 在空间尺寸上越来越小，但在通道尺寸上变得越来越深，这就是 VGG16 的特点。 

讲到 feature map 哦，就不得不提一下人脸识别领域里经常提到的 embedding. 它其实就是 feature map 被最后一层全连接层所提取到特征向量。深度学习鼻祖 hinton 于2006年发表于《SCIENCE 》上的一篇[论文](http://www.cs.toronto.edu/~hinton/science.pdf) 首次利用自编码网络实现了对 mnist 数据集特征的提取，得到的特征是一个2维或3维的向量。值得一提的是，也是这篇论文揭开了深度学习兴起的序幕。

<p align="center">
    <img width="50%" src="https://user-images.githubusercontent.com/30433053/62225873-d395b100-b3eb-11e9-8a3b-ac3fe9d75518.png" style="max-width:50%;">
    </a>
</p>

**下面就是上面这张图片里的数字在 CNN 空间里映射后得到的特征向量在2维和3维空间里的样子。如果你对这个过程感兴趣，可以参考这份[代码](https://github.com/YunYang1994/SphereFace)。**

| 2维空间| 3维空间 |
|---|---|
|![image](https://github.com/YunYang1994/SphereFace/blob/master/image/2D_Original_Softmax_Loss_embeddings.gif)|![weibo-logo](https://github.com/YunYang1994/SphereFace/blob/master/image/3D_Original_Softmax_Loss_embeddings.gif)|

> 每一种颜色代表一种数字，原来这些数字的图片信息是[28, 28, 1]维度的，现在经 CNN 网络特征映射后，居然得到的是一个2维或3维的特征向量，真是降维打击👊！

### 1.1.2 ROI 映射到 feature map

好了，我们现在大概知道特征映射是怎么回事了。现在需要讲讲 ROI 的概念，ROI 的全称是 Region Of Interest, 中文翻译过来叫感兴趣区域。说白了就是从图像中选择一块区域，这块区域是我们对图像分析所关注的重点， 比如我们前面提到的候选框区域，也可以认为是 ROI。**前面我们提到：CNN 网络在对图像自底向上提取特征时，得到的 feature map 一般都是在空间尺寸上越来越小，而在通道尺寸上变得越来越深。** 那么，为什么要这么做？

<p align="center">
    <img width="60%" src="https://user-images.githubusercontent.com/30433053/62198793-e4770000-b3b4-11e9-808f-d53703455def.png" style="max-width:60%;">
    </a>
</p>

其实，这就与 ROI 映射到 Feature map 有关。 在上面这幅图里：原图里的一块 ROI 在 CNN 网络空间里映射后，在 feature map 上空间尺寸会变得更小，甚至是一个点, 但是这个点的通道信息会很丰富，这些通道信息是 ROI 区域里的图片信息在 CNN 网络里映射得到的特征表示。由于图像中各个相邻像素在空间上的联系很紧密，从而在空间上造成具有很大的冗余性。**因此，我们往往会通过在空间上降维，而在通道上升维的方式来消除这种冗余性，尽量以最小的维度来获得它最本质的特征。**

![image](https://user-images.githubusercontent.com/30433053/62262236-84329d80-b44a-11e9-9c7f-cf225bed4754.png)

>原图左上角红色 ROI 经 CNN 映射后在 feature map 空间上只得到了一个点，但是这个点有85个通道。那么，ROI的维度由原来的 [32, 32, 3] 变成了现在的 85 维，这难道又不是降维打击么？👊

按照我的理解，这其实就是 CNN 网络对 ROI 进行特征提取后得到的一个85维的特征向量。这个特征向量前4个维度代表的是候选框信息，中间这个维度代表是判断有无物体的概率，后面80个维度代表的是对 80 个类别的分类概率信息。

### 1.1.3 YOLOv3 的网格思想

YOLOv3 对输入图片进行了粗、中和细网格划分，以便分别实现对大、中和小物体的预测。其实在下面这幅图里面，每一个网格对应的就是一块 ROI 区域。如果某个物体的中心刚好落在这个网格中，那么这个网格就负责预测这个物体.

>If the center of an object falls into a grid cell, that grid cell is responsible for detecting that object. 

假如输入图片的尺寸为 `416X416`, 那么得到粗、中和细网格尺寸分别为 `13X13`、`26X26`和`52X52`。这样一算，那就是在长宽尺寸上分别缩放了`32`、`16`和`8`倍，其实这些倍数正好也是这些 ROI 的尺寸大小。

| 粗网格| 中网格 | 细网格 |
|---|---|---|
|![image](https://user-images.githubusercontent.com/30433053/62338487-52cdd680-b50b-11e9-9ffe-86a42cfa9acb.png)|![image](https://user-images.githubusercontent.com/30433053/62338514-6bd68780-b50b-11e9-8085-be9fe9d84d77.png)|![image](https://user-images.githubusercontent.com/30433053/62338538-8446a200-b50b-11e9-8a73-0334271d73b7.png)|

## 1.2 Darknet-53 的网络结构

Darknet-53 有多牛逼？看看下面这张图，作者进行了比较，得出的结论是 Darknet-53 在精度上可以与最先进的分类器进行媲美，同时它的浮点运算更少，计算速度也最快。和 ReseNet-101 相比，Darknet-53 网络的速度是前者的1.5倍；虽然 ReseNet-152 和它性能相似，但是用时却是它的2倍以上。

<p align="center">
    <img width="60%" src="https://user-images.githubusercontent.com/30433053/62341417-d7bded80-b515-11e9-8f98-cd3a75e5be63.png" style="max-width:60%;">
    </a>
</p>

此外，Darknet-53 也可以实现每秒最高的测量浮点运算，这就意味着网络结构可以更好地利用 GPU，使其测量效率更高，速度也更快。

### 1.2.1 backbone 结构

Darknet-53 的主体框架如下图所示，它主要由 `Convolutional` 和 `Residual` 结构所组成。需要特别注意的是，最后三层 `Avgpool`、`Connected` 和 `softmax` layer 是用于在 `Imagenet` 数据集上作分类训练用的。当我们用 Darknet-53 层对图片提取特征时，是不会用到这三层的。

| 网络结构| [代码结构](https://github.com/YunYang1994/TensorFlow2.0-Examples/blob/master/4-Object_Detection/YOLOV3/core/backbone.py) |
|---|---|
|<img width="150%" src="https://raw.githubusercontent.com/YunYang1994/tensorflow-yolov3/master/docs/images/darknet53.png" style="max-width:150%;">|<img width="80%" src="https://user-images.githubusercontent.com/30433053/62342173-7ba89880-b518-11e9-8878-f1c38466eb39.png" style="max-width:70%;">|

>代码结构里的`downsample`参数的意思是下采样，表示 feature map 输入该层 layer 后尺寸会变小。例如在第二层 layer 的输入尺寸是 `256X256`，输出尺寸则变成了 `128X128`。

### 1.2.2 Convolutional 结构

Convolutional 结构其实很简单，就是普通的卷积层，其实没啥讲的。但是对于 `if downsample` 的情况，初学者可能觉得有点陌生， `ZeroPadding2D` 是什么层？

```python
def convolutional(input_layer, filters_shape, downsample=False, activate=True, bn=True):
    if downsample:
        input_layer = tf.keras.layers.ZeroPadding2D(((1, 0), (1, 0)))(input_layer)
        padding = 'valid'
        strides = 2
    else:
        strides = 1
        padding = 'same'

    conv = tf.keras.layers.Conv2D(filters=filters_shape[-1], kernel_size = filters_shape[0], strides=strides, padding=padding,
                                  use_bias=not bn, kernel_regularizer=tf.keras.regularizers.l2(0.0005),
                                  kernel_initializer=tf.random_normal_initializer(stddev=0.01),
                                  bias_initializer=tf.constant_initializer(0.))(input_layer)

    if bn: conv = BatchNormalization()(conv)
    if activate == True: conv = tf.nn.leaky_relu(conv, alpha=0.1)

    return conv
```
讲到 `ZeroPadding2D`层，我们得先了解它是什么，为什么有这个层。对于它的定义，[Keras](https://keras-cn.readthedocs.io/en/latest/layers/convolutional_layer/) 官方给了很好的解释:

>`keras.layers.convolutional.ZeroPadding2D(padding=(1, 1), data_format=None)`<br>
>说明: 对2D输入（如图片）的边界填充0，以控制卷积以后特征图的大小
<p align="center">
    <img width="20%" src="https://camo.githubusercontent.com/d8c0543ca968db698c9451b7c3cba9b7836e400e/68747470733a2f2f61736b2e71636c6f7564696d672e636f6d2f64726166742f313038323535352f3068753464326e6333352e676966" style="max-width:20%;">
    </a>
</p>

其实就是对图片的上下左右四个边界填充0而已，`padding=((top_pad, bottom_pad), (left_pad, right_pad))`。 很简单吧，快打开你的`ipython`试试吧！

```ipython
In [2]: x=tf.keras.layers.Input([416,416,3])                                                                                                                                                                

In [3]: tf.keras.layers.ZeroPadding2D(padding=((1,0),(1,0)))(x)                                                                                                                                             
Out[3]: <tf.Tensor 'zero_padding2d/Identity:0' shape=(None, 417, 417, 3) dtype=float32>

In [4]: tf.keras.layers.ZeroPadding2D(padding=((1,1),(1,1)))(x)                                                                                                                                             
Out[4]: <tf.Tensor 'zero_padding2d_1/Identity:0' shape=(None, 418, 418, 3) dtype=float32>
```

### 1.2.3 Residual 残差模块

残差模块最显著的特点是使用了 `short cut` 机制（**有点类似于电路中的短路机制**）来缓解在神经网络中增加深度带来的梯度消失问题，从而使得神经网络变得更容易优化。它通过恒等映射(identity mapping)的方法使得输入和输出之间建立了一条直接的关联通道，从而使得网络集中学习输入和输出之间的残差。

<p align="center">
    <img width="50%" src="https://user-images.githubusercontent.com/30433053/62363930-de1e8a80-b552-11e9-98e9-914da36e5922.png" style="max-width:50%;">
    </a>
</p>

```python
def residual_block(input_layer, input_channel, filter_num1, filter_num2):
    short_cut = input_layer
    conv = convolutional(input_layer, filters_shape=(1, 1, input_channel, filter_num1))
    conv = convolutional(conv       , filters_shape=(3, 3, filter_num1,   filter_num2))

    residual_output = short_cut + conv
    return residual_output
```

>不知道大家有没有注意，整个 `Darknet-53`  网络压根就没有使用 `Pooling` 层。

## 1.3 奇怪的 anchor 机制
讲到 `anchor` 机制，大家不会觉得有点奇怪吗？ 既然一个物体的特征很明显地摆在那里，神经网络居然还要通过一个先验候选框去学习如何找到它。这就好比当我们人第一眼看到物体的时候，居然还要拿着一把尺子去量，Is there anything more asshole than that in the world?

>所以，我更愿意相信 `anchor free` 机制。

### 1.3.1 边界框的预测
前面讲到，如果物体的中心落在了这个网格里，那么这个网格就要负责去预测它。在下面这幅图里：蓝色框代表先验框(anchor)，黑色虚线框表示的是预测框.
<p align="center">
    <img width="40%" src="https://user-images.githubusercontent.com/30433053/62366897-b8957f00-b55a-11e9-93e0-89e796c36200.png" style="max-width:40%;">
    </a>
</p>

- b_h 和 b_w 分别表示预测框的长宽，P_h 和 P_w 分别表示先验框的长和宽。
- t_x 和 t_y 表示的是物体中心距离网格左上角位置的偏移量，C_x 和 C_y 则代表网格左上角的坐标。

```python
def decode(conv_output, i=0):
    # 这里的 i=0、1 或者 2， 以分别对应三种网格尺度
    conv_shape       = tf.shape(conv_output)
    batch_size       = conv_shape[0]
    output_size      = conv_shape[1]

    conv_output = tf.reshape(conv_output, (batch_size, output_size, output_size, 3, 5 + NUM_CLASS))

    conv_raw_dxdy = conv_output[:, :, :, :, 0:2] # 中心位置的偏移量
    conv_raw_dwdh = conv_output[:, :, :, :, 2:4] # 预测框长宽的偏移量
    conv_raw_conf = conv_output[:, :, :, :, 4:5]
    conv_raw_prob = conv_output[:, :, :, :, 5: ]

    # 好了，接下来需要画网格了。其中，output_size 等于 13、26 或者 32
    y = tf.tile(tf.range(output_size, dtype=tf.int32)[:, tf.newaxis], [1, output_size])
    x = tf.tile(tf.range(output_size, dtype=tf.int32)[tf.newaxis, :], [output_size, 1])

    xy_grid = tf.concat([x[:, :, tf.newaxis], y[:, :, tf.newaxis]], axis=-1)
    xy_grid = tf.tile(xy_grid[tf.newaxis, :, :, tf.newaxis, :], [batch_size, 1, 1, 3, 1])
    xy_grid = tf.cast(xy_grid, tf.float32) # 计算网格左上角的位置
    # 根据上图公式计算预测框的中心位置
    pred_xy = (tf.sigmoid(conv_raw_dxdy) + xy_grid) * STRIDES[i] # 乘上缩放的倍数，如 8、16 和 32 倍。
    # 根据上图公式计算预测框的长和宽大小
    pred_wh = (tf.exp(conv_raw_dwdh) * ANCHORS[i]) * STRIDES[i]
    # 合并边界框的位置和长宽信息
    pred_xywh = tf.concat([pred_xy, pred_wh], axis=-1) 

    pred_conf = tf.sigmoid(conv_raw_conf) # 计算预测框里object的置信度
    pred_prob = tf.sigmoid(conv_raw_prob) # 计算预测框里object的类别概率

    return tf.concat([pred_xywh, pred_conf, pred_prob], axis=-1)
```

### 1.3.2 K-means 的作用有多大?

按照前文的思路，那么问题来了：先验框是怎么来的？对于这点，作者在 YOLOv2 论文里给出了很好的解释：

>we run k-means clustering on the training set bounding boxes to automatically find good priors.

其实就是使用 `k-means` 算法对训练集上的 boudnding box 尺度做聚类。此外，考虑到训练集上的图片尺寸不一，因此对此过程进行归一化处理。

`k-means` 聚类算法有个坑爹的地方在于，类别的个数需要人为事先指定。这就带来一个问题，先验框 `anchor` 的数目等于多少最合适？一般来说，`anchor` 的类别越多，那么 `YOLO` 算法就越能在不同尺度下与真实框进行回归，但是这样就会导致模型的复杂度更高，网络的参数量更庞大。

<p align="center">
    <img width="60%" src="https://user-images.githubusercontent.com/30433053/62404991-d222df00-b5cb-11e9-8347-f3808eb8f893.png" style="max-width:60%;">
    </a>
</p>

>We choose k = 5 as a good tradeoff between model complexity and high recall.
>If we use 9 centroids we see a much higher average IOU. This indicates that using k-means to generate our bounding box starts the model off with a better representation and makes the task easier to learn.

在上面这幅图里，作者发现 k = 5 时就能较好地实现高召回率与模型复杂度之间的平衡。由于在 `YOLOv3` 算法里一共有3种尺度预测，因此只能是3的倍数，所以最终选择了 9 个先验框。这里还有个问题需要解决，k-means 度量距离的选取很关键。距离度量如果使用标准的欧氏距离，大框框就会比小框产生更多的错误。在目标检测领域，我们度量两个边界框之间的相似度往往以 `IOU` 大小作为标准。因此，这里的度量距离也和 `IOU` 有关。**需要特别注意的是，这里的IOU计算只用到了 boudnding box 的长和宽**。在[我的代码](https://nbviewer.jupyter.org/github/YunYang1994/tensorflow-yolov3/blob/master/docs/Box-Clustering.ipynb)里，是认为两个先验框的左上角位置是相重合的。

<p align="center">
    <img width="40%" src="https://user-images.githubusercontent.com/30433053/62405048-760c8a80-b5cc-11e9-9d2c-1ba88ab4ad65.png" style="max-width:40%;">
    </a>
</p>

>如果两个边界框之间的`IOU`值越大，那么它们之间的距离就会越小。

```python
def kmeans(boxes, k, dist=np.median,seed=1):
    """
    Calculates k-means clustering with the Intersection over Union (IoU) metric.
    :param boxes: numpy array of shape (r, 2), where r is the number of rows
    :param k: number of clusters
    :param dist: distance function
    :return: numpy array of shape (k, 2)
    """
    rows = boxes.shape[0]

    distances     = np.empty((rows, k)) ## N row x N cluster
    last_clusters = np.zeros((rows,))

    np.random.seed(seed)

    # initialize the cluster centers to be k items
    clusters = boxes[np.random.choice(rows, k, replace=False)]

    while True:
        # 为每个点指定聚类的类别（如果这个点距离某类别最近，那么就指定它是这个类别)
        for icluster in range(k): # I made change to lars76's code here to make the code faster
            distances[:,icluster] = 1 - iou(clusters[icluster], boxes)

        nearest_clusters = np.argmin(distances, axis=1)
	# 如果聚类簇的中心位置基本不变了，那么迭代终止。
        if (last_clusters == nearest_clusters).all():
            break
            
        # 重新计算每个聚类簇的平均中心位置，并它作为聚类中心点
        for cluster in range(k):
            clusters[cluster] = dist(boxes[nearest_clusters == cluster], axis=0)

        last_clusters = nearest_clusters

    return clusters,nearest_clusters,distances
```

经常有人发邮件问我，到底要不要在自己的数据集上对先验框进行聚类，这个作用会有多大？我的答案是：不需要，直接默认使用 [COCO 数据集](https://github.com/YunYang1994/tensorflow-yolov3/issues/261)上得到的先验框即可。因为 YOLO 算法最本质地来说是去学习真实框与先验框之间的尺寸偏移量，即使你选的先验框再准确，也只能是网络更容易去学习而已。事实上，这对预测的精度没有什么影响，所以这个过程意义不大。我觉得作者在论文里这样写的原因在于**你总得告诉别人你的先验框是怎么来的**，并且让论文更具有学术性。

## 1.4 原来是这样预测的

### 1.4.1 准备图片
在将图片输入模型之前，需要将图片尺寸 resize 成固定的大小，如 416X416 或 608X608 。如果直接对图片进行 resize 处理，那么会使得图片扭曲变形从而降低模型的预测精度。

```python
def image_preporcess(image, target_size, gt_boxes=None):

    ih, iw    = target_size # resize 尺寸
    h,  w, _  = image.shape # 原始图片尺寸

    scale = min(iw/w, ih/h)
    nw, nh  = int(scale * w), int(scale * h) # 计算缩放后图片尺寸
    image_resized = cv2.resize(image, (nw, nh))
    # 制作一张画布，画布的尺寸就是我们想要的尺寸
    image_paded = np.full(shape=[ih, iw, 3], fill_value=128.0)
    dw, dh = (iw - nw) // 2, (ih-nh) // 2
    # 将缩放后的图片放在画布中央
    image_paded[dh:nh+dh, dw:nw+dw, :] = image_resized
    image_paded = image_paded / 255.

    if gt_boxes is None:
        return image_paded

    else:   # 训练网络时需要对 groudtruth box 进行矫正
        gt_boxes[:, [0, 2]] = gt_boxes[:, [0, 2]] * scale + dw
        gt_boxes[:, [1, 3]] = gt_boxes[:, [1, 3]] * scale + dh
        return image_paded, gt_boxes
```
| original_image (768X576)| letterbox_image (416X416)  |
|---|---|
|![image](../data/dog.jpg)|![image](https://user-images.githubusercontent.com/30433053/62408461-16c66e80-b5fc-11e9-8ae5-bb4d9963b43f.jpg)|

### 1.4.2 网络输出
下面这幅图就是 YOLOv3 网络的整体结构，在图中我们可以看到：尺寸为 416X416 的输入图片进入 Darknet-53 网络后得到了 3 个分支，这些分支在经过一系列的卷积、上采样以及合并等操作后最终得到了三个尺寸不一的 feature map，形状分别为 [13, 13, 255]、[26, 26, 255] 和 [52, 52, 255]。
![image](https://raw.githubusercontent.com/YunYang1994/tensorflow-yolov3/master/docs/images/levio.jpeg)
讲了这么多，还是不如看[代码](https://github.com/YunYang1994/TensorFlow2.0-Examples/blob/master/4-Object_Detection/YOLOV3/core/yolov3.py)来得亲切。

```python
def YOLOv3(input_layer):
    # 输入层进入 Darknet-53 网络后，得到了三个分支
    route_1, route_2, conv = backbone.darknet53(input_layer)
    # 见上图中的橘黄色模块(DBL)，一共需要进行5次卷积操作
    conv = common.convolutional(conv, (1, 1, 1024,  512))
    conv = common.convolutional(conv, (3, 3,  512, 1024))
    conv = common.convolutional(conv, (1, 1, 1024,  512))
    conv = common.convolutional(conv, (3, 3,  512, 1024))
    conv = common.convolutional(conv, (1, 1, 1024,  512))

    conv_lobj_branch = common.convolutional(conv, (3, 3, 512, 1024))
    # conv_lbbox 用于预测大尺寸物体，shape = [None, 13, 13, 255]
    conv_lbbox = common.convolutional(conv_lobj_branch, (1, 1, 1024, 3*(NUM_CLASS + 5)), activate=False, bn=False)

    conv = common.convolutional(conv, (1, 1,  512,  256))
    # 这里的 upsample 使用的是最近邻插值方法，这样的好处在于上采样过程不需要学习，从而减少了网络参数
    conv = common.upsample(conv)

    conv = tf.concat([conv, route_2], axis=-1)

    conv = common.convolutional(conv, (1, 1, 768, 256))
    conv = common.convolutional(conv, (3, 3, 256, 512))
    conv = common.convolutional(conv, (1, 1, 512, 256))
    conv = common.convolutional(conv, (3, 3, 256, 512))
    conv = common.convolutional(conv, (1, 1, 512, 256))

    conv_mobj_branch = common.convolutional(conv, (3, 3, 256, 512))
    # conv_mbbox 用于预测中等尺寸物体，shape = [None, 26, 26, 255]
    conv_mbbox = common.convolutional(conv_mobj_branch, (1, 1, 512, 3*(NUM_CLASS + 5)), activate=False, bn=False)

    conv = common.convolutional(conv, (1, 1, 256, 128))
    conv = common.upsample(conv)

    conv = tf.concat([conv, route_1], axis=-1)

    conv = common.convolutional(conv, (1, 1, 384, 128))
    conv = common.convolutional(conv, (3, 3, 128, 256))
    conv = common.convolutional(conv, (1, 1, 256, 128))
    conv = common.convolutional(conv, (3, 3, 128, 256))
    conv = common.convolutional(conv, (1, 1, 256, 128))

    conv_sobj_branch = common.convolutional(conv, (3, 3, 128, 256))
    # conv_sbbox 用于预测小尺寸物体，shape = [None, 52, 52, 255]
    conv_sbbox = common.convolutional(conv_sobj_branch, (1, 1, 256, 3*(NUM_CLASS +5)), activate=False, bn=False)

    return [conv_sbbox, conv_mbbox, conv_lbbox]
```


### 1.4.3 NMS 处理

<p align="center">
    <img width="60%" src="https://raw.githubusercontent.com/YunYang1994/tensorflow-yolov3/master/docs/images/iou.png" style="max-width:60%;">
    </a>
</p>

非极大值抑制（Non-Maximum Suppression，NMS），顾名思义就是抑制不是极大值的元素，说白了就是去除掉那些重叠率较高并且 score 评分较低的边界框。 NMS 的算法非常简单，迭代流程如下:

- 流程1: 判断边界框的数目是否大于0，如果不是则结束迭代；
- 流程2: 按照 socre 排序选出评分最大的边界框 A 并取出；
- 流程3: 计算这个边界框 A 与剩下所有边界框的 iou 并剔除那些 iou 值高于阈值的边界框，重复上述步骤；<br>

```python
# 流程1: 判断边界框的数目是否大于0
while len(cls_bboxes) > 0:
    # 流程2: 按照 socre 排序选出评分最大的边界框 A
    max_ind = np.argmax(cls_bboxes[:, 4])
    # 将边界框 A 取出并剔除
    best_bbox = cls_bboxes[max_ind]
    best_bboxes.append(best_bbox)
    cls_bboxes = np.concatenate([cls_bboxes[: max_ind], cls_bboxes[max_ind + 1:]])
	# 流程3: 计算这个边界框 A 与剩下所有边界框的 iou 并剔除那些 iou 值高于阈值的边界框
    iou = bboxes_iou(best_bbox[np.newaxis, :4], cls_bboxes[:, :4])
    weight = np.ones((len(iou),), dtype=np.float32)
    iou_mask = iou > iou_threshold
    weight[iou_mask] = 0.0
    cls_bboxes[:, 4] = cls_bboxes[:, 4] * weight
    score_mask = cls_bboxes[:, 4] > 0.
    cls_bboxes = cls_bboxes[score_mask]
```

最后所有取出来的边界框 A 就是我们想要的。不妨举个简单的例子：假如5个边界框及评分为: A: 0.9，B: 0.08，C: 0.8, D: 0.6，E: 0.5，设定的评分阈值为 0.3，计算步骤如下。

- 步骤1: 边界框的个数为5，满足迭代条件；
- 步骤2: 按照 socre 排序选出评分最大的边界框 A 并取出；
- 步骤3: 计算边界框 A 与其他 4 个边界框的 iou，假设得到的 iou 值为：B: 0.1，C: 0.7, D: 0.02, E: 0.09, 剔除边界框 C；
- 步骤4: 现在只剩下边界框 B、D、E，满足迭代条件；
- 步骤5: 按照 socre 排序选出评分最大的边界框 D 并取出；
- 步骤6: 计算边界框 D 与其他 2 个边界框的 iou，假设得到的 iou 值为：B: 0.06，E: 0.8，剔除边界框 E；
- 步骤7: 现在只剩下边界框 B，满足迭代条件；
- 步骤8: 按照 socre 排序选出评分最大的边界框 B 并取出；
- 步骤9: 此时边界框的个数为零，结束迭代。

最后我们得到了边界框 A、B、D，但其中边界框 B 的评分非常低，这表明该边界框是没有物体的，因此应当抛弃掉。在 postprocess_boxes 代码中，

```python
# # (5) discard some boxes with low scores
classes = np.argmax(pred_prob, axis=-1)
scores = pred_conf * pred_prob[np.arange(len(pred_coor)), classes]
score_mask = scores > score_threshold
```

### 2. YOLOv3 损失函数的理解

#### 2.1 边界框损失

#### 2.2 置信度损失

#### 2.3 分类损失

#### 2.4 原来是这样训练的

### 3. YOLOv3 的训练技巧

#### 3.1 权重初始化设置

#### 3.2 学习率的设置

#### 3.3 加载预训练模型

#### 3.4 其实好像也没那么难
