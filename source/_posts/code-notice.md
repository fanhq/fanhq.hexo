---
title: Java编码规则
date: 2019-04-15 17:25:31
tags:
    - java
---

+ 常量命名全部大写，单词间用下划线隔开，力求语义表达完整清楚，不要嫌名字长。  
正例：MAX_STOCK_COUNT  
反例：MAX_COUNT

+ 抽象类命名使用Abstract或Base开头；异常类命名使用Exception结尾；测试类命名以它要测试的类的名称开始，以Test结尾

+ POJO类中布尔类型的变量，都不要加is前缀，否则部分框架解析会引起序列化错误。 反例：定义为基本数据类型Boolean isDeleted的属性，它的方法也是isDeleted()，RPC框架在反向解析的时候，“误以为”对应的属性名称是deleted，导致属性获取不到，进而抛出异常

+ 接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的Javadoc注释。尽量不要在接口里定义变量，如果一定要定义变量，肯定是与接口方法相关，并且是整个应用的基础常量

+ 对于Service和DAO类，基于SOA的理念，暴露出来的服务一定是接口，内部的实现类用Impl的后缀与接口区别。   
正例：CacheServiceImpl实现CacheService接口

+ 如果是形容能力的接口名称，取对应的形容词为接口名（通常是–able的形式）。   
正例：AbstractTranslator实现 Translatable接口

+ 