---
title: JVM参数
date: 2019-03-25 14:02:59
tags:
    - jvm
    - java
---

### jvm参数简介
-XX 参数被称为不稳定参数，之所以这么叫是因为此类参数的设置很容易引起JVM 性能上的差异，使JVM 存在极大的不稳定性。如果此类参数设置合理将大大提高JVM 的性能及稳定性。

不稳定参数语法规则：
```
    1.布尔类型参数值
        -XX:+<option> '+'表示启用该选项
        -XX:-<option> '-'表示关闭该选项
    2.数字类型参数值：
        -XX:<option>=<number> 给选项设置一个数字类型值，可跟随单位，例如：'m'或'M'表示兆字节;'k'或'K'千字节;'g'或'G'千兆字节。32K与32768是相同大小的。
    3.字符串类型参数值：
        -XX:<option>=<string> 给选项设置一个字符串类型值，通常用于指定一个文件、路径或一系列命令列表。 例如：-XX:HeapDumpPath=./dump.core
```
<!-- more -->
### 项目中常用配置
| 参数设置 | 描述 | 配置格式 |
| ------ | ------ | ------ |
| -Xms | 初始化堆空间大小 | -Xms64m |
| -Xmx | 最大堆空间大小 | -Xmx128m |
| -Xmn | 年轻代的空间大小 | -Xmn32m |
| -Xss | 设置线程栈空间大小 | -Xss512k |
| -XX:PermSize|永久代空间大小（jdk8已废弃）|-XX:PermSize=256m|
|-XX:MaxPermSize|最大永久区大小|-XX:MaxPermSize=256m|
|-XX:+UseStringCache|启用缓存常用字符产|-|
|-XX:+UseConcMarkSweepGC|老年代使用cms收集器|-|
|-XX:+UseParNewGC|新生代使用并行收集器|-|
|-XX:+ParallelGCThreads|设置并行线程数量|-XX:+ParallelGCThreads=4|
|-XX:+CMSClassUnloadingEnabaled|允许对类元数据进行清理|-|
|-XX:+DisableExplicitGC|禁止显示GC|-|
|-XX:+UseCMSInitiatingOccupancyOnly|表示达到阀值之后才进行CMS回收|-|
|-XX:+CMSInitiatingOccupanyFraction|设置CMS老年代回收阀值百分比|-XX:+CMSInitiatingOccupanyFraction=68|
|-verbose:gc|输出虚拟机GC详情|-|
|-XX:+PrintGCDetails|打印GC详情|-|
|-XX:+PrintGCDateStamps|打印GC的耗时|
|-XX:+PrintTenuringDistribution|打印tenuring年龄信息|-|
|-XX:+HeapDumpOnOutOfMemoryError|当抛出OOM时进行HeapDump|-|
|-XX:+HeapDumpPath|指定HeapDump的文件路径和目录|-|

### 常用组合
| Young | Old | JVM Option |
| ------ | ------ | ------ |
|Serial|Serial|-XX:+UserSerialGC|
|Parallel|Parallel/Serial|-XX:+UseParallelGC  -XX:+UseParallelOldGC|
|Serial/Parnllel|CMS|-XX:+UseParNewGC -XX:+UseConcSweepGC|
|G1|-|-XX:+UseG1GC|

### 常用GC调用策略

 + GC 调优原则
 ```
    多数的 Java 应用不需要在服务器上进行 GC 优化；多数导致 GC 问题的 Java 应用，都不是因为我们参数设置错误，而是代码问题；在应用上线之前，先考虑将机器的 JVM 参数设置到最优（最适合）；减少创建对象的数量；减少使用全局变量和大对象；GC 优化是到最后不得已才采用的手段；在实际使用中，分析 GC 情况优化代码比优化 GC 参数要多得多。
 ```

  + GC 调优目的

