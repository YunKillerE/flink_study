实现目标：

* 看看能不能实现 producer exactly once，kafka 1.0后好像支持二阶段提交，研究一下

# maven编译时的scope指定

<scope>provided</scope> : 只在开发、测试阶段使用，idea run时可能会报找不到包

<scope>compile</scope> : 会打进jar包中，额外的jar需要执行这个，否则flink run时会找不到jar包

# error

```
Caused by: java.lang.NoClassDefFoundError: Could not initialize class org.apache.hadoop.hdfs.protocol.HdfsConstants
```

应该是jar包冲突，pom文件中将hdfs，mr的几个包的scope改为provided

```
java.util.zip.ZipException: Not in GZIP format
```

无解，因为输入的是gzip压缩流，有人说改为单字节的编码可以解决，但测试无效，还是有时会出现，据说彻底的解决办法是输出也用gzip

```
Could not load the TypeInformation for the class 'org.apache.hadoop.io.Writable'. You may be missing the 'flink-hadoop-compatibility' dependency. 
```

将flink-hadoop-compatibility_2.11-1.x.x.jar放到flink的lib目录中，一般在opt目录中，这解决方案比较无语，所有机器都得放一遍


```$xslt
Cannot instantiate user function
```

每个人遇到这个问题的原因可能都不一样，我这里是三个hdfs的包冲突，scope改为provided就行了，也有人说是资源不够也会导致这个问题

```$xslt
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.protocol.FSLimitException$MaxDirectoryItemsExceededException): The directory item limit of /tmp/ha is exceeded: limit=1048576 items=1048576
```

太牛了，吧flink玩挂了，不知道高可用那个目录中都存了什么东西，这么快就达到上限了,一个BUG，原因是开启ha，但是没有显式的指定checkpoint

```$xslt
org.apache.kafka.common.errors.SerializationException: Can't convert value of class [B to class org.apache.kafka.common.serialization.StringSerializer specified in value.serializer
```

哎，习惯了spark的producer，习惯性的加上了key/value.serializer，其实这里可以用加的，下面FlinkKafkaProducer会指定serialization schema

```$xslt
The directory item limit of /tmp/ha is exceeded: limit=1048576 items=1048576
```
开启ha，但是没有显式地指定checkpoint地址

```
java.lang.UnsatisfiedLinkError: org.apache.hadoop.util.NativeCodeLoader.buildSupportsSnappy()Z
```

flink环境无法加载BZip2Codec的包，需要将包打进jar包中，pom中将hdfs-common的scope改为compile

```
could not find implicit value for evidence parameter of type org.apache.flink.api.common.typeinfo.TypeInformation[String]
```
原因是上下文环境找不到TypeInformation对应类型（比如String）的隐式转换，

两个办法，一个是直接 ```import org.apache.flink.api.scala._```

或者创建一个对应类型的TypeInformation，比如我这里用的String类型：```implicit val typeInfo = TypeInformation.of(classOf[(String)])```