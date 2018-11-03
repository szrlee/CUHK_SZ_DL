# 含并行连结的网络（GoogLeNet）

在2014年的ImageNet竞赛中，一个名叫GoogLeNet的网络结构大放异彩 [1]。它虽然在名字上向LeNet致敬，但在网络结构上已经很难看到LeNet的影子。GoogLeNet吸收了NiN中网络嵌套网络的思想，并在此基础上做了很大改进。在随后的几年里，研究人员对GoogLeNet进行了数次改进，本节将介绍这个模型系列的第一个版本。


## Inception 块

GoogLeNet中的基础卷积块叫做Inception块，得名于同名电影《盗梦空间》（Inception）。与上一节介绍的NiN块相比，这个基础块在结构上更加复杂。

![Inception块的结构。](./inception.svg)

由图5.8可以看出，Inception块里有四条并行的线路。前三条线路使用窗口大小分别是$1\times 1$、$3\times 3$和$5\times 5$的卷积层来抽取不同空间尺寸下的信息。其中中间两个线路会对输入先做$1\times 1$卷积来减少输入通道数，以降低模型复杂度。第四条线路则使用$3\times 3$最大池化层，后接$1\times 1$卷积层来改变通道数。四条线路都使用了合适的填充来使得输入输出高宽一致。最后我们将每条线路的输出在通道维上连结，并输入到接下来的层中去。

Inception块中可以自定义的超参数是每个层的输出通道数，我们以此来控制模型复杂度。

```{.python .input  n=1}
import sys
sys.path.insert(0, '../../Week5/code/')

import gluonbook as gb
from mxnet import nd, init, gluon
from mxnet.gluon import nn

class Inception(nn.Block):
    # c1 - c4 为每条线路里的层的输出通道数。
    def __init__(self, c1, c2, c3, c4, **kwargs):
        super(Inception, self).__init__(**kwargs)
        # 线路 1，单 1 x 1 卷积层。
        self.p1_1 = nn.Conv2D(c1, kernel_size=1, activation='relu')
        # 线路 2，1 x 1 卷积层后接 3 x 3 卷积层。
        self.p2_1 = nn.Conv2D(c2[0], kernel_size=1, activation='relu')
        self.p2_2 = nn.Conv2D(c2[1], kernel_size=3, padding=1,
                              activation='relu')
        # 线路 3，1 x 1 卷积层后接 5 x 5 卷积层。
        self.p3_1 = nn.Conv2D(c3[0], kernel_size=1, activation='relu')
        self.p3_2 = nn.Conv2D(c3[1], kernel_size=5, padding=2,
                              activation='relu')
        # 线路 4，3 x 3 最大池化层后接 1 x 1 卷积层。
        self.p4_1 = nn.MaxPool2D(pool_size=3, strides=1, padding=1)
        self.p4_2 = nn.Conv2D(c4, kernel_size=1, activation='relu')

    def forward(self, x):
        p1 = self.p1_1(x)
        p2 = self.p2_2(self.p2_1(x))
        p3 = self.p3_2(self.p3_1(x))
        p4 = self.p4_2(self.p4_1(x))        
        return nd.concat(p1, p2, p3, p4, dim=1)  # 在通道维上连结输出。
```

```{.json .output n=1}
[
 {
  "name": "stderr",
  "output_type": "stream",
  "text": "/home/zli/anaconda3/lib/python3.6/site-packages/h5py/__init__.py:36: FutureWarning: Conversion of the second argument of issubdtype from `float` to `np.floating` is deprecated. In future, it will be treated as `np.float64 == np.dtype(float).type`.\n  from ._conv import register_converters as _register_converters\n"
 }
]
```

## GoogLeNet模型

GoogLeNet跟VGG一样，在主体卷积部分中使用五个块，每个块之间使用步幅为2的$3\times 3$最大池化层来减小输出高宽。第一块使用一个64通道的$7\times 7$卷积层。

```{.python .input  n=2}
b1 = nn.Sequential()
b1.add(nn.Conv2D(64, kernel_size=7, strides=2, padding=3, activation='relu'),
       nn.MaxPool2D(pool_size=3, strides=2, padding=1))
```

第二块使用两个卷积层：首先是64通道的$1\times 1$卷积层，然后是将通道增大3倍的$3\times 3$卷积层。它对应Inception块中的第二条线路。

```{.python .input  n=3}
b2 = nn.Sequential()
b2.add(nn.Conv2D(64, kernel_size=1),
       nn.Conv2D(192, kernel_size=3, padding=1),
       nn.MaxPool2D(pool_size=3, strides=2, padding=1))
```

第三块串联两个完整的Inception块。第一个Inception块的输出通道数为$64+128+32+32=256$，其中四条线路的输出通道数比例为$64:128:32:32=2：4：1：1$。其中第二、第三条线路先分别将输入通道数减小至$96/192=1/2$和$16/192=1/12$后，再接上第二层卷积层。第二个Inception块输出通道数增至$128+192+96+64=480$，每条线路的输出通道数之比为$128:192:96:64 = 4：6：3：2$。其中第二、第三条线路先分别将输入通道数减小至$128/256=1/2$和$32/256=1/8$。

```{.python .input  n=6}
b3 = nn.Sequential()
b3.add(Inception(64, (96, 128), (16, 32), 32),
       Inception(128, (128, 192), (32, 96), 64),
       nn.MaxPool2D(pool_size=3, strides=2, padding=1))
```

