# Distributed Runtime Environment

先看看官方的概念：https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/runtime.html

# Job Managers, Task Managers, Clients

![](..\images\57db7218.png)

# Flink社区培训-Flink Runtime 核心机制剖析


[PPT](https://files.alicdn.com/tpsservice/7bb8f513c765b97ab65401a1b78c8cb8.pdf)

[视频回放](https://www.bilibili.com/video/av52394455/) 

# Flink社区培训-Flink 作业执行解析

[PPT](https://files.alicdn.com/tpsservice/0833b0c40a3033c8df0d2ab6e717ea5c.pdf)

[视频回放](https://www.bilibili.com/video/av54603593/) 

# Flink作业执行流程

这里具体的可以看看上面的Flink作业执行流程的视频

这里重点关注Flink-On-YARN的执行流程，详细参考官方的FLIP-6

[FLIP-6 - Flink Deployment and Process Model - Standalone, Yarn, Mesos, Kubernetes, etc.](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077)

[FLIP-6 - 有个哥们翻译的中文版](http://www.whitewood.me/2018/06/17/FLIP6-%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6%E6%A8%A1%E5%9E%8B%E9%87%8D%E6%9E%84/)

![](..\images\e56980c5.png)

![](..\images\4be0ca19.png)

![](..\images\770cb2c6.png)

StreamGraph是对用户逻辑的映射。JobGraph在此基础上进行了一些优化，比如把一部分操作串成chain以提高效率。ExecutionGraph是为了调度存在的，加入了并行处理的概念。而在此基础上真正执行的是Task及其相关结构

还有一个哥们写的一篇文章参考看看，[Flink核心框架的执行流程](https://github.com/bethunebtj/flink_tutorial/blob/master/%E8%BF%BD%E6%BA%90%E7%B4%A2%E9%AA%A5%EF%BC%9A%E9%80%8F%E8%BF%87%E6%BA%90%E7%A0%81%E7%9C%8B%E6%87%82Flink%E6%A0%B8%E5%BF%83%E6%A1%86%E6%9E%B6%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.md)