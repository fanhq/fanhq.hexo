---
title: SpringCloud-Zookeeper
date: 2019-09-09 13:52:16
tags:
    - java
    - springcloud
    - zookeeper
---

### Spring Cloud介绍
Spring Cloud是一个基于SpringBoot实现的微服务架构开发工具。它为微服务架构中涉及的配置管理、服务治理、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式，基于http协议开发的开箱即用的微服务架构

### 注册中心
Spring Cloud 早期版本的注册中心主要使用Eureka，但在2.0过后，Netflix不再对Eureka更新维护，但是注册中心有很多实现方式，Spring Cloud Consul是其中一种，本文主要介绍Spring Cloud Zookeeper，在实际使用中也是遇到了很多坑。Zookeeper是一个高性能，分布式的，开源分布式应用协调服务。它提供了简单原始的功能，分布式应用可以基于它实现更高级的服务，比如同步，配置管理，集群管理，名空间。它被设计为易于编程，使用文件系统目录树作为数据模型

### 小试牛刀

* maven项目结构  
```
springboot-sample  
    |--springcloud-feign //消费者
    |--springcloud-zookeeper //服务提供者
    |--pom
```
<!-- more -->

* 服务提供者

     + 项目结构
        ```
        springcloud-zookeeper
            |--src
                |--main
                    |--java
                        |--com.fanhq.example
                            |--config
                            |--controller
             --pom        
       ``` 
    +  pom依赖
       ```
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
                <exclusions>
                    <exclusion>
                        <groupId>org.apache.curator</groupId>
                        <artifactId>curator-recipes</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
            </dependency>
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-recipes</artifactId>
                <version>4.2.0</version>
                <exclusions>
                    <exclusion>
                        <groupId>org.apache.zookeeper</groupId>
                        <artifactId>zookeeper</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.apache.zookeeper</groupId>
                <artifactId>zookeeper</artifactId>
                <version>3.5.5</version>
            </dependency>
        </dependencies>
       ```
       注意：pom依赖这样调整可以根据自己zookeeper的版本进行调整，不建议使用官网的方式
    + Java配置
      ```
        @Component
        public class RegistrationConfig {

            @Autowired
            private AbstractAutoServiceRegistration serviceRegistration;

            @EventListener(WebServerInitializedEvent.class)
            public void register(WebServerInitializedEvent event) {
                serviceRegistration.bind(event);
            }
        }
      ```
    注意：这里用的最新的springcloud版本进行踩坑，按照官网的例子，springcloud不会去zookeeper注册服务，通过阅读源码添加以上配置触发注册的事件

* 服务消费者(feign)

    + 项目结构
        ```
        springcloud-feign
            |--src
                |--main
                    |--java
                        |--com.fanhq.example
                            |--client
                            |--consumer
             --pom        
       ``` 
    +  pom依赖
       ```
            <dependencies>
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
                    <exclusions>
                        <exclusion>
                            <groupId>org.apache.curator</groupId>
                            <artifactId>curator-recipes</artifactId>
                        </exclusion>
                    </exclusions>
                </dependency>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </dependency>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-actuator</artifactId>
                </dependency>
                <dependency>
                    <groupId>org.apache.curator</groupId>
                    <artifactId>curator-recipes</artifactId>
                    <version>4.2.0</version>
                    <exclusions>
                        <exclusion>
                            <groupId>org.apache.zookeeper</groupId>
                            <artifactId>zookeeper</artifactId>
                        </exclusion>
                    </exclusions>
                </dependency>
                <dependency>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                    <version>3.5.5</version>
                </dependency>
            </dependencies>
       ```


###  附     
+ [项目gitbub地址](https://github.com/fanhq/springcloud-sample)