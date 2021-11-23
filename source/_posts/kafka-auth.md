---
title: kafka权限管理
date: 2021-11-23 14:25:33
tags:
    - kakka
---


## kafka权限管理概述
从0.9.0.0版本开始，Kafka社区添加了许多特性，无论是单独使用还是联合使用，都可以提高Kafka集群的安全性。目前支持以下安全策略:

+ SASL/GSSAPI (Kerberos) 
+ SASL/PLAIN
+ SASL/SCRAM-SHA-256 and SASL/SCRAM-SHA-512
+ SASL/OAUTHBEARER

本文主要介绍SCRAM-SHA-512安全管理

## 集群搭建

+ 2.8.0
+ [下载地址](https://apache.claz.org/kafka/2.8.0/kafka_2.13-2.8.0.tgz)

<!-- more -->

### zookeeper安装版本
+ 3.6.3
+ [下载地址](https://apache.claz.org/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz)

## 安装步骤（默认熟悉kafka的安装）
+ 按照上述要求，官网下载kafka、zookeeper对应的版本，并提前安装zookeeper

+ 配置kafka的管理员账号/密码
```$xslt
./kafka-configs.sh --zookeeper localhost:2181  --alter --add-config 'SCRAM-SHA-512=[password=ABCD@1234]' --entity-type users --entity-name admin
```

+ server.properties 配置

```$xslt
#日志策略，保留时间3天，大小1G
log.retention.hours=72
log.retention.bytes=1073741824
#开启授权
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
#配置超级管理员
super.users=User:admin
#配置鉴权方式
listeners=SASL_PLAINTEXT://127.0.0.1:9093
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
```

+ jaas配置
新增配置文件，这里命名未kafka_server_jaas.conf(可自行命名)，文件内容如下
```$xslt
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="ABCD@1234";
};
```

+ 修改启动脚本
指定jaas配置，在原来kafka-server-start.sh最后一行修改为
```$xslt
exec $base_dir/kafka-run-class.sh $EXTRA_ARGS -Djava.security.auth.login.config=$base_dir/../config/kafka_server_jaas.conf kafka.Kafka "$@"
```

+ 启动kafka
```$xslt
 ./kafka-server-start.sh ../config/server.properties &
```

+ 创建用户
    - 命令创建 

    ```
    ./kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --add-config 'SCRAM-SHA-512=[password=xxxxxx]' --entity-type users --entity-name alice
    ```
    - 代码创建
    ```
     Properties props = new Properties();
        props.setProperty(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9093");
        props.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"admin\" password=\"iot@10086\";");
        props.put(AdminClientConfig.SECURITY_PROTOCOL_CONFIG, "SASL_PLAINTEXT");
        props.put(SaslConfigs.SASL_MECHANISM, "SCRAM-SHA-512");
        props.put("request.timeout.ms", 600000);
        AdminClient adminClient = AdminClient.create(props);
     ScramCredentialInfo credentialInfo = new ScramCredentialInfo(ScramMechanism.SCRAM_SHA_512, 4096);
        UserScramCredentialUpsertion userScramCredentialUpsertion = new UserScramCredentialUpsertion("alice", credentialInfo, "xxxxxx");
        AlterUserScramCredentialsResult alterUserScramCredentialsResult = adminClient.alterUserScramCredentials(Arrays.asList(userScramCredentialUpsertion));
        alterUserScramCredentialsResult.all().get();
        alterUserScramCredentialsResult.all().isDone();
    ```
    kafkaAdmin提供了丰富的管理功能，可以自行探索
+ 用户授权
    - topic授权

    ```
    ./kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Alice  --operation Read --operation Write --topic test001
    ```
    
    - group授权

    ```
    ./kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Alice  --operation Read --operation Write --group grp001
    ```

    同样可以通过KafkaAdmin管理，这里就不再进行展示

## 客服端使用示例

+  依赖引入

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.8.0</version>
</dependency>

```

+ 代码样例

```
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "127.0.0.1:9093");
        props.setProperty("group.id", "grp001");
        props.setProperty("enable.auto.commit", "true");
        props.setProperty("auto.commit.interval.ms", "1000");
        props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"alice\" password=\"xxxxxx\";");
        props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, "SASL_PLAINTEXT");
        props.put(SaslConfigs.SASL_MECHANISM, "SCRAM-SHA-512");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("test001"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records){
                System.out.printf("partition =%d offset = %d, key = %s, value = %s%n", record.partition(), record.offset(), record.key(), record.value());
            }
        }
```