第四块更加复杂。它串联了五个Inception块，其输出通道数分别是$192+208+48+64=512$、$160+224+64+64=512$、$128+256+64+64=512$、$112+288+64+64=528$和$256+320+128+128=832$。这些线路的通道数分配和第三块中的类似：含$3\times 3$卷积层的第二条线路输出最多通道，其次是仅含$1\times 1$卷积层的第一条线路，之后是含$5\times 5$卷积层的第三条线路和含$3\times 3$最大池化层的第四条线路。其中第二、第三条线路都会先按比例减小通道数。这些比例在各个Inception块中都略有不同。

```{.python .input  n=7}
b4 = nn.Sequential()
b4.add(Inception(192, (96, 208), (16, 48), 64),
       Inception(160, (112, 224), (24, 64), 64),
       Inception(128, (128, 256), (24, 64), 64),
       Inception(112, (144, 288), (32, 64), 64),
       Inception(256, (160, 320), (32, 128), 128),
       nn.MaxPool2D(pool_size=3, strides=2, padding=1))
```

第五块有输出通道数为$256+320+128+128=832$和$384+384+128+128=1024$的两个Inception块。其中每条线路的通道数分配思路和第三、第四块中的一致，只是在具体数值上有所不同。需要注意的是，第五块的后面紧跟输出层，该块同NiN一样使用全局平均池化层来将每个通道的高和宽变成1。最后我们将输出变成二维数组后接上一个输出个数为标签类数的全连接层。

```{.python .input  n=8}
b5 = nn.Sequential()
b5.add(Inception(256, (160, 320), (32, 128), 128),
       Inception(384, (192, 384), (48, 128), 128),
       nn.GlobalAvgPool2D())

net = nn.Sequential()
net.add(b1, b2, b3, b4, b5, nn.Dense(10))
```

GoogLeNet模型的计算复杂，而且不如VGG那样便于修改通道数。本节里我们将输入高宽从224降到96来简化计算。下面演示各个块之间的输出的形状变化。

```{.python .input  n=9}
X = nd.random.uniform(shape=(1, 1, 96, 96))
net.initialize()
for layer in net:
    X = layer(X)
    print(layer.name, 'output shape:\t', X.shape)
```

```{.json .output n=9}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "sequential0 output shape:\t (1, 64, 24, 24)\nsequential1 output shape:\t (1, 192, 12, 12)\nsequential4 output shape:\t (1, 480, 6, 6)\nsequential5 output shape:\t (1, 832, 3, 3)\nsequential6 output shape:\t (1, 1024, 1, 1)\ndense0 output shape:\t (1, 10)\n"
 }
]
```

## 获取数据并训练

我们使用高和宽均为96像素的图像来训练GoogLeNet模型。

```{.python .input  n=10}
import mxnet as mx
ctx = mx.gpu(3)

lr, num_epochs, batch_size = 0.1, 5, 128
net.initialize(force_reinit=True, ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': lr})
train_iter, test_iter = gb.load_data_fashion_mnist(batch_size, resize=96)
gb.train_ch5(net, train_iter, test_iter, batch_size, trainer, ctx, num_epochs)
```

```{.json .output n=10}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "training on gpu(3)\nepoch 1, loss 1.7636, train acc 0.354, test acc 0.738, time 69.7 sec\nepoch 2, loss 0.5943, train acc 0.775, test acc 0.830, time 62.2 sec\nepoch 3, loss 0.4664, train acc 0.827, test acc 0.863, time 62.1 sec\nepoch 4, loss 0.3664, train acc 0.862, test acc 0.881, time 63.9 sec\nepoch 5, loss 0.3291, train acc 0.876, test acc 0.889, time 61.5 sec\n"
 }
]
```

## 小结

* Inception块相当于一个有四条线路的子网络。它通过不同窗口大小的卷积层和最大池化层来并行抽取信息，并使用$1\times 1$卷积层减少通道数以减小模型复杂度。

* GoogLeNet将多个设计精细的Inception块和其他层串联起来。其中Inception块的通道数分配之比是在ImageNet数据集上通过大量的实验得来的。

* GoogLeNet和它的后继者们一度是ImageNet上最高效的模型之一：它们在同样的测试精度下计算复杂度往往更低。

## 练习

* GoogLeNet有数个后续版本。尝试实现并运行它们，观察实验结果。这些后续版本包括加入批量归一化层（后面章节将介绍）[2]、对Inception块做调整 [3] 和加入残差连接（后面章节将介绍）[4]。

* 对比AlexNet、VGG和NiN、GoogLeNet的模型参数尺寸。为什么后两个网络可以显著减小模型？


## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1662)

![](../img/qr_googlenet-gluon.svg)

## 参考文献

[1] Szegedy, C., Liu, W., Jia, Y., Sermanet, P., Reed, S., & Anguelov, D. & Rabinovich, A.(2015). Going deeper with convolutions. In Proceedings of the IEEE conference on computer vision and pattern recognition (pp. 1-9).

[2] Ioffe, S., & Szegedy, C. (2015). Batch normalization: Accelerating deep network training by reducing internal covariate shift. arXiv preprint arXiv:1502.03167.

[3] Szegedy, C., Vanhoucke, V., Ioffe, S., Shlens, J., & Wojna, Z. (2016). Rethinking the inception architecture for computer vision. In Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (pp. 2818-2826).

[4] Szegedy, C., Ioffe, S., Vanhoucke, V., & Alemi, A. A. (2017, February). Inception-v4, inception-resnet and the impact of residual connections on learning. In Proceedings of the AAAI Conference on Artificial Intelligence (Vol. 4, p. 12).
