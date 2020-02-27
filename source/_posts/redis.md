---
title: redis入门
date: 2020-02-27 16:10:11
tags:
    - redis
---

### 基础数据结构
+ string  
字符串结构使用非常广泛，一个常见的用途就是缓存用户信息。我们将用户信息结构体使用 JSON 序列化成字符串，然后将序列化后的字符串塞进 Redis 来缓存
+ list (列表)  
Redis 的列表相当于 Java 语言里面的 LinkedList，注意它是链表而不是数组。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)，如果再深入一点，你会发现 Redis 底层存储的还不是一个简单的 linkedlist，而是称之为快速链表 quicklist 的一个结构。
+ hash (字典)  
Redis 的字典相当于 Java 语言里面的 HashMap，它是无序字典。内部实现结构上同 Java 的 HashMap 也是一致的，同样的数组 + 链表二维结构。第一维 hash 的数组位置碰撞时，就会将碰撞的元素使用链表串接起来。
+ set (集合)  
Redis 的集合相当于 Java 语言里面的 HashSet，它内部的键值对是无序的唯一的。它的内部实现相当于一个特殊的字典，字典中所有的 value 都是一个值NULL，zset有序列表，zset 内部的排序功能是通过「跳跃列表」数据结构来实现的，它的结构非常特殊，也比较复杂

### 应用场景
+ 分布式锁
    - 加锁  
    ```java
        public boolean tryLock() {
            //随机生成字符串
            String uuid= UUID.randomUUID().toString();
            //获取redis原始链接
            Jedis jedis = redisManager.getJedis();
            //使用setnx命令请求写值，并设置有效时间
            String ret=jedis.set(KEY,uuid,"NX","PX",1000);
            if("OK".equals(ret)){
                //供给unlock时从threadLocal获取此值,ThreadLocal完成数据共享
                local.set(uuid);
                return  true;
            }
            return false;
        }
    ```
    - 解锁
    ```java
        public void unlock() {
            //读取lua脚本
            String script= FileUtils.readFileByLines("lua脚本路径");
            Jedis jedis=redisManager.getJedis();
            //链接redis执行Lua脚本,value值无法从unlock方法入参了，因此用threadLocal来获取
            jedis.eval(script, Arrays.asList(KEY),Arrays.asList(local.get()));
        }  
    ```
    - lua脚本
    ```lua
        if redis.call("get",KEYS[1])==ARGV[1] then
        return redis.call("del",KEYS[1)
        else
        return 0
        end
    ```
