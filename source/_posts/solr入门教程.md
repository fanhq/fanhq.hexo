---
title: solr入门教程
date: 2018-11-07 14:34:18
tags:
    - solr
    - lucene
---

### solr简介
Solr是Apache下的一个顶级开源项目，采用Java开发，它是基于Lucene的全文搜索服务器。Solr提供了比Lucene更为丰富的查询语言，同时实现了可配置、可扩展，并对索引、搜索性能进行了优化

### solr安装
+ 官网下载安装包，最新的版本是7.5，点击[下载地址](http://lucene.apache.org/solr/downloads.html)进行下载，并解压文件，进入所解压的安装目录(window同理)    
``` shell
~$ ls solr*
solr-7.5.0.zip

~$ unzip -q solr-7.5.0.zip

~$ cd solr-7.5.0/

```
+ 启动solr  
   Unix or MacOS:
    ``` shell
    ~$ bin/solr start -e cloud
    ```  
   windows:
   ``` shell
   bin\solr.cmd start -e cloud
  ```
  ``` shell
    solr-7.5.0:$ ./bin/solr start -e cloud
    Welcome to the SolrCloud example!

    This interactive session will help you launch a SolrCloud cluster on your local workstation.
    To begin, how many Solr nodes would you like to run in your local cluster? (specify 1-4 nodes) [2]:
  ```
  输入本地集群启动安装的节点数量，默认为2
  ``` 
  Ok, let's start up 2 Solr nodes for your example SolrCloud cluster.
  Please enter the port for node1 [8983]:
  ```
  输入第一个节点的服务端口号，默认8983
  ```
  Please enter the port for node2 [7574]: 
  ```
  输入第二个节点的端口号，默认7574
  ```
    Starting up 2 Solr nodes for your example SolrCloud cluster.

    Creating Solr home directory /solr-7.5.0/example/cloud/node1/solr
    Cloning /solr-7.5.0/example/cloud/node1 into
    /solr-7.5.0/example/cloud/node2

    Starting up Solr on port 8983 using command:
    "bin/solr" start -cloud -p 8983 -s "example/cloud/node1/solr"

    Waiting up to 180 seconds to see Solr running on port 8983 [\]
    Started Solr server on port 8983 (pid=34942). Happy searching!


    Starting up Solr on port 7574 using command:
    "bin/solr" start -cloud -p 7574 -s "example/cloud/node2/solr" -z localhost:9983

    Waiting up to 180 seconds to see Solr running on port 7574 [\]
    Started Solr server on port 7574 (pid=35036). Happy searching!

    INFO  - 2017-07-27 12:28:02.835; org.apache.solr.client.solrj.impl.ZkClientClusterStateProvider; Cluster at localhost:9983 ready
  ```
  这个时候，一个两个节点的solr集群就已经启动完成，solr依靠zookeeper协调管理各个节点，solr会默认启动一个嵌套的zookeeper,启动完成后，会提示你创建集合，熟悉lucene的可以理解文创建document，用来创建数据的索引
  ```
   Now let's create a new collection for indexing documents in your 2-node cluster.
    Please provide a name for your new collection: [gettingstarted]

  ```
  我这里输入techproducts,和官网一致
  ```
  How many shards would you like to split techproducts into? [2]
  ```
  这里默认值为2，为了让索引均匀的分布再每个节点上

  ```
   How many replicas per shard would you like to create? [2]
  ```
  这里输入所以备份数量
  ```
  Please choose a configuration for the techproducts collection, available options are:
  _default or sample_techproducts_configs [_default]  
  ```
  输入配置文件夹名称,主要的配置文件有schema.xml和solrconfig.xml，并在solr-7.5.0\server\solr\configsets文件夹下创建相应的文件夹
  ```
  Uploading /solr-7.5.0/server/solr/configsets/_default/conf for config techproducts to ZooKeeper at localhost:9983

    Connecting to ZooKeeper at localhost:9983 ...
    INFO  - 2017-07-27 12:48:59.289; org.apache.solr.client.solrj.impl.ZkClientClusterStateProvider; Cluster at localhost:9983 ready
    Uploading /solr-7.5.0/server/solr/configsets/sample_techproducts_configs/conf for config techproducts to ZooKeeper at localhost:9983

    Creating new collection 'techproducts' using command:
    http://localhost:8983/solr/admin/collections?action=CREATE&name=techproducts&numShards=2&replicationFactor=2&maxShardsPerNode=2&collection.configName=techproducts

    {
    "responseHeader":{
        "status":0,
        "QTime":5460},
    "success":{
        "192.168.0.110:7574_solr":{
        "responseHeader":{
            "status":0,
            "QTime":4056},
        "core":"techproducts_shard1_replica_n1"},
        "192.168.0.110:8983_solr":{
        "responseHeader":{
            "status":0,
            "QTime":4056},
        "core":"techproducts_shard2_replica_n2"}}}

    Enabling auto soft-commits with maxTime 3 secs using the Config API

    POSTing request to Config API: http://localhost:8983/solr/techproducts/config
    {"set-property":{"updateHandler.autoSoftCommit.maxTime":"3000"}}
    Successfully set-property updateHandler.autoSoftCommit.maxTime to 3000

   SolrCloud example running, please visit: http://localhost:8983/solr
  ```
  至此，solrc创建techproduct 索引已经完成，可以访问solr的管理界面[http://localhost:8983/solr](http://localhost:8983/solr)，接下来可以向solr添加需要搜索的数据，生成反向索引数据

  Linux/Mac:  
  ```
  solr-7.5.0:$ bin/post -c techproducts example/exampledocs/*
  ```
  windows:  
  ```
   C:\solr-7.5.0> java -jar -Dc=techproducts -Dauto example\exampledocs\post.jar example\exampledocs\*
  ```
  example\exampledocs下的文件是solr提供测试演示的文档，正式使用的时候可以是我们自己的所需要分析的数据

  ```
    SimplePostTool version 5.0.0
    Posting files to [base] url http://localhost:8983/solr/techproducts/update...
    Entering auto mode. File endings considered are xml,json,jsonl,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log
    POSTing file books.csv (text/csv) to [base]
    POSTing file books.json (application/json) to [base]/json/docs
    POSTing file gb18030-example.xml (application/xml) to [base]
    POSTing file hd.xml (application/xml) to [base]
    POSTing file ipod_other.xml (application/xml) to [base]
    POSTing file ipod_video.xml (application/xml) to [base]
    POSTing file manufacturers.xml (application/xml) to [base]
    POSTing file mem.xml (application/xml) to [base]
    POSTing file money.xml (application/xml) to [base]
    POSTing file monitor.xml (application/xml) to [base]
    POSTing file monitor2.xml (application/xml) to [base]
    POSTing file more_books.jsonl (application/json) to [base]/json/docs
    POSTing file mp500.xml (application/xml) to [base]
    POSTing file post.jar (application/octet-stream) to [base]/extract
    POSTing file sample.html (text/html) to [base]/extract
    POSTing file sd500.xml (application/xml) to [base]
    POSTing file solr-word.pdf (application/pdf) to [base]/extract
    POSTing file solr.xml (application/xml) to [base]
    POSTing file test_utf8.sh (application/octet-stream) to [base]/extract
    POSTing file utf8-example.xml (application/xml) to [base]
    POSTing file vidcard.xml (application/xml) to [base]
    21 files indexed.
    COMMITting Solr index changes to http://localhost:8983/solr/techproducts/update...
    Time spent: 0:00:00.822
  ```
  到这里，我们的solr加入了分析的数据，并且solr会根据文件的特性，生成默认的field

+ 小试牛刀

   solr提供界面进行搜索，操作简单，同事提供接口对外进行搜索，这里展示接口的方式搜索

   * Search for a Single Term  
   根据关键字foundation进行搜索
   ```
    curl "http://localhost:8983/solr/techproducts/select?q=foundation"
   ```
   * Field Searches  
   搜索字段cat并等于electronics的文件
   ```
    curl "http://localhost:8983/solr/techproducts/select?q=cat:electronics"
   ```
   * Phrase Search   
   搜索包含CAS latency的文件
    ```
    curl "http://localhost:8983/solr/techproducts/select?q=\"CAS+latency\""
   ```
