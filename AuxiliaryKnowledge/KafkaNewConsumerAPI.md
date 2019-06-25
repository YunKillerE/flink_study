# 官方API文档

看flink kafka consumer代码前应该了解一下内容

https://kafka.apache.org/22/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html

开发人员都应该看看

这里copy一下要点

1. Cross-Version Compatibility

This client can communicate with brokers that are version 0.10.0 or newer. Older or newer brokers may not support certain features. For example, 0.10.0 brokers do not support offsetsForTimes, because this feature was added in version 0.10.1. You will receive an UnsupportedVersionException when invoking an API that is not available on the running broker version.

2. Consumer Groups and Topic Subscriptions

Kafka uses the concept of consumer groups to allow a pool of processes to divide the work of consuming and processing records. These processes can either be running on the same machine or they can be distributed over many machines to provide scalability and fault tolerance for processing. All consumer instances sharing the same group.id will be part of the same consumer group.
Each consumer in a group can dynamically set the list of topics it wants to subscribe to through one of the subscribe APIs. Kafka will deliver each message in the subscribed topics to one process in each consumer group. This is achieved by balancing the partitions between all members in the consumer group so that each partition is assigned to exactly one consumer in the group. So if there is a topic with four partitions, and a consumer group with two processes, each process would consume from two partitions.

Membership in a consumer group is maintained dynamically: if a process fails, the partitions assigned to it will be reassigned to other consumers in the same group. Similarly, if a new consumer joins the group, partitions will be moved from existing consumers to the new one. This is known as rebalancing the group and is discussed in more detail below. Group rebalancing is also used when new partitions are added to one of the subscribed topics or when a new topic matching a subscribed regex is created. The group will automatically detect the new partitions through periodic metadata refreshes and assign them to members of the group.

3. Detecting Consumer Failures

After subscribing to a set of topics, the consumer will automatically join the group when poll(Duration) is invoked. The poll API is designed to ensure consumer liveness. As long as you continue to call poll, the consumer will stay in the group and continue to receive messages from the partitions it was assigned. Underneath the covers, the consumer sends periodic heartbeats to the server. If the consumer crashes or is unable to send heartbeats for a duration of session.timeout.ms, then the consumer will be considered dead and its partitions will be reassigned.

It is also possible that the consumer could encounter a "livelock" situation where it is continuing to send heartbeats, but no progress is being made. To prevent the consumer from holding onto its partitions indefinitely in this case, we provide a liveness detection mechanism using the max.poll.interval.ms setting. Basically if you don't call poll at least as frequently as the configured max interval, then the client will proactively leave the group so that another consumer can take over its partitions. When this happens, you may see an offset commit failure (as indicated by a CommitFailedException thrown from a call to commitSync()). This is a safety mechanism which guarantees that only active members of the group are able to commit offsets. So to stay in the group, you must continue to call poll.

The consumer provides two configuration settings to control the behavior of the poll loop:

* max.poll.interval.ms: By increasing the interval between expected polls, you can give the consumer more time to handle a batch of records returned from poll(Duration). The drawback is that increasing this value may delay a group rebalance since the consumer will only join the rebalance inside the call to poll. You can use this setting to bound the time to finish a rebalance, but you risk slower progress if the consumer cannot actually call poll often enough.
* max.poll.records: Use this setting to limit the total records returned from a single call to poll. This can make it easier to predict the maximum that must be handled within each poll interval. By tuning this value, you may be able to reduce the poll interval, which will reduce the impact of group rebalancing.

# 测试代码

1. subscribe api 类似于以前的high level层次的api，partition由kafka管理分区分配及reblance

```
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "x.x.x.18:9092,x.x.x.19:9092");
        props.setProperty("group.id", "kk");
        props.setProperty("enable.auto.commit", "true");
        props.setProperty("auto.commit.interval.ms", "1000");
        props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("xxxout"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100L);
            for (ConsumerRecord<String, String> record : records)
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
        }
```

2. 手动assign api，需要自己分配consumer消费哪些partition，如果失败了，也需要手动管理

这里还可以调用 consumer.pause 停止消费数据，在有些场景还是挺有用的

```
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "x.x.x.18:9092,x.x.x.19:9092");
        props.setProperty("group.id", "kk");
        props.setProperty("enable.auto.commit", "false");
        props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        String topic = "xxxout";
        TopicPartition partition0 = new TopicPartition(topic, 0);
        TopicPartition partition1 = new TopicPartition(topic, 1);
        consumer.assign(Arrays.asList(partition0, partition1));
        
        final int minBatchSize = 200;
        consumer.pause(Collections.singleton(partition0));
        consumer.pause(Collections.singleton(partition1));
        consumer.resume(Collections.singleton(partition0));
        List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100L);
            for (ConsumerRecord<String, String> record : records) {
                buffer.add(record);
            }
            if (buffer.size() >= minBatchSize) {
                System.out.printf("value = %s%n", buffer);
                consumer.commitSync();
                buffer.clear();
            }
        }
```

3. topic相关信息查询

```
        //获取topic及其partition
        List<KafkaTopicPartition> partitions = new LinkedList<>();

        for (PartitionInfo partitionInfo : consumer.partitionsFor("xxxout")) {
            System.out.println(partitionInfo.topic() + "-------" + partitionInfo.partition());
            partitions.add(new KafkaTopicPartition(partitionInfo.topic(), partitionInfo.partition()));
        }

        //获取所有的topic
        System.out.println(consumer.listTopics().keySet());

        //根据时间获取offset
        final Map<KafkaTopicPartition, Long> result = new HashMap<>(partitions.size());
        Map<TopicPartition, Long> partitionOffsetsRequest = new HashMap<>(partitions.size());

        for (KafkaTopicPartition partition : partitions) {
            partitionOffsetsRequest.put(
                    new TopicPartition(partition.getTopic(), partition.getPartition()),
                    1559283017L);
        }

        for (Map.Entry<TopicPartition, OffsetAndTimestamp> partitionToOffset :
                consumer.offsetsForTimes(partitionOffsetsRequest).entrySet()) {

            result.put(
                    new KafkaTopicPartition(partitionToOffset.getKey().topic(), partitionToOffset.getKey().partition()),
                    (partitionToOffset.getValue() == null) ? null : partitionToOffset.getValue().offset());
        }

        System.out.println("result ==== "+result);

        //初始化partition 的 offset
        Map<KafkaTopicPartition, Long> subscribedPartitionsToStartOffsets = new HashMap<>();;

        for (KafkaTopicPartition seedPartition : partitions) {
            subscribedPartitionsToStartOffsets.put(seedPartition, -915623761773L);
        }

        System.out.println("subscribedPartitionsToStartOffsets==="+subscribedPartitionsToStartOffsets);


        //获取分配给当前consumer的partition
        final Map<TopicPartition, Long> oldPartitionAssignmentsToPosition = new HashMap<>();
        for (TopicPartition oldPartition : consumer.assignment()) {
            oldPartitionAssignmentsToPosition.put(oldPartition, consumer.position(oldPartition));
        }
        System.out.println("oldPartitionAssignmentsToPosition=="+oldPartitionAssignmentsToPosition);
```
















