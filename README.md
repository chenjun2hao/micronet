# Model-Compression-Deploy

*"目前在深度学习领域分类两个派别，一派为学院派，研究强大、复杂的模型网络和实验方法，为了追求更高的性能；另一派为工程派，旨在将算法更稳定、高效的落地在硬件平台上，效率是其追求的目标。复杂的模型固然具有更好的性能，但是高额的存储空间、计算资源消耗是使其难以有效的应用在各硬件平台上的重要原因。所以，深度神经网络日益增长的规模为深度学习在移动端的部署带来了巨大的挑战，深度学习模型压缩与部署成为了学术界和工业界都重点关注的研究领域之一"*


## 项目简介

**模型压缩与部署**

### 压缩

- 量化：qat，任意位数(16/8/4/2-bit)、三值/二值；ptq，8-bit(tensorrt)
- 剪枝：正常、规整、分组卷积结构剪枝
- 针对特征(A)二值量化的BN融合
- 任意位数(any-bit)量化的BN融合

### 部署

- tensorrt

## 目前提供

- 普通卷积和分组卷积结构
- 权重W和特征A的训练中/后量化。训练中量化，W(32/16/8/4/2-bit, 三/二值) 和 A(32/16/8/4/2-bit, 二值)任意组合；训练后量化，采用tensorrt，支持8-bit
- 针对三/二值的一些tricks：W二值/三值缩放因子，W/grad（ste、saturate_ste、soft_ste）截断，A二值时采用B-A-C-P可比C-B-A-P获得更高acc等
- 多种剪枝方式：正常、规整（比如model可剪枝为每层剩余filter个数为N(8,16等)的倍数）、分组卷积结构（剪枝后仍保证分组卷积结构）的通道剪枝
- batch normalization融合及融合前后model对比测试：非量化普通BN融合（训练后，BN层参数 —> conv的权重w和偏置b）、针对特征(A)二值量化的BN融合（训练量化后，BN层参数 —> conv的偏置b)、任意位数(bits)量化的BN融合（训练量化中，先融合再量化）
- tensorrt：fp32/fp16/int8(ptq-calibration)、op-adapt(upsample)、dynamic_shape等

## 代码结构

