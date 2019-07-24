# spark vs flink


#1. 宏观层面对比

设计动机思想都不同：
	
1. spark设计之初主要是为了解决MapReduce的缺陷（比如迭代、性能等）及和Hadoop生态圈的集成，然后逐步支持mirobatch streaming的计算及完善的一套自身的生态系统，比如Spark Core之上的Spark SQL，Spark MLIB，Spark Streaming，Spark Graph
		
2. flink设计之初就是为了解决流式计算场景，基于业界当时许多的流计算研究文献，尤其是Google DataFlow，然后逐步发展出了和spark相似的一套自身的生态系统
		
3. 一个是在批上支持流，一个是在流上支持批

#2. 微观层面对比

微观层面就比较多了
	
1. 核心开发语言方面，Flink是用java，Spark是用scala，但两者都支持java、scala、python等语言的api，相对来说感觉spark的支持度相对好一点，尤其是python、R及在机器学习领域Spark领先于Flink，但也不是绝对的，阿里加入后据说后续会对机器学习这一块有很多提升
	
2. 从流式计算的设计来说，spark里面有spark stremaing的微批的流计算及structured streaming来支持更低延迟的实时流计算，当然structured streaming有两种执行引擎，一个是和spark stremaing一样的micro batch执行引擎，另一个是continue streaming执行引擎，从性能上来continue streaming和flink更接近，但是continue stremaing目前还是实验性质，感觉spark streaming类似于flink的datastreamin api，当然这个没有权威性，因为flink datastream api里面有event time、window、trigger等流计算的重要概念，而spark streaming里面还不具备，但structured streaming也引入，下面从这几个流的重要概念来看看
	
3. 从流计算的几个重要概念event time、window、trigger来说，flink里面从底层的datastream api到table api再到SQL都支持这些概念，但是spark里面只有structured streaming支持，structured streaming有点类似于flink的table api和SQL两层，event time这一块两者都支持，但是flink更具定制性，flink支持event time、ingest time、processing time，可以适应多种场景，window方面，flink默认支持的更强大，比如session window、group window，在spark里面都要自己去实现，当然spark里面提供了mapGroupsWithState状态算子可以间接实现session window，最后trigger，触发策略，flink里面的设计更接近dataflow，默认支持更多场景的trigger，自定义也更方便，当然table api和sql层面是不支持自定义的，spark里面的trigger默认只有三种，once、continue、processing time。从流式细节的支持方面，flink是远远强于spark，能更好地致辞更多复杂的实时计算场景
	
4. 底层分布式通讯方面，flink用到了netty和akka，spark是netty，早期spark也是akka，后来主要是社区有人反馈说spark akka版本和环境的冲突，就改成netty了
	








