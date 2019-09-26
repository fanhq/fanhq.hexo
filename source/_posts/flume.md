---
title: Flume实践
date: 2019-09-26 15:18:25
tags:
    - flume
    - kafka
    - hdfs
---

### Flume简介
flume 作为 cloudera 开发的实时日志收集系统，受到了业界的认可与广泛应用。Flume 初始的发行版本目前被统称为 Flume OG（original generation），属于 cloudera。但随着 FLume 功能的扩展，Flume OG 代码工程臃肿、核心组件设计不合理、核心配置不标准等缺点暴露出来，尤其是在 Flume OG 的最后一个发行版本 0.9.4. 中，日志传输不稳定的现象尤为严重，为了解决这些问题，2011 年 10 月 22 号，cloudera 完成了 Flume-728，对 Flume 进行了里程碑式的改动：重构核心组件、核心配置以及代码架构，重构后的版本统称为 Flume NG（next generation）；改动的另一原因是将 Flume 纳入 apache 旗下，cloudera Flume 改名为 Apache Flume。
+ [官方网站](http://flume.apache.org/)
+ [用户文档](http://flume.apache.org/FlumeUserGuide.html)
+ [开发文档](http://flume.apache.org/FlumeDeveloperGuide.html)

### Flume的核心组件
<!-- more -->

![](/images/20190415215813979.png)

+ Source
    Source 负责接收 event 或通过特殊机制产生 event，并将 events 批量的放到一个或多个Channel， Flume提供了各种source的实现，包括Avro Source、 Exce Source、 SpoolingDirectory Source、 NetCat Source、 Syslog Source、 Syslog TCP Source、Syslog UDP Source、 HTTP Source、 HDFS Source， etc

+ Channel
    Flume Channel主要提供一个队列的功能，对source提供中的数据进行简单的缓存。 Flume对于Channel， 则提供了Memory Channel、 JDBC Chanel、 File Channel，etc

+ Sink
    Flume Sink取出Channel中的数据，进行相应的存储文件系统，数据库，或者提交到远程服务器，包括HDFS sink、 Logger sink、 Avro sink、 File Roll sink、 Null sink、 HBasesink、 etc

### 消费kafka推送到HDFS
   
+ 配置文件
    ```
        agent.sources = source_from_kafka
        # channels alias
        agent.channels = mem_channel
        # sink alias
        agent.sinks = hdfs_sink

        # define kafka source
        agent.sources.source_from_kafka.type = org.apache.flume.source.kafka.KafkaSource
        agent.sources.source_from_kafka.channels = mem_channel
        agent.sources.source_from_kafka.batchSize = 1000
        agent.sources.source_from_kafka.kafka.consumer.auto.offset.reset = latest
        # set kafka broker address  
        agent.sources.source_from_kafka.kafka.bootstrap.servers = 127.0.0.1:9092
        # set kafka topic
        agent.sources.source_from_kafka.kafka.topics = intelligence-building
        # set kafka groupid
        agent.sources.source_from_kafka.kafka.consumer.group.id = building-group
                
        # defind hdfs sink
        agent.sinks.hdfs_sink.type = hdfs         
        # specify the channel the sink should use  
        agent.sinks.hdfs_sink.channel = mem_channel
        # set store hdfs path
        agent.sinks.hdfs_sink.hdfs.path = hdfs://127.0.0.1:9000/data/flume/kafka/%Y%m%d   
            
        # set file size to trigger roll
        agent.sinks.hdfs_sink.hdfs.rollSize = 0 
        agent.sinks.hdfs_sink.hdfs.rollCount = 0  
        agent.sinks.hdfs_sink.hdfs.rollInterval = 3600  
        agent.sinks.hdfs_sink.hdfs.threadsPoolSize = 16
        agent.sinks.hdfs_sink.hdfs.fileType = DataStream
        agent.sinks.hdfs_sink.hdfs.writeFormat = Text
        agent.sinks.hdfs_sink.hdfs.callTimeout = 3600000
        agent.sinks.hdfs_sink.hdfs.useLocalTimeStamp = true    
        agent.sinks.hdfs_sink.hdfs.filePrefix = FlumeData_%H
        #agent.sinks.hdfs_sink.hdfs.fileSuffix = .log
        # define channel from kafka source to hdfs sink 
        agent.channels.mem_channel.type = memory
        # channel store size
        agent.channels.mem_channel.capacity = 100000
        # transaction size
        agent.channels.mem_channel.transactionCapacity = 10000
        agent.channels.mem_channel.keep-alive = 60
        agent.channels.mem_channel.capacity = 1000000

    ```

+ 启动脚本
    ```
        ./bin/flume-ng agent --conf ./conf --name agent --conf-file ./conf/flume-hdfs.example -Dflume.root.logger=INFO,console >log.log 2>&1 &
    ```
