# what is RDD?

Resilient Distributed Dataset, the basic abstraction in Spark. Represents an immutable,partitioned collection of elements that can be operated on in parallel

这是RDD类中对RDD的简单解释

RDD是spark中核心基础抽象，代表一个可并行执行的不可变的、可分区的弹性分布式数据集

所谓“弹性”，一种简单解释是指RDD是横向多分区的，纵向当计算过程中内存不足时可刷写到磁盘等外存上，可与外存做灵活的数据交换；而另一种个人更偏向的解释是RDD是由虚拟数据结构组成，并不包含真实数据本体，RDD使用了一种“血统”的容错机制，在结构更新和丢失后可随时根据血统进行数据模型的重建。

所谓“分布式”，就是可以分布在多台机器上进行并行计算。

从空间结构上，可以理解为是一组只读的、可分区的分布式数据集合，该集合内包含了多个分区。分区就是依照特定规则，将具有相同属性的数据记录放在一起。每个分区相当于一个数据集片段


# RDD 内部结构

![](..\images\2c8745a7.png)

RDD 的内部结构图，它是一个只读、有属性的数据集。

它的属性用来描述当前数据集的状态，数据集由数据的分区（partition）组成，并由（block）映射成真实数据。

RDD 的主要属性可以分为 3 类：

1. 与其他 RDD 的关系（parents、dependencies）；
2. 数据(partitioner、checkpoint、storage level、iterator 等)；
3. RDD 自身属性(sparkcontext、sparkconf)。

## RDD自身属性

从自身属性说起，SparkContext 是 Spark job 的入口，由 Driver 创建在 client 端，包括集群连接、RDD ID、累加器、广播变量等信息。SparkConf 是参数配置信息，包括:

1. Spark api，控制大部分的应用程序参数；
2. 环境变量，配置IP地址、端口等信息；
3. 日志配置，通过 log4j.properties 配置。

## 数据

RDD 内部的数据集合在逻辑上和物理上被划分成多个小子集合，这样的每一个子集合我们将其称为分区（Partitions），分区的个数会决定并行计算的粒度，而每一个分区数值的计算都是在一个单独的任务中进行的，因此并行任务的个数也是由 RDD分区的个数决定的。

但事实上 RDD 只是数据集的抽象，分区内部并不会存储具体的数据。

Partition 类内包含一个 index 成员，表示该分区在 RDD 内的编号，通过 RDD 编号+分区编号可以确定该分区对应的唯一块编号，再利用底层数据存储层提供的接口就能从存储介质（如：HDFS、Memory）中提取出分区对应的数据。

***RDD 的分区方式*** 主要包含两种：Hash Partitioner 和 Range Partitioner，这两种分区类型都是针对 Key-Value 类型的数据，如是非 Key-Value 类型则分区函数为 None。Hash 是以 Key 作为分区条件的散列分布，分区数据不连续，极端情况也可能散列到少数几个分区上导致数据不均等；Range 按 Key 的排序平衡分布，分区内数据连续，大小也相对均等。

***Preferred Location 是一个列表*** 用于存储每个 Partition 的优先位置。对于每个 HDFS 文件来说，这个列表保存的是每个 Partition 所在的块的位置，也就是该文件的「划分点」。

***Storage Level*** 是 RDD 持久化的存储级别，RDD 持久化可以调用两种方法：cache 和 persist：persist 方法可以自由的设置存储级别，默认是持久化到内存；cache 方法是将 RDD 持久化到内存，cache 的内部实际上是调用了persist 方法，由于没有开放存储级别的参数设置，所以是直接持久化到内存。

***Checkpoint*** 是 Spark 提供的一种缓存机制，当需要计算依赖链非常长又想避免重新计算之前的 RDD 时，可以对 RDD 做 Checkpoint 处理，检查 RDD 是否被物化或计算，并将结果持久化到磁盘或 HDFS 内。Checkpoint 会把当前 RDD 保存到一个目录，要触发 action 操作的时候它才会执行。在 Checkpoint 应该先做持久化（persist 或者 cache）操作，否则就要重新计算一遍。若某个 RDD 成功执行 checkpoint，它前面的所有依赖链会被销毁。

