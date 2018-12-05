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
    