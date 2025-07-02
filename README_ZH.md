 <h1 align="center">hadoop分布式计算平台</h1>
  <p align="center">
    <a href="README.md"><strong>English</strong></a> | <strong>简体中文</strong>
  </p>


## 目录

- [仓库简介](#项目介绍)
- [前置条件](#前置条件)
- [镜像说明](#镜像说明)
- [获取帮助](#获取帮助)
- [如何贡献](#如何贡献)

## 项目介绍

[Hadoop](https://github.com/apache/hadoop)  是一个开源的分布式计算平台，用于处理大规模数据的存储和计算‌。本商品基于鲲鹏服务器的Huawei Cloud EulerOS 2.0 64bit系统，提供开箱即用的hadoop计算平台。

## 核心组件与功能

1.HDFS（Hadoop Distributed File System）‌
- 
- **存储架构‌：** 采用主从结构，NameNode管理元数据，DataNode存储实际数据块（默认128MB/块），支持多副本冗余（默认3份）以保障容错性。‌‌‌‌
- **适用场景‌：** 适合大文件批处理（如视频存储、日志分析），但低延迟访问和小文件存储效率较低。‌‌‌‌

2.MapReduce
-
- **计算模型‌：** 通过Map（数据分片处理）和Reduce（结果汇总）两阶段实现分布式计算，简化并行编程复杂度。‌‌‌‌

3.YARN（资源调度系统）‌
-
- **‌功能‌：** 动态分配集群资源（CPU、内存），支持多任务并发执行（如MapReduce、Spark），提升资源利用率

本项目提供的开源镜像商品 [**hadoop分布式计算平台**](https://marketplace.huaweicloud.com/contents/6bd70e0a-4bf5-4343-b483-500b10cbd1fb#productid=OFFI1123074339229200384) 已预先安装3.3.6版本的Hadoop及其相关运行环境，并提供部署模板。快来参照使用指南，轻松开启“开箱即用”的高效体验吧。


> **系统要求如下：**
> - CPU: 2vCPUs 或更高
> - RAM: 4GB 或更大
> - Disk: 至少 40GB

## 前置条件
[注册华为账号并开通华为云](https://support.huaweicloud.com/usermanual-account/account_id_001.html)

## 镜像说明

| 镜像规格                                                                                                       | 特性说明 | 备注 |
|------------------------------------------------------------------------------------------------------------| --- | --- |
| [hadoop-3.3.6-kunpeng](https://github.com/HuaweiCloudDeveloper/hadoop-image/tree/hadoop-3.3.6-kunpeng) | 基于鲲鹏服务器 + Huawei Cloud EulerOS 2.0 64bit 安装部署 |  |

## 获取帮助
- 更多问题可通过 [issue](https://github.com/HuaweiCloudDeveloper/hadoop-image/issues) 或 华为云云商店指定商品的服务支持 与我们取得联系
- 其他开源镜像可看 [open-source-image-repos](https://github.com/HuaweiCloudDeveloper/open-source-image-repos)

## 如何贡献
- Fork 此存储库并提交合并请求
- 基于您的开源镜像信息同步更新 README.md