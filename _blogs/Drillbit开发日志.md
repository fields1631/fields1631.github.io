---
title: "Drillbit开发日志"
collection: blogs
type: "blogs"
permalink: /blogs/Drillbit开发日志
date: 2021-03-06
---

## Drillbit开发日志

### 目的

仿造hivemall，使用drill开发机器学习的库。

函数的输入应该是一行数据，输出是一个模型。输入的数据可以是一个矩阵等，还有标签。

### Hivemall分析

推荐系统fm（factorized matrix）是输入一行用户信息，输出一行推荐表的。logistic regression也是输入一行数据，即各个特征，然后输出一行特征的。

PredictionModel: 储存了一堆权重。根据具体model的不同，对权重的使用也不同。这样的model是最基本的存储方式，根据sparse或dense的PredictionModel，存在不同的方式储存weight，如hashtable和数组。因此在drillbit中需要不同形式的model的proto来进行序列化。在返回时就返回了序列化的结果。在使用这个model来预测时就反序列化重建模型。尽管相比hivemall不需要序列化有额外的开销，但protobuf应该能够尽量减小。

FeatureValue：主要是hivemall规定了index:value的字符串来表示，所以FeatureValue会有解析这样的字符串的功能。

hivemall.GeneralLearnerBaseUDTF: 完成学习任务。每次接收样本时，hive的UDTF规定调用process方法，而GeneralLearnerBaseUDTF在process方法中调用recordTrainSampleToTempFile方法，暂存接收到的样本。在UDTF中规定在接收所有行后调用close方法，而在close方法中，GeneralLearnerBaseUDTF训练第二轮直到规定的迭代次数。对应到drill的udf中，可以使用DrillAggFunction，process方法对应add方法，close方法对应output方法。因此我们的策略应该是：第一轮遍历，在add方法中暂存输入，遍历完成后调用output方法，在output中完成第二轮以及之后的训练。为了能使用Agg，我们需要一个mask，即在表后添加一列相同的数据，作为GROUP BY的类别。这个mask可以是使用SQL编写的函数，不必是udf，因为想象上不太可能。

### drillbit的类设计

#### 模型储存与序列化

drillbit.model.Model: interface，含有设置权重、读取权重的方法，这两个方法调用AbstractModelWeight的设置读取权重的方法。读取、设置模型信息，如是否dense的方法。含有根据byte[] 来重建模型的方法（Builder），这个方法调用AbstractModelWeight的反序列化的方法。含有将模型序列化成byte[]的方法，这个方法调用AbstractModelWeight的序列化的方法。

drillbit.model.AbstractModel: abstract class，实现drillbit.model.Model，根据上面的思路实现Model中的方法。

drillbit.model.Weight: interface，含有权重设置的方法，比如根据特征来设置某一个权重。并且能够从byte[]重建权重。并且根据是Dense还是Sparse实现drillbit.model.AbstractWeight实现Weight中的方法。

drillbit.model.Weighst: protobuf编译产生的类，可以储存权重，序列化以及反序列化。目前有两种weights：dense和sparse。其写入都是使用了builder，序列化时需要用CodecOutputStream来接收结果，使用CodecOutputStream的方法如"http://itindex.net/detail/30996-序列化-代码-测试中"所示.基本上是通过声明一个byte[] result，其大小为getSerializeSize()计算得到，然后使用result构建CodecOutputStream，写入后即可得到byte[]。而在反序列化时直接使用这个byte[]来得到Weights，然后使用getHashMap之类的方法就行了。

drillbit.model.ExtendedWeight: abstract class extends AbstractWeight, 实现了。covar、sum_of_squared_gradients等都是属于扩展后的weight，因此将他们全部归为ExtendedWeight中。

使用一个weightFactory来产生weight。

#### 回归模型

训练、更新是和具体模型存储分离的，并可仿照hivemall。

drillbit.AggFuncWithCLI: abstract class，继承AggFunc，添加命令行参数，用于指定训练的学习率、损失函数等超参数。

drillbit.LearnerAggFunc: abstract class, 继承了drillbit.AggFuncWithCLI