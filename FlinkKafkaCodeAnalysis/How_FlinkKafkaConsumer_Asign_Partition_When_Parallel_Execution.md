# Flink Kafka Consumer并行模式下是怎么分配partition的

前段时间在看Flink Kafka Comsumer的源码时一直在纠结在分布式环境下是怎么运行的？已经多并行度下是怎么分配partition的？

这里大概过一下代码，前面分析的时候也没具体讲分区发现这一块，从FlinkKafkaConsumerBase的open()方法开始，后面再看看开启分区发现的情况

```FlinkKafkaConsumerBase的open()方法里面：```

```
		// create the partition discoverer
		this.partitionDiscoverer = createPartitionDiscoverer(
			topicsDescriptor,
			getRuntimeContext().getIndexOfThisSubtask(),
			getRuntimeContext().getNumberOfParallelSubtasks());
		this.partitionDiscoverer.open();

		subscribedPartitionsToStartOffsets = new HashMap<>();
		final List<KafkaTopicPartition> allPartitions = partitionDiscoverer.discoverPartitions();
```

这里会根据topic、当前subtask的编号、并行度来初始化partitionDiscoverer，partitionDiscoverer.open()会初始化kafka consumer连接

然后调用discoverPartitions()方法来获取分配给当前subtask的分区

```AbstractPartitionDiscoverer的discoverPartitions()方法：```

```
	public List<KafkaTopicPartition> discoverPartitions() throws WakeupException, ClosedException {
		if (!closed && !wakeup) {
			try {
				List<KafkaTopicPartition> newDiscoveredPartitions;

				// (1) get all possible partitions, based on whether we are subscribed to fixed topics or a topic pattern
				if (topicsDescriptor.isFixedTopics()) {
					newDiscoveredPartitions = getAllPartitionsForTopics(topicsDescriptor.getFixedTopics());
				} else {
					List<String> matchedTopics = getAllTopics();

					// retain topics that match the pattern
					Iterator<String> iter = matchedTopics.iterator();
					while (iter.hasNext()) {
						if (!topicsDescriptor.isMatchingTopic(iter.next())) {
							iter.remove();
						}
					}

					if (matchedTopics.size() != 0) {
						// get partitions only for matched topics
						newDiscoveredPartitions = getAllPartitionsForTopics(matchedTopics);
					} else {
						newDiscoveredPartitions = null;
					}
				}

				// (2) eliminate partition that are old partitions or should not be subscribed by this subtask
				if (newDiscoveredPartitions == null || newDiscoveredPartitions.isEmpty()) {
					throw new RuntimeException("Unable to retrieve any partitions with KafkaTopicsDescriptor: " + topicsDescriptor);
				} else {
					Iterator<KafkaTopicPartition> iter = newDiscoveredPartitions.iterator();
					KafkaTopicPartition nextPartition;
					while (iter.hasNext()) {
						nextPartition = iter.next();
						if (!setAndCheckDiscoveredPartition(nextPartition)) {
							iter.remove();
						}
					}
				}

				return newDiscoveredPartitions;
			} catch (WakeupException e) {
				// the actual topic / partition metadata fetching methods
				// may be woken up midway; reset the wakeup flag and rethrow
				wakeup = false;
				throw e;
			}
		} else if (!closed && wakeup) {
			// may have been woken up before the method call
			wakeup = false;
			throw new WakeupException();
		} else {
			throw new ClosedException();
		}
	}
```
这里首先会获取所有传进去的topic的所有partition，然后排除掉老的已经分配了的partition以及不应该分配到当前subtask的partition

来看一下是怎么排除法，调用的是setAndCheckDiscoveredPartition这个函数

```setAndCheckDiscoveredPartition排除partition逻辑```

```
	public boolean setAndCheckDiscoveredPartition(KafkaTopicPartition partition) {
		if (isUndiscoveredPartition(partition)) {
			discoveredPartitions.add(partition);

			return KafkaTopicPartitionAssigner.assign(partition, numParallelSubtasks) == indexOfThisSubtask;
		}

		return false;
	}
```
```
	public static int assign(KafkaTopicPartition partition, int numParallelSubtasks) {
		int startIndex = ((partition.getTopic().hashCode() * 31) & 0x7FFFFFFF) % numParallelSubtasks;

		// here, the assumption is that the id of Kafka partitions are always ascending
		// starting from 0, and therefore can be used directly as the offset clockwise from the start index
		return (startIndex + partition.getPartition()) % numParallelSubtasks;
	}
```

主要执行下面这个计算逻辑确定计算结果是否等于当前subtask的编号，来决定是否将只保留在newDiscoveredPartitions中

计算逻辑偏底层，大概是对topic名称进行hashCode，然后和long的最大值进行与（&）运算，最后除以并行度得出的一个值加上partition的编号，再除以并行度作为返回值

来看一下下面这段代码的测试结果就就明白了,分区数为8，并行度为4

