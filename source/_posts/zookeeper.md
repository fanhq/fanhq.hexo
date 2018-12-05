---
title: zookeeper
date: 2018-11-01 14:22:49
tags: 
    - zookeeper 
    - Curator
---
##### zookeeper 简介
ZooKeeper（动物园管理员），顾名思义，是用来管理Hadoop（大象）、Hive（蜜蜂）、Pig（小猪）的管理员，同时Apache HBase、Apache Solr、LinkedIn Sensei等众多项目中都采用了ZooKeeper。
ZooKeeper曾是Hadoop的正式子项目，后发展成为Apache顶级项目，与Hadoop密切相关但却没有任何依赖。它是一个针对大型应用提供高可用的数据管理、应用程序协调服务的分布式服务框架，基于对Paxos算法的实现，使该框架保证了分布式环境中数据的强一致性，提供的功能包括：配置维护、统一命名服务、状态同步服务、集群管理等。
在分布式应用中，由于工程师不能很好地使用锁机制，以及基于消息的协调机制不适合在某些应用中使用，因此需要有一种可靠的、可扩展的、分布式的、可配置的协调机制来统一系统的状态。Zookeeper的目的就在于此

ZooKeeper可以理解为类似redis的缓存数据库，只是相对于redis存储数据量小，额外增加了存储节点的机制，
常用于分布式协调服务


##### zookeeper客户端 - Curator

+ Curator简介  
  Apache Curator is a Java/JVM client library for Apache ZooKeeper, a distributed coordination service. It includes a highlevel API framework and utilities to make using Apache ZooKeeper much easier and more reliable. It also includes recipes for common use cases and extensions such as service discovery
<!-- more -->

+ Curator常用api  
  * 创建客户端  
    ```
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder()
                                .connectString("127.0.0.1:2181")
                                .sessionTimeoutMs(5000)
                                .connectionTimeoutMs(5000)
                                .retryPolicy(retryPolicy)
                                .build();
    client.start();
    ```
  * 创建节点数据  
    ```//创建节点
       //PERSISTENT：持久化 默认模式
       //PERSISTENT_SEQUENTIAL：持久化并且带序列号
       //EPHEMERAL：临时
       //EPHEMERAL_SEQUENTIAL：临时并且带序列号
       //创建节点并递归创建父节点，并指定创建模式
       client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/data/path", "hello".getBytes());
    ```
  * 删除节点数据  
    ```
    client.delete().deletingChildrenIfNeeded().forPath("/data/path");
    ```
  * 更新节点数据
    ```
     client.setData().forPath("/data/path", "world".getBytes());
    ```
  * 查询节点数据
    ```
    byte[] data = client.getData().forPath("/data/path");//获取指定节点数据
    List<String> childs = client.getChildren().forPath("/")//获取子节点
    ```
+ Curator事件(cache)  

  ZooKeeper原生支持通过注册Watcher来进行事件监听，但是其使用并不是特别方便，需要开发人员自己反复注册Watcher，比较繁琐。Curator引入了Cache来实现对ZooKeeper服务端事件的监听。Cache是Curator中对事件监听的包装，其对事件的监听其实可以近似看作是一个本地缓存视图和远程ZooKeeper视图的对比过程。同时Curator能够自动为开发人员处理反复注册监听，从而大大简化了原生API开发的繁琐过程
  
  * 事件监听示例代码
    ```
    
    private static final String PATH_CACHE = "/example/pathCache";

    private static final String NODE_CACHE = "/example/nodeCache";

    private static final String TREE_CACHE = "/example/treeCache";

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();

        //PathChildrenCache
        System.out.println("=========================PathChildrenCache==========================");
        PathChildrenCache pCache = new PathChildrenCache(client, PATH_CACHE, true);
        pCache.start();
        pCache.getListenable().addListener((c, e) -> {
            System.out.println("事件类型：" + e.getType());
            if (null != e.getData()) {
                System.out.println("节点数据：" + e.getData().getPath() + " = " + new String(e.getData().getData()));
            }
        });
        client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test01", "01".getBytes());
        Thread.sleep(100);
        client.create().creatingParentsIfNeeded().forPath("/example/pathCache/test02", "02".getBytes());
        Thread.sleep(100);
        client.setData().forPath("/example/pathCache/test01", "01_V2".getBytes());
        Thread.sleep(100);
        for (ChildData data : pCache.getCurrentData()) {
            System.out.println("getCurrentData:" + data.getPath() + " = " + new String(data.getData()));
        }
        client.delete().forPath("/example/pathCache/test01");
        Thread.sleep(100);
        client.delete().forPath("/example/pathCache/test02");
        Thread.sleep(2000);
        pCache.close();

        //Node Cache
        System.out.println("=========================NodeCache==========================");
        client.create().creatingParentsIfNeeded().forPath(NODE_CACHE);
        NodeCache nCache = new NodeCache(client, NODE_CACHE);
        nCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                ChildData data = nCache.getCurrentData();
                if (null != data) {
                    System.out.println("节点数据：" + new String(nCache.getCurrentData().getData()));
                } else {
                    System.out.println("节点被删除!");
                }
            }
        });
        nCache.start();
        client.setData().forPath(NODE_CACHE, "01".getBytes());
        Thread.sleep(100);
        client.setData().forPath(NODE_CACHE, "02".getBytes());
        Thread.sleep(100);
        client.delete().deletingChildrenIfNeeded().forPath(NODE_CACHE);
        Thread.sleep(2000);
        nCache.close();

        //Tree cache
        System.out.println("=========================TreeCache==========================");
        client.create().creatingParentsIfNeeded().forPath(TREE_CACHE);
        TreeCache cache = new TreeCache(client, TREE_CACHE);
        cache.getListenable().addListener((c, e) ->
                System.out.println("事件类型：" + e.getType() + " | 路径：" + (null != e.getData() ? e.getData().getPath() : null)));
        cache.start();
        client.setData().forPath(TREE_CACHE, "01".getBytes());
        Thread.sleep(100);
        client.setData().forPath(TREE_CACHE, "02".getBytes());
        Thread.sleep(100);
        client.delete().deletingChildrenIfNeeded().forPath(TREE_CACHE);
        Thread.sleep(1000 * 2);
        cache.close();

        client.close();

    }
    ```
   * 控制台输出展示
    ```
   =========================PathChildrenCache==========================
   事件类型：CONNECTION_RECONNECTED
   事件类型：CHILD_ADDED
   节点数据：/example/pathCache/test01 = 01
   事件类型：CHILD_ADDED
   节点数据：/example/pathCache/test02 = 02
   事件类型：CHILD_UPDATED
   节点数据：/example/pathCache/test01 = 01_V2
   getCurrentData:/example/pathCache/test01 = 01_V2
   getCurrentData:/example/pathCache/test02 = 02
   事件类型：CHILD_REMOVED
   节点数据：/example/pathCache/test01 = 01_V2
   事件类型：CHILD_REMOVED
   节点数据：/example/pathCache/test02 = 02
   =========================NodeCache==========================
   节点数据：01
   节点数据：02
   节点被删除!
   =========================TreeCache==========================
   事件类型：NODE_ADDED | 路径：/example/treeCache
   事件类型：INITIALIZED | 路径：null
   事件类型：NODE_UPDATED | 路径：/example/treeCache
   事件类型：NODE_UPDATED | 路径：/example/treeCache
   事件类型：NODE_REMOVED | 路径：/example/treeCache
    ```
