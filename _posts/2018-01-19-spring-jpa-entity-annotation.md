---
layout: post
title: Spring Boot + JPA Entity Annotation不生效问题记
tags: ["spring data jpa"]
categories: ["spring data"]
---
* TOC
{:toc}

# Spring boot + JPA Entity Annotation 不生效问题记
## 版本
 - spring boot: 1.5.9
 - hibernate： 5.2.12
## 问题描述
在Entity的field上添加注解`@Enumerated`、`@Column`、`@Lob`不生效，但加在getter方法上有效,有用采用lombok, 写getter方法就多余了

## 解决过程
查询大部分资料是说hibernate naming strategy调整导致的，但怎么调整都不对不生效，后来发现加在getter方法上发现有效。发现原来是Jpa访问策略默认是`AccessType.PROPERTY`

### `@Access注解`
JPA的`@Access`批注,其值定义在`AccessType`枚举类中,包括`AccessType.FIELD`及`AccessType.PROPERTY`,这两个类型定义了实体的访问模式(Access mode)

## 解决方案
在Entity上增加`@Access(AccessType.FIELD)`注解


参考:
- [spring boot naming issues](https://github.com/spring-projects/spring-boot/issues/2129)
- [Difference between adding JPA annotations to fields vs getters?](https://stackoverflow.com/questions/43256489/difference-between-adding-jpa-annotations-to-fields-vs-getters)
