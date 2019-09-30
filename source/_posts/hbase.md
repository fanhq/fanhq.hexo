---
title: Hbase实践
date: 2019-09-30 08:37:36
tags:
    - 大数据
    - Hbase
---

### Hbase是什么  
HBase是一种构建在HDFS之上的分布式、面向列的存储系统。HBase从另一个角度处理伸缩性问题。它通过线性方式从下到上增加节点来进行扩展。HBase不是关系型数据库，也不支持SQL，但是它有自己的特长，这是RDBMS不能处理的，HBase巧妙地将大而稀疏的表放在商用的服务器集群上。HBase 是Google Bigtable 的开源实现，与Google Bigtable 利用GFS作为其文件存储系统类似， HBase 利用Hadoop HDFS 作为其文件存储系统；Google 运行MapReduce 来处理Bigtable中的海量数据， HBase 同样利用Hadoop MapReduce来处理HBase中的海量数据；Google Bigtable 利用Chubby作为协同服务，HBase利用Zookeeper作为对应。  

### 存储结构  
+ HBase以表的形式存储数据，表有行和列组成，列划分为若干个列族/列簇(column family)

+ 表结构  
    Row Key | column-family1 | column-family2 
    -|-|-
    key1 | f1:列簇1第1个字段;f2:列簇1第2个字段 |  f1:列簇2第1个字段;f2:列簇2第2个字段 
    key2 | f1:列簇1第1个字段,;2:列簇1第2个字段 |  f1:列簇2第1个字段;f2:列簇2第2个字段
<!-- more -->

### HBase集群整体架构

+ 架构图
    ![](/images/e2ea998c0979d447310785c2b98f48b4.png)

+ 核心组件
    * HMaster  
        1. HMaster没有单节点问题，HBase中可以启动多个HMaster，通过Zookeeper的Master Election机制保证总有一个Master在运行，主要负责Table和Region的管理工作。如何启动多个HMaster？通过hbase-daemons.sh启动，步骤如下：1）在hbase/conf目录下编辑backup-masters；2）编辑内容为自己的主机名；3）保存后，执行如下命令：bin/hbase-daemons.sh start master-backup
        2. 管理用户对表的增删改查操作
        3. 管理HRegionServer的负载均衡，调整Region分布（在命令行里面有一个tools，tools这个分组命令其实全部都是Master做的事情
        4. Region Split后，负责新Region的分布
        5. 在HRegionServer停机后，负责失效HRegionServer上Region迁移工作        

    * HRegionServer
        1. 维护HRegion，处理HRegion的IO请求，向HDFS文件系统中读写数据
        2. 负责切分运行过程中变得过大的HRegion
        3. Client访问HBase上数据的过程并不需要Master参与（寻址访问zookeeper和HRegionServer，数据读写访问HRegionServer），HMaster仅仅维护着table和Region的元数据信息，负载很低    

### Hbase集群搭建
Hbase集群依赖hadoop，zookeeper，安装Hbase集群前首先要准备好hadoop和zookeeper集群

+ 服务器列表
    IP | host | 服务  
    -|-| -
    10.19.3.194 | hadoop01 | HMaster
    10.19.3.195 | hadoop02 | HRegionServer
    10.19.3.196 | hadoop03 | HRegionServer 

    注意：在相应的服务器上配置好host

+ 准备工作

    * 下载安装包  
        [Hbase下载](https://hbase.apache.org/downloads.html)

    * 解压安装包  
        tar -zvxf  hbase-2.2.1-bin.tar.gz 

+ 安装步骤

    * 进入安装目录
        ``` shell
            [app@hadoop01 hbase]$ pwd
            /usr/local/hbase
            [app@hadoop01 hbase]$ ls
            bin  CHANGES.md  conf  hbase-webapps  LEGAL  lib  LICENSE.txt  logs  NOTICE.txt  README.txt  RELEASENOTES.md
            [app@hadoop01 hbase]$ 
        ```        
    * 编辑conf目录下的hbase-site.xml
        ```
            <configuration>
                <property>
                    <name>hbase.rootdir</name>
                    <value>hdfs://10.19.3.194:9000/hbase</value>
                </property>

                <property>
                    <name>hbase.zookeeper.quorum</name>
                    <value>10.19.3.194,10.19.3.195,10.19.3.196</value>
                </property>

                <property>
                    <name>hbase.zookeeper.property.clientPort</name>
                    <value>2181</value>
                </property>

                <property>
                    <name>hbase.cluster.distributed</name>
                    <value>true</value>
                </property>

            </configuration>

        ```

    * 在conf目录下的hbase-env.sh末尾增加配置
        ```
            export JAVA_HOME=/opt/jdk1.8.0_131
            export HBASE_MANAGES_ZK=false
        ```
    
    * 编辑conf目录下的regionservers
        ```
            hadoop02
            hadoop03
        ```
    * 将配置好的hbase分别复制到另外两台服务器
        ```
            scp -r hbase app@10.19.3.195:/usr/local/hbase
            scp -r hbase app@10.19.3.196:/usr/local/hbase     
        ```
    * 启动
        ```
            ./bin/start-hbase.sh
        ```
    * 查看相关进程是否正常  
        
        1. hadoop01
        ``` shell
            [app@hadoop01 hbase]$ jps
            24113 SecondaryNameNode
            23878 NameNode
            9398 Jps
            28685 HMaster
            24335 ResourceManager
            [app@hadoop01 hbase]$ 
        ```
        2. hadoop02 
        ```
            [app@hadoop02 hbase]$ jps
            20049 Jps
            16779 DataNode
            9308 HRegionServer
            16910 NodeManager
            [app@hadoop02 hbase]$ 
        ```
        3. hadoop03
        ```
            [app@hadoop03 hbase]$ jps
            20049 Jps
            16779 DataNode
            9308 HRegionServer
            16910 NodeManager
            [app@hadoop03 hbase]$ 
        ```
    * 可以通过访问http://10.19.3.194:16010查看集群信息

+ 验证集群(java代码)

    * 引入pom依赖
        ```
            <dependency>
                <groupId>org.apache.hbase</groupId>
                <artifactId>hbase-client</artifactId>
                <version>2.2.1</version>
            </dependency>
            <dependency>
                <groupId>org.apache.hbase</groupId>
                <artifactId>hbase-server</artifactId>
                <version>2.2.1</version>
            </dependency>
            <dependency>
                <groupId>org.apache.hbase</groupId>
                <artifactId>hbase-common</artifactId>
                <version>2.2.1</version>
            </dependency>
        ```

    * 代码判断是否存在表test_table
    ``` java
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "10.19.3.194,10.19.3.195,10.19.3.196");
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        Connection connection = ConnectionFactory.createConnection(configuration);
        Admin admin = connection.getAdmin();
        TableName tableName = TableName.valueOf("test_table");
        System.out.println(admin.tableExists(tableName));
        admin.close();
        connection.close();
    ```

    * 控制台输出
    ```
        false
    ```

###  附      
    
+ [Hbase官网地址](https://hbase.apache.org/)       