+ zookeeper选举机制
  
  * 流程示意图  
    
    ![](/images/316c0262c7de86fbfe2cb9d062a17b0.png)
  
  * 流程解析 
    - zookeeper提供三种选举机制：LeaderElection,AuthFastLeaderElection，FastLeaderElection。默认采用的机制是FastLeaderElection,本文主要分析该机制。举例描述之前先明白几个概念：
    1. 服务器ID:比如有三台服务器，编号分别是1,2,3, 值编号越大在选择算法中的权重越大
    2. 数据ID:服务器中存放的最大数据ID,值越大说明数据越新，在选举算法中数据越新权重越大
    3. 逻辑时钟：或者叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值做出不同的判断
    4. 选举状态:LOOKING，竞选状态;FOLLOWING，随从状态，同步leader状态，参与投票;OBSERVING，观察状态,同步leader状态，不参与投票;LEADING，领导者状态  

    选举完成后会将以上信息发给集群中的每个节点,默认是采用投票数大于半数则胜出的逻辑，所以zookeeper集群的节点数一般都是单数
  
    - 假设zookeeper集群有五个实例，FastLeaderElection选举的流程如下：  
     1. 服务器1启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于Looking
     2. 服务器2启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务器2胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING
     3. 服务器3启动，给自己投票，同时与之前启动的服务器1,2交换信息，由于服务器3的编号最大所以服务器3胜出，此时投票数正好大于半数，所以服务器3成为领导者，服务器1,2成为小弟
     4.  服务器4启动，给自己投票，同时与之前启动的服务器1,2,3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4只能成为小弟
     5. 服务器5启动，后面的逻辑同服务器4成为小弟


+ zookeeper 分布式锁应用

    * zookeeper特性
        1. 有序节点：假如当前有一个父节点为/lock，我们可以在这个父节点下面创建子节点；zookeeper提供了一个可选的有序特性，例如我们可以创建子节点“/lock/node-”并且指明有序，那么zookeeper在生成子节点时会根据当前的子节点数量自动添加整数序号，也就是说如果是第一个创建的子节点，那么生成的子节点为/lock/node-0000000000，下一个节点则为/lock/node-0000000001，依次类推。
        2. 临时节点：客户端可以建立一个临时节点，在会话结束或者会话超时后，zookeeper会自动删除该节点。
        3. 事件监听：在读取数据时，我们可以同时对节点设置事件监听，当节点数据或结构变化时，zookeeper会通知客户端。当前zookeeper有如下四种事件：1）节点创建；2）节点删除；3）节点数据修改；4）子节点变更
   
    * 分布式锁原理
        1. 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推。
        2. 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听刚好在自己之前一位的子节点删除消息，获得子节点变更通知后重复此步骤直至获得锁；
        3. 执行业务代码；
        4. 完成业务流程后，删除对应的子节点释放锁

    * 基于Curator分布式锁代码展示

    ```
    //创建客户端
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client =
            CuratorFrameworkFactory.builder()
                    .connectString("127.0.0.1:2181")
                    .sessionTimeoutMs(5000)
                    .connectionTimeoutMs(5000)
                    .retryPolicy(retryPolicy)
                    .build();
    client.start();
    //创建分布式锁, 锁空间的根节点路径为/curator/lock
    InterProcessMutex mutex = new InterProcessMutex(client, "/curator/lock");
    mutex.acquire();
    //获得了锁, 进行业务流程
    //todo
    //完成业务流程, 释放锁
    mutex.release();
    
    //关闭客户端
    client.close();

    ```
