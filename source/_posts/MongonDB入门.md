---
title: MongonDB入门
date: 2018-12-05 08:46:11
tags:
    - mongondb
    - springboot
---

### MongonDB介绍
MongoDB 是一个基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

### MongonDB概念介绍

+ database mongon里面数据库的概念和其他关系型数据库一样，如mysql、oracle等

+ collection mongon里面collection类似于关系型数据库中的表，一类数据的集合，和其他数据库的区别就是数据库字段可以不固定

+ document mongon本身是面向文档的存储的介于关系与非关系型数据库，这里的document类似于mysql中的一条记录

+ field 数据库字段，和关系型数据一样

### MongonDB使用

+ springboot

    * 添加pom依赖
    ```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
    ```

    * 创建数据库实体
    ``` java
    @Document(collection = "user")    
    public class User {

        @Id
        private ObjectId id; //数据库id 与数据库字段_id对应,org.bson.types.ObjectId

        @Field(value = "user_name")
        private String username;

        @Field(value = "age")
        private Integer age;

        // 省略getter和setter
   }
    ```
    * 添加mongon配置
    ```
    spring:
        data:
            mongodb:
                host: localhost
                port: 26007
                database: test 
    ```

    * 创建repository
    ``` java
    public interface UserRepository extends MongoRepository<User, ObjectId>{
    }
    ```
    * 单元测试
    ``` java 

    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringApplicationConfiguration(Application.class)
    public class ApplicationTests {

        @Autowired
        private UserRepository userRepository;

        @Before
        public void setUp() {
            userRepository.deleteAll();
        }

        @Test
        public void test() throws Exception {

            // 创建user,并保存
            User user = new User();
            user.setAge(10)
            user.setUsername("hachel")
            userRepository.save(user);
          
            // 删除一个User，再验证User总数
            u = userRepository.findByUsername("hachel");
            userRepository.delete(u);
        
        }

    }
    ```