![code_structure](https://github.com/666DZY666/Model-Compression-Deploy/blob/master/readme_imgs/code_structure.jpg)


## 项目进展
- **2019.12.4**，初次提交
- **12.8**，任意位数特征(A)量化前先进行缩放(* 0.1)，然后再截断，以减小截断误差
- **12.11**，增加项目代码结构图
- 12.12，完善使用示例
- 12.14，增加:1、BN融合量化情况(W三值/二值)可选，即训练量化时选择W三/二值，这里则对应选择；2、BN融合时对卷积核(conv)不带偏置(bias)的处理
- **12.17**，增加模型压缩前后数据对比(示例)
- 12.20，增加设备可选(cpu、gpu(单卡、多卡))
- **12.27**，补充相关论文
- 12.29，取消任意位数量化8bits以内的限制，即现在可以量化至10bits、16bits等
- **2020.2.17**，1、精简W三值/二值量化代码；2、加速W三值量化训练
- **2.18**，优化针对特征(A)二值的BN融合:去除对BN层gamma参数的限制，即现在此情况下融合时BN可正常训练
- **2.24**，再次优化三/二值量化代码组织结构，增强可移植性，旧版确实不太好移植。目前移植方法：将想要量化的卷积用./WbWtAb/models/util_w_t_b_conv.py中的Conv2d_Q替换即可，可参照该路径下nin_gc.py中的使用方法
- **3.1**，新增：1、google任意位数(bits)量化方法；2、任意位数量化的BN融合
- **3.2、3.3**，规整量化代码整体结构，目前所有量化方法都可采取类似的移植方式：将想要量化的卷积(或全连接，目前dorefa中有量化全连接，其他方法类似可写)用models/util_xx.py中的Conv2d_Q替换即可，可分别参照该路径下nin_gc.py中的使用方法进行移植（分类、检测等均适用，但需要据实际情况具体调试）
- **3.4**，规整优化WbWtAb/bn_folding中“针对特征(A)二值的BN融合”的相关实现代码，可进行BN融合及融合前后模型对比测试(精度/速度/(大小))
- 3.11，调整WqAq/IAO中的BN层momentum参数(0.1 —> 0.01),削弱batch统计参数占比,一定程度抑制量化带来的抖动。经实验,量化训练更稳定,acc提升1%左右
- **3.13**，更新代码结构图
- 4.6，修正二值量化训练中W_clip的相关问题(之前由于这个，导致二值量化训练精度上不去，现在已可正常使用)(同时修正无法找到一些模块如models/util_xx.py的问题)
- **12.14**，1、improve code structure；2、add deploy-tensorrt(main module, but not running yet)

## 环境要求

- python >= 3.5
- torch >= 1.1.0
- torchvison >= 0.3.0
- numpy
- onnx == 1.6.0
- tensorrt == 7.0.0.11


## 使用

```
git clone https://github.com/666DZY666/Model-Compression-Deploy.git
```

```
cd Model-Compression-Deploy
```

### 压缩

#### 量化

##### W（FP32/三/二值）、A（FP32/二值）

--W --A, 权重W和特征A量化取值

```
cd compression/quantization/WbWtAb
```

- WbAb

```
python main.py --W 2 --A 2
```

- WbA32

```
python main.py --W 2 --A 32
```

- WtAb

```
python main.py --W 3 --A 2
```

- WtA32

```
python main.py --W 3 --A 32
```

##### W（FP32/16/8/4/2 bits）、A（FP32/16/8/4/2 bits）

--Wbits --Abits, 权重W和特征A量化位数

```
cd compression/quantization/WqAq/dorefa 或 cd compression/quantization/WqAq/IAO
```

###### dorefa

- W16A16

```
python main.py --Wbits 16 --Abits 16
```

- W8A8

```
python main.py --Wbits 8 --Abits 8
```

- W4A4

```
python main.py --Wbits 4 --Abits 4
```

- 其他bits情况类比

###### IAO

*量化位数选择同上*

--q_type, 量化类型; --bn_fold, 量化中bn融合标志

- 对称量化, bn不融合

```
python main.py --q_type 0 --bn_fold 0
```

- 非对称量化, bn融合

```
python main.py --q_type 1 --bn_fold 1
```

- 其他组合情况类比
  
#### 剪枝

*稀疏训练  —>  剪枝  —>  微调*

```
cd compression/pruning
```

##### 正常训练

```
python main.py
```

##### 稀疏训练

-sr 稀疏标志, --s 稀疏率(需根据dataset、model情况具体调整)

- nin(正常卷积结构)

```
python main.py -sr --s 0.0001
```

- nin_gc(含分组卷积结构)

```
python main.py -sr --s 0.001
```

##### 剪枝

--percent 剪枝率, --normal_regular 正常、规整剪枝标志及规整剪枝基数(如设置为N,则剪枝后模型每层filter个数即为N的倍数), --model 稀疏训练后的model路径, --save 剪枝后保存的model路径（路径默认已给出, 可据实际情况更改）

- 正常剪枝

```
python normal_regular_prune.py --percent 0.5 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 规整剪枝

```
python normal_regular_prune.py --percent 0.5 --normal_regular 8 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

或

```
python normal_regular_prune.py --percent 0.5 --normal_regular 16 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 分组卷积结构剪枝

```
python gc_prune.py --percent 0.4 --model models_save/nin_gc_preprune.pth
```

##### 微调

--refine 剪枝后的model路径（在其基础上做微调）

```
python main.py --refine models_save/nin_prune.pth
```

#### 剪枝 —> 量化（注意剪枝率和量化率平衡）

*剪枝完成后，加载保存的模型参数在其基础上再做量化*

##### 剪枝 —> 量化（16/8/4/2 bits）（剪枝率偏大、量化率偏小）

```
cd compression/quantization/WqAq/dorefa 或 cd compression/quantization/WqAq/IAO
```

###### W8A8

- nin(正常卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../../../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../../../prune/models_save/nin_gc_refine.pth
```

###### 其他bits情况类比

##### 剪枝 —> 量化（三/二值）（剪枝率偏小、量化率偏大）

```
cd compression/quantization/WbWtAb
```

###### WbAb

- nin(正常卷积结构)

```
python main.py --W 2 --A 2 --refine ../../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --W 2 --A 2 --refine ../../prune/models_save/nin_gc_refine.pth
```

###### 其他取值情况类比

#### BN融合

```
cd compression/quantization/WbWtAb/bn_folding
```

--W 权重W量化取值(据量化训练时W量化取值(FP32/三值/二值)情况对应选择)

##### 融合并保存融合前后model

- Wb
  
```
python bn_folding.py --W 2
```

- Wt

```
python bn_folding.py --W 3
```

##### 融合前后model对比测试

```
python bn_folding_model_test.py
```

#### 设备选取

*现支持cpu、gpu(单卡、多卡)*

--cpu 使用cpu，--gpu_id 使用并选择gpu

- cpu

```
python main.py --cpu
```

- gpu单卡

```
python main.py --gpu_id 0
```

或

```
python main.py --gpu_id 1
```

- gpu多卡

```
python main.py --gpu_id 0,1
```

或

```
python main.py --gpu_id 0,1,2
```

*默认：使用服务器全卡*


### 部署

#### TensorRT

*目前仅提供相关**核心模块**代码，后续再加入完整可运行demo*

##### 相关解读
- [tensorrt-基础](https://zhuanlan.zhihu.com/p/336256668)
- [tensorrt-op/dynamic_shape](https://zhuanlan.zhihu.com/p/335829625)


## 模型压缩数据对比（仅供参考）

*以下为cifar10测试，可在更冗余模型、更大数据集上尝试其他组合压缩方式*

|            类型             |  Acc   | GFLOPs | Para(M) | Size(MB) | 压缩率 | 损失  |                        备注                        |
| :-------------------------: | :----: | :----: | :-----: | :------: | :----: | :---: | :------------------------------------------------: |
|        原模型（nin）        | 91.01% |  0.15  |  0.67   |   2.68   |  ***   |  ***  |                       全精度                       |
|  采用分组卷积结构(nin_gc)   | 90.88% |  0.15  |  0.58   |   2.32   | 13.43% | 0.13% |                       全精度                       |
|            剪枝             | 90.26% |  0.09  |  0.32   |   1.28   | 44.83% | 0.62% |                       全精度                       |
|        量化(W/A二值)        | 90.02% |  ***   |   ***   |   0.18   | 92.21% | 0.86% |                  W二值含缩放因子                   |
|      量化(W三值/A二值)      | 87.68% |  ***   |   ***   |   0.26   | 88.79% | 3.20% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
|   剪枝+量化(W三值/A二值)    | 86.13% |  ***   |   ***   |   0.19   | 91.81% | 4.75% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
| 分组+剪枝+量化(W三值/A二值) | 86.13% |  ***   |   ***   |   0.19   | 92.91% | 4.88% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |

*注：nin_gc 和 最后一行 的压缩率为与原模型（nin）相比，其余情况的压缩率均为与 nin_gc 相比*


## 相关资料

### 压缩

#### 量化

##### QAT

###### 二值

- [BinarizedNeuralNetworks: TrainingNeuralNetworkswithWeightsand ActivationsConstrainedto +1 or−1](https://arxiv.org/abs/1602.02830)

- [XNOR-Net:ImageNetClassiﬁcationUsingBinary ConvolutionalNeuralNetworks](https://arxiv.org/abs/1603.05279)

- [AN EMPIRICAL STUDY OF BINARY NEURAL NETWORKS’ OPTIMISATION](https://openreview.net/forum?id=rJfUCoR5KX)

- [A Review of Binarized Neural Networks](https://www.semanticscholar.org/paper/A-Review-of-Binarized-Neural-Networks-Simons-Lee/0332fdf00d7ff988c5b66c47afd49431eafa6cd1)

###### 三值

- [Ternary weight networks](https://arxiv.org/abs/1605.04711)

###### 任意位数

- [DoReFa-Net: Training Low Bitwidth Convolutional Neural Networks with Low Bitwidth Gradients](https://arxiv.org/abs/1606.06160)
- [Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference](https://arxiv.org/abs/1712.05877)
- [Quantizing deep convolutional networks for efficient inference: A whitepaper](https://arxiv.org/abs/1806.08342)

##### PTQ

###### tensorrt

- [ptq](https://on-demand.gputechconf.com/gtc/2017/presentation/s7310-8-bit-inference-with-tensorrt.pdf)

#### 剪枝

- [Learning Efficient Convolutional Networks through Network Slimming](https://arxiv.org/abs/1708.06519)
- [RETHINKING THE VALUE OF NETWORK PRUNING](https://arxiv.org/abs/1810.05270)

#### 适配专用芯片的模型压缩

- [Convolutional Networks for Fast, Energy-Efficient Neuromorphic Computing](https://arxiv.org/abs/1603.08270)

### 部署

#### TensorRT

- [github](https://github.com/NVIDIA/TensorRT)
- [ptq](https://on-demand.gputechconf.com/gtc/2017/presentation/s7310-8-bit-inference-with-tensorrt.pdf)
- [tensorrt-基础](https://zhuanlan.zhihu.com/p/336256668)
- [tensorrt-op/dynamic_shape](https://zhuanlan.zhihu.com/p/335829625)
- [summary](https://github.com/mileistone/study_resources/blob/master/engineering/tensorrt/tensorrt.md)


## 后续

- tensorrt完整demo
- 其他压缩算法(量化/剪枝/NAS/蒸馏等)
- 其他部署框架(mnn/tnn/tengine等)