```
     GC低停顿；GC低频率；低内存占用；高吞吐量。
```
+ GC调优策略  
  - 将新对象预留在新生代，由于 Full GC 的成本远高于 Minor GC，因此尽可能将对象分配在新生代是明智的做法，实际项目中根据 GC 日志分析新生代空间大小分配是否合理，适当通过“-Xmn”命令调节新生代大小，最大限度降低新对象直接进入老年代的情况  
  - 大对象进入老年代，虽然大部分情况下，将对象分配在新生代是合理的。但是对于大对象这种做法却值得商榷，大对象如果首次在新生代分配可能会出现空间不足导致很多年龄不够的小对象被分配的老年代，破坏新生代的对象结构，可能会出现频繁的 full gc。因此，对于大对象，可以设置直接进入老年代（当然短命的大对象对于垃圾回收老说简直就是噩梦）。-XX:PretenureSizeThreshold 可以设置直接进入老年代的对象大小
  - 合理设置进入老年代对象的年龄，-XX:MaxTenuringThreshold 设置对象进入老年代的年龄大小，减少老年代的内存占用，降低 full gc 发生的频率。
  - 设置稳定的堆大小，堆大小设置有两个参数：-Xms 初始化堆大小，-Xmx 最大堆大小。
  - 如果满足下面的指标，则一般不需要进行 GC 优化：
  ```
    MinorGC 执行时间不到50ms；
    Minor GC 执行不频繁，约10秒一次；
    Full GC 执行时间不到1s；
    Full GC 执行频率不算频繁，不低于10分钟1次。
  ```

### jvm常用命令
  + jps
    jps：显示当前用户的所有java进程的PID  
    jps -v 3331：显示虚拟机参数  
    jps -m 3331：显示传递给main()函数的参数  
    jps -l 3331：显示主类的全路径  

  + jstat 
    jstat -gc 3331 250 20 ：查询进程2764的垃圾收集情况，每250毫秒查询一次，一共查询20次  
    jstat -gccause：额外输出上次GC原因  
    jstat -calss：件事类装载、类卸载、总空间以及所消耗的时间  

  + jmap
    jmap -histo 3331：查看堆内存(histogram)中的对象数量及大小  
    jmap -heap 3331：查看java 堆（heap）使用情况  
    jmap -histo:live 3331：JVM会先触发gc，然后再统计信息，查看堆内存中的对象数量及大小  
    jmap -dump:format=b,file=heapDump 3331：将内存使用的详细情况输出到文件，之后一般使用其他工具进行分析  
    
  + jstack 
    jstack 3331：查看线程情况  
    jstack -F 3331：正常输出不被响应时，使用该指令  
    jstack -l 3331：除堆栈外，显示关于锁的附件信息  

### 常见问题定位过程

1. 使用uptime查看当前load，发现load飙高
    ```
    ➜  ~ uptime
    13:29  up 23:41, 3 users, load averages: 10 10 10
    ```

2. 使用top命令，查看占用CPU较高的进程ID
    ```
    ➜  ~ top
    PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
    1893 admin     20   0 7127m 2.6g  38m S 181.7 32.6  10:20.26 java
    ```
    发现PID为1893的进程占用CPU 181%,而且是一个Java进程，基本断定是软件问题
3. 使用 top命令，查看具体是哪个线程占用率较高
    ```
    ➜  ~ top -Hp 1893
    PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
    4519 admin     20   0 7127m 2.6g  38m R 18.6 32.6   0:40.11 java
    ```
4. 使用printf命令查看这个线程的16进制
    ```
    ➜  ~ printf %x 4519
    11a7
    ```
5. 使用jstack命令查看当前线程正在执行的方法 
    ```
    ➜  ~ jstack 1893 |grep -A 200 11a7
    ```
6. 使用jmap内存使用的详细情况输出到文件，之后一般使用其他工具进行分    析
   ```
   ➜ ~ jmap -dump:format=b,file=heapDump 1893
   ```
    + [在线分析地址](http://heaphero.io/index.jsp)
    + [在线生成jvm参数](http://xxfox.perfma.com)