```
        for (int i = 0; i < 8; i++) {
            Integer startIndex = (("topic".hashCode() * 31) & 0x7FFFFFFF) % 4;
            System.out.println("xxxxxxxxxxxxxxxxxxxxxxxx=="+startIndex);
            System.out.println("aaaaaaaaa===="+(startIndex + i) % 4);
        }
```
```
startIndex==1
返回值====1
startIndex==1
返回值====2
startIndex==1
返回值====3
startIndex==1
返回值====0
startIndex==1
返回值====1
startIndex==1
返回值====2
startIndex==1
返回值====3
startIndex==1
返回值====0
```

返回值和并行度编号相等的都会保留在newDiscoveredPartitions中用于分配给当前subtask，最后返回newDiscoveredPartitions这个List<KafkaTopicPartition>

最后会在KafkaConsumerThread中将分区assign给当前consumer。多个并发执行过程都是一样，只是它们获取到的分区不一样

```从这里可以看出，Flink Kafka Consumer用的是手动分配partition的方式```

```开启discovery情况```

开启discovery会在FlinkKafkaConsumerBase中的run()执行下面代码

```
	private void runWithPartitionDiscovery() throws Exception {
		final AtomicReference<Exception> discoveryLoopErrorRef = new AtomicReference<>();
		createAndStartDiscoveryLoop(discoveryLoopErrorRef);

		kafkaFetcher.runFetchLoop();

		// make sure that the partition discoverer is waked up so that
		// the discoveryLoopThread exits
		partitionDiscoverer.wakeup();
		joinDiscoveryLoopThread();

		// rethrow any fetcher errors
		final Exception discoveryLoopError = discoveryLoopErrorRef.get();
		if (discoveryLoopError != null) {
			throw new RuntimeException(discoveryLoopError);
		}
	}
```

这里主要看createAndStartDiscoveryLoop着函数

```
	private void createAndStartDiscoveryLoop(AtomicReference<Exception> discoveryLoopErrorRef) {
		discoveryLoopThread = new Thread(() -> {
			try {
				// --------------------- partition discovery loop ---------------------

				// throughout the loop, we always eagerly check if we are still running before
				// performing the next operation, so that we can escape the loop as soon as possible

				while (running) {
					if (LOG.isDebugEnabled()) {
						LOG.debug("Consumer subtask {} is trying to discover new partitions ...", getRuntimeContext().getIndexOfThisSubtask());
					}

					final List<KafkaTopicPartition> discoveredPartitions;
					try {
						discoveredPartitions = partitionDiscoverer.discoverPartitions();
					} catch (AbstractPartitionDiscoverer.WakeupException | AbstractPartitionDiscoverer.ClosedException e) {
						// the partition discoverer may have been closed or woken up before or during the discovery;
						// this would only happen if the consumer was canceled; simply escape the loop
						break;
					}

					// no need to add the discovered partitions if we were closed during the meantime
					if (running && !discoveredPartitions.isEmpty()) {
						kafkaFetcher.addDiscoveredPartitions(discoveredPartitions);
					}

					// do not waste any time sleeping if we're not running anymore
					if (running && discoveryIntervalMillis != 0) {
						try {
							Thread.sleep(discoveryIntervalMillis);
						} catch (InterruptedException iex) {
							// may be interrupted if the consumer was canceled midway; simply escape the loop
							break;
						}
					}
				}
			} catch (Exception e) {
				discoveryLoopErrorRef.set(e);
			} finally {
				// calling cancel will also let the fetcher loop escape
				// (if not running, cancel() was already called)
				if (running) {
					cancel();
				}
			}
		}, "Kafka Partition Discovery for " + getRuntimeContext().getTaskNameWithSubtasks());

		discoveryLoopThread.start();
	}
```
```
	public void addDiscoveredPartitions(List<KafkaTopicPartition> newPartitions) throws IOException, ClassNotFoundException {
		List<KafkaTopicPartitionState<KPH>> newPartitionStates = createPartitionStateHolders(
				newPartitions,
				KafkaTopicPartitionStateSentinel.EARLIEST_OFFSET,
				timestampWatermarkMode,
				watermarksPeriodic,
				watermarksPunctuated,
				userCodeClassLoader);

		if (useMetrics) {
			registerOffsetMetrics(consumerMetricGroup, newPartitionStates);
		}

		for (KafkaTopicPartitionState<KPH> newPartitionState : newPartitionStates) {
			// the ordering is crucial here; first register the state holder, then
			// push it to the partitions queue to be read
			subscribedPartitionStates.add(newPartitionState);
			unassignedPartitionsQueue.add(newPartitionState);
		}
	}
```

这里新起了个线程间隔循环执行partitionDiscoverer.discoverPartitions()，有新分区时执行addDiscoveredPartitions

将新分区加入了unassignedPartitionsQueue，在KafkaConsumerThread中就能读取出来分配给当前consumer


# 给Flink Kafka Consumer加上日志分析

