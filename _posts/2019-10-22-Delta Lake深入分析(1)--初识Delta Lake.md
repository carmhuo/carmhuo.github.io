---
layout: post
title: Delta Lake深入分析(1)
subtitle: 初识Delta Lake
author: Carm
date: 2019-10-22 20:41:42 +0800
header-img: img/home-bg.jpg
categories: 大数据
tag:
  - spark
  - delta
---
# Delta Lake深入分析(1)--初识Delta Lake
---

## 什么是Delta Lake?

**Delta Lake is an open-source storage layer that brings ACID transactions to Apache Spark™ and big data workloads.**

Delta Lake是一个开源的存储层，提供***ACID事务能力***，***支持批/流处理***，并且***完全兼容Spark API***，致力于***提高数据湖的性能、质量和可靠性***。


***2019.10.16***，Delta Lake开始由Linux基金会进行托管，但仍遵循Apache 2.0软件许可证。


本次基于Delta Lake 0.4（当前最新版本）进行分析


![delta + linux](/img/delta/Delta-Lake3-Linux.png)


Delta商业架构为：

![商业架构](/img/delta/Delta-Lake-marketecture-0423c.png)

> Tips:
***数据湖（data lake）:*** 数据湖是一个中心位置，它以原始格式保存大量数据，并且是一种组织高度多样化数据的方式。与将数据存储在文件或文件夹中的分层数据仓库（data warehouse）相比，数据湖使用平面架构来存储数据。数据湖支持所有类型（结构化、半结构化和非结构化）数据的存储，这意味着数据可以更灵活的格式保存，因此我们可以在准备使用它们时对其进行转换。您可以将各种类型的分析应用于数据，例如SQL查询，大数据分析，全文检索，实时分析，甚至可以使用机器学习来发现新的视野。
术语“数据湖”通常与面向Hadoop的对象存储相关联。
Hadoop数据湖是一个数据管理平台，它跨一组集群计算节点将数据存储在Hadoop分布式文件系统（HDFS）中，它的主要用途是处理和存储非关系数据。 可以处理的数据类型包括日志文件，网页点击记录，传感器数据，JSON对象，图片和社交媒体帖子。

## Delta Lake特性
1.  ACID事物支持
2.  完全支持DML操作(UPDATE、DELETE和MERGE INTO)
3.  完全兼容Apache Spark API
4. 	底层使用Apache Parquet作为数据存储格式
5.  支持Time Travel
6. 	对历史数据的变化提供审计功能（版本控制）
7.	Session级别快照隔离
8.	支持批处理和流处理