与 Spark 提供的另一种缓存机制 cache 相比：cache 缓存数据由 executor 管理，若 executor 消失，它的数据将被清除，RDD 需要重新计算；而 checkpoint 将数据保存到磁盘或 HDFS 内，job 可以从 checkpoint 点继续计算。Spark 提供了 rdd.persist(StorageLevel.DISK_ONLY) 这样的方法，相当于 cache 到磁盘上，这样可以使 RDD 第一次被计算得到时就存储到磁盘上，它们之间的区别在于：persist 虽然可以将 RDD 的 partition 持久化到磁盘，但一旦作业执行结束，被 cache 到磁盘上的 RDD 会被清空；而 checkpoint 将 RDD 持久化到 HDFS 或本地文件夹，如果不被手动 remove 掉，是一直存在的。

***Compute*** 函数实现方式就是向上递归「获取父 RDD 分区数据进行计算」，直到遇到检查点 RDD 获取有缓存的 RDD。

***Iterator*** 用来查找当前 RDD Partition 与父 RDD 中 Partition 的血缘关系，并通过 Storage Level 确定迭代位置，直到确定真实数据的位置。它的实现流程如下：

* 若标记了有缓存，则取缓存，取不到则进行 computeOrReadCheckpoint(计算或读检查点)。完了再存入缓存，以备后续使用。
* 若未标记有缓存，则直接进行 computeOrReadCheckpoint。
* computeOrReadCheckpoint 这个过程也做两个判断：有做过 checkpoint 和没有做过 checkpoint，做过 checkpoint 则可以读取到检查点数据返回，没做过则调该 RDD 的实现类的 compute 函数计算。


## 血统关系

一个作业从开始到结束的计算过程中产生了多个 RDD，RDD 之间是彼此相互依赖的，我们把这种父子依赖的关系称之为「血统」。

RDD 只支持粗颗粒变换，即只记录单个块（分区）上执行的单个操作，然后创建某个 RDD 的变换序列（血统 lineage）存储下来。

变换序列指每个 RDD 都包含了它是如何由其他 RDD 变换过来的以及如何重建某一块数据的信息。

因此 RDD 的容错机制又称「血统」容错。 要实现这种「血统」容错机制，最大的难题就是如何表达父 RDD 和子 RDD 之间的依赖关系。

![](..\images\7bb7100b.png)

如图所示

* 父 RDD 的每个分区最多只能被子 RDD 的一个分区使用，称为窄依赖（narrow dependency）
* 若父 RDD 的每个分区可以被子 RDD 的多个分区使用，称为宽依赖（wide dependency）

简单来讲，窄依赖就是父子RDD分区间「一对一」的关系，而宽依赖就是「一对多」关系。从失败恢复来看，窄依赖的失败恢复起来更高效，因为它只需找到父 RDD 的一个对应分区即可，而且可以在不同节点上并行计算做恢复；宽依赖牵涉到父 RDD 的多个分区，需要得到所有依赖的父 RDD 分区的 shuffle 结果，恢复起来相对复杂些。

![](..\images\9aac42a6.png)

根据 RDD 之间的宽窄依赖关系引申出 Stage 的概念，Stage 是由一组 RDD 组成的执行计划。如果 RDD 的衍生关系都是窄依赖，则可放在同一个 Stage 中运行，若 RDD 的依赖关系为宽依赖，则要划分到不同的 Stage。这样 Spark 在执行作业时，会按照 Stage 的划分, 生成一个最优、完整的执行计划。




来源：

[RDD原理与基本操作](https://www.toutiao.com/i6595708367690269187/?timestamp=1559207879&app=news_article&group_id=6595708367690269187&req_id=20190530171759010152045154221E925)

[RDD的数据结构模型](https://www.jianshu.com/p/dd7c7243e7f9?from=singlemessage)
