---
title: Springboot-Kafka
date: 2019-10-18 09:30:51
tags:
    - java
    - springboot
    - kafka
---

### kafka介绍
Apache Kafka是一个分布式的发布-订阅消息系统，能够支撑海量数据的数据传递，在离线和实时的消息处理业务系统中，Kafka都有广泛的应用

### springboot快速集成

+ pom
    
    ```
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.kafka</groupId>
			<artifactId>spring-kafka</artifactId>
		</dependency>
    ```
<!-- more -->
+ 配置文件application.properties
    ```
        spring.kafka.bootstrap-servers=10.19.16.67:9092,10.19.16.68:9092,10.19.16.69:9092
        spring.kafka.consumer.group-id=myGroup
        spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
        spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
        spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
        spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
    ```
+ 消费者
    ``` java

        @Component
        public class Consumer {
            @KafkaListener(topics = "topic_test")
            public void listener(ConsumerRecord<String, String> record) {
                String value = record.value();
                log.info("【receive】:{}", value);
            }
        }
    ```
+ 生产者
    ``` java
        @Component
        public class Producer {

            @Autowired
            private KafkaTemplate<String, String> kafkaTemplate;

            public void sendMessage(String message) {
                kafkaTemplate.send("topic_test", message);
            }

            public void sendMessage(String key, String message) {
                kafkaTemplate.send("topic_test", key, message);
            }
        }

    ```
### 启动流程解析

+ 回顾springboot启动  
    srpingboot启动的时候会扫描calsspath下的META-INF/spring.factories所有文件，通过加载spring.factories文件初始化spring容器的bean

    ``` java 
        private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        int var10 = var9.length;

                        for(int var11 = 0; var11 < var10; ++var11) {
                            String factoryImplementationName = var9[var11];
                            result.add(factoryTypeName, factoryImplementationName.trim());
                        }
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var13) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
            }
        }
    }
    ```
+ kafka的spring.factories配置
    ```
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
    ```
    kafka通过KafkaAutoConfiguration自动注入kafka的生产者、消费者需要的组件，如ProducerFactory、KafkaTemplate、ConsumerFactory等bean

    ``` java
        @Configuration(proxyBeanMethods = false)
        @ConditionalOnClass(KafkaTemplate.class)
        @EnableConfigurationProperties(KafkaProperties.class)
        @Import({ KafkaAnnotationDrivenConfiguration.class, KafkaStreamsAnnotationDrivenConfiguration.class })
        public class KafkaAutoConfiguration {

            private final KafkaProperties properties;

            public KafkaAutoConfiguration(KafkaProperties properties) {
                this.properties = properties;
            }

            @Bean
            @ConditionalOnMissingBean(KafkaTemplate.class)
            public KafkaTemplate<?, ?> kafkaTemplate(ProducerFactory<Object, Object> kafkaProducerFactory,
                    ProducerListener<Object, Object> kafkaProducerListener,
                    ObjectProvider<RecordMessageConverter> messageConverter) {
                KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<>(kafkaProducerFactory);
                messageConverter.ifUnique(kafkaTemplate::setMessageConverter);
                kafkaTemplate.setProducerListener(kafkaProducerListener);
                kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic());
                return kafkaTemplate;
            }

            @Bean
            @ConditionalOnMissingBean(ProducerListener.class)
            public ProducerListener<Object, Object> kafkaProducerListener() {
                return new LoggingProducerListener<>();
            }

            @Bean
            @ConditionalOnMissingBean(ConsumerFactory.class)
            public ConsumerFactory<?, ?> kafkaConsumerFactory() {
                return new DefaultKafkaConsumerFactory<>(this.properties.buildConsumerProperties());
            }

            @Bean
            @ConditionalOnMissingBean(ProducerFactory.class)
            public ProducerFactory<?, ?> kafkaProducerFactory() {
                DefaultKafkaProducerFactory<?, ?> factory = new DefaultKafkaProducerFactory<>(
                        this.properties.buildProducerProperties());
                String transactionIdPrefix = this.properties.getProducer().getTransactionIdPrefix();
                if (transactionIdPrefix != null) {
                    factory.setTransactionIdPrefix(transactionIdPrefix);
                }
                return factory;
            }

            @Bean
            @ConditionalOnProperty(name = "spring.kafka.producer.transaction-id-prefix")
            @ConditionalOnMissingBean
            public KafkaTransactionManager<?, ?> kafkaTransactionManager(ProducerFactory<?, ?> producerFactory) {
                return new KafkaTransactionManager<>(producerFactory);
            }

            @Bean
            @ConditionalOnProperty(name = "spring.kafka.jaas.enabled")
            @ConditionalOnMissingBean
            public KafkaJaasLoginModuleInitializer kafkaJaasInitializer() throws IOException {
                KafkaJaasLoginModuleInitializer jaas = new KafkaJaasLoginModuleInitializer();
                Jaas jaasProperties = this.properties.getJaas();
                if (jaasProperties.getControlFlag() != null) {
                    jaas.setControlFlag(jaasProperties.getControlFlag());
                }
                if (jaasProperties.getLoginModule() != null) {
                    jaas.setLoginModule(jaasProperties.getLoginModule());
                }
                jaas.setOptions(jaasProperties.getOptions());
                return jaas;
            }

            @Bean
            @ConditionalOnMissingBean
            public KafkaAdmin kafkaAdmin() {
                KafkaAdmin kafkaAdmin = new KafkaAdmin(this.properties.buildAdminProperties());
                kafkaAdmin.setFatalIfBrokerNotAvailable(this.properties.getAdmin().isFailFast());
                return kafkaAdmin;
            }

        }
    ```
### 自定义配置kafka
按照springboot的快速接入，可以在平时的开发中快乐的使用kafka，应对大部分场景。如果项目中有多个kafka，则需要自定义配置，多样化使用kafka

+ pom
    ```
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.kafka</groupId>
			<artifactId>spring-kafka</artifactId>
		</dependency>
    ```
+ 配置文件application.properties
    ```
        #producer
        kafka.producer.bootstrapServers=10.19.16.67:9092,10.19.16.68:9092,10.19.16.69:9092
        # consumer
        kafka.consumer.bootstrapServers=10.19.3.194:9092,10.19.3.195:9092,10.19.3.196:9092
        kafka.consumer.groupId=myGroup
    ```
+ 消费者配置
    ``` java
        @EnableKafka
        @Configuration
        public class KafkaConsumerConfig {

            @Value("${kafka.consumer.bootstrapServers}")
            private String consumerBootstrapServers;

            @Value("${kafka.consumer.groupId}")
            private String consumerGroupId;
            @Bean
            public ConsumerFactory<String, String> consumerFactory() {
                Map<String, Object> props = new HashMap<>();
                props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, consumerBootstrapServers);
                props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);
                props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
                props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
                return new DefaultKafkaConsumerFactory<>(props);
            }

            @Bean
            public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
                ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
                factory.setConcurrency(3);
                factory.setConsumerFactory(consumerFactory());
                return factory;
            }
        }
    ```
+ 生产者配置
    ``` java
        @Configuration
        public class KafkaProducerConfig {

            @Value("${kafka.producer.bootstrapServers}")
            private String producerBootstrapServers;

            @Bean
            public ProducerFactory<String, String> producerFactory(){
                Map<String, Object> configs = new HashMap<>();
                configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, producerBootstrapServers);
                configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
                configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
                return new DefaultKafkaProducerFactory<>(configs);
            }

            @Bean
            public KafkaTemplate<String, String> kafkaTemplate() {
                return new KafkaTemplate<>(producerFactory());
            }
        }
    ```
