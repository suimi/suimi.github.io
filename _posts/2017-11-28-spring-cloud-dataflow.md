---
title: Spring Cloud Dataflow
tags: ["spring cloud","spring cloud dataflow"]
categories: ["MQ","spring cloud"]
---
* TOC
{:toc}

# 环境
- jdk 1.8
- 数据库(使用默认h2)
- redis
- rabbitmq
- maven
- spring cloud dataflow

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dataflow-dependencies</artifactId>
    <version>1.2.3.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-task-dependencies</artifactId>
    <version>1.2.1.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

# Data flow server

1. pom.xml

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-dataflow-server-local</artifactId>
</dependency>
```

2. `@EnableDataFlowServer`

```java
@SpringBootApplication
@EnableDataFlowServer
public class DataflowServerLocalApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataflowServerLocalApplication.class, args);
    }
}

```

3. redis配置

```
spring:
  redis:
    host: 192.168.6.20
    port: 7399
    password: risetekpassok
    database: 1
```
4. 启动服务

 服务器以端口9393启动,具体默认配置参见`spring-cloud-starter-dataflow-server-local#dataflow-server.yml` 与`spring-cloud-dataflow-server-core#dataflow-server-defaults.yml`。dashboard 地址`http://localhost:9393/dashboard`

5. maven 配置

由于在app 注册过程(后面会提到)会采用maven路径,在本地测试过程或是其他情形，需要更改maven配置(如localRepository, remote, repositoryes, proxy等),可以在server启动时，加入相应参数,如:
```
--maven.localRepository=mylocal
--maven.remote-repositories.repo1.url=https://repo1
--maven.remote-repositories.repo1.auth.username=user1
--maven.remote-repositories.repo1.auth.password=pass1
--maven.remote-repositories.repo1.snapshot-policy.update-policy=daily
--maven.remote-repositories.repo1.snapshot-policy.checksum-policy=warn
--maven.remote-repositories.repo1.release-policy.update-policy=never
--maven.remote-repositories.repo1.release-policy.checksum-policy=fail
--maven.remote-repositories.repo2.url=https://repo2
--maven.remote-repositories.repo2.policy.update-policy=always
--maven.remote-repositories.repo2.policy.checksum-policy=fail
--maven.proxy.host=proxy1
--maven.proxy.port=9010
--maven.proxy.auth.username=proxyuser1
--maven.proxy.auth.password=proxypass1
```

也可以采用环境变量方式。配置SPRING_APPLICATION_JSON的环境变量
```json
SPRING_APPLICATION_JSON='{
  "maven": {
    "local-repository": null,
    "remote-repositories": {
      "repo1": {
        "url": "https://repo1",
        "auth": {
          "username": "repo1user",
          "password": "repo1pass"
        }
      },
      "repo2": {
        "url": "https://repo2"
      }
    },
    "proxy": {
      "host": "proxyhost",
      "port": 9018,
      "auth": {
        "username": "proxyuser",
        "password": "proxypass"
      }
    }
  }
}'
```
- maven 的 localrepository 默认配置为`${user.home}/.m2/repository/`

# Data flow shell

## 构建

 构建过程与Data Flow Server一致,不同在于添加`@EnableDataFlowShell`注解,不需要配置redis

```java
 @SpringBootApplication
 @EnableDataFlowShell
 public class DataflowShellApplication {
     public static void main(String[] args) {
         SpringApplication.run(DataflowShellApplication.class, args);
     }
 }
```

## 启动shell, 配置server
在同一机器的情况下不需要配置，其他情况可采用下面方式配置：
```shell
dataflow:> config server http://localhost:9393
```

# Data flow Stream

## 创建source、processor、sink
创建过程与Data Flow Server一致,不同在于绑定各自channel,增加cloud stream依赖及配置
1. pom.xml

```
 <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```
2. rabbitmq 配置

```
spring:
  rabbitmq:
    host: 192.168.6.164
    port: 5672
    username: user
    password: pwd
```

## 注册 stream app

- 注册格式:`app register --name <app name> --type <type> --uri maven://<groupId>:<artifactId>[:<extension>[:<classifier>]]:<version>`

    `type` 支持 `source`,`processor`,`sink`,`task`

```
app register --name "source" --type source --uri maven://com.suimi.hello:dataflow-streams-source:0.0.1-SNAPSHOT
app register --name "processor" --type processor --uri maven://com.suimi.hello:dataflow-streams-processor:0.0.1-SNAPSHOT
app register --name "sink" --type sink --uri maven://com.suimi.hello:dataflow-streams-sink:0.0.1-SNAPSHOT
```
- 查看注册列表 app list

# 创建 stream 并部署

Stream DSL描述了数据流在系统中流转过程的线性序列。

例如，stream definition 为`http |transformer | cassandra`，每个管道符号连接应用程序的左右两边。命名通道可用于路由和将数据分发到多个消息传递目的地。

- 执行`stream create --name <stream name> --definition 'source | processor | sink'`
- stream list 可以查看定义的stream
- 执行 `stream deploy --name <stream name>` 部署stream

部署成功后可在server看到具体部署信息及各app日志路径,根据该路径查看具体app日志信息.



```
app register --name "task" --type task --uri maven://com.suimi.hello:dataflow-task:0.0.1-SNAPSHOT
task create myjob --difination task
task launch myjob

```

[github source](https://github.com/suimi/hello-mq/tree/master/data-flow)
