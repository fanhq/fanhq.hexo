---
title: Protobuf详解
date: 2018-12-05 14:42:42
tags:
    - protobuf
    - grpc
---

### Protobuf介绍 
Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC 数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式，被广泛应用在网络传输

### Protobuf编码原理

+ Message Buffer  
Message Buffer是指protobuf序列化后的二进制文件格式如下： 
![](/images/66a0e1d3ef88e052865f47b61c990c3.png)  
如图所示，消息经过序列化后会成为一个二进制数据流，该流中的数据为一系列的 Key-Value。protobuf采用Varint编码、ZigZag编码技术，使得这种Key-Pair 结构无需使用分隔符来分割不同的 Field。对于可选的 Field，如果消息中不存在该 field，那么在最终的 Message Buffer 中就没有该 field，这些特性都有助于节约消息本身的大小，protobuf利用巧妙的编码技术，压缩传输的字节数，可大大的提升网络传输效率

<!-- more -->
+ Varint编码
    * 原理介绍  
    varint是一种对数字进行编码的方案，编码后的数据是不定长的，值越小的数字使用越小的字节数，编码后的一般占在1~5个字节。最高位表示是否继续，继续是1，代表后面7位仍然表示数字，否则为0，后面7位用原码补齐，小字节序  
    * 小试牛刀  
    400对应的二进制为00000001 10010000（原码）
    1. 每个字节保留后7位，去掉最高位，有效编码向前移动，生成编码如：0000011 0010000
    2. 因为protobuf使用的是小字节序，所以要把低位字节写到高字节，最后一个字节高位补0，其余各字节高位补1，生成编码如：10010000 0000011
    * Varint编码缺点
    计算机在表示负数的时候，最高位是1，导致使用varint编码的时候，会当作很大的整数处理，从而导致浪费资源，为了解决该问题，protobuf编码引入zigzag编码

+ ZigZag编码

    * 原理介绍  
     Zigzag 编码用无符号数来表示有符号数字，正数和负数交错，如图所示:
     ![](/images/aee146123365b11b43c62190e5e9438.png)  
     使用 zigzag 编码，绝对值小的数字，无论正负都可以采用较少的 byte 来表示，充分利用了 Varint 这种技术。在实际使用过程中，先用zigzag编码后，再对编码后的数据进行varint编码，可以节省很大的空间

+ 字符串类型  
    字符串等则采用类似数据库中的 varchar 的表示方法，即用一个 varint 表示长度，然后将其余部分紧跟在这个长度部分之后即可

+ key的计算方式  

    * wireType

        ![](/images/9303524d3ec2a4cf399ca77c09ec914.png)  

    * 源码展示
    ``` java
    /** Makes a tag value given a field number and wire type. */
    static int makeTag(final int fieldNumber, final int wireType) {
        return (fieldNumber << TAG_TYPE_BITS) | wireType;
    }
    ```
    TAG_TYPE_BITS取值为3，也就是低位为wire_type，高位为field_number，举例说明：age声明为int32，age的field_number=1，所以wire_type =0,所以key=（1<<3 | 0 ）=0x08

### protobuf序列化
+ protobuf文件
    ```
            syntax ="proto3";
            package com.simple;
            option java_package="com.simple";
            option java_outer_classname="Person";
            message Person{
                int32 age= 1;
            }

    ```

+ 序列化
    ``` java
    Person.Builder builder = Person.newBuilder();
    builder.setAge(18);
    Person person =builder.build();
    byte[] byteArray = person.toByteArray();
    FileOutputStream outstream = new FileOutputStream(new File("Person.txt"));
    outstream.write(byteArray);
    outstream.close();
    ```
    打开Person.txt，使用十六进制查看：08 12

### grpc应用

+ 概要  

    protobuf提供了maven插件，可以利用插件生成对应的文件，如Java，可以生成对应的Java类，具体使用方法，这里不再累赘介绍

+ protobuf文件
    ``` 
        syntax = "proto3";
        option java_multiple_files = true;
        option java_package = "io.grpc.examples.helloworld";
        option java_outer_classname = "HelloWorldProto";
        option objc_class_prefix = "HLW";

        package helloworld;

        // The greeting service definition.
        service Greeter {
        // Sends a greeting
        rpc SayHello (HelloRequest) returns (HelloReply) {}
        }

        // The request message containing the user's name.
        message HelloRequest {
        string name = 1;
        }

        // The response message containing the greetings
        message HelloReply {
        string message = 1;
        }
    ```
+ maven依赖
    ```
         <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.5.0</version>
        </dependency>
    ```

+ maven配置protobuf插件  
    ```
        <!-- protobuf 编译组件 -->
        <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.5.1</version>
                <extensions>true</extensions>
                <configuration>
                    <pluginId>grpc-java</pluginId>
                    <protocArtifact>com.google.protobuf:protoc:3.5.0:exe:${os.detected.classifier}</protocArtifact>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.16.1:exe:${os.detected.classifier}</pluginArtifact>
                    <protoSourceRoot>${project.basedir}/src/main/resources/proto</protoSourceRoot>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
         </plugin>
    ```

### 附
+ [grpc官网地址](https://grpc.io/about/)
+ [protobuf官网地址](https://developers.google.com/protocol-buffers/)