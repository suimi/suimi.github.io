---
layout: post
title: 版本控制
tags: api-version
categories: spring
---
* TOC
{:toc}

# 概述
在实际的应用开发过程中，接口版本往往是比较头疼的一个问题，如何控制接口的版本，本人根据spring custom request condition，定义了一个annotation `@ApiVersion`,来指定接口版本信息。
在接口请求时，在request header中增加版本信息（eg:`api-version:2.1.0`）

# 使用方法
## 注解`@ApiVersion` 和配置`app.api.version = @project.version@`

`app.api.version `指定当前系统版本，注解可以在`Controller` 或是 方法上指定api版本信息
- 如果只是在`Controller`指定版本信息，则所有方法都是该版本
- 如果`Controller`和方法上同时指定了版本，则方法上指定的版本为该api版本
- 如果都没有指定，则系统当前版本为API 版本
    ```
    @ApiVersion("5")
    public class HelloController {

        @RequestMapping("hello")
        @ApiVersion("1")
        public String hello(String value) {
            log.info("haha1..........");
            return "hello 1 " + value;
        }
    ```

## 版本控制逻辑
api请求版本可通过请求头`api-version`指定，在没有带请求版本的情况下，默认匹配低于当前版本的最新版本。

可以通过`app.api.param-name`指定版本头参数名

- 如果请求版本高于当前系统版本，则返回404
- 请求版本低于或等于当前系统版本，匹配指定版本内的最新版本.
- 不带请求版本，匹配最新版本
- eg: 系统当前版本2.1.0-a， api有四个版本：1.2, 2.0,2.1.0-a 3.0

    |请求版本| 匹配版本|
    |:---:|:---:|
    |未指定|2.0|
    |3.0|无匹配|
    |2.0.1|2.0|
    |2.1.0-a|2.1.0-a|
    |2.1.0| 2.0|
    |1.8|1.2|
    |1.0|无匹配|

[GitHub](https://github.com/suimi/hello-demo/tree/master/api-version-demo)
