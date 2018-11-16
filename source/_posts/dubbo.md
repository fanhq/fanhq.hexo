---
title: dubbo
date: 2018-11-16 09:21:41
tags:
    - springboot 
    - dubbo
---

### dubbo介绍
Dubbo是 阿里巴巴公司开源的一个高性能优秀的服务框架，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 Spring框架无缝集成。
```
RPC（Remote Procedure Call）—远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC协议假定某些传输协议的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。
RPC采用客户机/服务器模式。请求程序就是一个客户机，而服务提供程序就是一个服务器。首先，客户机调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息到达为止。当一个调用信息到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。
```

### 小试牛刀

* maven项目结构  
```
springboot-dubbo  
    |--dubbo-api      //定义接口 
    |--dubbo-consumer //消费者
    |--dubbo-provider //服务提供者
    |--pom
```
* 提前准备  
dubbo支持很多种注册中心，如multicast、redis、zookeeper等，目前使用较多的是zookeeper，官网也强力推荐。本文也是使用zookeeper作为dubbo的注册中心，这里不再展示zookeeper的安装过程，zookeeper的版本为3.4.12,dubbo的版本为最新的
    ``` 
       <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.5</version>
        </dependency>
    ```


* 定义服务接口
    + 项目结构
        ```
        dubbo-api
            |--src
                |--main
                    |--java
                        |--com.fanhq.dubbo.api.IUserService
            |--pom
        ```
    + 用户服务接口
        ``` java
        public interface IUserService {
            String sayHello(String hello);
        }
        ```

* 服务提供创建
    + 项目结构
        ```
        dubbo-provider
            |--src
                |--main
                    |--java
                        |--com.fanhq.dubbo.provider
                            |--config
                            |--service
            |-- pom
        ```
    + 服务提供配置    
        ``` java
        @Configuration
        public class DubboConfiguration {

            @Bean
            public ApplicationConfig applicationConfig() {
                ApplicationConfig applicationConfig = new ApplicationConfig();
                applicationConfig.setName("provider-demo");
                return applicationConfig;
            }

            @Bean
            public RegistryConfig registryConfig() {
                RegistryConfig registryConfig = new RegistryConfig();
                registryConfig.setAddress("zookeeper://127.0.0.1:2181");
                registryConfig.setClient("curator");
                registryConfig.setProtocol("dubbo");
                return registryConfig;
            }
        }
        ```
    + 服务实现
        ``` java
        @Service(timeout = 5000)
        public class UserServiceImpl implements IUserService {

            @Override
            public String sayHello(String hello) {
                System.out.println(hello);
                return "hello world";
            }
        }
        ```    
        注意这里的@service注解是com.alibaba.dubbo.config.annotation.Service
* 消费者测试    
    这里在消费者项目里面创建单元测试进行测试  
    + 项目结构
        ```
        dubbo-consumer
            |--src
                |--main
                    |--java
                        |--com.fanhq.dubbo.consumer
                            |--config
                    |--test
                        |--com.fanhq.dubbo.consumer 
            |--pom   
       ```
    + 消费项目配置
        ``` java
        @Configuration
        public class DubboConfiguration {

            @Bean
            public ApplicationConfig applicationConfig() {
                ApplicationConfig applicationConfig = new ApplicationConfig();
                applicationConfig.setName("consumer-demo");
                return applicationConfig;
            }

            @Bean
            public ConsumerConfig consumerConfig() {
                ConsumerConfig consumerConfig = new ConsumerConfig();
                consumerConfig.setTimeout(3000);
                return consumerConfig;
            }

            @Bean
            public RegistryConfig registryConfig() {
                RegistryConfig registryConfig = new RegistryConfig();
                registryConfig.setAddress("zookeeper://127.0.0.1:2181");
                registryConfig.setClient("curator");
                return registryConfig;
            }
        }
        ```    
    + 单元测试
        ``` java 
        @RunWith(SpringRunner.class)
        @SpringBootTest(classes = ConsumerApplication.class)
        public class UserServiceTest {

            @Reference
            public IUserService userService;

            @Test
            public void userSeiviceTest(){
                String result = userService.sayHello("hello");
                System.out.println(result);
            }
        }
        ```
###  附     
+ [dubbo官网地址](http://dubbo.apache.org/zh-cn/index.html)
+ [项目gitbub地址](https://github.com/fanhq/springboot-dubbo)