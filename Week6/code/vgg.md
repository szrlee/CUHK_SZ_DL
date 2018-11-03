# 使用重复元素的网络（VGG）

AlexNet在LeNet的基础上增加了三个卷积层。但AlexNet作者对它们的卷积窗口、输出通道数和构造顺序均做了大量的调整。虽然AlexNet指明了深度卷积神经网络可以取得出色的结果，但并没有提供简单的规则以指导后来的研究者如何设计新的网络。我们将在接下来数个小节里介绍几种不同的网络设计思路。

本节我们介绍VGG，它名字来源于论文作者所在实验室Visual Geometry Group [1]。VGG提出了可以通过重复使用简单的基础块来构建深度模型。

## VGG块

VGG块的组成规律是：连续使用数个相同的填充为1、窗口形状为$3\times 3$的卷积层后接上一个步幅为2、窗口形状为$2\times 2$的最大池化层。卷积层保持输入的高和宽不变，而池化层则对其减半。我们使用`vgg_block`函数来实现这个基础的VGG块，它可以指定卷积层的数量`num_convs`和输出通道数`num_channels`。

```{.python .input  n=1}
import sys
sys.path.insert(0, '../../Week5/code/')

import gluonbook as gb
from mxnet import nd, init, gluon
from mxnet.gluon import nn

def vgg_block(num_convs, num_channels):
    blk = nn.Sequential()
    for _ in range(num_convs):
        blk.add(nn.Conv2D(num_channels, kernel_size=3, 
                          padding=1, activation='relu'))
    blk.add(nn.MaxPool2D(pool_size=2, strides=2))
    return blk
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

## VGG网络

VGG网络同AlexNet和LeNet一样是由卷积层模块后接全连接层模块构成。卷积层模块串联数个`vgg_block`，其超参数由变量`conv_arch`定义。该变量指定了每个VGG块里卷积层个数和输出通道数。全连接模块则跟AlexNet中的一样。

现在我们构造一个VGG网络。它有5个卷积块，前两块使用单卷积层，而后三块使用双卷积层。第一块的输出通道是64，之后每次对输出通道数翻倍，直到变为512。因为这个网络使用了8个卷积层和3个全连接层，所以经常被称为VGG-11。

```{.python .input  n=2}
conv_arch = ((1, 64), (1, 128), (2, 256), (2, 512), (2, 512))
```

下面我们实现VGG-11。

```{.python .input  n=3}
def vgg(conv_arch):
    net = nn.Sequential()
    # 卷积层部分。
    for (num_convs, num_channels) in conv_arch:
        net.add(vgg_block(num_convs, num_channels))
    # 全连接层部分。
    net.add(nn.Dense(4096, activation='relu'), nn.Dropout(0.5),
            nn.Dense(4096, activation='relu'), nn.Dropout(0.5),
            nn.Dense(10))
    return net

net = vgg(conv_arch)
```

下面构造一个高和宽均为224像素的单通道数据样本来观察每一层的输出形状。

```{.python .input  n=4}
net.initialize()
X = nd.random.uniform(shape=(1, 1, 224, 224))
for blk in net:
    X = blk(X)
    print(blk.name, 'output shape:\t', X.shape)
```

```{.json .output n=4}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "sequential1 output shape:\t (1, 64, 112, 112)\nsequential2 output shape:\t (1, 128, 56, 56)\nsequential3 output shape:\t (1, 256, 28, 28)\nsequential4 output shape:\t (1, 512, 14, 14)\nsequential5 output shape:\t (1, 512, 7, 7)\ndense0 output shape:\t (1, 4096)\ndropout0 output shape:\t (1, 4096)\ndense1 output shape:\t (1, 4096)\ndropout1 output shape:\t (1, 4096)\ndense2 output shape:\t (1, 10)\n"
 }
]
```

可以看到每次我们将输入的长和宽减半，直到最终高宽变成7后传入全连接层。与此同时，输出通道数每次翻倍，直到变成512。因为每个卷积层的窗口大小一样，所以每层的模型参数尺寸和计算复杂度与高、宽、输入通道数和输出通道数的乘积成正比。VGG这种高宽减半和通道翻倍的设计使得多数卷积层都有相同的模型参数尺寸和计算复杂度。

## 模型训练

因为VGG-11计算上比AlexNet更加复杂，出于测试的目的我们构造一个通道数更小，或者说更窄的网络在Fashion-MNIST上训练。

```{.python .input  n=5}
ratio = 4
small_conv_arch = [(pair[0], pair[1] // ratio) for pair in conv_arch]
net = vgg(small_conv_arch)
```

除了使用了稍大些的学习率，模型训练过程跟上一节AlexNet中的类似。

```{.python .input  n=6}
import mxnet as mx
ctx = mx.gpu(1)

lr, num_epochs, batch_size = 0.05, 5, 128
net.initialize(ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': lr})
train_iter, test_iter = gb.load_data_fashion_mnist(batch_size, resize=224)
gb.train_ch5(net, train_iter, test_iter, batch_size, trainer, ctx, num_epochs)
```

```{.json .output n=6}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "training on gpu(1)\nepoch 1, loss 0.9589, train acc 0.652, test acc 0.849, time 91.8 sec\nepoch 2, loss 0.4014, train acc 0.853, test acc 0.885, time 88.1 sec\nepoch 3, loss 0.3297, train acc 0.880, test acc 0.900, time 89.5 sec\nepoch 4, loss 0.2867, train acc 0.896, test acc 0.908, time 89.7 sec\nepoch 5, loss 0.2587, train acc 0.905, test acc 0.914, time 89.6 sec\n"
 }
]
```

## 小结

* VGG-11通过5个可以重复使用的卷积块来构造网络。根据每块里卷积层个数和输出通道数的不同可以定义出不同的VGG模型。

## 练习

* VGG的计算比AlexNet慢很多，也需要很多的GPU内存。试分析原因。
* 尝试将Fashion-MNIST中图像的高和宽由224改为96。这在实验中有哪些影响？
* 参考VGG论文里的表1来构造VGG其他常用模型，例如VGG-16和VGG-19 [1]。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1277)

![](../img/qr_vgg-gluon.svg)

## 参考文献

[1] Simonyan, K., & Zisserman, A. (2014). Very deep convolutional networks for large-scale image recognition. arXiv preprint arXiv:1409.1556.
