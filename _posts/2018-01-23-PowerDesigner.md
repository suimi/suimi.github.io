---
layout: post
title: PowerDesigner 使用记
tags: PowerDesigner
categories: ["Designer"]
---
* TOC
{:toc}

# 基本概念
- CDM(Conceptual Data Model):概念数据模型

    以实体为单元，进行实体以及实体对应关系的建立。即实体-联系图（E-R图），CDM就是以其自身方式来描述E-R图。
    - 此时不考虑物理实现的细节，只表示数据库的整体逻辑结构，独立于任何软件和数据存储结构。
    - 在CDM中用来标识实体的是属性（Attribute）
- LDM(Logical Data Model):逻辑数据模型

    逻辑模型是概念模型的延伸，逻辑模型中一方面显示了实体、实体的属性和实体之间的关系，另一方面又将继承、实体关系中的引用等在实体的属性中进行展示。逻辑模型主要是使得整个概念模型更易于理解，同时又不依赖于具体的数据库实现。

- PDM(Physical Data Model):物理数据模型

    PDM更接近与关系数据库里的关系表，PDM可以直接与RDBMS（关系型数据库管理系统）发生关联。PDM考虑了数据库的物理实现，包括软件和数据存储结构。
    - PDM的对象：表（Table）、表中的列（Table column）、主外键（Primary、Foreign key）、参照（Reference）、索引（Index）、视图（View）等。
    - 在PDM中用来表示实体属性的是列（Column）。
- OOM(Object-Oriented Model):面向对象模型

    一个OOM包含一系列包，类，接口 , 和他们的关系。 这些对象一起形成所有的( 或部份) 一个软件系统的逻辑的设计视图的类结构。 一个OOM 本质上是软件系统的一个静态的概念模型。可以直接生成JavaBean文件。

- BPM(Business Process Model):业务程序模型

# CDM -> OOM

1. Tools -> Generate Object-Oriented Model
2. Detail选项卡，取消`Convert names into codes`,否则生成的OOM name与code相同
3. Configure Model Options

    在Attribute/Class选项类别中，选择Code选项卡，定义`Naming Template`

    ![PowerDesigner Generate OOM](/static/img/PowerDesigner-config-oom.png)

参考:
- [PowerDesigner CDM LDM PDM OOM](http://blog.csdn.net/u010924834/article/details/48531669)
- [PowerDesigner 系列文章](http://www.cnblogs.com/sandea/p/4318540.html)
