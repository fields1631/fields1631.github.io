---
title: "Drillbit开发日志"
collection: blogs
type: "blogs"
permalink: /blogs/Drillbit开发日志
date: 2021-03-07
---

本文介绍drillbit系统的一些开发细节。

- 仿造hivemall，使用drill开发机器学习的库。
- 函数的输入应该是一行数据，输出是一个模型。输入的数据可以是一个矩阵等，还有标签。
- 首先推荐系统fm（factorized matrix）是输入一行用户信息，输出一行推荐表的。logistic regression也是输入一行数据，即各个特征，然后输出一行特征的。
- PredictionModel: 储存了一堆权重。根据具体model的不同，对权重的使用也不同。这样的model是最基本的存储方式，根据sparse或dense的PredictionModel，存在不同的方式储存weight，如hashtable和数组。因此在drillbit中需要不同形式的model的proto来进行序列化。在返回时就返回了序列化的结果。在使用这个model来预测时就反序列化重建模型。尽管相比hivemall不需要序列化有额外的开销，但protobuf应该能够尽量减小。
- FeatureValue：主要是hivemall规定了index:value的字符串来表示，所以FeatureValue会有解析这样的字符串的功能。
- hivemall.GeneralLearnerBaseUDTF: 完成学习任务。每次接收样本时，hive的UDTF规定调用process方法，而GeneralLearnerBaseUDTF在process方法中调用recordTrainSampleToTempFile方法，暂存接收到的样本。在UDTF中规定在接收所有行后调用close方法，而在close方法中，GeneralLearnerBaseUDTF训练第二轮直到规定的迭代次数。对应到drill的udf中，可以使用DrillAggFunction，process方法对应add方法，close方法对应output方法。因此我们的策略应该是：第一轮遍历，在add方法中暂存输入，遍历完成后调用output方法，在output中完成第二轮以及之后的训练。为了能使用Agg，我们需要一个mask，即在表后添加一列相同的数据，作为GROUP BY的类别。这个mask可以是使用SQL编写的函数，不必是udf，因为想象上不太可能。
- drillbit.model.Model: interface，含有设置权重、读取权重的方法，这两个方法调用AbstractModelWeight的设置读取权重的方法。读取、设置模型信息，如是否dense的方法。含有根据byte[] 来重建模型的方法（Builder），这个方法调用AbstractModelWeight的反序列化的方法。含有将模型序列化成byte[]的方法，这个方法调用AbstractModelWeight的序列化的方法。
- drillbit.model.AbstractModel: abstract class，实现drillbit.model.Model，根据上面的思路实现Model中的方法。
- drillbit.model.Weight: interface，含有权重设置的方法，比如根据特征来设置某一个权重。并且能够从byte[]重建权重。并且根据是Dense还是Sparse实现drillbit.model.AbstractWeight实现Weight中的方法。
- drillbit.model.Weighst: protobuf编译产生的类，可以储存权重，序列化以及反序列化。目前有两种weights：dense和sparse。其写入都是使用了builder，序列化时需要用CodecOutputStream来接收结果，使用CodecOutputStream的方法如"http://itindex.net/detail/30996-序列化-代码-测试中"所示.基本上是通过声明一个byte[] result，其大小为getSerializeSize()计算得到，然后使用result构建CodecOutputStream，写入后即可得到byte[]。而在反序列化时直接使用这个byte[]来得到Weights，然后使用getHashMap之类的方法就行了。
- drillbit.model.ExtendedWeight: abstract class extends AbstractWeight, 实现了。covar、sum_of_squared_gradients等都是属于扩展后的weight，因此将他们全部归为ExtendedWeight中。
- 使用一个weightFactory来产生weight。
- 具体回归模型：训练、更新是和具体模型存储分离的，并可仿照hivemall。
- drillbit.Learner: interface 含有指定命令行参数的Learner, 能够指定命令行参数（parseOptions）、处理命令行参数（processOptions）、创建模型（createModel）等。含有setup、add、output、reset四个AggFunc的方法。在实现udf时调用这几个方法。
- drillbit.GeneralLearner: abstract class, 实现上述函数体。
- 回归模型: drillbit.regression.GeneralRegressor：继承GeneralLearner，实现setup、add、output、reset。在add时要暂存数据、在output时要进行迭代的训练。
- 模型输入：每一行是一个输入，一行有两列，第一个元素是特征的集合，为字符串的list，每个字符串都被drillbit.model.FeatureValue解析，格式为“特征:数值”或“特征#类别”。第二个元素是输出，为double、int、float、string中的某一种。第三个输入是命令行参数，是一个constant的参数。针对第二个元素的不同，我们需要分辨数值输出和类别输出（value or category?），因此predict时存在不同。如果是float、double，可以认为是一个value的输出，可以在regressor中使用等，因此predict中是一个线性组合的形式。如果是int、string，则认为是一个category的输出，需要用一个labelMap来记录所有的label，predict时是对所有label进行打分，取最高的label作为输出，同时add的行为有所不同：需要记录所有的label后才可以开始训练（在output函数中）。
- Drill aggfunc说明：setup里面无法预先得知任何参数，包括需要指定的命令行参数，因此需要在add时解析命令行参数。参数的顺序是按照在class中声明的顺序来的，第一个声明的是第一个参数，第二个声明的是第二个参数，如此递推。如果有Null，一定要声明成NullableVarCharHolder。如果是指定命令行参数，则在@Param上要变成@Param(constant = true)，因为命令行参数对所有row是一样的。为了能够指定和不指定命令行参数，需要两个一样的class，@FunctionTemplate中的name一样，而仅仅缺少命令行参数的区别。
- 如何在udf中使用变长参数：”https://issues.apache.org/jira/browse/DRILL-7337”。将Param声明为NullableVarCharHolder[]. 调用时select func(array(...))即可
- feature values的序列化与反序列化：使用protobuf
- TODO: open hash table的性能与java的hashmap性能比较。采用哪个比较好？concurrent hash table
- workspace中的变量需要是一个holder，所以需要一个静态方法，可能不能在workspace里面使用learner。答案：使用ObjectHolder来规避这个问题。
- 在处理text feature时，我们使用feature hashing将text映射为一个整数。
- 最新的Learner设计： BaseLearner为基类，指定minibatch、optimizer。不指定模型形式、是否迭代训练（iters）、是否检查收敛（cvChk、cvState等）、模型dense和dims。GeneralLearner为BaseLearner子类，完成了迭代训练、检查收敛等功能。其他BaseLearner子类根据需要指定模型形式，可以是Model，也可以是Decision Tree，决策树不需要dense和dims，也不需要收敛检查等。这样将Decision Tree和Model统一起来了。需要迭代训练的BaseLearner子类需要完成finalizeTraining，可以在里面调用runIterativelyTraining。finalizeTraining需要重写。serializeModel则根据具体的模型内容来序列化并恢复。
- 序列化一共有两层。第一层是Decision Tree或者Model，Model又分为Dense和Sparse。它们是储存权重的。第二层是Learner附带的其他信息，比如模型大小和dense等等等。
- 关于多类别标签的情况，应该建立一个标签的集合，这样可以指定use_label，也可以直接使用label。应该是统一的，不需要这么复杂的东西。两个array，一个是label，另一个是coordinates。
- 做一个小实验：在setup中产生随机数，在add中输出随机数，查看随机数是否相同
- 关于并行，使用synchorized来做。