加日志来测试吧，在Flink Kafka Consumer里面加了不少日志，具体代码先项目：[flinkKafkaPartitionAsignAnalysis](https://github.com/YunKillerE/flinkKafkaPartitionAsignAnalysis)

主要对一下几个文件加日志：FlinkKafkaConsumerBaseY、FlinkKafkaConsumerY、KafkaConsumerThreadY、KafkaFetcherY

所有文件后面加了个Y，然后代码里面调用FlinkKafkaConsumerY来消费数据

测试环境两台机器，初始topic的partition是2，并行度是4，后面加到3，然后加到4，看日志


task1：

```
2019-06-24 16:14:09,740 INFO  org.apache.flink.runtime.taskmanager.Task                     - Ensuring all FileSystem streams are closed for task Source: Custom Source (1/4) (a5c1863564127c17ec833dae6c1a743b) [CANCELED]
2019-06-24 16:14:09,740 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Un-registering task and sending final execution state CANCELED to JobManager for task Source: Custom Source a5c1863564127c17ec833dae6c1a743b.
2019-06-24 16:14:09,975 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Free slot TaskSlot(index:4, state:ACTIVE, resource profile: ResourceProfile{cpuCores=1.7976931348623157E308, heapMemoryInMB=2147483647, directMemoryInMB=2147483647, nativeMemoryInMB=2147483647, networkMemoryInMB=2147483647}, allocationId: c48aa5dac020842a0757df3f156602e2, jobId: 7a1920a97535147e94747a4494dffc80).
2019-06-24 16:14:09,976 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Free slot TaskSlot(index:0, state:ACTIVE, resource profile: ResourceProfile{cpuCores=1.7976931348623157E308, heapMemoryInMB=2147483647, directMemoryInMB=2147483647, nativeMemoryInMB=2147483647, networkMemoryInMB=2147483647}, allocationId: 05779e6e4e4cea2a00abb6fb1914bd41, jobId: 7a1920a97535147e94747a4494dffc80).
2019-06-24 16:14:09,976 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Free slot TaskSlot(index:1, state:ACTIVE, resource profile: ResourceProfile{cpuCores=1.7976931348623157E308, heapMemoryInMB=2147483647, directMemoryInMB=2147483647, nativeMemoryInMB=2147483647, networkMemoryInMB=2147483647}, allocationId: 8eecf0ce4186b1d18679c0c5e0b5c06e, jobId: 7a1920a97535147e94747a4494dffc80).
2019-06-24 16:14:09,977 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Remove job 7a1920a97535147e94747a4494dffc80 from job leader monitoring.
2019-06-24 16:14:09,977 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Stopping ZooKeeperLeaderRetrievalService /leader/7a1920a97535147e94747a4494dffc80/job_manager_lock.
2019-06-24 16:14:09,977 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 7a1920a97535147e94747a4494dffc80.
2019-06-24 16:14:09,977 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 7a1920a97535147e94747a4494dffc80.
2019-06-24 16:14:09,977 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Cannot reconnect to job 7a1920a97535147e94747a4494dffc80 because it is not registered.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Receive slot request 2933096b8d0e4e738986ccf0c0a33f4b for job d2f97605e3580cd1a4ea6acf4e6631d9 from resource manager with leader id 94c6a5d8b3387a7378507d1cef384cd0.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Allocated slot for 2933096b8d0e4e738986ccf0c0a33f4b.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Add job d2f97605e3580cd1a4ea6acf4e6631d9 for job leader monitoring.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Starting ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,299 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Try to register at job manager akka.tcp://flink@x.x.x.252:39171/user/jobmanager_7 with leader id 4105bfad-524c-4bdd-8a47-4ba56a4e9992.
2019-06-24 16:17:29,302 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Resolved JobManager address, beginning registration
2019-06-24 16:17:29,302 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Registration at JobManager attempt 1 (timeout=100ms)
2019-06-24 16:17:29,307 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Successful registration at job manager akka.tcp://flink@x.x.x.252:39171/user/jobmanager_7 for job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,307 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Establish JobManager connection for job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,307 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Offer reserved slots to the leader of job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,309 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Activate slot 2933096b8d0e4e738986ccf0c0a33f4b.
2019-06-24 16:17:29,313 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Received task Source: Custom Source (3/4).
2019-06-24 16:17:29,313 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (3/4) (68da675ea747956faafb6f2e3e2bb1c8) switched from CREATED to DEPLOYING.
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskmanager.Task                     - Creating FileSystem stream leak safety net for task Source: Custom Source (3/4) (68da675ea747956faafb6f2e3e2bb1c8) [DEPLOYING]
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskmanager.Task                     - Loading JAR files for task Source: Custom Source (3/4) (68da675ea747956faafb6f2e3e2bb1c8) [DEPLOYING].
2019-06-24 16:17:31,293 INFO  org.apache.flink.runtime.taskmanager.Task                     - Registering task at network: Source: Custom Source (3/4) (68da675ea747956faafb6f2e3e2bb1c8) [DEPLOYING].
2019-06-24 16:17:31,294 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (3/4) (68da675ea747956faafb6f2e3e2bb1c8) switched from DEPLOYING to RUNNING.
2019-06-24 16:17:31,294 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - Loading state backend via factory org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory
2019-06-24 16:17:31,295 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using predefined options: DEFAULT.
2019-06-24 16:17:31,295 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using default options factory: DefaultConfigurableOptionsFactory{configuredOptions={}}.
2019-06-24 16:17:31,327 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition does not contain a setter for field topic
2019-06-24 16:17:31,327 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - Class class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition cannot be used as a POJO type because not all fields are valid POJO fields, and must be processed as GenericType. Please read the Flink documentation on "Data Types & Serialization" for details of the effect on performance.
2019-06-24 16:17:31,327 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - No restore state for FlinkKafkaConsumerY.
2019-06-24 16:17:31,341 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,377 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,378 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,379 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,444 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  start =============================================
2019-06-24 16:17:31,444 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init offsetCommitModeb  : KAFKA_PERIODIC
2019-06-24 16:17:31,444 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,444 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init allPartitions : []
2019-06-24 16:17:31,444 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  end =============================================
2019-06-24 16:17:31,445 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - Consumer subtask 2 initially has no partitions to read from.
2019-06-24 16:17:31,445 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open ifelse init subscribedPartitionsToStartOffsets  : []
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init threadName : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init handover : org.apache.flink.streaming.connectors.kafka.internal.Handover@cbfebe4    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init kafkaProperties : {key.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, auto.offset.reset=earliest, bootstrap.servers=x.x.x.18:9092,x.x.x.19:9092, group.id=xxxxxxx, value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, flink.partition-discovery.interval-millis=1000}    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init unassignedPartitionsQueue : []    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init pollTimeout: 100    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init running : true    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init start =================================================== 
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subtaskIndex : 2
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init kafkaFetcher  : consumerPartitionAsgin.KafkaFetcherY@34ec8e23
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init periodicWatermarkAssigner : null
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init punctuatedWatermarkAssigner : null
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init offsetCommitMode : KAFKA_PERIODIC
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init KAFKA_CONSUMER_METRICS_GROUP : KafkaConsumer
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init useMetrics : true
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init discoveryIntervalMillis : 1000
2019-06-24 16:17:31,449 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init end  =================================================== 
2019-06-24 16:17:31,452 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,455 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,455 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,455 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,458 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run consumer.assignment() : []    threadName  : Kafka Fetcher for Source: Custom Source (3/4)


2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run running else newPartitions : [Partition: KafkaTopicPartition{topic='partition3', partition=3}, KafkaPartitionHandle=partition3-3, offset=-915623761775]    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignPartitions : true    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : false    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY oldPartitionAssignmentsToPosition : {}    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY newPartitionAssignments : [partition3-3]    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,701 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : true    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,702 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-3]    threadName  : Kafka Fetcher for Source: Custom Source (3/4)
2019-06-24 16:21:07,708 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-3]    threadName  : Kafka Fetcher for Source: Custom Source (3/4)

```

这台机器分到了一个subtask，运行了一个consumer，初始化时没有分配到partition

这里开启了partition的discovery，后面有新分区加入时就会将partition 3分配给当前subtask

task2:

```
[BEGIN] 2019/6/24 16:17:06
[hdfs@qa-hdpdn06 flink-1.8.0]$ tail -f log/flink-hdfs-taskexecutor-2-qa-hdpdn06.exxx.com.log 
2019-06-24 16:14:09,730 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (3/4) (a237455c74722ce9947df3c3f5e579e0) switched from CANCELING to CANCELED.
2019-06-24 16:14:09,730 INFO  org.apache.flink.runtime.taskmanager.Task                     - Freeing task resources for Source: Custom Source (3/4) (a237455c74722ce9947df3c3f5e579e0).
2019-06-24 16:14:09,731 INFO  org.apache.flink.runtime.taskmanager.Task                     - Ensuring all FileSystem streams are closed for task Source: Custom Source (3/4) (a237455c74722ce9947df3c3f5e579e0) [CANCELED]
2019-06-24 16:14:09,731 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Un-registering task and sending final execution state CANCELED to JobManager for task Source: Custom Source a237455c74722ce9947df3c3f5e579e0.
2019-06-24 16:14:09,975 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Free slot TaskSlot(index:1, state:ACTIVE, resource profile: ResourceProfile{cpuCores=1.7976931348623157E308, heapMemoryInMB=2147483647, directMemoryInMB=2147483647, nativeMemoryInMB=2147483647, networkMemoryInMB=2147483647}, allocationId: a4761f3b9ab99db559b96a6cf0d3b8b4, jobId: 7a1920a97535147e94747a4494dffc80).
2019-06-24 16:14:09,975 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Remove job 7a1920a97535147e94747a4494dffc80 from job leader monitoring.
2019-06-24 16:14:09,975 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Stopping ZooKeeperLeaderRetrievalService /leader/7a1920a97535147e94747a4494dffc80/job_manager_lock.
2019-06-24 16:14:09,975 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 7a1920a97535147e94747a4494dffc80.
2019-06-24 16:14:09,976 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Close JobManager connection for job 7a1920a97535147e94747a4494dffc80.
2019-06-24 16:14:09,976 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Cannot reconnect to job 7a1920a97535147e94747a4494dffc80 because it is not registered.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Receive slot request f99f4d15290c4fcc320fc1e62afc20e0 for job d2f97605e3580cd1a4ea6acf4e6631d9 from resource manager with leader id 94c6a5d8b3387a7378507d1cef384cd0.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Allocated slot for f99f4d15290c4fcc320fc1e62afc20e0.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Add job d2f97605e3580cd1a4ea6acf4e6631d9 for job leader monitoring.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Starting ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,297 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Receive slot request faaa85631057031bc5e72ea2c2715852 for job d2f97605e3580cd1a4ea6acf4e6631d9 from resource manager with leader id 94c6a5d8b3387a7378507d1cef384cd0.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Allocated slot for faaa85631057031bc5e72ea2c2715852.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Add job d2f97605e3580cd1a4ea6acf4e6631d9 for job leader monitoring.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Stopping ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Starting ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Receive slot request b6ae1716b1c3a132c3858a9d80b9aaf5 for job d2f97605e3580cd1a4ea6acf4e6631d9 from resource manager with leader id 94c6a5d8b3387a7378507d1cef384cd0.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Allocated slot for b6ae1716b1c3a132c3858a9d80b9aaf5.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Add job d2f97605e3580cd1a4ea6acf4e6631d9 for job leader monitoring.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Stopping ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,298 INFO  org.apache.flink.runtime.leaderretrieval.ZooKeeperLeaderRetrievalService  - Starting ZooKeeperLeaderRetrievalService /leader/d2f97605e3580cd1a4ea6acf4e6631d9/job_manager_lock.
2019-06-24 16:17:29,301 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Try to register at job manager akka.tcp://flink@x.x.x.252:39171/user/jobmanager_7 with leader id 4105bfad-524c-4bdd-8a47-4ba56a4e9992.
2019-06-24 16:17:29,305 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Resolved JobManager address, beginning registration
2019-06-24 16:17:29,305 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Registration at JobManager attempt 1 (timeout=100ms)
2019-06-24 16:17:29,309 INFO  org.apache.flink.runtime.taskexecutor.JobLeaderService        - Successful registration at job manager akka.tcp://flink@x.x.x.252:39171/user/jobmanager_7 for job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,309 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Establish JobManager connection for job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,309 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Offer reserved slots to the leader of job d2f97605e3580cd1a4ea6acf4e6631d9.
2019-06-24 16:17:29,313 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Received task Source: Custom Source (1/4).
2019-06-24 16:17:29,313 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (1/4) (94c7a964ef15865f095976b4b6fd2006) switched from CREATED to DEPLOYING.
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskmanager.Task                     - Creating FileSystem stream leak safety net for task Source: Custom Source (1/4) (94c7a964ef15865f095976b4b6fd2006) [DEPLOYING]
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskmanager.Task                     - Loading JAR files for task Source: Custom Source (1/4) (94c7a964ef15865f095976b4b6fd2006) [DEPLOYING].
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Received task Source: Custom Source (2/4).
2019-06-24 16:17:29,314 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (2/4) (272ab8f42bd8c606ebcbe5e568a2b524) switched from CREATED to DEPLOYING.
2019-06-24 16:17:29,315 INFO  org.apache.flink.runtime.taskmanager.Task                     - Creating FileSystem stream leak safety net for task Source: Custom Source (2/4) (272ab8f42bd8c606ebcbe5e568a2b524) [DEPLOYING]
2019-06-24 16:17:29,315 INFO  org.apache.flink.runtime.taskmanager.Task                     - Loading JAR files for task Source: Custom Source (2/4) (272ab8f42bd8c606ebcbe5e568a2b524) [DEPLOYING].
2019-06-24 16:17:29,315 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Received task Source: Custom Source (4/4).
2019-06-24 16:17:29,315 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Activate slot b6ae1716b1c3a132c3858a9d80b9aaf5.
2019-06-24 16:17:29,316 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (4/4) (9e020349c387653ad1c9bd1ed075987b) switched from CREATED to DEPLOYING.
2019-06-24 16:17:29,316 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Activate slot f99f4d15290c4fcc320fc1e62afc20e0.
2019-06-24 16:17:29,316 INFO  org.apache.flink.runtime.taskmanager.Task                     - Creating FileSystem stream leak safety net for task Source: Custom Source (4/4) (9e020349c387653ad1c9bd1ed075987b) [DEPLOYING]
2019-06-24 16:17:29,316 INFO  org.apache.flink.runtime.taskexecutor.slot.TaskSlotTable      - Activate slot faaa85631057031bc5e72ea2c2715852.
2019-06-24 16:17:29,316 INFO  org.apache.flink.runtime.taskmanager.Task                     - Loading JAR files for task Source: Custom Source (4/4) (9e020349c387653ad1c9bd1ed075987b) [DEPLOYING].
2019-06-24 16:17:29,317 INFO  org.apache.flink.runtime.taskexecutor.TaskExecutor            - Received task Sink: Print to Std. Out (1/1).
2019-06-24 16:17:29,318 INFO  org.apache.flink.runtime.taskmanager.Task                     - Sink: Print to Std. Out (1/1) (2e5375f621b643fb4d8c1cb83e198acd) switched from CREATED to DEPLOYING.
2019-06-24 16:17:29,318 INFO  org.apache.flink.runtime.taskmanager.Task                     - Creating FileSystem stream leak safety net for task Sink: Print to Std. Out (1/1) (2e5375f621b643fb4d8c1cb83e198acd) [DEPLOYING]
2019-06-24 16:17:29,318 INFO  org.apache.flink.runtime.taskmanager.Task                     - Loading JAR files for task Sink: Print to Std. Out (1/1) (2e5375f621b643fb4d8c1cb83e198acd) [DEPLOYING].
2019-06-24 16:17:30,993 INFO  org.apache.flink.runtime.taskmanager.Task                     - Registering task at network: Source: Custom Source (4/4) (9e020349c387653ad1c9bd1ed075987b) [DEPLOYING].
2019-06-24 16:17:30,993 INFO  org.apache.flink.runtime.taskmanager.Task                     - Registering task at network: Source: Custom Source (1/4) (94c7a964ef15865f095976b4b6fd2006) [DEPLOYING].
2019-06-24 16:17:30,993 INFO  org.apache.flink.runtime.taskmanager.Task                     - Registering task at network: Source: Custom Source (2/4) (272ab8f42bd8c606ebcbe5e568a2b524) [DEPLOYING].
2019-06-24 16:17:30,993 INFO  org.apache.flink.runtime.taskmanager.Task                     - Registering task at network: Sink: Print to Std. Out (1/1) (2e5375f621b643fb4d8c1cb83e198acd) [DEPLOYING].
2019-06-24 16:17:30,995 INFO  org.apache.flink.runtime.taskmanager.Task                     - Sink: Print to Std. Out (1/1) (2e5375f621b643fb4d8c1cb83e198acd) switched from DEPLOYING to RUNNING.
2019-06-24 16:17:30,995 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (1/4) (94c7a964ef15865f095976b4b6fd2006) switched from DEPLOYING to RUNNING.
2019-06-24 16:17:30,995 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - Loading state backend via factory org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory
2019-06-24 16:17:30,995 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (2/4) (272ab8f42bd8c606ebcbe5e568a2b524) switched from DEPLOYING to RUNNING.
2019-06-24 16:17:30,995 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - Loading state backend via factory org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory
2019-06-24 16:17:30,995 INFO  org.apache.flink.runtime.taskmanager.Task                     - Source: Custom Source (4/4) (9e020349c387653ad1c9bd1ed075987b) switched from DEPLOYING to RUNNING.
2019-06-24 16:17:30,995 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - Loading state backend via factory org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory
2019-06-24 16:17:30,995 INFO  org.apache.flink.streaming.runtime.tasks.StreamTask           - Loading state backend via factory org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory
2019-06-24 16:17:30,995 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using predefined options: DEFAULT.
2019-06-24 16:17:30,995 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using predefined options: DEFAULT.
2019-06-24 16:17:30,995 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using default options factory: DefaultConfigurableOptionsFactory{configuredOptions={}}.
2019-06-24 16:17:30,996 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using default options factory: DefaultConfigurableOptionsFactory{configuredOptions={}}.
2019-06-24 16:17:30,996 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using predefined options: DEFAULT.
2019-06-24 16:17:30,996 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using predefined options: DEFAULT.
2019-06-24 16:17:30,996 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using default options factory: DefaultConfigurableOptionsFactory{configuredOptions={}}.
2019-06-24 16:17:30,996 INFO  org.apache.flink.contrib.streaming.state.RocksDBStateBackend  - Using default options factory: DefaultConfigurableOptionsFactory{configuredOptions={}}.
2019-06-24 16:17:31,030 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition does not contain a setter for field topic
2019-06-24 16:17:31,030 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition does not contain a setter for field topic
2019-06-24 16:17:31,030 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - Class class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition cannot be used as a POJO type because not all fields are valid POJO fields, and must be processed as GenericType. Please read the Flink documentation on "Data Types & Serialization" for details of the effect on performance.
2019-06-24 16:17:31,030 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition does not contain a setter for field topic
2019-06-24 16:17:31,030 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - Class class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition cannot be used as a POJO type because not all fields are valid POJO fields, and must be processed as GenericType. Please read the Flink documentation on "Data Types & Serialization" for details of the effect on performance.
2019-06-24 16:17:31,031 INFO  org.apache.flink.api.java.typeutils.TypeExtractor             - Class class org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition cannot be used as a POJO type because not all fields are valid POJO fields, and must be processed as GenericType. Please read the Flink documentation on "Data Types & Serialization" for details of the effect on performance.
2019-06-24 16:17:31,031 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - No restore state for FlinkKafkaConsumerY.
2019-06-24 16:17:31,031 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - No restore state for FlinkKafkaConsumerY.
2019-06-24 16:17:31,031 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - No restore state for FlinkKafkaConsumerY.
2019-06-24 16:17:31,045 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,045 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,046 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,085 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,085 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,085 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,086 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,086 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,087 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,087 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,087 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,087 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  start =============================================
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  start =============================================
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init offsetCommitModeb  : KAFKA_PERIODIC
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init offsetCommitModeb  : KAFKA_PERIODIC
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  start =============================================
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init offsetCommitModeb  : KAFKA_PERIODIC
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init allPartitions : []
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  end =============================================
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - allPartitions for : KafkaTopicPartition{topic='partition3', partition=0}
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - allPartitions for : KafkaTopicPartition{topic='partition3', partition=1}
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init allPartitions : [KafkaTopicPartition{topic='partition3', partition=0}]
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  end =============================================
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase init allPartitions : [KafkaTopicPartition{topic='partition3', partition=1}]
2019-06-24 16:17:31,154 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open init  end =============================================
2019-06-24 16:17:31,155 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - Consumer subtask 1 initially has no partitions to read from.
2019-06-24 16:17:31,155 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open ifelse init subscribedPartitionsToStartOffsets  : []
2019-06-24 16:17:31,155 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - Consumer subtask 0 will start reading the following 1 partitions from the committed group offsets in Kafka: [KafkaTopicPartition{topic='partition3', partition=1}]
2019-06-24 16:17:31,155 INFO  consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - Consumer subtask 3 will start reading the following 1 partitions from the committed group offsets in Kafka: [KafkaTopicPartition{topic='partition3', partition=0}]
2019-06-24 16:17:31,155 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open ifelse init subscribedPartitionsToStartOffsets  : [KafkaTopicPartition{topic='partition3', partition=0}]
2019-06-24 16:17:31,155 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase open ifelse init subscribedPartitionsToStartOffsets  : [KafkaTopicPartition{topic='partition3', partition=1}]
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init threadName : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init threadName : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init handover : org.apache.flink.streaming.connectors.kafka.internal.Handover@4bceee10    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init threadName : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init kafkaProperties : {key.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, auto.offset.reset=earliest, bootstrap.servers=x.x.x.18:9092,x.x.x.19:9092, group.id=xxxxxxx, value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, flink.partition-discovery.interval-millis=1000}    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init handover : org.apache.flink.streaming.connectors.kafka.internal.Handover@1049706    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init handover : org.apache.flink.streaming.connectors.kafka.internal.Handover@79b1099b    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init unassignedPartitionsQueue : [Partition: KafkaTopicPartition{topic='partition3', partition=0}, KafkaPartitionHandle=partition3-0, offset=-915623761773]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init kafkaProperties : {key.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, auto.offset.reset=earliest, bootstrap.servers=x.x.x.18:9092,x.x.x.19:9092, group.id=xxxxxxx, value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, flink.partition-discovery.interval-millis=1000}    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init pollTimeout: 100    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,160 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init kafkaProperties : {key.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, auto.offset.reset=earliest, bootstrap.servers=x.x.x.18:9092,x.x.x.19:9092, group.id=xxxxxxx, value.deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer, flink.partition-discovery.interval-millis=1000}    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init unassignedPartitionsQueue : []    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init running : true    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init unassignedPartitionsQueue : [Partition: KafkaTopicPartition{topic='partition3', partition=1}, KafkaPartitionHandle=partition3-1, offset=-915623761773]    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init start =================================================== 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init pollTimeout: 100    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init running : true    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init start =================================================== 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subtaskIndex : 1
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subtaskIndex : 3
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init pollTimeout: 100    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init running : true    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY init  ============================================ 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init start =================================================== 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subtaskIndex : 0
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init kafkaFetcher  : consumerPartitionAsgin.KafkaFetcherY@31b7c97b
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init kafkaFetcher  : consumerPartitionAsgin.KafkaFetcherY@62ef41e
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subscribedPartitionsToStartOffsets : []
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init periodicWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init punctuatedWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init kafkaFetcher  : consumerPartitionAsgin.KafkaFetcherY@7bc0729c
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subscribedPartitionsToStartOffsets : [KafkaTopicPartition{topic='partition3', partition=1}]
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init periodicWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init punctuatedWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init offsetCommitMode : KAFKA_PERIODIC
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init subscribedPartitionsToStartOffsets : [KafkaTopicPartition{topic='partition3', partition=0}]
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init periodicWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init punctuatedWatermarkAssigner : null
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init offsetCommitMode : KAFKA_PERIODIC
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init KAFKA_CONSUMER_METRICS_GROUP : KafkaConsumer
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init offsetCommitMode : KAFKA_PERIODIC
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init KAFKA_CONSUMER_METRICS_GROUP : KafkaConsumer
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init useMetrics : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init useMetrics : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init KAFKA_CONSUMER_METRICS_GROUP : KafkaConsumer
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init useMetrics : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init running : true
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init discoveryIntervalMillis : 1000
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init discoveryIntervalMillis : 1000
2019-06-24 16:17:31,162 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init end  =================================================== 
2019-06-24 16:17:31,161 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init discoveryIntervalMillis : 1000
2019-06-24 16:17:31,162 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init end  =================================================== 
2019-06-24 16:17:31,162 ERROR consumerPartitionAsgin.FlinkKafkaConsumerBaseY                - FlinkKafkaConsumerBase run init end  =================================================== 
2019-06-24 16:17:31,165 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,166 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,167 INFO  org.apache.kafka.clients.consumer.ConsumerConfig              - ConsumerConfig values: 
	auto.commit.interval.ms = 5000
	auto.offset.reset = earliest
	bootstrap.servers = [x.x.x.18:9092, x.x.x.19:9092]
	check.crcs = true
	client.id = 
	connections.max.idle.ms = 540000
	enable.auto.commit = true
	exclude.internal.topics = true
	fetch.max.bytes = 52428800
	fetch.max.wait.ms = 500
	fetch.min.bytes = 1
	group.id = xxxxxxx
	heartbeat.interval.ms = 3000
	interceptor.classes = null
	internal.leave.group.on.close = true
	isolation.level = read_uncommitted
	key.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer
	max.partition.fetch.bytes = 1048576
	max.poll.interval.ms = 300000
	max.poll.records = 500
	metadata.max.age.ms = 300000
	metric.reporters = []
	metrics.num.samples = 2
	metrics.recording.level = INFO
	metrics.sample.window.ms = 30000
	partition.assignment.strategy = [class org.apache.kafka.clients.consumer.RangeAssignor]
	receive.buffer.bytes = 65536
	reconnect.backoff.max.ms = 1000
	reconnect.backoff.ms = 50
	request.timeout.ms = 305000
	retry.backoff.ms = 100
	sasl.jaas.config = null
	sasl.kerberos.kinit.cmd = /usr/bin/kinit
	sasl.kerberos.min.time.before.relogin = 60000
	sasl.kerberos.service.name = null
	sasl.kerberos.ticket.renew.jitter = 0.05
	sasl.kerberos.ticket.renew.window.factor = 0.8
	sasl.mechanism = GSSAPI
	security.protocol = PLAINTEXT
	send.buffer.bytes = 131072
	session.timeout.ms = 10000
	ssl.cipher.suites = null
	ssl.enabled.protocols = [TLSv1.2, TLSv1.1, TLSv1]
	ssl.endpoint.identification.algorithm = null
	ssl.key.password = null
	ssl.keymanager.algorithm = SunX509
	ssl.keystore.location = null
	ssl.keystore.password = null
	ssl.keystore.type = JKS
	ssl.protocol = TLS
	ssl.provider = null
	ssl.secure.random.implementation = null
	ssl.trustmanager.algorithm = PKIX
	ssl.truststore.location = null
	ssl.truststore.password = null
	ssl.truststore.type = JKS
	value.deserializer = class org.apache.kafka.common.serialization.ByteArrayDeserializer

2019-06-24 16:17:31,173 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,173 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,173 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,173 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,174 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,174 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,174 WARN  org.apache.kafka.clients.consumer.ConsumerConfig              - The configuration 'flink.partition-discovery.interval-millis' was supplied but isn't a known config.
2019-06-24 16:17:31,174 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka version : 1.0.0
2019-06-24 16:17:31,174 INFO  org.apache.kafka.common.utils.AppInfoParser                   - Kafka commitId : aaa7af6d4a11b29d
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run consumer.assignment() : []    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run running else newPartitions : [Partition: KafkaTopicPartition{topic='partition3', partition=1}, KafkaPartitionHandle=partition3-1, offset=-915623761773]    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run consumer.assignment() : []    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignPartitions : true    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : false    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run consumer.assignment() : []    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY oldPartitionAssignmentsToPosition : {}    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run running else newPartitions : [Partition: KafkaTopicPartition{topic='partition3', partition=0}, KafkaPartitionHandle=partition3-0, offset=-915623761773]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignPartitions : true    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : false    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY newPartitionAssignments : [partition3-1]    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY oldPartitionAssignmentsToPosition : {}    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,176 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY newPartitionAssignments : [partition3-0]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,177 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : true    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,177 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : true    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,177 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-1]    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,177 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-0]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:17:31,180 INFO  org.apache.kafka.clients.consumer.internals.AbstractCoordinator  - [Consumer clientId=consumer-6, groupId=xxxxxxx] Discovered coordinator x.x.x.18:9092 (id: 2147483629 rack: null)
2019-06-24 16:17:31,181 INFO  org.apache.kafka.clients.consumer.internals.AbstractCoordinator  - [Consumer clientId=consumer-4, groupId=xxxxxxx] Discovered coordinator x.x.x.18:9092 (id: 2147483629 rack: null)
2019-06-24 16:17:31,186 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-1]    threadName  : Kafka Fetcher for Source: Custom Source (1/4)
2019-06-24 16:17:31,186 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-0]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)


2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run running else newPartitions : [Partition: KafkaTopicPartition{topic='partition3', partition=2}, KafkaPartitionHandle=partition3-2, offset=-915623761775]    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignPartitions : true    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : false    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY oldPartitionAssignmentsToPosition : {}    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY newPartitionAssignments : [partition3-2]    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : true    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,280 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-2]    threadName  : Kafka Fetcher for Source: Custom Source (2/4)
2019-06-24 16:19:51,284 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-2]    threadName  : Kafka Fetcher for Source: Custom Source (2/4)


2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY run running else newPartitions : [Partition: KafkaTopicPartition{topic='partition3', partition=4}, KafkaPartitionHandle=partition3-4, offset=-915623761775]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignPartitions : true    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : false    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY oldPartitionAssignmentsToPosition : {partition3-0=0}    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY newPartitionAssignments : [partition3-0, partition3-4]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY reassignmentStarted : true    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,393 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-0, partition3-4]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)
2019-06-24 16:21:23,589 ERROR consumerPartitionAsgin.KafkaFetcherY                          - FlinkKafkaConsumerBase KafkaConsumerThreadY consumerTmp.assignment : [partition3-0, partition3-4]    threadName  : Kafka Fetcher for Source: Custom Source (4/4)

[END] 2019/6/24 16:22:07

```














